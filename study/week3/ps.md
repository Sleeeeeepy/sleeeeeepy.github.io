# 문제 풀이
## 행렬 체인 곱셈
``` python
count = 0
def matrix_chain_order(i, j):
    if (i == j):
        return 0

    if dp[i][j] != float("inf"):
        return dp[i][j]
    
    result = 2147483647
    for k in range(i, j):
        result = min(result, 
                     matrix_chain_order(i, k) +
                     matrix_chain_order(k + 1, j) +
                     matrix[i][0] * matrix[k][1] * matrix[j][1])
        
    dp[i][j] = result
    return dp[i][j]

print(matrix_chain_order(0, len(matrix) - 1))

```
점화식을 알기만 하면 재귀를 이용하는 방법은 정말 쉽게할 수 있습니다. 책에 나온 그대로 재귀함수를 구성하면 끝. 이 풀이법은 시간복잡도가 $O(n^3)$입니다. 그러나 수학자들이 열심히 연구한 결과 이 문제를 다각형을 삼각형으로 분할하는 문제로 환원하여 $O(n\log n)$에 풀 수 있다는 사실을 발견했습니다. 이를 Hu & Shing 알고리즘이라고 합니다.

## 동전(9084) vs 동전 0(11047)
동전 문제는 **coin change problem**이라고 불리는 유명한 문제 중 하나입니다. 이 문제는 특수한 상황에서 그리디 알고리즘으로 최적해를 도출할 수 있지만 다른 상황에서는 그렇지 못합니다. 동전이 배수관계라면 잔돈 100원을 만들 때 100원 짜리 1개를 선택하는 것이 50원 짜리를 2개 선택하는 것보다, 10원 짜리를 10개 선택하는 것보다 좋습니다. 그러나 배수 관계가 아니라면 옳지 않은 결과를 도출합니다. 잔돈 15원을 1원, 5원, 12원짜리 동전을 이용해서 만들 때 동전의 갯수의 최솟값은 12원 한 개, 1원 3개를 고르는 것이 아니라 5원 3개를 고르는 것입니다.
``` python
def test_memo():
    num_coin_kinds = int(input())
    coins = list(map(int, input().split()))
    target_change = int(input())
    dp = [[-1 for _ in range(10002)] for _ in range(22)]
    def solve(cursor, change):
        if change < 0:
            return 0
        
        if change == 0:
            return 1
        
        if dp[cursor][change] != -1:
            return dp[cursor][change]
        
        dp[cursor][change] = 0
        for i in range(cursor, num_coin_kinds):
            dp[cursor][change] += solve(i, change - coins[i])
        return dp[cursor][change]
    
    print(solve(0, target_change))

for i in range(num_testcase):
    test_memo()
```

## 멀티탭 스케줄링
스케줄링에는 여러 방법이 있습니다. LRU(least recently used), random, second chance, MRU(most recently used), FIFO(first in first out)... 그러나 이 문제에서는 최적(optimal)의 상황을 구해야합니다. 가장 나중에 사용하는 전자기기의 플러그를 뽑으면 최적입니다. 이미 꽂혀있는 경우 cache hit입니다. 그냥 넘어갑니다. 멀티탭이 비어있는 경우 cold miss입니다. 멀티탭에 기기를 꽂고 넘어갑니다. 멀티탭이 가득 찼으면 capacity miss입니다. 이제 멀티탭에 꽂혀있는 전자기기 중 희생양을 고릅니다. 가장 오랫동안 사용하지 않는 것을 희생양으로 고릅니다. 만약 이러한 희생양이 둘 이상이 된다면 아무나 없애도 상관 없습니다. 그리고 이제 새 기기를 멀티탭에 꽂습니다. 아래는 이를 그대로 구현한 코드입니다.
``` python
import sys
input = sys.stdin.readline

num_socket, num_uses = map(int, input().split())
uses = list(map(int, input().split()))
socket = set()

answer = 0
for i in range(num_uses):
    # 이미 꽂힌 경우
    if uses[i] in socket:
        continue

    # 멀티탭이 비어있는 경우
    if len(socket) < num_socket:
        socket.add(uses[i])
        continue

    # 이미 멀티탭이 찬 경우
    # 꽂혀있는 전선 중 가장 오랫동안 사용하지 않는 것을 선택한다.
    victim = 0
    max_term = -1
    for j in socket:
        term = 0
        for k in range(i, num_uses):
            if j != uses[k]:
                term += 1
            else:
                break
        
        if max_term < term:
            victim = j
            max_term = term
    
    socket.remove(victim)
    socket.add(uses[i])
    answer += 1

print(answer)
```
