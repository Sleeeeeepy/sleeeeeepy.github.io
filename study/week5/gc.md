# 가비지 컬렉션
## 개요
가비지 컬렉션은 동적으로 할당된 메모리 영역 가운데 더 이상 접근할 수 없는 영역을 탐색해 삭제하는 기법입니다. 가비지 컬렉션은 언어 수준, 혹은 라이브러리 수준에서 지원되는 경우가 많습니다. 대표적으로 LISP, Java, C# 등의 언어는 언어 수준에서, C++의 경우에는 라이브러리 수준에서 가비지 컬렉션을 지원합니다. 어떤 언어들의 경우에는 가비지 컬렉션이 필요한 경우들이 있습니다. 대표적으로 함수형 프로그래밍 언어들이 있습니다.

## 장점
대표적인 장점은 프로그래머가 메모리 관리를 직접하지 않는다는 것에 있습니다. 공간 효율성에 대한 고려는 여전히 필요하지만, 더블 프리나 메모리 누수와 같은 문제를 막는다는 장점이 있습니다.

## 단점
특유의 비결정론적인 특성 때문에 메모리의 해제를 예측할 수 없어 실시간 시스템에서는 사용할 수 없습니다. 그리고 프로그램의 성능에 영향을 미치는 단점이 있습니다.

## 기법
가비지 컬렉션에는 여러 기법이 있는데, 이를 크게 두 가지로 나누면 레퍼런스 카운팅 방식과 트레이싱 방식이 있습니다.

### 레퍼런스 카운팅
레퍼런스 카운팅 가비지 컬렉션은 할당받은 메모리에 대해 참조 횟수를 셉니다. 가비지는 참조 횟수가 0입니다. 객체의 참조 횟수는 객체에 대한 참조가 생성될 때 증가하고 참조가 소멸될 때 감소합니다. 즉, 스택 영역이 정리될 때마다, 참조가 해제되어 카운트가 1씩 감소합니다. 대표적으로 C++의 `shared_ptr`이 있습니다. 단점은 순환 참조가 생기면 스택이 정리되더라도 카운트가 0이 되지 않을 수 있습니다. 이 때에는 메모리 누수가 생깁니다. 그럼에도 불구하고 다른 방법에 비해서 시간과 공간 코스트가 적고, 접근 불가능해지는 동시에 삭제된다는 장점이 있습니다. 그리고 어떤 구현에서는 트레이싱 기법과 결합되어 같이 사용되기도 합니다.

### 트레이싱 기법
메모리 할당을 그래프의 노드로 생각했을 때, 모든 메모리 할당은 하나의 루트 노드와 연결되어 있습니다. 만약 루트 노드로부터 시작해 도달할 수 없는 노드가 존재한다면 그것을 가비지로 간주하고 할당을 해제합니다. 기본적인 아이디어는 위와 같지만 트레이싱 기법은 컴퓨팅 자원을 많이 소모하고 특히 일반적인 구현에서는 프로그램 전체를 멈추고 쓰레기를 수집하기 떄문에 반응성이 좋아야하는 프로그램에겐 치명적일 수 있습니다. 그러나 대부분의 가비지 컬렉터 구현은 트레이싱 기법으로 구현되어 있으며, 간혹 레퍼런스 카운팅 기법과 같이 사용되기도 합니다.

#### Mark and Sweep
Mark&Sweep 기법은 앞에서 말한 일반적인 구현입니다. 메모리 할당에 대한 정보를 가지고 있다가 루트에서 탐색을 진행하여 도달 가능한 노드를 표시합니다. 그리고 표시되지 않은 노드를 전부 청소합니다.

#### Tri-color Marking
이번에는 노드를 백색, 회색, 흑색으로 칠합니다. 흑색은 루트에서 도달가능한 노드이고 흰색 노드를 가리키는 간선이 없습니다. 흰색은 가비지 컬렉션의 대상입니다. 회색은 아직 검사가 되지 않은 노드입니다. 회색 노드를 하나 선택하여 검은색으로 변경하고 이 객체가 가리키는 노드를 모두 회색으로 칠합니다. 회색 노드가 남지 않을 때 까지 반복합니다. 이후 흰색 객체를 전부 삭제합니다. 이 알고리즘은 프로그램의 실행 도중에도 같이 실행할 수 있다는 장점이 있습니다.

#### 세대 단위 가비지 컬렉션
Java, C#과 같은 언어에서는 객체를 할당된 시간에 따라 다른 영역에 나누어 관리합니다. 프로그램에서 새롭게 할당되는 메모리의 대부분은 빠르게 할당 해제된다는 특성을 이용한 것입니다. 이는 일부만을 대상으로 가비지 컬렉션을 수행하기 때문에 속도가 빠르다는 장점이 있습니다.

## 레퍼런스 카운팅 방식의 GC 구현하기
한가지 흥미로운 사실은 GCC와 Clang에서 cleanup atrribute를 이용하면 스택을 정리할 때 특정 함수를 호출할 수 있다는 것입니다. 이를 이용하면 레퍼런스 카운팅 방식의 GC를 구현할 수 있습니다. 구현의 정확성은 차치하고 대략적인 개념 증명을 위한 코드는 아래와 같습니다.

sptr.h
``` c
#include <stdlib.h>

#define smart __attribute__ ((__cleanup__(sptr_clean_up__)))

typedef struct mem_node_t {
    void *ptr;
    size_t count;
    struct mem_node_t *next;
} ref_node;

typedef struct mem_head_t {
    struct mem_node_t *head;
} ref_head;

void *make_ptr(void *);
void *ref(void *);
void sptr_clean_up__(void *);
```

sptr.c
``` c
#include <stdio.h>
#include <memory.h>
#include <stdlib.h>

#include "sptr.h"

static ref_head *ref_list = NULL;
void sptr_init__() {
    ref_list = (ref_head *)calloc(1, sizeof(ref_head));
    if (ref_list == NULL) {
        perror("mem_ref_init failed");
        abort();
        return;
    }
}

void *make_ptr(void *ptr) {
    if (ptr == NULL) {
        return NULL;
    }

    if (ref_list == NULL) {
        sptr_init__();
    }

    ref_node *node = (ref_node *)calloc(1, sizeof(ref_node));
    if (node == NULL) {
        perror("make_ptr failed");
        return NULL;
    }

    node->ptr = ptr;
    node->count = 1;
    node->next = ref_list->head;
    ref_list->head = node;

    return ptr;
}

void *ref(void *ptr) {
    for (ref_node *cursor = ref_list->head; cursor != NULL; cursor = cursor->next) {
        if (cursor->ptr == ptr) {
            cursor->count += 1;
            return ptr;
        }
    }

    return NULL;
}

void sptr_clean_up__(void *ptr) {
    void **p = (void **)ptr;
    ref_node *prev = NULL;

    if (ptr == NULL) {
        return;
    }

    if (ref_list == NULL) {
        return;
    }

    for (ref_node *cursor = ref_list->head; cursor != NULL; cursor = cursor->next) {
        if (cursor->ptr != *p) {
            prev = cursor;
            continue;
        }

        cursor->count -= 1;
        if (cursor->count != 0) {
            break;
        }
        
        if (prev != NULL) {
            prev->next = cursor->next;
        }

        if (ref_list->head == cursor) {
            ref_list->head = cursor->next;
        }

        free(cursor->ptr);
        free(cursor);
        break;
    }

    if (ref_list->head == NULL) {
        free(ref_list);
    }
}
```

## Keyword
Cheney's Algorithm

Strong/Weak Reference

Generational GC

Incremental GC

Stop-the-world

Concurrent GC

Compile-Time GC

Real-Time GC

Mark-Sweep-Compact
