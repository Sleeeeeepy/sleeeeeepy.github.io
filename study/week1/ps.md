# 문제 풀이
## 철로
주어진 문제 중에 가장 어려운 문제 중 하나였다. 철로 문제를 분석하던 도중 활동 선택 문제와 크게 다를 것이 없다는 것을 깨닫게 되었다. 일단 각 사람들의 집의 위치와 회사의 위치를 입력받게 되는데, 집의 위치와 회사의 위치의 순서는 중요하지 않으므로 둘 중에 작은 값이 튜플의 첫 요소가 되고, 큰 값이 튜플의 큰 요소가 되게 해야한다. 그리고 두 번째 요소를 기준으로 오름차순으로 정렬한다.

이제 이 정렬된 리스트에서 첫번째 회사의 위치를 가져오고 철로를 정확히 그 곳에 놓는다고 가정하자. 그렇다면 철로의 길이를 `L`, 집의 위치를 `h`, 회사의 위치를 `c`라고 두었을 때, `c - h`가 `L`보다 크다면 그 사람은 철로가 놓일 곳에 포함되지 않을 것이다. 그리고 `abs(c - h) <= L`이면 그 사람은 철로가 놓일 곳에 포함된다.

그렇게 계산을 마친 후 다음 사람의 회사의 위치를 기준으로 철로를 놓는다면 이전의 계산을 반복 해야한다. 즉, 우리가 순진하게 알고리즘을 작성한다면 시간 복잡도가 $O(n^2)$이 될 것이다. 그러나 이 방식으로는 제한 시간 안으로 문제를 해결할 수 없다.

이 문제를 해결하는 효과적인 방법은 집의 위치를 기준으로 하는 최소 힙을 사용하는 것이다. 각 사람에 대하여 `c - L <= h`를 만족하는 사람을 최소 힙에 집어넣고, 최소 힙에서 `c - L > h`를 만족하는 모든 원소를 제거한다. 우리는 사전에 두 번째 요소(회사의 위치)를 기준으로 리스트를 정렬하였기 때문에 최소 힙에서 제거된 원소는 다시 힙에 들어가지 않는다는 것이 보장된다. 즉 이전의 계산을 다시 수행하지 않아도 된다.

``` python
from queue import PriorityQueue

pq = PriorityQueue()
num_men = int(input())
home_to_company = []
for i in range(num_men):
    x, y = map(int, input().split())
    if x > y:
        x = x ^ y
        y = x ^ y
        x = x ^ y
    home_to_company.append((x, y))

rail_length = int(input())
home_to_company.sort(key=lambda x:(x[1]))
answer = 0
current = 0
for i in range(num_men):
    # c에서 시작한다고 가정
    h, c = home_to_company[i]
    if abs(c - h) > rail_length:
        continue
    
    if c - rail_length <= h:
        current += 1
        pq.put(h)
    
    while not pq.empty():
        home = pq.get()
        if c - rail_length > home:
            current -= 1
            continue
        else:
            pq.put(home)
            break

    answer = max(current, answer)

print(answer)
```

### 한줄평
Python의 `PriorityQueue`에는 왜 `peek()`연산이 없을까...

## 사냥꾼
동물의 위치와 사대의 위치, 총의 사거리가 주어지고, 사냥꾼이 모든 사대를 돌면서 총을 쏴 잡을 수 있는 동물의 수를 탐색하는 문제이다. 이 문제를 순진하게 탐색할 수 없는데, 가능한 `x`좌표와 `y`좌표가 너무 커서, 탐색을 수행하는데 너무 많은 시간이 필요하다.

이러한 문제에서는 이진 탐색을 수행한다. 문제는 어떤 것을 기준으로 이진 탐색을 수행하느냐는 것이다. 사대를 기준으로 동물을 찾든, 동물을 기준으로 사대를 찾든 둘 중 하나는 반드시 수행 해야한다.

``` python 
num_shooting_point, num_animal, gun_range = map(int, input().split())
shooting_points = list(map(int, input().split()))
animals = []
num_dead_aniaml = 0
shooting_points.sort()
for i in range(num_animal):
    x, y = map(int, input().split())
    animals.append((x, y))

# x의 어느 지점에 동물을 죽일수 있는 사대가 있는가?
def bsearch(animal, start, end):
    x, y = animal
    if start > end:
        return False
    
    mid = (start + end) // 2
    if can_kill(x, y, shooting_points[mid]):
        return True

    if x < shooting_points[mid]:
        return bsearch(animal, start, mid - 1)
    else:
        return bsearch(animal, mid + 1, end)

# 동물의 좌표와 사대의 본래 위치 shooting_point로 좌표를 계산합니다.
def can_kill(x, y, shooting_point):
    global gun_range
    result = gun_range - (abs(x - shooting_point))

    return result >= y

for animal in animals:
    if bsearch(animal, 0, len(shooting_points) - 1):
        num_dead_aniaml += 1

print(num_dead_aniaml)
```

## 괄호의 값
오랜만에 보는 스택 문제이다. 이제 괄호와 관련된 문제가 나오면 스택부터 생각하게 된다. 이 문제는 아래와 같은 문법으로 이루어져 있다.
```
term := () | [] | ( term ) | [ term ] | term term
```
이 때 두 항을 나란히 놓으면 더하기 연산이 되고, 하나의 항을 괄호 안에 감싸면 소괄호는 2를 곱하고, 대괄호는 3을 곱한다. 그리고 (), []는 각각 2와 3이라는 값을 갖는다. 처음에는 이를 해결하기 위해서 재귀 하강 파서를 구현하려고 하였다. 그러나 굳이 재귀 하강 파서까지 만들 필요는 없을 뿐더러, 이 문법에서는 왼쪽 재귀 문제가 있기 때문에 시간을 더 쓸 이유가 없다. 문제 자체는 스택 하나로 해결할 수 있기 때문이다.

규칙은 다음과 같다. 
1. 스택에는 세 종류의 문자만 들어간다. `(`, `[`, 그리고 숫자가 들어간다. 
2. 괄호가 짝이 맞지 않는 경우, 예를 들어 `())`, `[(])`와 같은 케이스를 제외한다. 
3. 스택에서 여는 괄호가 있고 현재 닫는 괄호가 있다면 여는 괄호를 스택에서 빼내고 숫자 2나 3을 스택에 집어넣는다.
4. 스택에서 숫자가 연속하여 있다면 모두 더한다. 그리고 괄호가 있는지 확인한다. 괄호가 있다면 규칙에 맞는 수를 곱하고 괄호를 스택에서 뺀다.

네 가지 규칙이 전부이다.

``` python
stack = []
brackets = input()

def error():
    print("0")
    exit()

for b in brackets:
    if b == "(":
        stack.append("(")
    elif b == "[":
        stack.append("[")
    elif b == ")":
        if not stack or stack[-1] == "[":
            error()
        if stack[-1].isnumeric():
            num = int(stack.pop())
            while stack and stack[-1].isnumeric():
                if stack and stack[-1].isnumeric():
                    num += int(stack.pop())
            if (stack and stack[-1] == "("):
                stack.pop()
            else:
                error()
            stack.append(f"{num * 2}")
        elif stack[-1] == "(":
            stack.pop()
            stack.append("2")
    elif b == "]":
        if not stack or stack[-1] == "(":
            error()
        if stack[-1].isnumeric():
            num = int(stack.pop())
            while stack and stack[-1].isnumeric():
                if stack and stack[-1].isnumeric():
                    num += int(stack.pop())
            if (stack and stack[-1] == "["):
                stack.pop()
            else:
                error()
            stack.append(f"{num * 3}")
        elif stack[-1] == "[":
            stack.pop()
            stack.append("3")

result = 0
for s in stack:
    if s.isnumeric():
        result += int(s)
    else:
        error()
print(result)
```