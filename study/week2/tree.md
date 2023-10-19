# 트리
## 개념
트리는 상하관계를 나타내는데 사용되는 추상 자료형입니다. 트리는 노드(node)와 가지(branch, edge)로 이루어져 있습니다. 즉, 트리는 그래프의 집합의 원소입니다.

1. 루트: 트리의 가장 위쪽에 있는 노드
2. 리프: 자식이 없는 노드
3. 비단말 노드: 리프 노드가 아닌 노드
4. 자식: 어떤 노드와 가지가 연결되었을 때 아랫쪽 노드
5. 부모: 어떤 노드와 가지가 연결되었을 때 위쪽 노드
6. 형제: 부모가 같은 노드
7. 조상: 어떤 노드의 상위 노드들, 루트 노드는 모든 노드의 조상입니다.
8. 자손: 어떤 노드의 하위 노드들, 루트 노드가 아닌 노드는 루트 노드의 자손입니다.
9. 레벨: 루트에서 얼마나 떨어져 있는가?
10. 차수: 각 노드가 갖는 자식의 수 중 제일 큰 것
11. 높이: 루트에서 가장 멀리 있는 리프까지의 길이
12. 서브트리: 어떤 노드를 루트로 하고 그 자손으로 구성된 트리
13. 빈 트리: 노드와 가지가 전혀 없는 트리
14. 포레스트: 서로 다른 루트를 가진 트리들의 모임

## 순회
마찬가지로, 트리 순회에도 여러 전략들이 있습니다. 대표적으로 전위 순회, 중위 순회, 후위 순회가 있습니다. 아래 예제에서는 이진 트리(트리의 차수가 2인 트리)를 순회합니다.
### 전위 순회
``` python
def preorder(node):
    print(node.name)
    if node.left:
        preorder(node.left)
    if node.right:
        preorder(node.right)    
```

### 중위 순회
```python
def inorder(node):
    if node.left:
        inorder(node.left)
    print(node.name)
    if node.right:
        inorder(node.right)    
```

### 후위 순회
```python
def postorder(node):
    if node.left:
        postorder(node.left)
    if node.right:
        postorder(node.right)
    print(node.name)    
```
