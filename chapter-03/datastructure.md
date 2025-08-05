# Redis 자료구조
## 1. String

### 개념

* Redis의 가장 기본적인 자료형
* **최대 512MB**까지 저장 가능
* 바이너리 세이프(Binary Safe) → 이미지, JSON, 직렬화 데이터도 저장 가능

### 주요 커맨드

```shell
SET key value
GET key
SETNX key value      # key가 없을 때만 저장 (간단 Lock 구현 시 사용)
INCR key             # 숫자형 문자열 증가
INCRBY key 10
MSET key1 v1 key2 v2
MGET key1 key2
```

### 예시 (간이 분산 락)

```shell
SETNX lock:userid123 "lock-value"   # 성공 시 1 반환
EXPIRE lock:userid123 10            # TTL 설정 필수 (락이 안풀리는 상황 방지한 TTL)
```

### [추가] `INCR`의 원자성

* Redis는 단일 스레드 기반의 명령 처리 구조이므로, 하나의 명령어는 실행 중에 중단되지 않는다.
* `INCR`, `INCRBY`는 내부적으로 `String`을 숫자로 변환 → 증가 → 다시 저장 과정을 거치지만, 이 전체가 한 단위로 처리되기 때문에 **Atomic** 연산이다.

---

## 2. List

### 개념

* **양방향 연결 리스트**로 동작 (과거에 내부 구조는 `ziplist`, 현재는 `quicklist`를 사용하고 있음)
* 채팅 메시지, 큐, 로그 저장 등 순차적 데이터 처리에 적합하다.

### 주요 커맨드

```shell
LPUSH mylist "A"       # 왼쪽 삽입
RPUSH mylist "B"       # 오른쪽 삽입
LPOP mylist            # 왼쪽 pop
LRANGE mylist 0 -1     # 전체 조회
LTRIM mylist 0 10      # 일부만 유지
LINSERT mylist BEFORE "B" "C"
LSET mylist 0 "Z"
```

### [심화] List를 블로킹 큐로 활용하기

```shell
BLPOP myqueue 5        # 큐에 항목이 없으면 최대 5초 동안 대기
BRPOP myqueue 0        # 무한 대기 가능 (비동기 이벤트 처리에 활용 가능)
```

- 실무 예시
    - 이벤트 큐 → 생산자 서버가 `RPUSH`, 소비자 서버가 `BLPOP`으로 처리하면서 작업을 조절한다.


여기서 블로킹 큐라는 말에 블로킹 큐 관련 연산이 Redis 메인 스레드를 블로킹하는거 아닌가? 하는 의문이 생길 수 있다.


그러나 이는 서버 입장에서 블로킹이 아니고, 클라이언트가 블로킹(대기 상태)되는 것으로 이해하면 된다.

> 관련 공식 문서 참고
>
> [(1) 레디스 모듈과 블로킹 커맨드](https://redis.io/docs/latest/develop/reference/modules/modules-blocking-ops/)
> [(2) BLPOP 커맨드](https://redis.io/docs/latest/commands/blpop/)
> 
> 더 자세한 내용은 다른 챕터에서 공부할 예정


### [추가] 시간 복잡도

* `LPUSH`, `RPUSH`, `LPOP` → O(1)
* `LSET`, `LINSERT`, `LRANGE` → O(N)
* 내부 구조는 Redis 3.2 이후 기본적으로 `quicklist`를 사용 중이다. 이는 기존의 `ziplist + linked list` 를 혼합한 구조였으나 7.x 버전 이후로 `listpack` 사용으로 바뀌었다. (과거 버전 설명 주의)
* [참고 자료 - redis github 내부 list 코드](https://github.com/redis/redis/blob/unstable/src/quicklist.h)


---

## 3. Hash

### 개념

* 키 안에 다수의 필드를 저장하는 구조
* 테이블형 데이터, 유저 프로필 정보 등에 적합

### 주요 커맨드

```shell
HSET user:123 name "Jinju" age "26"
HGET user:123 name
HGETALL user:123
HDEL user:123 age
HEXISTS user:123 email
```

- 실무 예시
    - 유저 상태 캐싱, 특히 내부 필드가 변경되는 구조에 용이
    - 여러 필드 중에 특정 필드를 가져와야하거나 `HGET`, 여러 필드들을 한 번에 가져오고 싶을 때 `HGETALL` 활용하기 좋음

---

## 4. Set

### 개념

* 중복 없는 원소 집합, 순서 없음 (그래서 꺼내는 연산하면 `SPOP` 무작위 원소가 꺼내짐)
* 태그, 친구 목록, 유저 ID 집합 등 활용 가능 (특히 집합 연산이 효율적인 곳에 활용하면 좋다)

### 주요 커맨드

```shell
SADD tags "java" "redis"
SREM tags "java"
SMEMBERS tags
SPOP tags          # 무작위 제거
SUNION set1 set2
SINTER set1 set2
SDIFF set1 set2
```

- 실무 예시
    - 특정 그룹의 참가자 목록 관리 (집합 연산 활용)


---

## 5. Sorted Set (ZSet)

### 개념

* **Score를 기준으로 정렬된 Set**
* 중복 불가, score만 다르면 갱신
* 내부적으로 **Skip List + Hash Table**로 구현되어 있음

### 주요 커맨드

```shell
ZADD rank 100 "user1"
ZADD rank NX 110 "user2"      # NX: 존재하지 않을 때만 추가
ZRANGE rank 0 1 WITHSCORES
ZREVRANGE rank 0 2
ZREM rank "user1"
```

### [심화] 시간복잡도와 내부 구조

* ZADD, ZREM, ZRANGE → **O(logN)**
* 이유: **SkipList**를 사용하여 정렬된 상태 유지
* [SkipList에 대한 추가 자료](https://github.com/Pearl-K/spring-redis-study/blob/main/week5_Redis_Data_Type/Redis_SortedSet.md)
* 인덱스 기반 접근 가능 `O(logN)`, score 범위 검색 가능



- 실무 예시
    - 게임 랭킹 시스템, 실시간 리더 보드 등
    - 최근 활동 시간순으로 정렬된 유저 목록

---

## 6. Bitmap

### 개념

* 각 비트를 0/1로 표현 (`Boolean` 데이터 표현에 최적)
* 매우 작은 메모리로 많은 상태를 표현 가능 (1억 유저 로그인 여부도 수십 MB면 가능함)


### 주요 커맨드

```shell
SETBIT active_users 123 1     # 123번째 비트 활성화
GETBIT active_users 123
BITCOUNT active_users         # 1의 개수 세기
BITFIELD active_users GET u8 100
```


- 실무 예시
    - 유저 출석 여부 (일간 출석 체크 등)
    - 회원가입 시 휴대폰 번호 중복 여부 (전화번호 마지막 4자리로 offset, bit연산으로 빠르게)

---

## 7. HyperLogLog

### 개념

* 중복을 허용한 채로 고유한 원소 개수(근사값)를 계산하는 구조
* 내부적으로 해시 함수 + bucket을 이용해 cardinality 를 근사적으로 추정함 (정확한 값 아님)

### 주요 커맨드

```shell
PFADD unique_visitors user1 user2 user3
PFCOUNT unique_visitors
PFMERGE all_visitors v1 v2 v3
```

- PF는 Probabilistic Filter의 약자이다.
- 실무 예시
    - 실시간 방문자 수 추정 (UV, DAU)
    - 대규모 유입에서 정확도보다 추정치만 알아도 되는 경우


---

## 8. GeoSpatial

### 개념

* Redis에서 위도/경도 좌표를 저장하고, 반경 검색, 거리 계산, 정렬 등의 기능을 제공
* 내부적으로는 `Sorted Set`을 활용하여 구현 (score를 GeoHash로 변환해 저장)

### 주요 커맨드

```shell
GEOADD places 126.9780 37.5665 "Seoul"
GEOADD places 127.0330 37.4991 "Gangnam"

GEODIST places "Seoul" "Gangnam" km
GEORADIUS places 127.0 37.5 10 km
GEOPOS places "Seoul"
GEORADIUSBYMEMBER places "Seoul" 5 km
```

- 사용 시, 거리 단위에 주의하기

### [심화] 내부 구조

* `GeoHash`를 기반으로 `Sorted Set`의 score에 좌표를 변환하여 저장
* 거리 계산 시 Haversine 공식을 기반으로 동작

- 실무 예시
    - 지도 기반 POI(주변 위치 추천)
    - 배달 앱: 사용자 반경 내 배달 가능 상점 조회
    - 주변 사용자 탐색

---

## 9. Stream

### 개념

* Redis 5.0부터 도입된 로그 기반 append-only 자료구조
* Kafka처럼 생산자-소비자 모델, 메시지 ID 기반 읽기, 컨슈머 그룹 등 지원
* 비동기 이벤트 처리, 로그 수집, 알림 시스템에 유용

### 주요 커맨드

```shell
XADD mystream * sensor-id 1234 temperature 22.5
XRANGE mystream - +
XREAD COUNT 2 STREAMS mystream 0
XGROUP CREATE mystream mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 STREAMS mystream >
XACK mystream mygroup 123456789-0
```

- ID: `timestamp-sequence` 형식 (예: `1658412345678-0`)
- `*` 사용 시 Redis가 자동 ID 생성

### [심화] 구조 및 특징

* 내부적으로는 `Radix Tree` 기반 구조
* 하나의 stream entry는 `ID + key-value pairs`로 구성
* `XGROUP`, `XREADGROUP`을 사용해 컨슈머 그룹 모델 구현 가능 (Kafka 유사)
* 더 구체적인 내용은 추후 Stream 챕터에서 다룰 예정


- 실무 예시
    - 실시간 주문/결제 이벤트 처리
    - 채팅 메시지 수집 (Kafka보다 가벼운 use case)
    - 로그 수집 및 분석 (ELK 대체 목적 소규모 시스템)


---
## Summary

| 자료형         | 특징                 | 주요 커맨드                   | 실무 활용 예시  |
| ----------- | ------------------ | ------------------------ | --------- |
| String      | 기본형, 바이너리 가능       | `SET`, `GET`, `INCR`, `MSET`     | 키-값 캐시    |
| List        | 순서 O, 양방향 리스트      | `LPUSH`, `LPOP`, `LRANGE`      | 채팅, 큐     |
| Hash        | Map 구조             | `HSET`, `HGET`, `HGETALL`      | 유저 정보     |
| Set         | 중복 X, 순서 X         | `SADD`, `SREM`, `SUNION`       | 태그, 친구 목록 |
| Sorted Set  | 정렬된 Set (score 기반) | `ZADD`, `ZRANGE`, `ZREM`       | 랭킹 시스템    |
| Bitmap      | 비트 단위 상태 표현        | `SETBIT`, `GETBIT`, `BITCOUNT` | 출석 체크 등   |
| HyperLogLog | 고유 원소 개수 추정 (근사치)  | `PFADD`, `PFCOUNT`           | UV 추정     |
| GeoSpatial | 위치 좌표 저장 및 검색 | `GEOADD`, `GEORADIUS`, `GEODIST` | 주변 검색, 위치 기반 추천 |
| Stream     | 실시간 로그, 메시지 큐 | `XADD`, `XREAD`, `XGROUP`, `XACK`  | 비동기 이벤트, 로그 처리  |


