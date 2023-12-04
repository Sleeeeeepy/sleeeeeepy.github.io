# PintOS 문제 해결
## 고려사항
### 어떻게 모든 스레드를 관리해야 하는가
MLFQS를 구현하기 위해서는 모든 스레드에 대해서 우선 순위를 재계산해야합니다. 하지만 스레드는 `ready_list`, `waiters`, `block_list`등에 흩어져 있습니다. 문제는 컨디션이나 세마포어에 대기하고 있는 스레드들인 `waiters`에 대한 정보는 각 컨텍스트만이 알고 있다는 사실입니다. 이를 해결하기 위해서 아래 방법들 중 하나를 선택할 수 있습니다.

1. 모든 세마포어의 리스트를 만듭니다.
2. 모든 스레드의 리스트를 만듭니다.
3. 세마포어에서 대기하고 있는 모든 스레드를 관리하는 새로운 리스트 `thread_list`를 만듭니다.

무엇을 선택하건 이 문제를 해결할 수는 있습니다. 저는 이 중에서 가장 쉬운 방법인 2번을 택했습니다. 스레드가 생성되는 타이밍부터 리스트에 추가되어 관리가 쉬워지기 때문입니다. 그래서 아래처럼 수정했습니다.
``` c
struct thread {
    // ...
    struct list_elem telem;
    // ...
};
```
그리고 스레드를 생성할 때 리스트에 추가하기 위해서 아래와 같이 했습니다.
``` c
tid_t
thread_create (const char *name, int priority,
		thread_func *function, void *aux) {
    // ...
    list_push_back (&thread_list, &t->telem);
}
```
그리고 스레드가 소멸할 때 리스트에서도 지워야합니다. 그렇지 않으면 올바르지 않은 주소에 엑세스를 시도하며 커널 패닉을 유발합니다.
```c
static void
do_schedule(int status) {
    // ...
	while (!list_empty (&destruction_req)) {
		struct thread *victim =
			list_entry (list_pop_front (&destruction_req), struct thread, elem);
		list_remove (&victim->telem);
		palloc_free_page(victim);
	}
    // ...
}
```
문제는 지금부터 시작됩니다. `mlfqs-load-1` 테스트에서 무한 루프가 발생합니다. 그리고 `load_avg`가 제대로 오르지 않는 현상이 발생합니다. 몇 시간이 지난 후 메인 스레드가 `thread_list`에 없다는 사실을 발견했습니다. 메인 스레드는 `thread_create`를 호출하지 않고 `thread_init`만을 호출합니다. `thread_create` 대신 `thread_init`에서 리스트에 원소를 추가하도록 수정하였습니다.