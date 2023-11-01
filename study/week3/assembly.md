# x86-64 Assembly
x86-64는 긴 역사를 가진 명령어 집합(instruction set)입니다. 따라서 ISA를 갈아엎고 나타난 프로세서들에 비해서 명령어의 종류도 다양하고, 작동을 이해하기 더욱 어렵습니다. 특히 CS:APP에서는 x86-64 어셈블리에 대해서 설명하고 있는데, 이는 학습상의 편의를 위해서 MIPS를 가르치는 많은 다른 책들과 다릅니다. 하지만 가장 쉽게 접할 수 있는 어셈블리가 x86-64 어셈블리인만큼, 배워두면 많은 도움이 될 것입니다.

## 레지스터
1 워드가 32 비트인 MIPS와는 다르게, x86-64에서는 1 워드가 16 비트입니다. 이는 x86-64가 하위 호환성을 유지하면서 16비트, 32비트, 64비트 아키텍쳐로 발전해왔기 때문입니다. 다른 아키텍쳐와 마찬가지로 레지스터에는 프로그램 카운터와 정수 레지스터, 부동소수점 계산을 위한 백터 레지스터가 있습니다. 그리고 조건 코드를 저장하기 위한 상태 레지스터를 가지고 있습니다. x86-64의 특이한 점은 하나의 레지스터를 크기에 따라 4개로 나누어 부릅니다.
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
피연산자에도 형식이 있습니다. 피연산자는 레지스터(`%rax`와 같이 표기함), 숫자(`$14`와 같이 표기함), 메모리(`(%rax)` 혹은, `0x1234`와 같이 표기함)가 올 수 있습니다. 그리고 메모리 주소의 쉬운 계산을 위해서 `(%base, %index, scale)` 처럼 쓸 수 있는데 이는 `%base + %index * scale`을 계산합니다. 그리고 오프셋은 `8(%rax)`처럼 괄호 앞에다 붙입니다.

## 데이터의 이동
x86-64의 특이한 점은 이뿐만이 아닙니다. `load`, `store`, `move` 명령이 `mov` 명령에 통합 되어있습니다. 즉, `mov` 명령어로 메모리에서 레지스터로 값을 읽어오거나, 레지스터에 있는 값을 메모리에 쓰거나, 레지스터에 있는 값을 다른 레지스터로 옮길 수 있습니다. 단, 메모리에 있는 값을 메모리에 쓰는 것은 불가능합니다. 물론 immediate값을 메모리에 쓰거나 레지스터에 쓰는 것은 가능합니다. 아래 어셈블리는 다음과 같은 C 코드와 같습니다다.
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
## 유효 주소 계산
`lea` 명령어는 주소 계산에 사용되며 메모리 주소를 계산해서 레지스터에 로드한다. 실제로 메모리를 쓰거나 읽지 않음에 주의해야한다.
``` 
lea src, dest 
```
`lea`를 사용하면 메모리 주소 계산을 하나의 명령어로 간편하게 할 수 있다. 가령 아래와 같은 두 명령어가 있다고 가정하자.
``` asm
leaq 7(%rdx), %rax
movq 7(%rdx), %rax
```
여기에서 `leaq`를 사용한 결과는 `%rdx`의 값에 `7`을 더한 것이다. 그러나 `mov`는 `%rdx + 7`이 가르키고 있는 주소에 있는 값을 %rax로 가져온다. 이러한 `lea`의 성질 때문에 몇몇 컴파일러는 간단한 곱셈 연산을 `lea`를 이용해 하기도 한다. 다음은 이 명령어를 사용해 계산을 하는 예제이다.
``` asm
scale:
    leaq (%rsi, %rsi, 9), %rbx
    leaq (%rbx, %rdx), %rbx
    leaq (%rbx, %rdi, %rsi), %rbx
    ret
```
``` c
short scale3(short x, short y, short z) {
    short t = 10 * y + z + x * y; 
    return t;
}
```

## 단항 및 이항 연산
연산 부분은 간단합니다. `inc`(increase), `dec`(decrease), `neg`(negative), `not` 명령어는 단항 연산을 수행하며 하나의 피연산자만을 취합니다. 반면 `add`, `sub`, `imul`(integer multiply), `xor`, `or`, `and`는 이항 연산자이며 두번째 피연산자와 첫번째 피연산자를 연산하여 두번째 피연산자에 결과를 넣습니다.
그리고 `sal`(shift arithmetic left), `shl`(shift left), `sar`(shift arithmetic right), `shr`(shift right)는 첫 번째 피연산자가 값, 혹은 `%cl` 레지스터만을 취합니다. 다음은 4만큼 왼쪽으로 시프트하고, n만큼 오른쪽으로 시프트하는 예제입니다.
``` asm
shift_left4_rightn:
    movq %rdi, %rax
    salq $4, %rax
    movl %esi, %ecx
    sarq %cl, %rax
```
``` c
long shift_left4_rightn(long x, long n) {
    x <<= 4;
    x >>= n;
    return x;
}
```

## 조건
조건문을 만들기 위해서는 비교와 PC를 변경하는 점프 명령어가 필요합니다. 당연히 x86_64에도 이런 명령어가 존재합니다.

### 비교, 그리고 PC의 이동
x86_64의 특이한 점은 플래그를 나타내는 레지스터들이 존재한다는 것입니다. 이러한 레지스터는 가장 최근의 연산결과에 따른 플래그들을 의미합니다. `CF`(carry flag), `ZF`(zero flag), `SF`(sign flag), `OF`(overflow flag) 네 개의 플래그 레지스터가 존재합니다. 다만, 이 때 `leaq`는 주소 연산을 위해 사용되므로 이러한 플래그를 변경시키지 않습니다.

비교를 수행하는데 쓰이는 연산자는 `cmp`와 `test`입니다. `cmp`는 두 피 연산자의 차를 계산하여 비교하고 `test`는 and 연산을 통해 두 연산자를 비교합니다. 이후 `je`, `jg`, `jge`, `jl`, `jle`와 같은 명령어를 통해 분기를 수행합니다. 그리고 `set`은 이러한 비교 연산의 결과를 레지스터로 가져오는 명령어입니다.

한편, 몇몇 연산들은 점프 명령 없이 수행할 수 있습니다. 바로 조건부 이동입니다. 조건부 이동을 하기 전, 비교를 수행하고 `cmovge`와 같은 조건부 이동을 수행할 수 있습니다.

### 조건문과 반복문
비교 명령어와 조건 명령어를 잘 사용하면 조건문과 반복문을 만들 수 있습니다.
```
.section .data
msg1:
    .asciz "Number is greater than 10\n"
msg2:
    .asciz "Number is not greater than 10\n"

.section .text
.global _start

_start:
    # 숫자를 비교할 레지스터에 저장
    mov $15, %rax

    # 비교 연산: 만약 rax가 10보다 크면 ZF(Zero Flag)가 설정됩니다.
    cmp $10, %rax

    # 조건부 점프 명령을 사용하여 조건 검사
    jg greater_than_10

    # rax가 10보다 크지 않을 때 실행되는 부분
    mov $4, %eax
    mov $1, %ebx
    lea msg2, %rdx
    mov $30, %rcx
    syscall

    jmp end_if

greater_than_10:
    # rax가 10보다 크거나 같을 때 실행되는 부분
    mov $4, %eax
    mov $1, %ebx
    lea msg1, %rdx
    mov $29, %rcx
    syscall

end_if:
    # 프로그램 종료
    mov $60, %rax
    xor %rdi, %rdi
    syscall

```
위 프로그램은 숫자가 10보다 크면 `greater_than_10`으로 분기하고 아니라면 계속 진행합니다. 참고로, `%rax`의 값이 `4`이므로 `write` 시스템 콜을 호출합니다.

## 프로시저
### 스택
프로그램은 스택 메모리를 가지고 있으며 스택에 값을 넣거나 빼는 식으로 작동합니다. 데이터를 넣는 것은 `push` 명령어를 이용하고 데이터를 빼는 것은 `pop` 명령을 사용합니다. `push` 명령어는 아래 두 명령어와 동작이 같습니다.
``` asm
subq $size, %rsp
movq %rbp, (%rsp)
```
그리고 `pop` 명령어는 아래와 같습니다.
``` asm
movq (%rsp), %rax
addq $size, %rsp
```

### 프로시저 호출
함수라고도 일컫는 프로시저의 호출은 `call` 명령어를 이용합니다. 먼저 프로시저를 호출하기 위해서 모든 인자를 8 바이트로 확장하고 스택에 저장합니다. `call` 명령어를 사용하면 스택에 복귀 주소(해당 `call` 명령어의 다음 주소)를 기록하고 해당 프로시저로 점프합니다. 그리고 보존 레지스터가 스택에 저장됩니다. 이후에는 프로시저가 실행되고, 반환값을 `eax` 레지스터에 넣습니다. 그리고 스택에 저장된 값을 레지스터로 복구하고 스택을 정리합니다. 하지만 이 과정은 함수의 호출 방법에 따라 작동 방식이 조금씩 다릅니다.

#### 함수 호출 규약
함수의 호출 규약은 프로그래밍 언어와 CPU 아키텍처에 따라 달라질 수 있습니다. 그러나 `stdcall`과 `cdecl` 규약은 수많은 언어와 아키텍처에서 폭넓게 사용됩니다. 

##### cdecl
1. 함수를 호출한 곳에서 스택을 정리합니다.
2. 매개변수를 스택에 넣어 함수 호출 시 전달합니다.

##### stdcall
1. 호출된 함수에서 스택을 정리합니다.
2. 매개변수를 스택에 넣어 함수 호출 시 전달합니다.

##### fastcall
1. 호출된 함수에서 스택을 정리합니다.
2. 일부 매개변수는 레지스터에, 다른 부분은 스택에 넣어 전달합니다.

여기까지가 x32의 함수 호출 규약들이였습니다. 하지만 x86_64에서는 기본적으로 `fastcall`만 이용합니다. 대략적으로는 이렇지만, 운영체제, 아키텍쳐, 컴파일러에 따라 달라질 수 있습니다.