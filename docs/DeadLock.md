# [Bug] Deadlock - 멀티스레드 자원 경쟁으로 인한 프로세스 무응답

## 1. Description

Ubuntu 22.04 Docker 환경에서 agent-leak-app을 실행한 결과, 여러 스레드가 공유 자원을 점유한 상태에서 서로 다른 자원의 해제를 기다리면서 작업이 중단되는 현상이 발생하였다.

실행 환경은 다음과 같다.

```text
MEMORY_LIMIT=512
CPU_MAX_OCCUPY=10
MULTI_THREAD_ENABLE=true
```

애플리케이션은 모든 Boot Check를 정상적으로 통과하였으나, Resource Check에서는 멀티스레드 활성화에 대한 경고가 출력되었다.

이후 Worker-Thread-1과 Worker-Thread-2가 각각 다른 공유 자원을 획득하고 작업을 진행하였다.

그러나 두 스레드가 작업을 완료하기 위해 상대방이 보유한 자원을 추가로 요구하면서 대기 상태에 진입하였다.

두 스레드 모두 WAITING / BLOCKED 상태가 되었으며, 이후 작업 완료 로그가 출력되지 않았다.

이때 프로세스의 PID는 유지되었으나 CPU와 메모리 사용량에 큰 변화가 없는 무응답 상태가 지속되었다.

## 2. Evidence & Logs

### 2.1 애플리케이션 실행 로그

애플리케이션 실행 당시 Resource Check에서는 다음과 같은 경고가 출력되었다.

```text
==================================================
 [ Agent Initiate ] Resource Check
==================================================
 [ MEMORY ] Limit: 512MB                [ OK ]
 [ CPU    ] Limit: 10%                  [ OK ]
 [ THREAD ] Concurrency: True           [ WARNING ]
--------------------------------------------------
 >>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
==================================================
```

이후 멀티스레드 기반 작업이 시작되었다.

```text
2026-10-06 08:42:04,806 [WARNING] [AgentWorker] Initializing concurrent transaction processors...
2026-10-06 08:42:04,806 [WARNING] [System] CAUTION: Strict resource locking is enabled.
```

각 스레드는 서로 다른 공유 자원을 획득하였다.

```text
2026-10-06 08:42:09,832 [INFO] [Worker-Thread-1] Process Started. Attempting to lock [Shared_Memory_A]...
2026-10-06 08:42:09,832 [INFO] [AgentWorker][Worker-Thread-1] LOCK ACQUIRED: [Shared_Memory_A]. (Holding...)
2026-10-06 08:42:09,832 [INFO] [AgentWorker][Worker-Thread-2] Process Started. Attempting to lock [Socket_Pool_B]...
2026-10-06 08:42:09,832 [INFO] [AgentWorker][Worker-Thread-1] Processing critical data in Memory A...
2026-10-06 08:42:09,833 [INFO] [AgentWorker] Waiting for worker threads to complete transactions...
2026-10-06 08:42:09,833 [INFO] [AgentWorker][Worker-Thread-2] LOCK ACQUIRED: [Socket_Pool_B]. (Holding...)
2026-10-06 08:42:09,834 [INFO] [AgentWorker][Worker-Thread-2] Establishing network connections in Pool B...
```

이후 두 스레드는 각각 상대방이 보유한 자원을 요구하기 시작하였다.

```text
2026-10-06 08:42:11,842 [INFO] [AgentWorker][Worker-Thread-1] Need resource [Socket_Pool_B] to finish job.
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-2] Need resource [Shared_Memory_A] to write logs.
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
```

마지막 로그에서 두 스레드 모두 WAITING / BLOCKED 상태임을 확인하였다.

이후 추가적인 작업 진행 로그나 작업 완료 로그가 출력되지 않았다.

### 2.2 monitor.sh 관제 로그

애플리케이션의 로그 출력이 멈춘 이후에도 프로세스가 실제로 존재하는지 확인하기 위해 monitor.sh를 이용하여 프로세스 상태를 관측하였다.

관제 결과 부모 프로세스 PID 5300과 자식 프로세스 PID 5302가 유지되고 있었다.

```text
[2026-10-06 08:42:18] PID:5300 PPID:1386 CPU:0.3% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:18] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:18] TOTAL_RSS:20216KB

[2026-10-06 08:42:19] PID:5300 PPID:1386 CPU:0.3% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:19] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:19] TOTAL_RSS:20216KB

[2026-10-06 08:42:20] PID:5300 PPID:1386 CPU:0.2% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:20] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:20] TOTAL_RSS:20216KB

[2026-10-06 08:42:21] PID:5300 PPID:1386 CPU:0.2% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:21] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:21] TOTAL_RSS:20216KB

[2026-10-06 08:42:22] PID:5300 PPID:1386 CPU:0.2% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:22] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:22] TOTAL_RSS:20216KB

[2026-10-06 08:42:23] PID:5300 PPID:1386 CPU:0.2% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:23] PID:5302 PPID:5300 CPU:0.2% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:23] TOTAL_RSS:20216KB

[2026-10-06 08:42:24] PID:5300 PPID:1386 CPU:0.2% MEM:0.0% RSS:2132KB VSZ:3000KB
[2026-10-06 08:42:24] PID:5302 PPID:5300 CPU:0.1% MEM:0.2% RSS:18084KB VSZ:170168KB
[2026-10-06 08:42:24] TOTAL_RSS:20216KB
```

관제 결과 전체 RSS는 20216KB로 일정하게 유지되었으며, CPU 사용률 역시 낮은 수준으로 유지되었다.

이러한 결과는 프로세스가 종료된 상태가 아니라 별도의 작업을 진행하지 못하고 대기 중인 상태라는 판단을 뒷받침한다.

### 2.3 스레드 상태 확인

교착상태가 발생한 이후 동일한 설정으로 다시 실행하여 ps -L 명령어로 스레드별 상태를 확인하였다.

실행한 명령어는 다음과 같다.

```bash
ps -L -C agent-leak-app -o pid,ppid,tid,stat,%cpu,rss,wchan:25,comm
```

출력 결과는 다음과 같다.

```text
    PID    PPID     TID STAT %CPU   RSS WCHAN                     COMMAND
   5522    1386    5522 S+    0.4  2148 do_wait                   agent-leak-app
   5524    5522    5524 SNl+  0.2 18048 futex_do_wait             agent-leak-app
   5524    5522    5526 SNl+  0.0 18048 futex_do_wait             agent-leak-app
   5524    5522    5527 SNl+  0.0 18048 futex_do_wait             agent-leak-app
```

부모 프로세스 PID 5522와 자식 프로세스 PID 5524가 존재하였으며, 자식 프로세스의 여러 스레드가 futex_do_wait 상태임을 확인하였다.

futex는 Linux에서 스레드 간 동기화를 위해 사용되는 메커니즘으로, futex_do_wait는 스레드가 동기화 조건을 기다리며 대기하고 있음을 나타낸다.

따라서 애플리케이션 로그에서 확인한 WAITING / BLOCKED 상태와 운영체제의 스레드 대기 상태가 서로 일치하였다.

다만 futex_do_wait 자체가 항상 Deadlock을 의미하는 것은 아니며, 이번 경우에는 서로 다른 자원을 보유한 두 스레드의 순환 대기 로그를 함께 확인하여 교착상태로 판단하였다.

## 3. Root Cause Analysis

수집된 로그를 분석한 결과, 서로 다른 두 스레드가 각자 공유 자원을 점유한 상태에서 상대방이 점유한 자원을 요구하면서 교착상태가 발생한 것으로 판단된다.

Deadlock은 여러 프로세스 또는 스레드가 서로의 자원 해제를 기다리면서 어느 쪽도 작업을 진행하지 못하는 상태를 의미한다.

이번 프로그램에서는 두 개의 공유 자원이 사용되었다.

- Shared_Memory_A
- Socket_Pool_B

Worker-Thread-1은 Shared_Memory_A를 획득한 상태에서 Socket_Pool_B가 필요했다.

반대로 Worker-Thread-2는 Socket_Pool_B를 획득한 상태에서 Shared_Memory_A가 필요했다.

자원 점유 및 대기 관계는 다음과 같다.

```text
Worker-Thread-1
  ├─ Shared_Memory_A 보유
  └─ Socket_Pool_B 대기

Worker-Thread-2
  ├─ Socket_Pool_B 보유
  └─ Shared_Memory_A 대기
```

이로 인해 Worker-Thread-1은 Worker-Thread-2가 Socket_Pool_B를 반환하기를 기다리고, Worker-Thread-2는 Worker-Thread-1이 Shared_Memory_A를 반환하기를 기다리는 순환 대기가 발생하였다.

Deadlock이 발생하기 위한 4가지 조건은 다음과 같다.

1. 상호 배제(Mutual Exclusion): 한 번에 하나의 스레드만 특정 자원을 점유할 수 있는 상태
2. 점유 대기(Hold and Wait): 자원을 점유한 상태에서 다른 자원의 획득을 기다리는 상태
3. 비선점(No Preemption): 다른 스레드가 점유한 자원을 강제로 회수할 수 없는 상태
4. 순환 대기(Circular Wait): 각 스레드가 서로 상대방이 점유한 자원을 기다리는 상태

이번 실행에서는 각 스레드가 독립적으로 Lock을 획득하여 자원을 점유하였고, 자신이 점유한 자원을 반환하지 않은 상태에서 상대방의 자원을 기다리고 있었다.

또한 로그에서 두 스레드 모두 상대방 자원을 기다리는 순환 관계가 확인되었다.

Lock이 강제로 회수되거나 대기가 해소되는 동작은 관측되지 않았으며, 두 스레드는 계속 BLOCKED 상태로 유지되었다.

이후 ps -L 명령어에서도 스레드들이 futex_do_wait 상태로 대기하고 있음을 확인하였다.

따라서 이번 장애는 공유 자원의 획득 순서가 서로 다르게 구성되면서 발생한 Deadlock으로 판단된다.

장애 발생 과정은 다음과 같이 정리할 수 있다.

1. 멀티스레드 작업 시작
2. Thread-1이 Shared_Memory_A 획득
3. Thread-2가 Socket_Pool_B 획득
4. Thread-1이 Socket_Pool_B 요청
5. Thread-2가 Shared_Memory_A 요청
6. 두 스레드가 서로 상대방의 자원을 기다림
7. 작업 진행 중단 및 무응답 상태 지속

## 4. Workaround & Verification

멀티스레드 활성화가 교착상태 발생에 영향을 주는지 확인하기 위해 MULTI_THREAD_ENABLE을 true에서 false로 변경하였다.

MEMORY_LIMIT과 CPU_MAX_OCCUPY는 기존 설정을 유지하였다.

### 4.1 Before

기존 환경변수는 다음과 같다.

```text
MEMORY_LIMIT=512
CPU_MAX_OCCUPY=10
MULTI_THREAD_ENABLE=true
```

실행 시 멀티스레드 활성화에 대한 경고가 출력되었다.

```text
[ THREAD ] Concurrency: True           [ WARNING ]

>>> SYSTEM WARNING: POTENTIAL DEADLOCK IN CONCURRENT MODE.
```

이후 두 스레드가 서로 다른 공유 자원을 획득하고 상대방의 자원을 기다리면서 작업이 중단되었다.

```text
2026-10-06 08:42:11,842 [INFO] [AgentWorker][Worker-Thread-1] Need resource [Socket_Pool_B] to finish job.
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-1] WAITING for [Socket_Pool_B]... (Status: BLOCKED)
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-2] Need resource [Shared_Memory_A] to write logs.
2026-10-06 08:42:11,843 [INFO] [AgentWorker][Worker-Thread-2] WAITING for [Shared_Memory_A]... (Status: BLOCKED)
```

이후 PID는 유지되었으나 작업이 진행되지 않았으며, 관제 결과 CPU와 RSS도 거의 변화하지 않았다.

### 4.2 After

MULTI_THREAD_ENABLE을 false로 변경한 뒤 애플리케이션을 재실행하였다.

```bash
export MULTI_THREAD_ENABLE=false
```

변경 후 실행 환경은 다음과 같다.

```text
MEMORY_LIMIT=512
CPU_MAX_OCCUPY=10
MULTI_THREAD_ENABLE=false
```

Resource Check에서는 모든 설정이 정상으로 표시되었다.

```text
[ MEMORY ] Limit: 512MB                [ OK ]
[ CPU    ] Limit: 10%                  [ OK ]
[ THREAD ] Concurrency: False          [ OK ]

>>> SYSTEM STATUS: STABLE. STARTING WORKLOAD MONITORING...
```

이후 정상적인 안정성 테스트가 시작되었다.

```text
2026-10-06 08:44:13,168 [INFO] >>> Scenario Selected: [Healthy System Monitoring]

>>> [SYSTEM] ALL CONFIGURATIONS OPTIMAL. RUNNING STABILITY TEST... <<<

2026-10-06 08:44:13,168 [INFO] [Scheduler] Task Scheduler Initialized.
2026-10-06 08:44:13,168 [INFO] [Scheduler] Registered Tasks: ['Thread-A', 'Thread-B', 'Thread-C']
2026-10-06 08:44:13,168 [INFO] [Scheduler] Starting task execution...
```

기존에는 두 스레드가 서로 다른 자원을 기다리며 작업이 중단되었지만, 변경 후에는 등록된 작업이 모두 정상적으로 완료되었다.

```text
2026-10-06 08:44:13,168 [INFO] [Thread-A] Task Started. Calculating... (20%)
2026-10-06 08:44:13,219 [INFO] [Thread-A] Calculating... (40%)
2026-10-06 08:44:13,269 [INFO] [Thread-A] Calculating... (60%)
2026-10-06 08:44:13,320 [INFO] [Thread-A] Calculating... (80%)
2026-10-06 08:44:13,371 [INFO] [Thread-A] Task Completed. (100%)

2026-10-06 08:44:13,421 [INFO] [Thread-B] Task Started. Calculating... (20%)
2026-10-06 08:44:13,472 [INFO] [Thread-B] Calculating... (40%)
2026-10-06 08:44:13,523 [INFO] [Thread-B] Calculating... (60%)
2026-10-06 08:44:13,574 [INFO] [Thread-B] Calculating... (80%)
2026-10-06 08:44:13,624 [INFO] [Thread-B] Task Completed. (100%)

2026-10-06 08:44:13,675 [INFO] [Thread-C] Task Started. Calculating... (20%)
2026-10-06 08:44:13,726 [INFO] [Thread-C] Calculating... (40%)
2026-10-06 08:44:13,776 [INFO] [Thread-C] Calculating... (60%)
2026-10-06 08:44:13,827 [INFO] [Thread-C] Calculating... (80%)
2026-10-06 08:44:13,878 [INFO] [Thread-C] Task Completed. (100%)

2026-10-06 08:44:13,928 [INFO] [Scheduler] All tasks completed.
```

작업 완료 이후에도 애플리케이션은 정상적으로 실행되었으며, MemoryWorker와 CpuWorker가 계속 동작하였다.

```text
2026-10-06 08:45:11,482 [INFO] [MemoryWorker] Current Heap: 500MB
2026-10-06 08:45:13,122 [INFO] [CpuWorker] Current Load: 10.00%
2026-10-06 08:45:14,511 [INFO] [MemoryWorker] Current Heap: 525MB
2026-10-06 08:45:14,511 [WARNING] [MemoryWorker] Memory Usage Reached Limit (525MB). Starting cleanup...
2026-10-06 08:45:14,533 [INFO] [System] Memory Cache Flushed. Process Stabilized.

>>> [SYSTEM] MEMORY RECOVERED (Cache Cleared) <<<

2026-10-06 08:45:16,239 [INFO] [CpuWorker] Current Load: 6.94%
2026-10-06 08:45:19,355 [INFO] [CpuWorker] Current Load: 5.07%
2026-10-06 08:45:19,567 [INFO] [MemoryWorker] Current Heap: 25MB
```

따라서 MULTI_THREAD_ENABLE을 비활성화한 이후 기존의 교착상태가 재현되지 않았으며, 작업 완료 후에도 애플리케이션이 정상적으로 실행되는 것을 확인하였다.

### 4.3 결과 및 결론

| 항목 | Before | After |
|---|---|---|
| MULTI_THREAD_ENABLE | true | false |
| Resource Check | WARNING | OK |
| 스레드 동작 | WAITING / BLOCKED | 작업 정상 완료 |
| 동기화 대기 | futex_do_wait 확인 | 작업 진행 로그 확인 |
| 작업 완료 여부 | 완료되지 않음 | 모든 작업 완료 |
| 프로세스 상태 | PID 유지, 무응답 | 정상 실행 |
| Deadlock 발생 | 확인 | 관측되지 않음 |

MULTI_THREAD_ENABLE을 true에서 false로 변경한 결과, 기존의 교착상태가 발생하지 않고 모든 작업이 정상적으로 완료되었다.

따라서 이번 장애는 멀티스레드 환경에서 발생한 공유 자원 간 순환 대기와 관련된 것으로 판단된다.

다만 멀티스레드 기능을 비활성화하는 것은 교착상태를 회피하기 위한 임시 조치이며, 동시에 여러 작업을 처리해야 하는 실제 서비스에서는 기능을 비활성화하는 것만으로 문제를 해결하기 어려울 수 있다.

근본적인 해결을 위해서는 다음과 같은 방법을 고려할 수 있다.

1. 여러 스레드가 공유 자원을 획득하는 순서를 통일하여 순환 대기를 방지한다.
2. Lock을 보유하는 시간을 최소화하고 불필요한 중첩 Lock을 제거한다.
3. Lock 획득 시 타임아웃을 설정하여 무한 대기를 방지한다.

이를 통해 멀티스레드 기능을 유지하면서도 교착상태 발생 가능성을 줄일 수 있다.