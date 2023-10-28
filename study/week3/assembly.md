# x86-64 Assembly
x86-64는 긴 역사를 가진 명령어 집합(instruction set)이다. 따라서 ISA를 갈아엎고 나타난 프로세서들에 비해서 명령어의 종류도 다양하고, 작동을 이해하기 더욱 어렵다.

특히 CS:APP에서는 x86-64 어셈블리에 대해서 설명하고 있는데, 이는 학습상의 편의를 위해서 MIPS를 가르치는 많은 다른 책들과 다르다. 하지만 가장 쉽게 접할 수 있는 어셈블리가 x86-64 어셈블리인만큼, 배워두면 많은 도움이 될 것이다.

## 레지스터
1 워드가 32 비트인 MIPS와는 다르게, x86-64에서는 1 워드가 16 비트이다. 이는 x86-64가 하위 호환성을 유지하면서 16비트, 32비트, 64비트 아키텍쳐로 발전해왔기 때문이다. 

다른 아키텍쳐와 마찬가지로 레지스터에는 프로그램 카운터와 정수 레지스터, 부동소수점 계산을 위한 백터 레지스터가 있다. 그리고 조건 코드를 저장하기 위한 상태 레지스터를 가지고 있다.

x86-64의 특이한 점은 하나의 레지스터를 크기에 따라 4개로 나누어 부른다.
| 64 | 32 | 16 | 8 | 비고 |
|---|---|---|---|---|
| rax | eax | ax | al | return value |
| rbx | ebx | bx | bl | callee saved |
| rcx | ecx | cx | cl | 4th arg |
| rdx | edx | dx | dl | 3rd arg |
| rsi | esi | si | sil | 2nd arg |
| rdi | edi | di | dil | 1st arg |
| rbp | ebp | bp | bpl | callee saved |
| r8 | r8d | r8w | r8b | callee saved |
| r9 | r9d | r9w | r9b | callee saved |
| r10 | r10d | r10w | r10b | callee saved |
| r11 | r11d | r11w | r11b | callee saved |
| r12 | r12d | r12w | r12b | callee saved |
| r13 | r13d | r13w | r13b | callee saved |
| r14 | r14d | r14w | r14b | callee saved |
| r15 | r15d | r15w | r15b | callee saved |

## 피연산자
피연산자에도 형식이 있다. 피연산자는 레지스터(`%rax`와 같이 표기함), 숫자(`$14`와 같이 표기함), 메모리(`(%rax)` 혹은, `0x1234`와 같이 표기함)가 올 수 있다. 그리고 메모리 주소의 쉬운 계산을 위해서 `(%base, %index, scale)` 처럼 쓸 수 있는데 이는 `%base + %index * scale`을 계산한다. 그리고 오프셋은 `8(%rax)`처럼 괄호 앞에다 붙인다.

## 데이터의 이동
x86-64의 특이한 점은 이뿐만이 아니다. `load`, `store`, `move` 명령이 `mov` 명령에 통합 되어있다. 즉, `mov` 명령어로 메모리에서 레지스터로 값을 읽어오거나, 레지스터에 있는 값을 메모리에 쓰거나, 레지스터에 있는 값을 다른 레지스터로 옮길 수 있다. 단, 메모리에 있는 값을 메모리에 쓰는 것은 불가능하다. 물론 immediate값을 메모리에 쓰거나 레지스터에 쓰는 것은 가능하다. 다음 아래 어셈블리는 다음과 같은 C 코드와 같다.
``` asm
decode1:
    movq (%rdi), %r8
    movq (%rsi), %rcx    
    movq (%rdx), %rax
    movq %r8, (%rsi)
    movq %rcx, (%rdx)
    movq %rax, (%rdi)    
```
``` c
void decode(long *xp, long *yp, long *zp) {
    long r8 = *xp;
    long rcx = *yp;
    long rax = *zp;

    *yp = r8;
    *zp = rcx;
    *xp = rax;
}
```
(작성중..)
``` c
short scale3(short x, short y, short z) {
    short t = 9 * y + y + z + x * y; 
    return t;
}
```

``` asm
shift_left4_rightn:
    movq %rdi, %rax
    salq $4, %rax
    movl %esi, %ecx
    sarq %cl, %rax
```