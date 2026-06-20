---
title: 프로그래머스 배달
date: 2026-06-20 22:07
category: CP
tags:
- Programmers
- Dijkstra
---

[문제 바로가기](https://school.programmers.co.kr/learn/courses/30/lessons/12978)

---

## 풀이 과정

```text
N개 마을
양방향 통행
K 이하만 카운트
---
그래프 생성
1번이 기준
distances 처음빼고 다 최대값 - float('inf')
deque - 우선순위 큐

pq 반복
    - 그래프에서 거리, 도착노드 pop
    - 이미 최단 거리면 건너뛰기(메모리 관리)
    그래프에서 현재 갈 수 있는 최단 거리의 노드 찾아 반복
	       
        - 현재까지 온 거리+각 노드의 거리
        - distances에 있는 거리보다 작으면
	        - distances에 넣기
	        - 해당 노드 heap 넣기(어떻게든 넣어도 pop하면 최소값 부터 나옴)(거리, 노드)
(grpah, 1) 시작
K 이하 노드만 카운트
```

---

## 코드

```python
import heapq

def solution(N, road, K):
    
    graph = [[] for _ in range(N+1)]
    for s, e, w in road:
        graph[s].append((e, w)) # s: 출발노드, e: 도착노드, w: 거리(가중치)
        graph[e].append((s, w))
        
    def dijkstra(graph, start):
        dsts = [float('inf')] * (N+1)
        dsts[start] = 0
        pq = [(0, start)]
        
        while pq:
            cur_dst, cur_node = heapq.heappop(pq)
            
            if cur_dst > dsts[cur_node]:
                continue
                
            for neighbor, weight in graph[cur_node]:
                dst = cur_dst + weight
                
                if dst < dsts[neighbor]:
                    dsts[neighbor] = dst
                    heapq.heappush(pq, (dst, neighbor))
        return dsts
    
    dsts = dijkstra(graph, 1)
    
    return sum(1 for d in dsts[1:] if d <= K)
```
