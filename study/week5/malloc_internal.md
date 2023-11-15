# Malloc Internal
명시적 할당기를 구현하다가 86점에서 더 이상 점수가 오르지 않아서 `libc`의 `realloc` 구현을 살펴보기로 했습니다.

## Re-allocation Implementation of libc
대략적으로 아래와 같은 구현을 따른다. 이 때, libc의 구현에서는 아레나와 청크 단위로 할당을 수행합니다.

### _int_realloc
mmap으로 할당된 블록의 경우
1. mremap과 munmap을 이용하여 OS로부터 할당받은 메모리 사이즈를 조정합니다. 이후 아래 작업이 실행됩니다.

확장

1. 청크가 mmap 시스템 콜을 이용하여 할당된 경우, 에러를 발생시킵니다 이 함수가 실행되기 전에 이 케이스는 이미 앞서 처리되었습니다.
2. 남은 공간이 충분히 크다면 할당된 청크과 가용 청크으로 나눕니다.
3. 확장을 시도합니다. 다음 청크로 확장을 시도합니다.
4. 확장이 어렵다면, 다시 할당받고 값을 복사한 후 `free`함수를 호출하여 기존 공간을 반환합니다.

축소

1. 만약, 확장 작업 이후 청크가 축소가 가능하다면 축소를 수행합니다.
    - 이전 청크가 축소가 가능하다면 마찬가지로 축소를 수행한다.
2. 축소하기 충분한 사이즈인 경우, 청크를 가용 청크와 할당된 청크로 나눈다.

간단한 알고리즘이지만 굉장히 빠르게 작동하며, 효율적입니다. 이와 유사한 알고리즘은 이미 구현했지만, 점수가 추가로 오르지는 않았습니다.

## Internal Implementation of malloc

1. 만약 알맞는 청크가 있다면 바로 반환합니다. 이 때에는 알맞다라는 말은 정확히 크기가 일치한다는 의미입니다.
2. 충분히 큰 크기의 요청이 온다면 `mmap` 시스템 콜을 이용하여 OS에게 메모리를 요청합니다. 이 때 '충분히 큰 크기'는 동적으로 결정됩니다.
3. 여기서에는 분리 가용 리스트를 사용하는데 이를 `bin`이라고 일컫습니다. 만약 `fastbin`에 청크가 있으면 사용합니다.
4. 만약 `smallbin`에 크기가 맞는 청크가 있으면 사용합니다.
5. 큰 요청이 온다면 `fastbin`에 있는 모든 것들을 `unsorted bin`으로 옮기고 합칩니다.
6. `unsorted bin`에서 청크를 가져와서 `large/small bin`으로 이동하면서 합칩니다. 적절한 크기의 청크가 있으면 이를 사용합니다.
7. 큰 요청이 온다면 `large bin`을 검색하여 큰 청크를 찾습니다.
8. 아직 `fastbin`에 청크가 남아있으면 이를 합칩니다.
9. top 청크의 일부를 분리하고 필요한 경우 확장합니다.

libc의 malloc 구현도 분리 가용 리스트를 사용합니다. 대신 수많은 최적화를 통해서 공간 활용도와 할당 시간을 크게 줄였습니다.

## other ideas
레드 블랙 트리, 혹은 이진 탐색 트리를 이용하여 알맞은 크기의 블록을 찾는 아이디어가 있습니다. 공간 탐색에 $O(\log n)$ 시간이 걸려서, 처리량이 뛰어납니다. 그리고, best-fit에 가깝게 구현되어 공간 활용도가 높습니다.

특히 7번, 8번 트레이스가 단순한 분리 가용 리스트에게 상당히 치명적으로 작동하는데, 이를 해결합니다.

이 코드를 제외하고는 거짓말처럼 좋은 점수를 받은 모든 코드가 비슷합니다. 그래서 그 코드가 원래 어떤 사람의 코드인 지 모르지만, 아무튼,  재할당을 받을 때 `extend_heap`을 호출하여 청크 사이즈보다 클 수 있는 큰 블록을 새로 할당받습니다. 이후 이를 사용하고, `reallocated block`이라고 기록합니다. 이 블록은 `coalesce`가 발생하지 않습니다. 만약 `realloc`이 한 번만 호출된다면 많은 공간의 메모리가 버려지게 됩니다. 이 블록은 `free`를 호출하하여 지우지 않는 한 절대로 변경되거나, 합쳐지지 않습니다.

아이디어 자체는 재미있지만, 범용 할당기에는 적합하지 않은 것 같습니다. 이 책의 수준에서는 80점 중후반이나 90점 초반이 나오도록 테스트 케이스를 조정한 것 같습니다. 




