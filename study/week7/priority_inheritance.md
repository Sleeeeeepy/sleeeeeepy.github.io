# 우선 순위 상속
## 개요
우선 순위 스케줄링을 수행할 때 주의할 점이 있습니다. 동기화를 수행할 때 우선 순위가 높은 프로세스 $H$가 우선 순위가 낮은 프로세스 $L$를 대기하고 있는 상황에서 $H$과 $L$의 사이의 우선 순위를 가진 프로세스 $M_1$, $M_2$, $M_3, ...$가 계속해서 들어오면, 우선 순위가 높은 프로세스 $H$는 작업을 수행하지 못하는 문제가 있습니다.

이를 해결하기 위해서 몇가지 방법을 생각할 수 있습니다.
1. 비선점 스케줄링
2. 우선 순위 상속 적용
3. 주기적으로 우선 순위를 최대로 높임
4. 허용된 소비 시간을 기준으로 우선 순위를 결정

1번 방법은 문제를 해결한다기보다, 문제 상황을 제거하는 방법이기 때문에 고려하지 않겠습니다. 따라서 일반적으로는 2번, 3번, 4번 중 하나를 선택하게 됩니다. 3번과 4번의 경우, 실시간 시스템에서 우선 순위가 높은 일을 빠르게 처리해야 하기 때문에 해당 방법들을 곧장 적용하기는 어렵습니다.

가장 빠르고 직관적인 해결 방법은 우선 순위 상속입니다. 우선 순위 상속이란 상기 예제에서 프로세스 $H$의 우선 순위를 lock을 가진 프로세스 $L$에게 넘기는 것입니다. 그리고 임계 영역에서 빠져나오면 $L$은 원래의 우선 순위로 복귀합니다.

### 상황 1: 하나의 프로세스가 여러 lock을 홀딩
이 경우는 자신과 프로세스가 홀딩하고 있는 lock을 기다리고 있는 다른 프로세스 중에서 가장 큰 값을 선택하면 됩니다.

### 상황 2: 재귀적 lock
이 경우는 재귀적으로 lock이 걸려있는 상황입니다. 프로세스 $A$가 $l_1$을 가지고 있고, 프로세스 $B$가 $l_2$를 가지고 $l_1$을 대기하고 있는 상황에서 프로세스 $C$가 $l_2$를 대기하고 있는 경우입니다.

이 경우에는 재귀적으로 타고 내려가면서 현재 재귀에서 가장 큰 우선 순위를 전파합니다.

상황 1과 상황 2를 동시에 고려하면 우선 순위 상속을 구현할 수 있습니다.

#### lock_acquire
``` c
void
lock_acquire (struct lock *lock) {
	enum intr_level old_level;
	struct thread *curr;
	struct thread *holder;
	int max_priority;
	
	ASSERT (lock != NULL);
	ASSERT (!intr_context ());
	ASSERT (!lock_held_by_current_thread (lock));
	
	old_level = intr_disable ();
	curr = thread_current ();
	holder = lock->holder;
	max_priority = PRI_MIN;
	
	if (holder) {
		curr->lock = lock;
		list_insert_ordered (&holder->donations, &curr->delem, 
						cmp_thrd_donation_priorities, NULL);
		max_priority = curr->priority;

		struct thread *cursor = holder;
		while (cursor->lock != NULL) {
			if (max_priority < cursor->priority) {
				max_priority = cursor->priority;
			}

			cursor->priority = max_priority;
			cursor = cursor->lock->holder;
		}

		if (cursor != NULL) {
			cursor->priority = max_priority;
		}

		holder->priority = max_priority;
	}

	sema_down (&lock->semaphore);
	intr_set_level (old_level);
	curr->lock = NULL;
	lock->holder = thread_current ();
}

```

#### lock_release
``` c
void
lock_release (struct lock *lock) {
	struct thread *holder;
	int max_priority;

	ASSERT (lock != NULL);
	ASSERT (lock_held_by_current_thread (lock));

	holder = lock->holder;
	holder->priority = holder->prev_priority;
	for (struct list_elem *e = list_begin (&holder->donations);
		e != list_end (&holder->donations);) {

		struct thread* t = list_entry (e, struct thread, delem);
		if (t->lock == lock) {
			e = list_remove (e);
			continue;
		}

		e = list_next (e);
	}

	if (!list_empty (&holder->donations)) {
		list_sort (&holder->donations, cmp_thrd_donation_priorities, NULL);
		max_priority = list_entry (list_begin (&holder->donations), 
					struct thread, delem)->priority;
		if (holder->priority < max_priority) {
			holder->priority = max_priority;
		}
	}

	lock->holder = NULL;
	sema_up (&lock->semaphore);
}
```