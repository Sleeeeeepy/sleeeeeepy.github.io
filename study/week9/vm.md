# 가상 메모리
## 현재 상황 톺아보기
### 작업 전환
x86_64는 물리 메모리 주소에 있는 데이터에 직접 접근할 수 있는 방법을 제공하지 않습니다. PintOS는 커널 가상 메모리와 물리 메모리를 그대로 매핑합니다. 그리고 유저 가상 메모리와 물리 메모리는 실제로 1:1로 매핑되지 않습니다. 프로세스 A와 프로세스 B가 있다고 가정합시다. 두 프로세스는 같은 가상 주소를 요청할 수 있습니다. 그러나 실제로는 다른 물리 메모리에 접근합니다. 우리는 매 컨텍스트 스위칭마다 가상 주소 테이블을 업데이트하기 위해 `process_activate`를 호출합니다.
``` c
void
process_activate (struct thread *next) {
    /* Activate thread's page tables. */
    pml4_activate (next->pml4);

    /* Set thread's kernel stack for use in
    processing interrupts. */
    tss_update (next);
}

/* Loads page directory PD into the CPU's page directory base
 * register. */
void
pml4_activate (uint64_t *pml4) {
	lcr3 (vtop (pml4 ? pml4 : base_pml4));
}

__attribute__((always_inline))
static __inline void lcr3(uint64_t val) {
	__asm __volatile("movq %0, %%cr3" : : "r" (val));
}
```
#### Control Register
1. CR0: 보호 모드, FPU 관련 제어, 작업 전환, 읽기 전용 페이지를 위한 쓰기 보호, 캐시 정책, **페이징** 제어
2. CR1: 사용하지 않음
3. CR2: Page Fault Linear Address. 에러 발생 시 주소 저장
4. CR3: 가상 주소 지정이 활성화된 경우, 가상 주소를 물리 주소로 변환하는 데 필요한 페이지 테이블 정보를 저장
5. CR4: 보호 모드 관련 제어
6. CR5: 사용하지 않음
7. CR6: 사용하지 않음
8. CR7: 사용하지 않음

즉, `process_activate` 함수는 페이지 테이블을 설정하는 역할을 수행합니다. 이를 통해 우리는 같은 가상 주소에 여러 물리 주소를 매핑할 수 있습니다.

### 메모리 레이아웃
PintOS는 256 TiB의 주소를 매핑(48 bits) 매핑할 수 있으며 이 중 각각의 유저 프로세스에게 할당된 영역은 0x400000~0x47480000입니다. 이 주소는 컴파일 시간에 결정됩니다.

#### 컴파일 타임 바인딩
몇몇 코드들은 사전에 메모리 위치가 이미 알려져 있다면 절대 주소를 쓰는 코드(absolute code)를 생성할 수 있습니다. 만약 이러한 주소가 변경된다면 반드시 다시 컴파일해야 합니다. 이러한 주소를 보려면 다음 명령어를 이용합니다.

``` bash
readelf -l filename
```

#### 로드 타임 바인딩
어떤 주소들은 로드 타임에 바인딩 되기도 합니다. 이 주소들은 오프셋을 가지고 있으며 베이스 주소는 로드 타임에 결정됩니다. 주소를 결정하는 것을 재배치라고 하며 이러한 주소들은 재배치 가능한 주소(relocatable address)라고 합니다. 이러한 주소를 보려면 다음 명령어를 이용합니다.

``` bash
readelf -r filename
```
하지만 PintOS 프로젝트에서 컴파일한 모든 유저 프로그램은 재배치 가능한 주소를 사용하지 않습니다. 이는 PintOS가 로드 타임 바인딩을 지원하지 않기 때문에 그렇습니다.

#### 런 타임 바인딩
어떤 것들은 로드 이후에도 바인딩되지 않은 경우가 있습니다. 메모리를 적재할 때 한 번에 주소 변환을 하지 않고 프로세스의 명령어가 실행되고자 할 때 주소 바인딩이 이루어집니다. 이때 주소 변환은 하드웨어가 지원합니다. 그리고 공유 라이브러리를 생성해서 여러 프로세스가 하나의 라이브러리를 공유하여 메모리를 절약하기도 합니다. 우리는 이러한 라이브러리를 동적 연결 라이브러리라고 부릅니다. 
현재 프로그램의 공유 라이브러리 의존성은 아래와 같은 방법으로 확인할 수 있습니다.
``` bash
objdump -p filename

혹은

readelf -d filename

혹은

ldd filename
```
마찬가지로 PintOS 프로젝트에서는 이러한 타입의 바인딩을 지원하지 않습니다.

### 스택 사이즈
현재 스택의 사이즈는 하나의 페이지로 제한되어 있습니다. 총 4096 바이트의 스택 사이즈를 가질 수 있고 그 이상은 가질 수 없습니다. 스택 사이즈를 늘리기 위해서 새로운 페이지를 할당해야 합니다. 수정해야 할 부분은 setup_stack ()입니다.

### 페이지 즉시 할당
페이지를 필요할 때 할당하지 않고 즉시 할당합니다. 이를 실제로 사용할 때 할당하도록 변경해야 합니다. 수정해야 할 부분은 load_segment ()입니다.

### Supplemental Page Table
각각의 페이지에 대해서 추가적인 정보를 가지고 있는 페이지입니다. 페이지 폴트가 발생하였을 때 폴트가 발생한 가상 페이지에 어떤 데이터가 있어야 하는지를 찾습니다. 그리고 프로세스가 종료될 때 어떤 리소스가 해제되어야 하는지를 결정합니다. 페이지에 대한 추가적인 정보를 담고 있는 supplemental page table을 찾기 위해 몇 가지 선택지가 있습니다.

1. 배열
2. 리스트
3. 해시 테이블

주로 해시 테이블을 이용한 구현이 시간과 메모리 효율성 두 측면에서 뛰어난 성능을 발휘합니다. 이 자료 구조 중 하나를 이용하여 다음과 같은 알고리즘을 따를 것 입니다.

1. 페이지 폴트가 발생합니다.
    - 주소의 유효성을 확인합니다.
2. 새로운 프레임을 할당합니다.
    - 만약 파일 맵드 페이지라면 파일을 읽어옵니다.
    - 만약 익명 페이지라면 스왑 디스크에서 읽어옵니다.
    - 초기화 되지 않은 페이지라면 초기화를 수행합니다.
    - 만약 찾을 수 없다면 에러가 발생하고 프로세스가 종료됩니다. 커널 단에서 발생하면 커널 패닉이 발생합니다.
3. 업데이트 되지 않은 정보가 있다면 업데이트를 수행합니다. (2-3)
4. 페이지 테이블에 페이지를 install 합니다.

### 페이징
현재 x86_64 프로세서에서는 **4단계 페이징**을 사용합니다. 하나의 단계를 더 추가해서 128 PiB까지 지원하는 *5단계 페이징*도 있지만, 비교적 최근 지원되기 시작했습니다. PintOS에서는 마찬가지로 4단계 페이징을 이용합니다. 페이지 테이블은 가상 주소와 실제 주소를 매핑하는 곳으로, 각 매핑을 페이지 테이블 엔트리(page table entry)라고 부릅니다. 페이지 테이블을 관리하는 코드는 `mmu.c`에 존재합니다. 여기에서 메모리를 5개의 조각으로 나눕니다. 메모리를 5개의 조각으로 나누는 이유는, 단순히 페이지 크기(4096 byte or 4 MiB)로 나누면 메모리 매핑을 위해서 엄청난 양의 메모리($2^{64-12}$ byte or $2^{64-22}$ byte)가 필요하기 때문입니다.

#### 4단계 페이징의 경우
```
63~48: sign ext
47~39: pml4 offset
38~30: pdp offset
29~21: pgdir offset
20~12: pte offset
11~00: frame offset
```

#### Appendix: 5단계 페이징의 경우
```
63~58: sign ext
57~48: pgd offset
47~39: p4d offset
38~30: pud offset
29~21: pmd offset
20~12: pte offset
11~00: frame offset
```

#### 페이지 삽입
``` c
bool
pml4_set_page (uint64_t *pml4, void *upage, void *kpage, bool rw) {
	ASSERT (pg_ofs (upage) == 0);
	ASSERT (pg_ofs (kpage) == 0);
	ASSERT (is_user_vaddr (upage));
	ASSERT (pml4 != base_pml4);

	uint64_t *pte = pml4e_walk (pml4, (uint64_t) upage, 1);

	if (pte)
		*pte = vtop (kpage) | PTE_P | (rw ? PTE_W : 0) | PTE_U;
	return pte != NULL;
}
```

#### 페이지 검색
``` c
void *
pml4_get_page (uint64_t *pml4, const void *uaddr) {
	ASSERT (is_user_vaddr (uaddr));

	uint64_t *pte = pml4e_walk (pml4, (uint64_t) uaddr, 0);

	if (pte && (*pte & PTE_P))
		return ptov (PTE_ADDR (*pte)) + pg_ofs (uaddr);
	return NULL;
}
```

#### 페이지 삭제
``` c
void
pml4_clear_page (uint64_t *pml4, void *upage) {
	uint64_t *pte;
	ASSERT (pg_ofs (upage) == 0);
	ASSERT (is_user_vaddr (upage));

	pte = pml4e_walk (pml4, (uint64_t) upage, false);

	if (pte != NULL && (*pte & PTE_P) != 0) {
		*pte &= ~PTE_P;
		if (rcr3 () == vtop (pml4))
			invlpg ((uint64_t) upage);
	}
}
```

#### 페이지 테이블 파괴
``` c
void
pml4_destroy (uint64_t *pml4) {
	if (pml4 == NULL)
		return;
	ASSERT (pml4 != base_pml4);

	/* if PML4 (vaddr) >= 1, it's kernel space by define. */
	uint64_t *pdpe = ptov ((uint64_t *) pml4[0]);
	if (((uint64_t) pdpe) & PTE_P)
		pdpe_destroy ((void *) PTE_ADDR (pdpe));
	palloc_free_page ((void *) pml4);
}
```

#### 페이지 테이블 탐색 과정
페이지 테이블을 탐색할 때 pml4, pdpe, pgidr, pte를 차례로 탐색합니다. 이는 각각 `pml4e_walk`, `pdpe_walk`, `pgdir_walk` 함수가 구현하고 있습니다. 각각의 함수를 차례대로 호출하면서 탐색을 수행합니다. 만약 페이지 테이블에 해당 유저 테이블이 없다면 `NULL`을 반환합니다.