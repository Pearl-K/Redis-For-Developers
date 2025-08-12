# Sorted Set Recent Search
유저 별 최근 검색 기록을 RDB로 제공하려면 요구사항이 많아 까다로울 수 있다.

- 유저별 다른 키워드 노출
- 중복 제거된 검색 내역이 필요
- 가장 최근 검색한 특정 개수 키워드만 사용자에게 노출


해당 조건들을 모두 쿼리로 제공하는 것보다, Sorted Set을 활용하여 쉽게 제공할 수 있다.

또한, Sorted Set 내부에 일정 개수를 유지하고 싶다면, 음수 인덱스 삭제를 통해서 구현할 수 있다.

---
## 실제 활용 예시

### 키 설계

* `recent:search:{userId}` (ZSET) — 멤버=검색어(정규화된 문자열), 점수=UNIX epoch millis

### 기본 연산

* **추가(중복 제거+시간 갱신은 자동)**

  ```bash
  # ts = (millis) 서버에서 생성
  ZADD recent:search:{userId} ts "redis cluster"
  ```

  * 같은 멤버면 점수만 갱신됨 → 중복 제거 효과.

* **최신 K개 조회**

  ```bash
  ZRANGE recent:search:{userId} +inf -inf BYSCORE REV LIMIT 0 K
  # 또는
  ZREVRANGE recent:search:{userId} 0 K-1
  ```

* **사이즈 K로 트림(가장 오래된 것 삭제)**

  ```bash
  # K=10이면
  ZREMRANGEBYRANK recent:search:{userId} 0 -11   # 0 ~ (뒤에서 11번째)까지 제거 → 상위 10개만 유지
  ```

  또는

  ```bash
  # 과잉 개수만큼 바로 제거
  # excess = max(ZCARD - K, 0)
  ZPOPMIN recent:search:{userId} excess
  ```

* **기간 만료(예: 30일치만 보존)**

  ```bash
  # now = millis, keepMillis = 30d
  ZREMRANGEBYSCORE recent:search:{userId} -inf (now-keepMillis)
  ```

* **키 자체 TTL(유저 장기 미사용 시 정리)**

  ```bash
  EXPIRE recent:search:{userId} 2592000   # 30d
  ```

  > 주의: `ZREMRANGEBYSCORE`는 내용만 정리, `EXPIRE`는 키 전체 만료.

### 운영 주의사항

* **정규화**: 소문자/trim/공백 단일화, 길이 제한으로 → 동일 의미 검색어의 중복을 회피하는 것이 좋다.
* **민감정보 필터**: 비즈니스적으로 중요한 사항
* **원자 처리**: “추가→트림→기간정리”를 **Lua Script**로 묶으면 race condition이 없으나 성능 저하가 없는지 체크해보도록 하자.


---
# Sorted Set과 Set을 활용한 Tag
일반적인 Set과 Sorted Set을 이용하여 효과적은 게시물 Tag 기능을 구현할 수 있다.

- Set: 특정 게시물에 달린 태그 집합(키워드) 관리
- Sorted Set: 태그(키워드)별로 최신 연관 게시물을 관리할 때 쓸 수 있다.


- 예를 들어, "Redis" 태그 하위에 작성된 최신 게시글 10개를 가져오는 기능이 가능하다. 
- 여러 태그 동시 검색을 원한다면, Set 연산 (합집합, 교집합, 차집합 등)을 통해 지원할 수 있다.
- 해당 기능을 RDB 쿼리로 제공하고자 하면, 테이블 크기에 따라서 DB 부하가 심해질 수 있으므로 간단하게 위와 같은 방식의 구현으로 대체할 수 있다.


단, 역시 게시물이 너무 많이 저장되는 상황(메모리 관리 측면)이나 샤딩 환경에서 사용할 때는 주의하도록 하자.


---

## 실제 활용 예시 (Set + Sorted Set)

### 키 설계

* `post:{postId}:tags` (SET) — 게시물의 태그 집합
* `tag:{tag}:posts` (ZSET) — 멤버=postId, 점수=작성 시각(epoch millis)

### 쓰기 흐름

* **게시물 생성/수정**

  ```bash
  SADD post:{postId}:tags "redis" "cache" "cluster"

  # 태그 ZSET에 역인덱스 반영(최신순)
  ZADD tag:redis:posts   ts {postId}
  ZADD tag:cache:posts   ts {postId}
  ZADD tag:cluster:posts ts {postId}
  ```

* **게시물 삭제/숨김**

  ```bash
  SMEMBERS post:{postId}:tags           # 연결된 태그 나열
  ZREM tag:{tag}:posts {postId}         # 각 태그 ZSET에서 제거
  DEL post:{postId}:tags
  ```

### 조회

* **태그별 최신 N개**

  ```bash
  ZRANGE tag:redis:posts +inf -inf BYSCORE REV LIMIT 0 N WITHSCORES
  ```

* **여러 태그 교집합(모두 포함한 최신순) — Redis 6.2+**

  ```bash
  # 결과를 바로 반환
  ZINTER 2 tag:redis:posts tag:cluster:posts AGGREGATE MAX WITHSCORES
  # → 클라이언트에서 상위 N개 슬라이싱 or LIMIT 사용(7.0+ 지원)
  ```

* **여러 태그 합집합(하나라도 포함, 최신순)**

  ```bash
  ZUNION 2 tag:redis:posts tag:cache:posts AGGREGATE MAX WITHSCORES
  ```

> 클러스터 환경 주의: `ZINTER/ZUNION`은 **같은 슬롯** 키만 서버-사이드에서 조합 가능. 태그 조합이 많으면 **애플리케이션 병합(K-way merge)** 전략을 병행.

### 정리/trim

* **태그별 상한(예: 최신 10만 개만 유지)**

  ```bash
  ZREMRANGEBYRANK tag:{tag}:posts 0 -100001
  ```

* **오래된 게시물 제거(예: 90일 이전)**

  ```bash
  ZREMRANGEBYSCORE tag:{tag}:posts -inf (now-90d)
  ```

* **키 TTL(비추천)**: 태그 키는 공유 인덱스기 때문에 만료되는 TTL보단 **랭크/시간 trim**이 안전.

---
## 운영 주의사항

* **멤버 압축**: `postId`는 정수형(문자열도 짧게), 태그는 키 이름에 직접 포함(빈도 높은 태그만 역인덱스 운영).
* **정합성**: RDB 트랜잭션 후 Redis 반영은 **outbox/event** 기반 비동기 반영을 추천. 실패 시 재처리 큐.


