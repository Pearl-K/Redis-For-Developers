# Redis Key 관리
## Redis의 키 생성과 삭제

### 1. 키 자동 생성

* Redis는 없는 키에 연산을 수행하면 자동 생성한다.

  ```shell
  INCR count        # count 키가 없으면 0으로 간주하고 생성 후 1 증가
  LPUSH list "a"    # list 키가 없으면 리스트 생성 후 삽입
  ```

### 2. 키 자동 삭제

* 컬렉션 타입(List, Set 등)에서 모든 요소를 삭제하면 키도 자동으로 사라진다.

  ```shell
  LPOP list         # 마지막 요소까지 pop되면 키가 사라짐
  ```

---

## 키 관련 주요 커맨드 정리

### `EXISTS key`

* 키 존재 여부 확인 (1 또는 0 반환)
* 여러 키 넣으면 존재하는 키 개수 반환

```shell
EXISTS key1 key2
```

---

### `KEYS pattern` ⚠️ 실무 사용시 위험

* O(N)으로 전체 키 순회 → 단일 스레드 연산인 Redis 성능에 치명적인 병목

```shell
KEYS user:*    # 전체 키 중 prefix가 user:인 키 반환
```

### [심화] – 왜 O(N)이나 걸릴까?

* Redis는 **Hash Table**로 키를 관리하지만, `KEYS`는 모든 슬롯을 순회하며 패턴을 직접 매칭한다.
* 내부적으로는 `dictScan()` 호출하여 모든 해시 슬롯을 순회하기 때문에 키 수가 많을수록 심각한 STW 지연이 일어난다.
* 대안으로 `SCAN`을 사용할 수 있다.


---

### `SCAN cursor [MATCH pattern] [COUNT n] [TYPE type]`

* **커서 기반** 반복 조회 (비차단, 점진적 탐색)
* `COUNT`는 힌트일 뿐 정확히 카운트 수만큼 반환하지 않을 수 있다.

```shell
SCAN 0 MATCH user:* COUNT 100
```

- 주의사항
    - 반복 조회할 때는 클라이언트가 커서 저장하고 loop 처리 필요
    - 중복 키가 반환될 수 있음 → `Set`에 수집하거나 중복 제거 처리 필요
     - `MATCH`, `TYPE` 조건 추가 시 속도 저하에 유의해야함

---

### `SORT key [BY pattern] [LIMIT start count] [ASC|DESC] [ALPHA]`

* 리스트, 셋, 정렬된 셋에 대해 정렬 수행

```shell
SORT user_ids BY user:*->score DESC LIMIT 0 10
```

- 주의사항
    - 인메모리 정렬이므로 대상이 크면 메모리 폭증, 속도 위험
    - `BY`, `GET`으로 외부 키를 조회하면 I/O 비용 급증

---

### `RENAME`, `RENAMENX`

```shell
RENAME old new         # 기존 키를 새 키로 변경 (덮어쓰기 허용)
RENAMENX old new       # 새 키가 없을 때만 변경
```

---

### `COPY source destination [REPLACE]`

* Redis 6.2+ 부터 사용 가능
* 키를 복제한다 (같은 DB 또는 다른 DB로)

```shell
COPY user:1 user:1:backup
```

---

### `OBJECT`

* 키의 메타데이터를 조회할 수 있다.
* 아래와 같은 옵션이 존재한다.

```shell
OBJECT ENCODING key
OBJECT IDLETIME key
```

---

### `FLUSHALL` ⚠️ 사용 X

* 모든 DB의 모든 키를 삭제한다.
* 운영 환경에서 절대 사용하지 않도록 하자.


---

### `DEL key`

* 동기 방식으로 키를 삭제한다.
* 데이터가 많으면 성능 문제 생기므로 주의

```shell
DEL biglist
```

### `UNLINK key` ✅ `DEL` 대안책

* Redis 4.0+ 부터 사용 가능
* **비동기 삭제**. 큰 키를 지울 때 메인 쓰레드를 차단하지 않기 때문에 이를 잘 활용하자.

```shell
UNLINK biglist
```

---

### TTL 관련 명령어

| 커맨드                      | 설명                                   |
| ------------------------ | ------------------------------------ |
| `EXPIRE key seconds`     | 지정된 초 후에 키 만료                        |
| `EXPIREAT key timestamp` | 특정 유닉스 타임스탬프에 만료                     |
| `TTL key`                | 남은 TTL (초) 확인. -1이면 만료 없음, -2면 키 없음  |
| `EXPIRETIME key`         | 만료 예정 Unix timestamp 반환 (Redis 7.0+) |

```shell
EXPIRE session:123 600
TTL session:123
EXPIRETIME session:123
```


---
### 실무 관점의 팁

* **KEYS 조회는 테스트 환경에서만** → 실무에서는 `SCAN`
* **큰 키를 지울 땐 반드시 `UNLINK`** → STW 방지
* **TTL 설정은 보안/자원 회수에 필수적** → 만료 정책 항상 고려
