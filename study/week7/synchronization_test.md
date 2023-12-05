# 동기화 테스트
## 필요한 부분에서만 동기화를 수행하라
### Don't
``` cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex lock;
int result = 0;

void threadFunc(int numThreads) {
    for (int i = 0; i < 1000000000 / numThreads; i++) {
        lock.lock();
        result++;
        lock.unlock();
    }
}

int main() {
    const int numThreads = 1;
    std::thread threads[numThreads];
    auto startsAt = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < numThreads; i++) {
        threads[i] = std::thread(threadFunc, numThreads);
    }

    for (int i = 0; i < numThreads; i++) {
        threads[i].join();
    }
    auto endsAt = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(endsAt - startsAt);
    std::cout << "Time: " << duration.count() << " ms" << std::endl;
    return 0;
}
```
|numThreads|Time|
|----------|----|
1 | 7110 ms |
2 | 12337 ms |
4 | 15792 ms |
8 | 21354 ms |
16 | 26454 ms |
32 | 26302 ms |
NO LOCK | 908 ms |

동기화 객체에 접근하는 것은 일반적인 접근에 비해 압도적으로 느립니다. 심지어 싱글 스레드인 상황에서 `lock`을 씌우는 것만으로도 속도가 느려집니다. 따라서 동기화는 반드시 필요한 부분에서 수행 되어야 합니다. result에 매 번 접근하는 것 보다, 전부 다 더하고 난 후 한 번만 접근하는 것이 유리합니다.

### Do
``` cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex lock;
int result = 0;

void threadFunc(int numThreads) {
    int temp = 0;
    for (int i = 0; i < 1000000000 / numThreads; i++) {
        temp++;
    }

    lock.lock();
    result += temp;
    lock.unlock();
}

int main() {
    const int numThreads = 1;
    std::thread threads[numThreads];
    auto startsAt = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < numThreads; i++) {
        threads[i] = std::thread(threadFunc, numThreads);
    }

    for (int i = 0; i < numThreads; i++) {
        threads[i].join();
    }
    auto endsAt = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(endsAt - startsAt);
    std::cout << "Time: " << duration.count() << " ms" << std::endl;
    return 0;
}
```
|numThreads|Time|
|----------|----|
1 | 918 ms |
2 | 492 ms |
4 | 269 ms |
8 | 231 ms |
16 | 222 ms |
32 | 218 ms |
NO LOCK | 895 ms |

## OoOE를 주의하라
동기화를 위한 피터슨 알고리즘은 알고리즘의 정확성이 증명되어있지만, 실제로는 제대로 작동하지 않습니다. 현대 CPU는 파이프라인을 최대한 꽉 채우기 위해서 명령어들의 순서를 임의로 바꾸어 실행합니다. 이는 인스트럭션 수준 병렬성을 효과적으로 달성하지만, 멀티스레드 환경에서는 문제를 발생시키기 아주 좋습니다.

### Don't
``` cpp
#include <iostream>
#include <thread>

volatile bool flag[2] = {false, false};
volatile int turn = 0;
int result = 0;

void lock(int id) {
    int other = 1 - id;
    flag[id] = true;
    turn = other;

    while (flag[other] && turn == other) {}
}

void unlock(int id) {
    flag[id] = false;
}

void process(int id) {
    for (int i = 0; i < 100000000; i++) {
        lock(id);
        result++;
        unlock(id);
    }
}

int main() {
    std::thread t0(process, 0);
    std::thread t1(process, 1);

    t0.join();
    t1.join();

    std::cout << "Result: " << result << std::endl;
    return 0;
}
```
결과는 199999962가 나왔습니다. 이는 이 알고리즘이 정확하게 작동하지 않는다는 증거입니다.

### Do
``` cpp
#include <iostream>
#include <thread>

volatile bool flag[2] = {false, false};
volatile int turn = 0;
int result = 0;

void lock(int id) {
    int other = 1 - id;
    flag[id] = true;
    turn = other;

    std::atomic_thread_fence(std::memory_order_seq_cst);
    while (flag[other] && turn == other) {}
}
```
메모리 명령의 순차적 일관성을 `std::atomic_thread_fence(std::memory_order_seq_cst)`를 이용하여 보장한다면 피터슨 알고리즘은 정확하게 작동합니다.

## 데이터를 캐시라인 크기에 정렬하라
데이터를 정렬하는데에는 이유가 있습니다. 여기에는 두 가지 문제가 있기 때문입니다. 첫번째는 두 캐시라인에 걸쳐있는 하나의 데이터의 값을 가져올 떄 문제가 발생할 수 있습니다. 두번째는 값이 정확하더라도, 엄청난 성능하락을 유발하는 문제가 발생할 수 있습니다.
### Don't
``` cpp
#include <iostream>
#include <thread>
#include <chrono>

typedef struct dataset {
    int data1 = 0;
    int data2 = 0;
    int data3 = 0;
    int data4 = 0;
} Dataset;

constexpr int N = 100000000;
volatile Dataset dataset;

void func1() {
    for (int i = 0; i < N; i++) {
        dataset.data1 = dataset.data1 + 1;
    }
}

void func2() {
    for (int i = 0; i < N; i++) {
        dataset.data2 = dataset.data2 + 1;
    }
}

void func3() {
    for (int i = 0; i < N; i++) {
        dataset.data3 = dataset.data3 + 1;
    }
}

void func4() {
    for (int i = 0; i < N; i++) {
        dataset.data4 = dataset.data4 + 1;
    }
}

int main() {
    std::thread threads[4];

    /* Single thread increment. */

    std::cout << "No Threads" << std::endl;
    auto start = std::chrono::high_resolution_clock::now();
    
    func1();
    func2();
    func3();
    func4();

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << duration.count() << "ms" << std::endl;

    /* Threaded increment */

    std::cout << "4 Threads" << std::endl;
    start = std::chrono::high_resolution_clock::now();
    
    threads[0] = std::thread(func1);
    threads[1] = std::thread(func2);
    threads[2] = std::thread(func3);
    threads[3] = std::thread(func4);

    threads[0].join();
    threads[1].join();
    threads[2].join();
    threads[3].join();

    end = std::chrono::high_resolution_clock::now();
    duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << duration.count() << "ms" << std::endl;
}
```

|numThreads|Time|arch|
|----------|----|----|
1 | 271 ms | x86_64
4 | 4840 ms | x86_64
1 | 779 ms | arm64
4 | 225 ms | arm64

멀티 코어 프로세서를 잘 관찰하면 하나의 코어가 하나의 L1 캐시(이하 로컬 캐시)를 가지고 있음을 알 수 있습니다. 만약 두 로컬 캐시가 메인 메모리의 동일한 부분 $P$를 가지고 있다고 합시다. 이에 대한 $n$번 코어의 로컬 캐시의 부분은 $P_n$이라고 합시다. 어떤 코어가 프로그램을 실행하면서 이 부분을 읽는다고 하더라도 두 로컬 캐시가 가지고 있는 $P_0$, $P_1$는 서로 동일합니다. 하지만 0번 코어가 이 부분의 값을 수정하면, 이제 $P_0$과 $P_1$은 서로 동일하지 않습니다. 만약 이 상태에서 연산을 수행하면 잘못된 결과를 얻을 것입니다. 한 가지 생각할 수 있는 해결 방법은 어떤 코어가 캐시라인을 수정하면 캐시라인을 무효로 표기하는 것입니다. 그러면 다른 코어가 해당 데이터를 얻기 위해 접근을 하면, 무효로 표기된 것을 확인하고 메인 메모리 접근을 수행할 것입니다. 따라서 두 코어가 메모리 일관성을 위해 지속적으로 번갈아가며 캐시라인을 엑세스합니다.

그리고 캐시라인 경계에 걸친 데이터들은 제대로 갱신되지 않을 가능성이 있습니다. 캐시 라인 $L_0$가 다른 캐시 라인 $L_1$에 걸쳐있다면 갱신이 일어나기전에 데이터를 가져오는 문제가 발생할 수 있습니다. 이러한 이유로 변경가능한 데이터들은 반드시 캐시 라인 크기에 정렬되어야 합니다.

앞선 결과에서 x86_64에서 문제가 발생하는 것이 관찰되지만, 그렇다고 해서 arm 아키텍쳐를 가진 CPU는 문제가 없다는 것은 아닙니다. 프로세서, 운영체제별로 존재할 수 있는 여러 문제가 있습니다.

### Do
``` cpp
constexpr int CACHE_LINE_SIZE = 64;
typedef struct dataset {
    int alignas(CACHE_LINE_SIZE) data1 = 0;
    int alignas(CACHE_LINE_SIZE) data2 = 0;
    int alignas(CACHE_LINE_SIZE) data3 = 0;
    int alignas(CACHE_LINE_SIZE) data4 = 0;
} Dataset;
```

