# 문제 풀이
## 줄 세우기 vs 구슬 찾기
줄 세우기 문제는 방향 비순환 그래프에서 줄을 세운 결과를 출력하면 됩니다. 방향 비순환 그래프라고 판단한 이유는 대소 관계가 주어졌고 사이클이 없어야 하기 때문입니다. 사이클이 있으면 논리적으로 모순이 생깁니다. 예를 들어 '1이 2보다 크다'와 '2가 3보다 크다'는 관계가 있을 때 '3이 1보다 크다'라는 관계가 더 있다면 서로 모순을 일으키기 때문입니다. 그래서 사이클이 발생하는 것이 불가능 합니다. 어쨌든, 줄을 세운 결과만 출력하면 되므로 위상 정렬을 사용하는 것이 타당합니다.

마찬가지로 구슬 찾기 문제도 방향 비순환 그래프일 것이므로 위상 정렬을 사용할 수 있을 것처럼 보입니다. 그러나 구슬 찾기 문제는 다릅니다. 위상 정렬을 수행한다고 해서 무게가 중간이 될 수 없는 구슬의 개수를 구할 수 없습니다. 그런 구슬을 찾기 위해서 추가적으로 그래프 탐색을 진행 해야 합니다. 위상 정렬의 결과를 이용하는 것도 아니고, 위상 정렬을 위한 탐색 과정에서 문제 해결에 필요한 정보를 얻는 것도 아닙니다.

따라서 구슬 찾기 문제는 바로 그래프 탐색을 진행하거나, 플로이드 워셜 알고리즘을 이용합니다. 플로이드 워셜 알고리즘을 이용하면 상당히 간편하게 풀 수 있습니다. 플로이드 워셜 알고리즘은 모든 쌍의 최단 거리를 구합니다. 쌍 `(i, j)`에 최단 거리가 있다고 하는 것은 `i`에서 `j`로 가는 경로가 존재한다는 것입니다. 이 말을 구슬 찾기 문제에 대입해보면 플로이드 워셜 알고리즘의 결과에서 쌍 `(i, j)`가 올바른 결과라면 구슬 `i`와 `j`를 비교할 수 있다는 뜻입니다. 

이후에는 모든 구슬에 대해서 좌, 우에 있는 구슬의 갯수가 전체 구슬의 수의 절반을 넘는지 확인하는 작업만 수행하면 됩니다. 만약 넘는다면 무게가 중간인 구슬이 될 수 없기 때문입니다.

또한 구슬 찾기 문제는 DFS로도 풀 수 있습니다. 각각의 구슬에 대해서 DFS를 수행하고 찾은 구슬의 갯수가 전체 구슬의 절반을 넘는지를 확인하면 됩니다.

## 동전 2
Coin change making problem이라는 유명한 문제입니다. 그러나 어떤 수의 공배수 관계로 이루어진 경우에만 탐욕적으로 가장 가치가 큰 동전부터 선택해 나가는 것이 최적해임을 보장합니다. 하지만 이 문제는 그 방법을 적용할 수 없어서 보통은 DP를 이용해서 풀이를 하는데 이 문제의 분류가 BFS인 것을 보고 한참 고민했습니다. 정말 정말 생각이 안나서 인터넷을 좀 찾아봤는데, 결국에는 BFS를 이용해서 메모하며 풉니다.
``` python
from collections import deque

num_coin, target_value = map(int, input().split())
coins = list(int(input()) for _ in range(num_coin))
memo = [0 for _ in range(10001)]
q = deque()

for i in range(num_coin):
    if coins[i] < 10001:
        memo[coins[i]] = 1
        q.append(coins[i])


def find():
    global num_coin 
    global target_value
    global q    

    while q:
        c = q.popleft()

        for i in range(num_coin):
            new_value = coins[i] + c
            if new_value < 10001:
                if memo[new_value] == 0:
                    memo[new_value] = memo[c] + 1
                    q.append(new_value)
find()
print(-1 if memo[target_value] == 0 else memo[target_value])
```

## 탈출
매 시간마다 지형이 계속 바뀌므로 고슴도치가 물에 빠졌는지 빠지지 않았는지를 고려해야한다. 고슴도치(S)가 빠지게 되는 경로는 물(*)로 덮어버리고 경로에서 제하면 된다. 그래서 아래와 같이 작동한다. 예제 1에서
```
Phase 1 (time = 0)
D . *
. . .
. S .

=========
Phase 2 (time = 1)
D * *
. S *
S S S

=========
Phase 3 (time = 2)
D * *
S * *
S S *

=========
Phase 4 (time = 3)
D * *
* * *
* * * 
```
마지막 단계에 고슴도치가 비버의 굴에 도착하게 되면서 3이 출력된다. 각 단계마다 고슴도치가 먼저 이동하고, 그 다음 물이 퍼지게 한다. 그러면 자연스럽게 고슴도치가 이동했으면서 다음에 물에 잠기는 경로가 제외된다. 마지막으로, 고슴도치가 도착할 수 없는 경우에 프로그램을 종료해야하는데, 간단하게 이전 숲을 각 이동마다 미리 복제해놓고, 숲과 이전 숲이 같은 상태이면 종료하게 만들었다. 아마도 한 번의 BFS로 푸는 방법이 있을 것이지만, 페이즈를 나눠서 생각하는게 편하기도 하고, 게임을 만드는 것 같아서 더 좋았다. 그런데 코딩 테스트 때에는 한 번의 BFS로 풀어야겠지...

## 임계 경로
문제 이름부터 위상 정렬을 쓰라고 힌트를 줬다. 위상 정렬을 하여 최장경로를 정하고 도착지부터 시작지까지 DFS로 탐색하면 된다. 이 때 최장 경로가 아니게 되는 부분은 제외하고 탐색하면 되므로 시간 복잡도가 크게 감소한다.

## 미로 만들기
미로 만들기를 처음 봤을 떄 예전에 풀었던 [연구소](https://www.acmicpc.net/problem/14502) 문제와 비슷하다는 생각이 들었다. 그러나 미로 만들기 문제는 연구소 문제와 비슷하게 풀려고 시도하면 시간 초과가 발생할 것이다. 단순히 BFS를 한 번만 돌리면 해결할 수 있는 문제였다.

``` python
import sys
from collections import deque

input = sys.stdin.readline
MAX = float("inf")
n = int(input())
dx = [0, 0, 1, -1]
dy = [1, -1, 0, 0]
maze = []
for i in range(n):
    maze.append(list(map(int, list(input().split('\n')[0]))))

visited = [[MAX for _ in range(n)] for _ in range(n)]
start = (0, 0)
visited[0][0] = 0
end = (n - 1, n - 1)
q = deque()
q.append(start)
while q:
    x, y = q.popleft()
    for i in range(len(dx)):
        nx = x + dx[i]
        ny = y + dy[i]

        if not (0 <= nx < n and 0 <= ny < n):
            continue

        if visited[nx][ny] > visited[x][y] and maze[nx][ny] == 1:
            visited[nx][ny] = visited[x][y]
            q.append((nx, ny))
            continue

        if visited[nx][ny] > visited[x][y] + 1 and maze[nx][ny] == 0:
            visited[nx][ny] = visited[x][y] + 1
            q.append((nx, ny))

result = visited[n - 1][n - 1] if visited[n - 1][n - 1] != MAX else 0
print(result)
```

0-1 너비 우선 탐색이라는 기술도 사용할 수 있는데 이 경우에는 최소 힙을 사용하는 것이다. 최소의 비용으로 갈 수 있는 노드를 계속해서 집어넣으면 비용이 큰 노드는 자연스럽게 잘 선택되지 않는다. 그렇게 첫번째로 도착하는 경로를 찾으면 그것이 최적이 된다.

