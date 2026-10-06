## 1. 문서 개요

본 실습에서는 Linux 환경에서 실행되는 agent-leak-app을 대상으로 메모리 누수(Memory Leak), CPU 과점유(CPU Spike), 교착상태(Deadlock) 세 가지 장애 상황을 재현하고 분석하였다.

각 장애는 애플리케이션 실행 로그와 Linux 시스템 명령어, monitor.sh를 통해 수집한 프로세스 자원 사용량을 근거로 분석하였다. 이후 장애와 관련된 환경변수를 변경하여 동일한 조건에서 다시 실행하고, 변경 전후의 프로세스 상태와 생존 여부를 비교하였다.

실습은 Ubuntu 22.04 Docker 컨테이너에서 진행하였으며, 애플리케이션은 일반 사용자 계정인 agent-admin으로 실행하였다.

### 실행 환경

[ 자세한 실행환경 세팅 ](/docs/Setting.md)

```text
OS                  : Ubuntu 22.04 (Docker)
Application         : agent-leak-app
User                : agent-admin
AGENT_PORT          : 15034
MEMORY_LIMIT        : 256MB
CPU_MAX_OCCUPY      : 80%
MULTI_THREAD_ENABLE : true
```

### 장애 분석 방법

각 장애는 다음 순서로 분석하였다.

```text
장애 재현
→ 애플리케이션 로그 확인
→ 프로세스 CPU / Memory 상태 확인
→ 장애 원인 분석
→ 관련 환경변수 변경
→ 동일 조건 재실행
→ Before / After 비교
```

## 2. Memory Leak
[ 보고서 ](/docs/Memory%20Leak.md)

## 3. CPU Spike
[ 보고서 ](/docs/CPUSpike.md)

## 4. DeadLock
[ 보고서 ](/docs/DeadLock.md)