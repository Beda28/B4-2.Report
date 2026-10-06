# [Bug] OOM Crash - 메모리 사용량 증가에 따른 MemoryGuard 프로세스 종료

## 1. Description

Ubuntu 22.04 Docker 환경에서 agent-leak-app을 실행한 결과, 시간 경과에 따라 메모리 사용량이 지속적으로 증가하다가 프로세스가 종료되는 현상이 발생하였다.

실행 환경은 다음과 같다.

```text
MEMORY_LIMIT=256
CPU_MAX_OCCUPY=80
MULTI_THREAD_ENABLE=true
```

애플리케이션은 모든 Boot Check를 정상적으로 통과하였으며, Agent READY 상태로 실행되었다.

실행 이후 MemoryWorker의 Heap 사용량이 약 3초마다 25MB씩 증가하였으며, 설정된 MEMORY_LIMIT인 256MB를 초과한 275MB에 도달하자 MemoryGuard가 임계치 초과를 감지하였다.

이후 애플리케이션 내부의 보호 정책에 따라 프로세스가 종료되었다.

## 2. Evidence & Logs

### 2.1 애플리케이션 실행 로그

애플리케이션 실행 시 모든 Boot Check가 정상적으로 완료되었다.

```text
>>> Starting Agent Boot Sequence...
[1/6] Checking User Account               [OK]
   ... Running as service user 'agent-admin' (uid=1000)
[2/6] Verifying Environment Variables     [OK]
   ... All required Envs correct
[3/6] Checking Required Files             [OK]
   ... Verified 'secret.key' with correct key string.
[4/6] Checking Port Availability          [OK]
   ... Port 15034 is available.
[5/6] Verifying Log Permission            [OK]
   ... Log directory is writable: /var/log/agent-app
[6/6] Verifying Mission Environment       [OK]
   ... MEMORY_LIMIT=256MB, CPU_MAX_OCCUPY=80%, MULTI_THREAD_ENABLE=True

All Boot Checks Passed!
Agent READY
```

실행 당시 Resource Check에서는 다음과 같은 경고가 출력되었다.

```text
[ MEMORY ] Limit: 256MB       [ WARNING: Recommend Over 256MB ]
[ CPU    ] Limit: 80%         [ WARNING: Recommend Under 50% ]
[ THREAD ] Concurrency: True  [ WARNING ]
```

애플리케이션 실행 이후 MemoryWorker의 Heap 사용량은 다음과 같이 증가하였다.

```text
2026-10-06 03:30:38,685 [INFO] [MemoryWorker] Current Heap: 25MB
2026-10-06 03:30:41,710 [INFO] [MemoryWorker] Current Heap: 50MB
2026-10-06 03:30:44,735 [INFO] [MemoryWorker] Current Heap: 75MB
2026-10-06 03:30:47,763 [INFO] [MemoryWorker] Current Heap: 100MB
2026-10-06 03:30:50,792 [INFO] [MemoryWorker] Current Heap: 125MB
2026-10-06 03:30:53,821 [INFO] [MemoryWorker] Current Heap: 150MB
2026-10-06 03:30:56,850 [INFO] [MemoryWorker] Current Heap: 175MB
2026-10-06 03:30:59,882 [INFO] [MemoryWorker] Current Heap: 200MB
2026-10-06 03:31:02,912 [INFO] [MemoryWorker] Current Heap: 225MB
2026-10-06 03:31:05,941 [INFO] [MemoryWorker] Current Heap: 250MB
2026-10-06 03:31:08,970 [INFO] [MemoryWorker] Current Heap: 275MB
```

최종적으로 Heap 사용량이 275MB에 도달하자 다음 로그와 함께 프로세스가 종료되었다.

```text
2026-10-06 03:31:08,970 [CRITICAL] [MemoryGuard] Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-10-06 03:31:08,971 [CRITICAL] [MemoryGuard] Self-terminating process 863 to prevent system instability.

>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<

Killed
```

### 2.2 monitor.sh 관제 로그

애플리케이션 내부 로그와 별개로 운영체제에서 관측되는 메모리 사용량을 확인하기 위해 monitor.sh를 사용하였다.

동일한 MEMORY_LIMIT=256MB 조건에서 재실행한 뒤 ps 명령어를 이용하여 프로세스별 PID, PPID, CPU, MEM, RSS, VSZ를 1초 간격으로 기록하였다.

관제 결과 agent-leak-app은 부모 프로세스와 자식 프로세스로 실행되고 있었다.

부모 프로세스인 PID 2317의 RSS는 약 2MB 수준으로 유지되었으며, 실제 메모리 사용량 증가는 자식 프로세스인 PID 2318에서 발생하였다.

다음은 관제 로그에서 메모리 사용량이 증가하는 구간을 3초 간격으로 발췌한 결과이다.

```text
[2026-10-06 05:09:12] PID:2317 PPID:1386 CPU:0.3% MEM:0.0% RSS:2068KB VSZ:3000KB
[2026-10-06 05:09:12] PID:2318 PPID:2317 CPU:0.6% MEM:2.1% RSS:171700KB VSZ:176324KB
[2026-10-06 05:09:12] TOTAL_RSS:173768KB

[2026-10-06 05:09:15] PID:2317 PPID:1386 CPU:0.2% MEM:0.0% RSS:2068KB VSZ:3000KB
[2026-10-06 05:09:15] PID:2318 PPID:2317 CPU:0.6% MEM:2.4% RSS:197304KB VSZ:201928KB
[2026-10-06 05:09:15] TOTAL_RSS:199372KB

[2026-10-06 05:09:18] PID:2317 PPID:1386 CPU:0.2% MEM:0.0% RSS:2068KB VSZ:3000KB
[2026-10-06 05:09:18] PID:2318 PPID:2317 CPU:0.5% MEM:2.8% RSS:222908KB VSZ:227532KB
[2026-10-06 05:09:18] TOTAL_RSS:224976KB

[2026-10-06 05:09:21] PID:2317 PPID:1386 CPU:0.2% MEM:0.0% RSS:2068KB VSZ:3000KB
[2026-10-06 05:09:21] PID:2318 PPID:2317 CPU:0.5% MEM:3.1% RSS:248512KB VSZ:253136KB
[2026-10-06 05:09:21] TOTAL_RSS:250580KB

[2026-10-06 05:09:24] PID:2317 PPID:1386 CPU:0.2% MEM:0.0% RSS:2068KB VSZ:3000KB
[2026-10-06 05:09:24] PID:2318 PPID:2317 CPU:0.5% MEM:3.4% RSS:274116KB VSZ:278740KB
[2026-10-06 05:09:24] TOTAL_RSS:276184KB
```

자식 프로세스의 RSS는 171700KB에서 274116KB까지 증가하였다.

반면 CPU 사용률은 약 0.5~0.6% 수준으로 큰 변화가 없었으므로, 해당 구간에서는 CPU 과점유보다 메모리 사용량 증가가 두드러지는 것을 확인하였다.

이후 프로세스가 종료되면서 다음과 같은 로그가 기록되었다.

```text
[2026-10-06 05:09:27] PROCESS:NOT_FOUND
[2026-10-06 05:09:28] PROCESS:NOT_FOUND
```

애플리케이션 실행 로그와 운영체제 관제 로그는 서로 다른 실행 회차에서 수집되었으나, 동일한 메모리 제한 조건에서 메모리 증가 이후 프로세스가 종료되는 현상을 반복적으로 확인할 수 있었다.

## 3. Root Cause Analysis

수집된 로그를 분석한 결과, 메모리 사용량의 지속적인 증가가 MemoryGuard의 보호 정책을 작동시킨 직접적인 원인으로 판단된다.

애플리케이션 내부에서는 MemoryWorker가 일정한 간격으로 Heap 사용량을 증가시키고 있었으며, 운영체제 관제 결과에서도 자식 프로세스의 RSS가 지속적으로 상승하였다.

Heap은 프로그램 실행 중 동적으로 생성되는 데이터가 저장되는 메모리 영역이다.

프로그램이 더 이상 사용하지 않는 데이터를 계속 참조하거나 메모리를 적절하게 반환하지 않을 경우, 사용량이 지속적으로 증가하는 Memory Leak 현상이 발생할 수 있다.

이번 실험에서는 Heap 사용량과 RSS가 함께 증가하였으므로, 애플리케이션 내부에서 메모리가 지속적으로 누적되는 현상이 발생한 것으로 판단하였다.

다만 제공된 바이너리의 내부 코드를 분석하지 않았으므로, 어떤 객체나 데이터가 해제되지 않았는지까지는 특정할 수 없다.

애플리케이션의 MEMORY_LIMIT은 256MB로 설정되어 있었으며, MemoryWorker가 보고한 Heap 사용량이 275MB에 도달하자 MemoryGuard가 제한 초과를 감지하였다.

```text
Memory limit exceeded (275MB >= 256MB)
```

이후 MemoryGuard는 시스템의 불안정을 방지하기 위한 보호 정책에 따라 프로세스를 종료하였다.

```text
Self-terminating process 863 to prevent system instability.
```

따라서 이번 종료는 시스템 전체의 메모리 부족으로 Linux 커널의 OOM Killer가 개입한 것으로 확인된 상황이 아니라, 애플리케이션 내부의 MemoryGuard가 자체적으로 종료를 수행한 상황이다.

장애 발생 과정은 다음과 같이 정리할 수 있다.

1. MemoryWorker 실행
2. Heap 메모리 사용량 지속 증가
3. 자식 프로세스의 RSS 증가
4. MEMORY_LIMIT=256MB 초과
5. MemoryGuard의 메모리 임계치 초과 감지
6. 애플리케이션 자체 보호 정책에 따른 프로세스 종료

## 4. Workaround & Verification

메모리 임계치가 프로세스 종료에 미치는 영향을 확인하기 위해 MEMORY_LIMIT 환경변수를 기존 256MB에서 512MB로 변경하고 애플리케이션을 재실행하였다.

### 4.1 Before

기존 환경변수는 다음과 같다.

```text
MEMORY_LIMIT=256
```

실행 당시 Resource Check 결과는 다음과 같았다.

```text
[ MEMORY ] Limit: 256MB [ WARNING: Recommend Over 256MB ]
```

실행 결과 Heap 사용량이 275MB에 도달하면서 설정된 제한을 초과하였다.

```text
2026-10-06 03:31:05,941 [INFO] [MemoryWorker] Current Heap: 250MB
2026-10-06 03:31:08,970 [INFO] [MemoryWorker] Current Heap: 275MB

2026-10-06 03:31:08,970 [CRITICAL] [MemoryGuard] Memory limit exceeded (275MB >= 256MB) / (Recommend Over 256MB)
2026-10-06 03:31:08,971 [CRITICAL] [MemoryGuard] Self-terminating process 863 to prevent system instability.

>>> [SYSTEM] SELF-TERMINATED (Memory Limit Exceeded) <<<

Killed
```

### 4.2 After

MEMORY_LIMIT 환경변수를 다음과 같이 변경하였다.

```bash
export MEMORY_LIMIT=512
```

변경 후 애플리케이션을 재실행한 결과, Resource Check에서 메모리 설정이 정상 상태로 표시되었다.

```text
[ MEMORY ] Limit: 512MB                [ OK ]
[ CPU    ] Limit: 80%                  [ WARNING: Recommend Under 50% ]
[ THREAD ] Concurrency: True           [ WARNING ]
```

이후 애플리케이션은 CPU 작업을 실행하였다.

```text
2026-10-06 05:27:55,141 [INFO] [CpuWorker] Started. Maximum CPU Limit: 80%
2026-10-06 05:27:55,141 [INFO] [CpuWorker] Current Load: 5.00%
2026-10-06 05:27:58,257 [INFO] [CpuWorker] Current Load: 10.89%
2026-10-06 05:28:01,373 [INFO] [CpuWorker] Current Load: 12.87%
2026-10-06 05:28:04,490 [INFO] [CpuWorker] Current Load: 12.93%
```

변경 후 실행에서는 MemoryGuard에 의한 종료 없이 CpuWorker가 동작하는 것을 확인하였다.

다만 해당 실행에서는 MemoryWorker의 메모리 증가 과정이 출력되지 않았으므로, 동일한 메모리 누수 상황에서 512MB 설정이 종료 시점을 얼마나 지연시키는지까지는 직접 비교하지 못하였다.

### 4.3 결과 및 결론

MEMORY_LIMIT을 256MB에서 512MB로 변경한 결과, Resource Check의 메모리 제한 상태가 WARNING에서 OK로 변경되었으며, 변경 후 관측한 실행에서는 MemoryGuard 종료가 발생하지 않았다.

이를 통해 메모리 제한 설정을 변경하여 애플리케이션의 실행 조건을 조정할 수 있음을 확인하였다.

다만 동일한 MemoryWorker 부하에서의 생존 시간 비교는 추가 검증이 필요하다.

또한 메모리 임계치를 증가시키는 것은 메모리 누수 자체를 해결하는 방법이 아니라 허용 가능한 메모리 사용량을 증가시키는 임시 조치이다.

근본적인 해결을 위해서는 애플리케이션 내부에서 메모리가 지속적으로 증가하는 원인을 파악하고, 불필요한 객체나 데이터가 메모리에 계속 남지 않도록 개선해야 한다.