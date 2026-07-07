# Mini Redis 구축 - 학습 정리

## 1. 개요

Redis가 빠른 이유인 내부 자료구조(해시맵, 이중 연결 리스트, 힙)를 직접 구현하고,
이를 조합해서 **LRU 캐시**와 **TTL 만료**가 동작하는 CLI 기반 Mini Redis를 만든 과제.

**제약사항**: `dict`, `set`, `collections` 사용 금지 → 해시맵도 배열(`list`) 기반으로 직접 구현.

**파일 구조 (6개, 각 자료구조 독립 모듈)**

```
entry.py               # 공유 데이터 모델 (해시맵/LRU가 같이 참조)
doubly_linked_list.py   # 이중 연결 리스트 (LRU 순서 관리)
hashmap.py              # 해시맵 (체이닝 방식)
min_heap.py             # 최소 힙 (TTL 만료 관리)
redis_db.py             # 위 3개 조립하는 엔진 (MiniRedis 클래스)
main.py                 # REPL / 파싱 / 출력 포맷
```

---

## 2. 요구사항 ↔ 구현 매핑

| 요구사항 | 구현 위치 |
|---|---|
| 이중 연결 리스트 O(1) 연산 | `doubly_linked_list.py` |
| 해시맵 체이닝 + 로드팩터 0.75 확장 | `hashmap.py` |
| 최소 힙 (expire_at, key) | `min_heap.py` |
| SET/GET/DEL/EXISTS/DBSIZE/KEYS | `redis_db.py` cmd_* 메서드 |
| CONFIG SET maxmemory / INFO memory | `redis_db.py` cmd_config_set_maxmemory / cmd_info_memory |
| EXPIRE / TTL | `redis_db.py` cmd_expire / cmd_ttl |
| CLI REPL | `main.py` |

---

## 3. 파일별 핵심 로직

### 3-1. `entry.py` — 공유 데이터 모델

세 자료구조가 각자 다른 목적으로 참조하는 값 하나를 표현. 해시맵 체이닝용 `hash_next`와
LRU 순서용 `lru_prev`/`lru_next`를 **한 객체에 통합**해서, Entry 자신이 곧 DLL의 노드 역할도 겸함.

```python
class Entry:
    def __init__(self, key, value):
        self.key = key
        self.value = value
        self.expire_at = None      # TTL 없으면 None

        self.lru_prev = None       # DLL 순서 관리용
        self.lru_next = None

        self.hash_next = None      # 해시맵 체이닝용
```

### 3-2. `doubly_linked_list.py` — LRU 순서 관리

별도의 Node 클래스 없이 `Entry` 자체를 노드로 사용. `head`=MRU(최근 사용), `tail`=LRU(오래된 것).

```python
def move_to_front(self, entry: Entry):
    """지정된 엔트리를 맨 앞으로 이동시킵니다. O(1)"""
    if self.size <= 1 or entry is self.head:
        return
    if entry.lru_prev:
        entry.lru_prev.lru_next = entry.lru_next
    if entry.lru_next:
        entry.lru_next.lru_prev = entry.lru_prev
    else:
        self.tail = entry.lru_prev
    entry.lru_next = self.head
    entry.lru_prev = None
    self.head.lru_prev = entry
    self.head = entry
```

- 앞뒤 포인터만 갈아끼우는 방식이라 리스트 길이와 무관하게 **O(1)**.
- GET 성공 시, SET으로 값 갱신 시 이 메서드로 MRU 갱신.

### 3-3. `hashmap.py` — 체이닝 해시맵

```python
def _hash(self, key):
    h = 0
    for ch in key:
        h = ((h * 31) + ord(ch))
    return h

def put(self, key, entry):
    idx = self.index_for(key)
    current = self.bucket[idx]
    while current is not None:
        if current.key == key:
            current.value = entry.value   # 같은 key면 값만 갱신
            return current
        current = current.hash_next

    entry.hash_next = self.bucket[idx]    # 체인 맨 앞에 삽입
    self.bucket[idx] = entry
    self.size += 1

    if (self.size / self.bucket_size) > self.load_factor:
        self._resize()
```

- 해시 함수: 31 곱하는 다항 해시 (Java `String.hashCode()`랑 같은 방식, 31이 소수라 분포 안정적).
- 충돌 시 `hash_next`로 체이닝, 로드팩터(0.75) 넘으면 `_resize()`로 버킷 2배 확장 + 전체 재해싱.

### 3-4. `min_heap.py` — TTL 관리용 최소 힙

```python
def _heapify_up(self, idx):
    while idx > 0:
        parent = (idx - 1) // 2
        if self.data[idx][0] < self.data[parent][0]:
            self.data[idx], self.data[parent] = self.data[parent], self.data[idx]
            idx = parent
        else:
            break
```

- 배열 기반 완전 이진트리. `(expire_at, key)` 튜플을 push하면 만료 시각이 가장 빠른 게 항상 root.
- 삭제된 키의 heap 항목은 그대로 두고(lazy), `redis_db.py`에서 pop할 때 "현재도 유효한 만료 정보인지" 재검증.

### 3-5. `redis_db.py` — 세 자료구조를 접착하는 엔진

```python
def _evict_lru(self):
    """메모리 초과 시 LRU 알고리즘 데이터 청소"""
    while self.maxmemory > 0 and self.used_memory > self.maxmemory and self.lru.size > 0:
        victim_node = self.lru.tail          # LRU = 맨 뒤
        if victim_node is None:
            break
        self._delete_key_internal(victim_node.key)
        self.evicted_keys += 1
```

- `used_memory` 계산: `len(utf8(key)) + len(utf8(value))`의 합.
- SET 후 `maxmemory` 초과 시 tail(LRU)부터 하나씩 삭제.

### 3-6. `main.py` — REPL

```python
SPECS = {
    "SET": (3, "(error) ERR wrong number of arguments for 'SET' command"),
    "GET": (2, "(error) ERR wrong number of arguments for 'GET' command"),
    ...
}
if cmd in SPECS:
    expected_len, err_msg = SPECS[cmd]
    if len(tokens) != expected_len:
        return err_msg
```

- 명령어별 예상 토큰 개수를 미리 정의해두고, 다르면 표준화된 에러 메시지 리턴.
- 큰따옴표로 감싼 인자도 하나로 인식하는 간이 파서(`split_arguments`) 포함.

---

## 4. SET / GET 실행 흐름 (파일 간 호출 순서)

### SET name jaehyun

```
1. main.py
   └─ input()으로 "SET name jaehyun" 받음
   └─ split_arguments() 로 ["SET", "name", "jaehyun"] 토큰화
   └─ dispatch() 에서 SPECS["SET"] 확인 (토큰 3개 맞음)
   └─ db.cmd_set("name", "jaehyun") 호출

2. redis_db.py :: MiniRedis.cmd_set()
   └─ _sweep_expired_heap()            # 힙 top 확인, 만료된 키 있으면 선삭제
   └─ entry_size = _size_of(key, value) # utf-8 바이트 길이 계산
   └─ maxmemory 초과 여부(OOM) 체크
   └─ existing = self.table.get(key)    # ↓ hashmap.py 호출

3. hashmap.py :: HashMap.get()
   └─ index_for(key) 로 버킷 인덱스 계산  # ↓ _hash() 내부 호출
   └─ 해당 버킷 체인(hash_next)을 순회하며 key 비교
   └─ 없으면 None 리턴 → redis_db.py로 복귀

4. redis_db.py (existing이 None인 경우, 신규 키)
   └─ entry.py :: Entry(key, value) 생성   # 새 Entry 객체 생성
   └─ self.lru.insert_front(entry)         # ↓ doubly_linked_list.py 호출

5. doubly_linked_list.py :: DoublyLinkedList.insert_front()
   └─ entry.lru_next = 기존 head, head.lru_prev = entry
   └─ self.head = entry  (새 Entry가 MRU가 됨)
   └─ redis_db.py로 복귀

6. redis_db.py
   └─ self.table.put(key, entry)          # ↓ hashmap.py 호출

7. hashmap.py :: HashMap.put()
   └─ 버킷에 entry.hash_next로 체이닝 삽입
   └─ size += 1, 로드팩터 0.75 초과 시 _resize() 실행
   └─ redis_db.py로 복귀

8. redis_db.py
   └─ used_memory += entry_size
   └─ _evict_lru()   # maxmemory 설정돼 있으면 LRU 청소
   └─ "OK" 리턴 → main.py 로 복귀 → print("OK")
```

### GET name

```
1. main.py
   └─ dispatch() → db.cmd_get("name") 호출

2. redis_db.py :: MiniRedis.cmd_get()
   └─ _purge_if_expired(key)              # ↓ 만료 여부 lazy 체크
        └─ self.table.get(key)  (hashmap.py get() 재사용)
        └─ entry.expire_at 확인, 만료면 _delete_key_internal() 호출 후 삭제

   └─ entry = self.table.get(key)         # ↓ hashmap.py get() 다시 호출
        (hashmap.py 내부 동작은 SET 때와 동일: index_for → 체인 순회)

   └─ entry가 None이면 "(nil)" 바로 리턴

   └─ entry가 있으면 self.lru.move_to_front(entry)   # ↓ doubly_linked_list.py 호출

3. doubly_linked_list.py :: DoublyLinkedList.move_to_front()
   └─ entry가 이미 head면 아무것도 안 함
   └─ 아니면 entry를 현재 위치에서 떼어내고(prev/next 재연결) head 앞에 재삽입
   └─ redis_db.py로 복귀

4. redis_db.py
   └─ f'"{entry.value}"' 형태로 포맷 → main.py 로 리턴 → print('"jaehyun"')
```

**핵심 포인트**: `cmd_set`/`cmd_get` 둘 다 **hashmap.py의 get()을 재사용**하고,
LRU 갱신이 필요한 시점(신규 삽입/GET 성공/기존 키 덮어쓰기)에만 `doubly_linked_list.py`의
`insert_front`/`move_to_front`를 호출하는 구조. TTL 체크(`_purge_if_expired`)는
GET/EXISTS/DEL/EXPIRE/TTL 등 **키를 다루는 모든 명령어 진입 시점에 공통으로 먼저 실행**됨.

### DEL name

```
1. main.py
   └─ dispatch() → db.cmd_del("name") 호출

2. redis_db.py :: MiniRedis.cmd_del()
   └─ _purge_if_expired(key)
        └─ hashmap.py get() 으로 entry 조회 → 만료면 _delete_key_internal() 로 이미 삭제
        └─ 이미 만료돼서 지워진 거면 "(integer) 0" 바로 리턴 (없던 키 취급)

   └─ (안 만료됐으면) _delete_key_internal(key) 호출

3. redis_db.py :: _delete_key_internal()
   └─ hashmap.py get() 으로 entry 조회
   └─ used_memory -= 바이트 크기
   └─ self.lru.remove_node(entry)          # ↓ doubly_linked_list.py 호출
   └─ self.table.remove(key)               # ↓ hashmap.py 호출

4. doubly_linked_list.py :: remove_node()
   └─ entry가 head/tail/중간 노드인지 판별해서 prev/next 재연결, size -= 1

5. hashmap.py :: remove()
   └─ index_for(key)로 버킷 찾고 체인(hash_next) 순회하며 key 일치하는 노드 끊어냄, size -= 1

6. redis_db.py
   └─ "(integer) 1" 리턴 → main.py → print
```

### EXISTS name

```
1. main.py → dispatch() → db.cmd_exists("name")

2. redis_db.py :: cmd_exists()
   └─ _purge_if_expired(key)              # hashmap.get() 재사용, 만료면 삭제
   └─ self.table.contains(key)            # ↓ hashmap.py 호출

3. hashmap.py :: contains()
   └─ 내부적으로 get(key) 그대로 호출해서 None 여부만 판단

4. redis_db.py
   └─ 있으면 "(integer) 1", 없으면 "(integer) 0" → main.py
```

### DBSIZE

```
1. main.py → dispatch() → db.cmd_dbsize()

2. redis_db.py :: cmd_dbsize()
   └─ _sweep_expired_heap()               # ↓ min_heap.py 호출

3. min_heap.py :: peek() / pop()
   └─ heap top의 (expire_at, key)가 이미 지났으면 pop()
   └─ redis_db.py가 그 key로 hashmap.get() 해서 진짜 만료 상태인지 재확인 후 _delete_key_internal()
   └─ top이 아직 안 지났으면 while 루프 종료 (break)

4. redis_db.py
   └─ self.table.size 속성값 그대로 "(integer) N" 형태로 리턴 (hashmap.py 호출 없이 속성 직접 참조)
```

### KEYS

```
1. main.py → dispatch() → format_keys_output(db.cmd_keys())

2. redis_db.py :: cmd_keys()
   └─ _sweep_expired_heap()               # DBSIZE와 동일 (min_heap.py 호출)
   └─ self.table.keys()                   # ↓ hashmap.py 호출

3. hashmap.py :: keys()
   └─ 버킷 배열 처음부터 끝까지 순회하며 각 체인(hash_next) 따라가서 key 전부 수집

4. main.py :: format_keys_output()
   └─ 리스트를 "1) "key1"\n2) "key2"" 형식의 Redis 스타일 문자열로 변환 후 print
```

### CONFIG SET maxmemory 100

```
1. main.py
   └─ dispatch() 에서 tokens[1]=="SET", tokens[2]=="maxmemory" 형식 검증
   └─ parse_int(tokens[3]) 로 정수 파싱 (실패하면 에러 즉시 리턴, redis_db.py 호출 안 함)
   └─ db.cmd_config_set_maxmemory(100) 호출

2. redis_db.py :: cmd_config_set_maxmemory()
   └─ self.maxmemory = 100
   └─ self._evict_lru() 호출             # 바로 이 시점에 초과분 있으면 즉시 청소

3. redis_db.py :: _evict_lru()
   └─ self.lru.tail 로 LRU 노드 확인       # ↓ doubly_linked_list.py 의 tail 속성 참조
   └─ 초과하는 동안 _delete_key_internal() 반복 (위 DEL 흐름의 3~5단계와 동일)

4. redis_db.py
   └─ "OK" 리턴 → main.py
```

### INFO memory

```
1. main.py → dispatch() → db.cmd_info_memory()

2. redis_db.py :: cmd_info_memory()
   └─ 다른 자료구조 호출 없이, 이미 들고 있는 self.used_memory / self.maxmemory /
      self.evicted_keys 값 3개를 그대로 f-string으로 포맷해서 리턴
```

### EXPIRE foo 20

```
1. main.py
   └─ dispatch() 에서 parse_int(tokens[2]) 로 "20" 파싱
   └─ db.cmd_expire("foo", 20) 호출

2. redis_db.py :: cmd_expire()
   └─ _purge_if_expired(key)              # hashmap.get() 재사용
   └─ entry = self.table.get(key)         # ↓ hashmap.py 호출 (없으면 "(integer) 0")
   └─ seconds <= 0 이면 _delete_key_internal() 후 즉시 리턴

   └─ (정상 케이스) expire_at = 현재시각 + 20
   └─ entry.expire_at = expire_at         # Entry 객체 필드에 직접 기록
   └─ self.expire_heap.push((expire_at, key))   # ↓ min_heap.py 호출

3. min_heap.py :: push()
   └─ 배열 맨 끝에 추가 후 _heapify_up() 으로 부모와 비교하며 위로 이동, 최소 힙 성질 유지

4. redis_db.py
   └─ "(integer) 1" 리턴 → main.py
```

### TTL foo

```
1. main.py → dispatch() → db.cmd_ttl("foo")

2. redis_db.py :: cmd_ttl()
   └─ _purge_if_expired(key)              # hashmap.get() + 만료 시 _delete_key_internal()
   └─ entry = self.table.get(key)         # ↓ hashmap.py 호출

   └─ entry 없으면 "(integer) -2" (키 자체가 없음)
   └─ entry.expire_at 이 None 이면 "(integer) -1" (TTL 설정 안 된 키)

   └─ remaining = entry.expire_at - 현재시각
   └─ remaining <= 0 이면 _delete_key_internal() 후 "(integer) -2"
   └─ 아니면 math.ceil(remaining) 으로 올림 처리해서 "(integer) N" 리턴 → main.py
```

**공통 패턴 정리**

| 명령어 | hashmap.py 호출 | doubly_linked_list.py 호출 | min_heap.py 호출 |
|---|---|---|---|
| SET | get + put | insert_front / move_to_front | (만료 스윕 시) |
| GET | get | move_to_front | (만료 스윕 시) |
| DEL | get + remove | remove_node | - |
| EXISTS | contains(=get) | - | (만료 스윕 시) |
| DBSIZE | size 속성만 | - | peek/pop |
| KEYS | keys() | - | peek/pop |
| CONFIG SET maxmemory | (evict 시 remove) | (evict 시 remove_node) | - |
| INFO memory | - | - | - |
| EXPIRE | get | - | push |
| TTL | get | - | - |

→ **`_purge_if_expired`(지연 만료)** 와 **`_sweep_expired_heap`(능동 만료)** 이 두 개가
사실상 모든 명령어의 진입점에서 "만료된 키부터 치우고 시작"하는 공통 게이트 역할을 함.

---

## 5. 트러블슈팅

### 5-1. `ImportError: cannot import name 'DLLNode'`

- 처음엔 Entry와 별개로 DLL 노드를 감싸는 `DLLNode` 클래스를 쓰려고 했는데,
  `doubly_linked_list.py`를 Entry 자체가 노드 역할을 겸하는 구조로 단순화하면서
  `redis_db.py`의 import 문을 안 고쳐서 발생.
- **해결**: `from doubly_linked_list import DoublyLinkedList` 만 남기고,
  `entry.lru_node` 참조하던 부분들을 전부 `entry` 자체를 넘기는 방식으로 수정.

### 5-2. `hashmap.py` 내부 속성 이름 불일치

- `__init__`에서는 `self.bucket`(단수), `self.size`, `self.bucket_size`로 선언했는데
  다른 메서드들에서 `self.buckets`(복수), `self.count`, `self.capacity`를 참조 → `AttributeError`.
- **해결**: `__init__`에서 선언한 이름 기준으로 전체 통일.

### 5-3. `size()`를 메서드처럼 호출

- `self.lru.size`, `self.table.size`는 **속성(int)**인데 `self.lru.size()`처럼 괄호를 붙여서
  `TypeError: 'int' object is not callable` 발생.
- **해결**: 괄호 제거.

### 5-4. `put()`이 값 갱신을 안 함

- 같은 key로 `put()` 호출 시 주석은 "값 업데이트"라고 되어 있는데 실제로는
  `return current`만 하고 `value` 갱신 로직이 빠져있었음.
- **해결**: `current.value = entry.value` 추가.

---

## 6. 실행 로그

### 6-1. 기본 CRUD

```
mini-redis> SET name jaehyun
OK
mini-redis> GET name
"jaehyun"
mini-redis> EXISTS name
(integer) 1
mini-redis> DEL name
(integer) 1
mini-redis> GET name
(nil)
mini-redis> DBSIZE
(integer) 0
```

### 6-2. TTL 실제 만료 (EXPIRE 20초로 여유 있게 테스트)

```
mini-redis> set foo bar
OK
mini-redis> expire foo 20
(integer) 1
mini-redis> ttl foo
(integer) 16
mini-redis> get foo
"bar"
mini-redis> ttl foo
(integer) 1
mini-redis> ttl foo
(integer) -2
mini-redis> get foo
(nil)
```

- TTL이 16 → 1로 줄어드는 건 타이핑하는 사이 시간이 흐른 것.
- 완전히 만료되면 TTL은 "TTL 없음(-1)"이 아니라 **"키 없음(-2)"** 으로 바뀜 → 만료된 키는
  `_purge_if_expired()`에 의해 실제로 삭제되기 때문.

### 6-3. LRU eviction + 메모리 관리 (maxmemory=12바이트)

```
mini-redis> CONFIG SET maxmemory 12
OK
mini-redis> SET k1 v1
OK
mini-redis> SET k2 v2
OK
mini-redis> SET k3 v3
OK
mini-redis> GET k1
"v1"
mini-redis> SET k4 v4
OK
mini-redis> KEYS
1) "k3"
2) "k4"
3) "k1"
mini-redis> EXISTS k2
(integer) 0
mini-redis> INFO memory
used_memory:12
maxmemory:12
evicted_keys:1
```

- k1/k2/k3 순서로 넣고 GET k1으로 k1을 MRU로 갱신했더니, SET k4로 용량 초과됐을 때
  **LRU였던 k2가 제거**되고 k1은 살아남음 (`evicted_keys: 1`로 정확히 반영).

### 6-4. 해시맵 리사이징 (bucket_size=8, load_factor=0.75)

```python
>>> from redis_db import MiniRedis
>>> db = MiniRedis()
>>> for i in range(15):
...     print('SET key%d val%d' % (i, i), '->', db.cmd_set(f'key{i}', f'val{i}'))
...
(SET key0 ~ key14 모두 OK)
>>> print('bucket_size:', db.table.bucket_size)
bucket_size: 32
>>> print('size:', db.table.size)
size: 15
```

- 계산: 7번째 삽입(7/8=0.875 > 0.75) → 8 → 16 확장 / 13번째 삽입(13/16=0.8125 > 0.75) → 16 → 32 확장.
- 15개 넣고 최종 `bucket_size=32`, `size=15` → 이론값과 정확히 일치.