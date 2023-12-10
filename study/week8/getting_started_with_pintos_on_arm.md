# aarch64 docker에서 PintOS 시작하기
이 글을 PintOS 프로젝트가 시작된 지 대략 3주만에 작성하는 이유는 아래와 같습니다.

1. AWS 등의 클라우드 환경에서 원격 디버거가 자주 프리징됩니다.
2. 원격 데스크탑을 사용했을 때 환경이 달라 잦은 작업 전환이 힘듭니다.
3. x86_64 아키텍처의 docker(Ubuntu)에서 Sonoma 업데이트 이후에 qemu 실행이 불가능합니다.
4. 가상 머신을 이용하는 경우 알 수 없는 이유로 가상 머신이 갑자기 꺼집니다.

특히 3번의 경우 qemu 실행 시 `rosetta error: Unimplemented syscall number 282` 오류가 발생하는 문제가 있었습니다. x86_64 애뮬레이트를 할 때 docker는 macOS에서 Rosetta 2 위에서 작동하는데, 이와 관련된 문제이므로 aarch64 아키텍쳐로 바꾸어주면 문제가 없을 것입니다. 그리고 컴파일 시 x86_64로 크로스 컴파일하고, qemu를 이용하면 됩니다.

## 크로스 컴파일 환경 설정
1. `gcc-multilib-arm-linux-gnueabi` 혹은 `gcc-multilib-arm-linux-gnueabihf`를 설치합니다.
2. `gcc-x86-64-linux-gnu`를 설치합니다.
3. `gdb-multiarch`를 설치합니다.
4. `/usr/bin/x86_64-linux-gnu-`로 시작하는 모든 실행파일에 대해 링크를 생성합니다.
5. `CC`, `OBJDUMP`, `OBJCOPY`, `LD` 등 환경 변수를 설정합니다.

이 과정을 거치면 크로스 컴파일이 가능합니다. PintOS의 Makefile을 수정하지 않기 위해서는 위 과정을 거쳐야합니다. 다만, 4번과 5번 과정을 거치면서 다른 아키텍처를 대상으로하는 gcc를 설치하는 경우 문제가 발생할 수 있습니다. 이 이미지를 통해 만드는 컨테이너에서 aarch64를 대상으로 컴파일 해야하는 경우가 없도록 해야합니다. 

## 요약
다음은 상기 과정이 담긴 `Dockerfile`입니다.

``` dockerfile
# Dockerfile for setting up the development environment on aarch64
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y python3 \
    make \
    git \
    gdb \
    gdb-multiarch \
    gcc-multilib-arm-linux-gnueabi \
    gcc-x86-64-linux-gnu \
    g++-x86-64-linux-gnu \
    qemu;

RUN for file in /usr/bin/x86_64-linux-gnu-*; do \
        link_name=$(basename $file | sed 's|x86_64-linux-gnu-||'); \
        ln -s $file /usr/bin/$link_name; \
    done

ENV CROSS_COMPILE x86_64-linux-gnu-
ENV CC ${CROSS_COMPILE}gcc
ENV CXX ${CROSS_COMPILE}g++
ENV LD ${CROSS_COMPILE}ld
ENV AR ${CROSS_COMPILE}ar
ENV AS ${CROSS_COMPILE}as
ENV RANLIB ${CROSS_COMPILE}ranlib
ENV NM ${CROSS_COMPILE}nm
ENV OBJCOPY ${CROSS_COMPILE}objcopy
ENV OBJDUMP ${CROSS_COMPILE}objdump
ENV STRIP ${CROSS_COMPILE}strip
```

상기 `Dockerfile`을 저장한 뒤 다음 지침에 따르세요.
1. `docker build -t <image_name> .`
2. `docker run -it --name <container_name> <image_name>`

## 기타
디버깅은 `gdb`가 아니라 `gdb-multiarch`를 이용합니다.