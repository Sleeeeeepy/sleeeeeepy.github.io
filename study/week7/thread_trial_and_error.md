# PintOS 문제 해결

## 1. List Remove
리스트에서 원소를 제거하는 `list_remove`는 아래와 같이 구현되었습니다.
``` c
struct list_elem *
list_remove (struct list_elem *elem) {
	ASSERT (is_interior (elem));
	elem->prev->next = elem->next;
	elem->next->prev = elem->prev;
	return elem->next;
}
```
`elem`를 삭제하고 `elem->next`를 반환함에 주의해야합니다. 따라서 아래와 같이 리스트에서 원소를 삭제하는 전통적인 루프는 작동하지 않습니다.

``` c
struct list_elem *e = list_begin (&list);
for (; e != list_end (&list); e = list_next (e)) {
	e = list_remove (e);
}
```
따라서 아래와 같이 해야합니다.
``` c
struct list_elem *e = list_begin (&list);
for (; e != list_end (&list);) {
	if (predicate) {
		e = list_remove (e);
		continue;
	}
	e = list_next (e);
}
```

## 2. Interrupt
언제 인터럽트를 걸어야하는 지 판단하는데 어려움을 겪었습니다. 이를 해결하기 위해서 문서를 조금 참조했습니다.

### 인터럽트의 종류
인터럽트는 내부 인터럽트와 외부 인터럽트로 나뉩니다. 내부 인터럽트는 CPU 인터럽트에 의해서 발생하는 인터럽트이며 동기식입니다. 예를 들어 시스템 콜, 페이지 오류, 0으로 나누는 행위들은 내부 인터럽트를 발생시킵니다. `intr_disable()`을 사용해도 내부 인터럽트는 꺼지지 않습니다. 외부 인터럽트는 CPU 외부에서 발생하는 인터럽트입니다. 타이머 인터럽트, 키보드 입력, 디스크 읽기 등 하드웨어 작업에서 발생합니다. 외부 인터럽트는 비동기식이고, 인터럽트를 잠시 끌 수 있습니다.

### 인터럽트 끄기
현재 실행 흐름을 멈추고 인터럽트 핸들러에서 인터럽트를 처리한 뒤 다른 스레드로 문맥 전환이 일어나는 경우 발생합니다. 이 때 두 쓰레드가 같은 공유 자원에 접근할 경우 문제가 생길 수 있습니다. 이러한 경우는 안전합니다.

``` c
void safe () {
	struct thread *curr = thread_current ();

	// Working with curr.
	// It is okay to call current_thread () multiple times within this context.
}
```

일반적으로 이러한 컨텍스트는 보존되기 때문에 괜찮습니다. 하지만 전역변수나 다른 쓰레드의 정보를 수정하는 경우에는 인터럽트를 꺼야합니다. 예를 들어 `ready_list`에 원소를 집어넣거나 빼는 경우는 안전하지 않을 가능성이 있습니다. 기존 코드를 분석에서 `ready_list`를 수정하는 모든 작업들에서 인터럽트를 끄고 있음을 관찰할 수 있습니다. 이를 작성일(2023.11.29)에 발견하였고, 전역 변수에 접근하는 모든 코드에 대해서 인터럽트를 끌 예정입니다.