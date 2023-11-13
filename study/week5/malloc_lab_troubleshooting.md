# malloc lab 문제 해결

malloc lab 과제 수행 도중 발생했던 문제에 대한 해결 방법을 적어보았습니다. 문제가 발생한 환경은 aarch64(m2) 아키텍쳐의 Ubuntu 입니다.

CMU의 유명한 과제 중 하나인 malloc lab은 x86 Unix/Linux를 기준으로 되어 다른 환경에서는 몇 가지 문제가 있습니다.

## 증상: mdriver를 실행하면 무조건 usage가 나옵니다.
### 원인 및 해결 방법
aarch64 환경의 gcc에서는 `getopt`의 결과값이 -1이면 이를 255로 변환하여 저장합니다. 이를 해결하기 위해서 컴파일 옵션을  수정해야 합니다.
``` bash
gcc -fsigned-char ...
```

### See also
반대로 arm-gcc가 아닌 다른 환경에서 이를 테스트하고 싶다면 아래와 같이 컴파일 옵션을 주면 됩니다.
``` bash
gcc -funsigned-char ...
```

## 증상: unrecognized command-line option ‘-m32’
### 원인 및 해결 방법
x64 환경에서 x86로 컴파일하기 위해서 컴파일러에 -m32 옵션을 주었습니다. 이를 x64 아키텍처가 아닌 버전의 gcc로 실행하면 이러한 에러가 발생할 수 있습니다. -m32 옵션을 제거하세요.

## 증상: explicit 구현 시 segfault가 발생합니다.
### 원인 및 해결 방법
mm.c는 x86를 기준으로 작성되었습니다. 다음 코드를 확인해보세요.
``` c
#define WSIZE     4
#define DSIZE     8
```
x64, 혹은 aarch64에서는 WSIZE가 8이 되어야 합니다. 따라서 아래와 같이 수정하세요
``` c
#define WSIZE     8
#define DSIZE     16
```

### See also
단, 이를 그대로 원격 저장소에 올리면 다른 팀원이 곤란을 겪을 수 있습니다. 이를 일반화하기 위해 다음 코드를 사용해보세요.
``` c
#define WSIZE sizeof(void*)
#define DSIZE (2*WSIZE)
```

## 증상: 디버깅 시 브레이크 포인트가 작동하지 않습니다.
### 원인 및 해결 방법
디버깅 정보를 생성해야 합니다. 컴파일 시 `-g` 옵션을 주세요.
``` bash
gcc -g ...
```

## 증상: 디버깅 시 optimized out이 뜨며 변수의 값을 관찰할 수 없습니다.
### 원인 및 해결 방법
말 그대로 컴파일러 최적화에 의한 문제입니다. 컴파일 시 최적화를 수행하지 않아야 합니다.
``` bash
gcc -O0 ...
```