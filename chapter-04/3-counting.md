# Redis에서 다양한 Counting 기능 만들기
## 1. 좋아요 처리
좋아요를 단순히 개수로 표현하고자 하면, 특정 키에 대한 원자적 연산으로 관리할 수 있을 것이다. 그러나 특정 유저가 좋아요를 했는지 안했는지를 반영해야 하는 상황이 있다.


해당 상황은 게시글이나 댓글을 기준으로, Set을 통해서 구현할 수 있다. `SADD`를 통해 `user_id`를 추가하면 되고, `SCARD` 연산을 통해 전체 좋아요 수를 가져올 수 있다.

- 아이디어
    - 게시물/댓글별로 “좋아요한 유저 집합”을 SET으로 관리
    조회 시 `SCARD`로 개수, `SISMEMBER`로 개인의 좋아요 여부 확인


- 키 설계
    - `like:post:{postId}` (SET) — 멤버: `userId`


```bash
# 좋아요 누르기
SADD like:post:42 1001

# 좋아요 취소
SREM like:post:42 1001

# 유저가 좋아요 했는지
SISMEMBER like:post:42 1001

# 총 좋아요 수
SCARD like:post:42
```

- 운영 팁
    - 트래픽이 높고 SCARD 비용을 더 줄이고 싶으면, SET 유지와 동시에 INCR/DECR 카운터를 Lua로 원자적 업데이트 하면 된다.


---
## 2. 읽지 않은 메시지 수 카운팅
채팅 메시지는 빠르게 쌓인다. 해당 정보를 매번 DB에 반영한다면 DB에 쓰기와 변경이 자주 일어날 수 있다. 따라서, Redis 등에 일시적으로 저장하고 필요한 시점에 한 번에 반영할 수도 있다.


- 아이디어
    - 대화방/유저별 미확인 메시지 카운터를 Redis에 두고, 주기적으로 DB에 플러시.


- 키 설계
    - `unread:{roomId}` (HASH)
    - field: `userId`, value: `count`
    - 유저 전체 합: `unread:user:{userId}` (STRING)


```bash
# 새 메시지 도착 → 해당 방에서 나(1001)를 제외한 참가자들의 미확인 +1
HINCRBY unread:room:77 1002 1
HINCRBY unread:room:77 1003 1

# 유저 1002가 room 77 읽음 → 해당 방 카운터 0으로 초기화
HDEL unread:room:77 1002

# 유저 전체 미확인 메시지 합계를 별도로 관리할 경우
INCRBY unread:user:1002 1        # 증가
DECRBY unread:user:1002 {readN}  # 읽은 만큼 감소
```



---
## 3. DAU 구하기
Redis의 Bitmap을 사용하여 실시간 DAU를 측정할 수 있다. (이 때 사용자 아이디는 0 이상의 정수형이어야 한다.)


레디스 string의 최대 길이는 512MB이기 때문에, 하나의 키를 사용해 1천 만명 정도의 사용자를 카운팅 할 수 있다. (약 1.2MB 소모)


```bash
# 로그인 시 유저 1001의 비트를 1로
SETBIT dau:2025-08-12 1001 1

# 해당 날짜의 DAU
BITCOUNT dau:2025-08-12

# 주간/월간 고유 사용자(합집합) — OR로 합치고 카운트
BITOP OR dau:2025-08 week32 dau:2025-08-11 dau:2025-08-12 dau:2025-08-13

BITCOUNT dau:2025-08 week32
```


---
## 4. hyperloglog를 이용한 애플리케이션 미터링
Redis hyperloglog를 사용하면 특정 API 호출 횟수를 쉽게 알 수 있다.

- 특정 유저의 월별 API 호출 횟수를 계산하는 방법
- user_id를 구분 키로 사용, API 호출할 때마다 생성되는 로그의 유일한 식별자를 hyperloglog에 저장할 수 있다.


```bash
# 고유 방문자 기록
PFADD uv:2025-08 1001
PFADD uv:2025-08 1002

# 월간 고유 방문자 추정치
PFCOUNT uv:2025-08

# 여러 집합 합치기(여러 API, 여러 일자 → 월간)
PFMERGE uv:api:search:2025-08 uv:api:search:2025-08-01 uv:api:search:2025-08-02

# API별 월간 고유 호출자
PFCOUNT uv:api:search:2025-08
```