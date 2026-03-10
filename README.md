# xv6 분석 Report

## 목차

1. [System Call이란?](#1-system-call이란)
2. [xv6에 System Call 추가하기 (getppid)](#2-xv6에-system-call-추가하기-getppid)
3. [Scheduler 구현 및 분석](#3-scheduler-구현-및-분석)

---

# 1. System Call이란?

> 상세 내용: [`1. what is System Call?.md`](./1.%20what%20is%20System%20Call%3F.md)

System Call은 Application이 Kernel을 **제어**할 수 있는 인터페이스이다.

- User Process는 하드웨어를 직접 제어할 수 없기 때문에 Kernel에게 System Call을 통해 H/W를 제어한다.
- System Call을 호출하면 커널 모드로 전환되어, 커널에서만 접근 가능한 데이터를 처리한 후 결과값을 반환한다.

> **System Call의 흐름**: 사용자 프로세스 → 시스템 콜 → 커널 처리 → 결과 반환

---

# 2. xv6에 System Call 추가하기 (getppid)

> 상세 내용: [`2. add system calls in xv6 보고서.md`](./2.%20add%20system%20calls%20in%20xv6%20%EB%B3%B4%EA%B3%A0%EC%84%9C.md)

## 과제 목표

**목표**: xv6 OS에서 `getppid` System Call을 추가

| 단계 | 내용 |
|------|------|
| 1 | Linux Ubuntu에서 xv6 설치 및 구동 |
| 2 | System Call에 대한 이해 |
| 3 | getppid System Call 기능 분석 |
| 4 | System Call 추가 방법 분석 |
| 5 | getppid System Call 구현 |
| 6 | System Call 호출 과정 정리 |

## 실습 환경

- **플랫폼**: [goorm.io](http://goorm.io) 가상 컨테이너 (Ubuntu)
- **에뮬레이터**: QEMU (`qemu-kvm`)
- **xv6 소스**: `git clone https://github.com/mit-pdos/xv6-public`
- **빌드 및 실행**: `make qemu-nox`

## getppid 구현 원리

`proc` 구조체에는 부모 프로세스 포인터(`struct proc *parent`)가 존재하며, 이를 통해 부모의 PID를 반환한다.

```c
// sysproc.c
int
sys_getppid(void)
{
  return myproc()->parent->pid;
}
```

## 수정 파일 목록

| 파일 | 역할 |
|------|------|
| `user.h` | System Call 함수 프로토타입 선언 (`int getppid(void);`) |
| `usys.S` | System Call 어셈블리 Wrapper (`SYSCALL(getppid)`) |
| `syscall.h` | System Call 번호 매핑 (`#define SYS_getppid 22`) |
| `syscall.c` | 함수 포인터 배열에 등록 (`[SYS_getppid] sys_getppid`) |
| `sysproc.c` | 실제 구현부 (`sys_getppid()`) |

## System Call 호출 흐름

```
pidtest.c  →  getppid()  →  usys.S SYSCALL(getppid)
  →  eax = SYS_getppid(22)  →  int $T_SYSCALL
  →  syscall()  →  syscalls[22]()  →  sys_getppid()
  →  myproc()->parent->pid 반환
```

## 실행 결과

```
$ pidtest
PID  : 4
PPID : 2
```

---

# 3. Scheduler 구현 및 분석

> 상세 내용: [`3. Scheduler.md`](./3.%20Scheduler.md)

## 과제 목표

**목표**: xv6 OS에서 Scheduler의 동작 원리를 파악하고, 스케줄링 로직을 수정

| 목표 | 내용 |
|------|------|
| 1 | 동적 우선순위 조정이 가능한 SSU 스케줄링 구현 |
| 2 | 새로운 스케줄링 알고리즘의 시스템 성능 영향 분석 |
| 3 | xv6의 프로세스 관리 및 스케줄링 기법 이해 |

## SSU 스케줄러 구현

PID의 홀수/짝수에 따라 프로세스를 그룹으로 나누어 번갈아 실행하는 스케줄러.

```c
// proc.c - SSUScheduler 핵심 로직
for(i = 0; i < 2; i++) {         // 짝수 그룹, 홀수 그룹 순서로 2회 순환
  for(p = ptable.proc; ...; p++){
    if(p->state != RUNNABLE) continue;
    if(p->pid % 2 == i) continue; // i=0: 짝수 skip → 홀수 실행
                                   // i=1: 홀수 skip → 짝수 실행
    // ... 프로세스 실행
  }
}
```

## 성능 분석

50개의 자식 프로세스를 생성하여 각 100tick 동안 실행 후 성능을 비교한 결과 (1 tick ≈ 10ms):

| 스케줄러 | Wait Time (avg) | Turnaround Time (avg) | Response Time (avg) |
|----------|----------------:|----------------------:|--------------------:|
| SSU Scheduler | 370.44 tick | 125.98 tick | 186.7 tick |
| xv6 기본 Scheduler | 336.56 tick | 126.8 tick | 233.88 tick |

- **Response Time**: SSU 스케줄러가 약 47tick(~470ms) 단축
- **Wait Time**: xv6 기본 스케줄러가 약 34tick 낮음
- **Turnaround Time**: 두 스케줄러가 거의 유사

## 성능 측정 구현

`proc` 구조체에 시간 측정 변수를 추가하여 분석:

```c
struct proc {
  // ... 기존 멤버 ...
  int runnable_start;  // RUNNABLE 상태 진입 시각
  int waiting_time;    // 총 대기 시간

  int start_time;      // fork() 시점 (Turnaround 측정용)
  int end_time;        // exit() 시점

  int response_time;   // 첫 RUNNING 진입 시각
  int has_run;         // 최초 실행 여부 플래그
};
```

| 측정 항목 | 시작 지점 | 종료 지점 |
|-----------|-----------|-----------|
| Waiting Time | `sched()` - RUNNABLE 진입 시 | `scheduler()` - 프로세스 선택 시 |
| Turnaround Time | `fork()` - 프로세스 생성 시 | `exit()` - 종료 시 |
| Response Time | `fork()` - 프로세스 생성 시 | `scheduler()` - 최초 RUNNING 진입 시 |

## xv6 스케줄러 상세 분석

### 부팅 및 초기화 흐름

```
main() → pinit() → userinit() → mpmain() → scheduler()
                ↓
         initcode 프로세스 실행 → init 프로세스 exec()
                                → sh(shell) 프로세스 fork()+exec()
```

### 스케줄러 동작 원리

```
scheduler() 무한루프
  └─ ptable 순회 → RUNNABLE 프로세스 발견
       └─ swtch() → 프로세스 실행
            └─ 타이머 인터럽트 발생 (10,000,000 클럭마다)
                 └─ trap() → yield() → sched() → scheduler()로 복귀
```

### Shell 명령어 실행 흐름

```
$ [명령어 입력]
  └─ getcmd() → gets() 로 입력 수신
       └─ fork1() → 자식 프로세스 생성
            └─ runcmd(parsecmd(buf)) → exec() → 명령어 실행
                                                → exit()
       └─ wait() → 자식 종료 대기 → 다시 $ 출력
```
