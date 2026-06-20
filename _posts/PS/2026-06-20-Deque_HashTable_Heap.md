---
title: 자료 구조 2
date: 2026-06-20 16:13
category: PS
tags:
- Data Structure
- Deque
- Hash Tabale
- Heap
---

> "덱, 해시 테이블, 힙"

---

## 목표

> 덱, 해시 테이블, 힙의 특징과 메서드에 대해 알아봅니다.

---

## Deque

- Double Ended Queue(양방향 큐)이다.
- 양쪽 끝에서 값을 넣고 뺄 수 있다.
- 리스트로도 가능은 한데, **모든 요소를 이동**해야하기에 큐나 스택으로 사용할 때는 덱이 시간적으로 유리하다.

#### 큐와 스택

```python
[큐 - FIFO]

from collections import deque

dq = deque()
dq.append(1)   # 뒤쪽 추가
dq.append(2)
dq.append(3)

# 앞쪽에서 제거 (popleft)
print(dq.popleft())  # 1
print(dq.popleft())  # 2
print(dq.popleft())  # 3
```

```python
[스택 - LIFO]

from collections import deque

dq = deque()
dq.append(1)   # 뒤쪽 추가
dq.append(2)
dq.append(3)

# 뒤쪽에서 제거 (pop)
print(dq.pop())  # 3
print(dq.pop())  # 2
print(dq.pop())  # 1
```

---

#### 메서드

```python
from collections import deque

a = deque([1, 2, 3, 4, 5]) # 메서드 쓰면 항상 초기화한다고 가정

# append: 오른쪽 끝에 값 추가
a.append(6)         # deque([1, 2, 3, 4, 5, 6])

# appendleft: 왼쪽 끝에 값 추가
a.appendleft(0)     # deque([0, 1, 2, 3, 4, 5])

# pop: 오른쪽 끝 값 반환 후 삭제
b = a.pop()         # a = deque([1, 2, 3, 4])
b                   # 5

# popleft: 왼쪽 끝 값 반환 후 삭제
b = a.popleft()     # a = deque([2, 3, 4, 5])
b                   # 1

# extend: 오른쪽에 다른 리스트(반복가능객체) 연결
a.extend([6, 7])    # deque([1, 2, 3, 4, 5, 6, 7])

# extendleft: 왼쪽에 연결 (역순으로 들어감 주의!)
a.extendleft([0, -1])  # deque([-1, 0, 1, 2, 3, 4, 5])

# insert: 특정 위치에 추가
a.insert(0, 10)     # deque([10, 1, 2, 3, 4, 5])

# remove: 특정 값 제거 (제일 먼저 있는 값만)
a.remove(1)         # deque([2, 3, 4, 5])

# count: 해당 값 세기
b = deque([1, 1, 1, 2, 3])
b.count(1)          # 3

# index: 해당 값의 위치 (제일 먼저 찾은 값만)
a.index(2)          # 1

# reverse: 뒤집기 (원본 변경, None 반환)
a.reverse()         # deque([5, 4, 3, 2, 1])

# rotate: 회전 (오른쪽으로 n칸, 음수면 왼쪽)
a.rotate(1)         # deque([5, 1, 2, 3, 4])
a.rotate(-1)        # deque([2, 3, 4, 5, 1])

# clear: 모든 값 삭제
a.clear()           # deque([])
```

---

## Hash Table

- 키(Key)에 데이터(Value)를 저장하는 데이터 구조이다.
- Key를 통해 데이터를 바로 받아올 수 있다.
- 파이썬에서는 해시를 별도 구현할 필요 없다.(딕셔너리 타입)
- **Set과 Dictionary는 해시 기반**이다.

#### 작동 원리

```text
입력 키    → 해시 함수 → 해시값 (숫자) → 인덱스 계산 → 메모리 위치
'cherry'   → hash()    → 987654321    → 987654321 % 8 = 5 → bucket[5]
'apple'    → hash()    → 123456789    → 123456789 % 8 = 1 → bucket[1]
```

#### Set

- 순서가 보장되지 않는다.
- 중복을 허용하지 않는다.

##### 메서드

```python
# 주의: set은 순서가 없어서 인덱스 접근(a[0]) 불가!

a = {1, 2, 3, 4, 5}# 메서드 쓰면 항상 초기화한다고 가정

# add: 값 하나 추가
a.add(6)            # {1, 2, 3, 4, 5, 6}
a.add(1)            # {1, 2, 3, 4, 5}  이미 있으면 변화 없음(중복 불가)

# update: 여러 값 한번에 추가 (리스트/셋 등)
a.update([6, 7])    # {1, 2, 3, 4, 5, 6, 7}

# remove: 특정 값 제거 (없으면 KeyError 에러!)
a.remove(1)         # {2, 3, 4, 5}

# discard: 특정 값 제거 (없어도 에러 안 남)
a.discard(1)        # {2, 3, 4, 5}
a.discard(100)      # {1, 2, 3, 4, 5}  없어도 조용히 넘어감

# pop: 임의의 값 반환 후 삭제 (어떤 값일지 모름!)
b = a.pop()         # a = {2, 3, 4, 5} (예시), b = 1
                    # 맨 앞이 아니라 '임의' 원소

# clear: 모든 값 삭제
a.clear()           # set()

# --- 집합 연산 (set의 핵심) ---
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

# union: 합집합 (| 연산자도 동일)
a.union(b)          # {1, 2, 3, 4, 5, 6}
a | b               # {1, 2, 3, 4, 5, 6}

# intersection: 교집합 (& 연산자)
a.intersection(b)   # {3, 4}
a & b               # {3, 4}

# difference: 차집합 (- 연산자)
a.difference(b)     # {1, 2}  (a에는 있고 b에는 없는 것)
a - b               # {1, 2}

# symmetric_difference: 대칭차집합 (^ 연산자)
a.symmetric_difference(b)  # {1, 2, 5, 6}  (교집합 빼고 전부)
a ^ b                      # {1, 2, 5, 6}

# issubset / issuperset: 부분집합 / 상위집합 판별
{1, 2}.issubset(a)         # True  ({1,2}가 a에 포함되나?)
a.issuperset({1, 2})       # True  (a가 {1,2}를 포함하나?)
```

#### Dictionary

- 순서를 가지지 않는다.
- key-value 쌍으로 이루어져있다.
- type은 `dict` 로 표시된다.

```python
a = {'x': 1, 'y': 2, 'z': 3} # 메서드 쓰면 항상 초기화한다고 가정

# 접근/추가/수정: 키로 직접
a['x']              # 1
a['w'] = 4          # {'x':1, 'y':2, 'z':3, 'w':4}  없는 키면 추가
a['x'] = 100        # {'x':100, 'y':2, 'z':3}  있는 키면 수정

# get: 키로 값 조회 (없는 키도 에러 안 남)
a.get('x')          # 1
a.get('w')          # None       없으면 None
a.get('w', 0)       # 0          없을 때 줄 기본값 지정 가능

# keys / values / items: 키, 값, 쌍 꺼내기
a.keys()            # dict_keys(['x', 'y', 'z'])
a.values()          # dict_values([1, 2, 3])
a.items()           # dict_items([('x',1), ('y',2), ('z',3)])

# in: 키 존재 여부 (값이 아니라 '키' 기준! O(1))
'x' in a            # True
1 in a              # False  ← 값 1이 아니라 키 1을 찾음

# pop: 특정 키 제거 후 값 반환 (없으면 KeyError)
b = a.pop('x')      # a = {'y':2, 'z':3}, b = 1
b = a.pop('w', -1)  # 없을 때 기본값 주면 에러 안 남, b = -1

# update: 다른 dict로 한번에 추가/수정
a.update({'y': 20, 'k': 5})  # {'x':1, 'y':20, 'z':3, 'k':5}

# setdefault: 키 있으면 값 반환, 없으면 추가하고 반환
a.setdefault('x', 99)   # 1   (이미 있어서 기존값)
a.setdefault('w', 99)   # 99  (없어서 추가됨) → a에 'w':99 생김

# clear: 모든 값 삭제
a.clear()           # {}
```

---

## Heap

- **최댓값과 최솟값을 찾는 연산을 빠르게** 하기 위해 고완된 완전 이진트리를 기본으로 한다.

#### 작동 원리

```python
import heapq

# 최소 힙 생성
heap = []
heapq.heappush(heap, 5)
heapq.heappush(heap, 1)
heapq.heappush(heap, 3)

print(heap)  # [1, 5, 3] (최소 힙: 루트 = 1) good

# heappop → 가장 작은 값 반환
print(heapq.heappop(heap))  # 1 (가장 작은 값) good
print(heapq.heappop(heap))  # 3
print(heapq.heappop(heap))  # 5
```

- heapq는 기본 **min_heap**으로 설정되어 있다.(최소힙)
- 그래서, 최대힙으로 하고 싶으면 음수로 하면 된다.
- 파이썬은 실제 트리구조가 아니라 배열을 힙 규칙으로 정렬한다.

```python
# 최소 힙 (Python heapq):
heap = [1, 2, 3, 10, 5]

# 규칙: 모든 k 에 대해
# heap[k] <= heap[2*k+1] AND heap[k] <= heap[2*k+2]

# 인덱스 0 (값 1):
#   heap[0] <= heap[1] → 1 <= 2 good
#   heap[0] <= heap[2] → 1 <= 3 good

# 인덱스 1 (값 2):
#   heap[1] <= heap[3] → 2 <= 10 good
#   heap[1] <= heap[4] → 2 <= 5 good
```

#### 메서드

```python
import heapq

# 핵심: heapq는 '최소 힙' — 항상 가장 작은 값이 맨 앞(a[0])

a = [1, 2, 3, 4, 5] # 함수 쓰면 항상 초기화한다고 가정

# heapify: 일반 리스트를 힙 구조로 변환 (O(n), 원본 변경)
heapq.heapify(a)        # [1, 2, 3, 4, 5] 힙 구조로 재배열

# heappush: 값 추가 (O(log n))
heapq.heappush(a, 0)    # 0이 들어가고 a[0] == 0

# heappop: 최솟값 반환 후 삭제 (O(log n))
b = heapq.heappop(a)    # b = 1 (가장 작은 값), a에서 제거됨

# 최솟값 확인만 (꺼내지 않고 보기, O(1))
a[0]                    # 현재 가장 작은 값

# heappushpop: push 먼저, 그 다음 pop (한 번에)
b = heapq.heappushpop(a, 10)  # 10 넣고 최솟값 꺼냄

# heapreplace: pop 먼저, 그 다음 push (순서 반대!)
b = heapq.heapreplace(a, 10)  # 최솟값 꺼내고 10 넣음

# nlargest / nsmallest: 큰/작은 n개 (정렬된 리스트로 반환)
heapq.nlargest(3, a)    # 가장 큰 3개 [내림차순]
heapq.nsmallest(3, a)   # 가장 작은 3개 [오름차순]
```