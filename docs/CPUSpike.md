# [Bug] CPU Spike - CPU 부하 증가에 따른 Watchdog 프로세스 종료

## 1. Description

Ubuntu 22.04 Docker 환경에서 agent-leak-app을 실행한 결과, CpuWorker가 기록하는 CPU 부하 수치가 지속적으로 증가하다가 프로세스가 종료되는 현상이 발생하였다.

실행 환경은 다음과 같다.

```text
MEMORY_LIMIT=512
CPU_MAX_OCCUPY=80
MULTI_THREAD_ENABLE=false
```

애플리케이션은 모든 Boot Check를 정상적으로 통과하였으나, Resource Check에서 CPU 제한 설정에 대한 경고가 출력되었다.

이후 CpuWorker가 약 3초 간격으로 부하 수치를 증가시키기 시작하였으며, 초기 5%에서 52.03%까지 상승하였다.

이 과정에서 CPU Threshold Violated 로그가 발생하였고, 프로세스가 종료되었다.

다만 설정된 CPU_MAX_OCCUPY는 80%였으므로, 단순히 설정값 80%를 초과하여 종료되었다고 판단할 수는 없다.

## 2. Evidence & Logs

### 2.1 애플리케이션 실행 로그

실행 당시 Resource Check 결과는 다음과 같았다.

```text
==================================================
 [ Agent Initiate ] Resource Check
==================================================
 [ MEMORY ] Limit: 512MB                [ OK ]
 [ CPU    ] Limit: 80%                  [ WARNING: Recommend Under 50% ]
 [ THREAD ] Concurrency: False          [ OK ]
--------------------------------------------------
 >>> SYSTEM STATUS: STABLE. STARTING WORKLOAD MONITORING...
==================================================
```

애플리케이션은 정상적으로 실행되었으나 CPU 설정값이 권장 범위인 50% 미만을 벗어난 상태였다.

이후 CpuWorker의 부하 수치는 다음과 같이 증가하였다.

```text
2026-10-06 08:35:44,588 [INFO] [CpuWorker] Started. Maximum CPU Limit: 80%
2026-10-06 08:35:44,588 [INFO] [CpuWorker] Current Load: 5.00%
2026-10-06 08:35:47,705 [INFO] [CpuWorker] Current Load: 6.52%
2026-10-06 08:35:50,821 [INFO] [CpuWorker] Current Load: 8.73%
2026-10-06 08:35:53,937 [INFO] [CpuWorker] Current Load: 14.15%
2026-10-06 08:35:57,053 [INFO] [CpuWorker] Current Load: 17.91%
2026-10-06 08:36:00,169 [INFO] [CpuWorker] Current Load: 19.83%
2026-10-06 08:36:03,281 [INFO] [CpuWorker] Current Load: 29.27%
2026-10-06 08:36:06,397 [INFO] [CpuWorker] Current Load: 35.07%
2026-10-06 08:36:09,514 [INFO] [CpuWorker] Current Load: 37.35%
2026-10-06 08:36:12,628 [INFO] [CpuWorker] Current Load: 40.06%
2026-10-06 08:36:15,745 [INFO] [CpuWorker] Current Load: 42.41%
2026-10-06 08:36:18,859 [INFO] [CpuWorker] Current Load: 52.03%
2026-10-06 08:36:18,960 [CRITICAL] [CpuWorker] CPU Threshold Violated! (52.029999999999994%).
```

약 34초 동안 CpuWorker의 부하 수치가 5%에서 52.03%까지 증가하였으며, 최종적으로 보호 조건 위반 로그가 출력되었다.

이전에 동일한 CPU_MAX_OCCUPY=80 설정에서 실행한 별도 실험에서도 유사한 현상이 발생하였다.

```text
2026-10-06 05:25:22,022 [INFO] [CpuWorker] Current Load: 47.77%
2026-10-06 05:25:25,133 [INFO] [CpuWorker] Current Load: 57.20%
2026-10-06 05:25:25,234 [CRITICAL] [CpuWorker] CPU Threshold Violated! (57.199999999999996%).

>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<

Terminated
```

이전 실행은 MULTI_THREAD_ENABLE=true 상태였으므로 최신 실험과 실행 조건이 완전히 같지는 않다.

다만 두 실험에서 CpuWorker의 부하 수치가 약 50%를 초과한 이후 Threshold Violation이 발생하였으며, 이전 실행에서는 Watchdog이 SIGTERM을 이용해 프로세스 종료를 시작한 사실까지 확인하였다.

### 2.2 monitor.sh 관제 로그

애플리케이션 내부 부하 수치와 별개로 운영체제에서 측정되는 CPU 사용률을 확인하기 위해 monitor.sh를 사용하였다.

관제 결과 부모 프로세스 PID 4019와 자식 프로세스 PID 4021이 확인되었다.

다음은 프로세스 종료 직전의 관제 로그이다.

```text
[2026-10-06 08:36:08] PID:4019 PPID:1386 CPU:0.2% MEM:0.0% RSS:2152KB VSZ:3000KB
[2026-10-06 08:36:08] PID:4021 PPID:4019 CPU:0.6% MEM:0.2% RSS:17696KB VSZ:23728KB
[2026-10-06 08:36:08] TOTAL_RSS:19848KB

[2026-10-06 08:36:11] PID:4019 PPID:1386 CPU:0.2% MEM:0.0% RSS:2152KB VSZ:3000KB
[2026-10-06 08:36:11] PID:4021 PPID:4019 CPU:0.7% MEM:0.2% RSS:17696KB VSZ:23728KB
[2026-10-06 08:36:11] TOTAL_RSS:19848KB

[2026-10-06 08:36:14] PID:4019 PPID:1386 CPU:0.1% MEM:0.0% RSS:2152KB VSZ:3000KB
[2026-10-06 08:36:14] PID:4021 PPID:4019 CPU:0.7% MEM:0.2% RSS:17696KB VSZ:23728KB
[2026-10-06 08:36:14] TOTAL_RSS:19848KB

[2026-10-06 08:36:18] PID:4019 PPID:1386 CPU:0.1% MEM:0.0% RSS:2152KB VSZ:3000KB
[2026-10-06 08:36:18] PID:4021 PPID:4019 CPU:0.8% MEM:0.2% RSS:17696KB VSZ:23728KB
[2026-10-06 08:36:18] TOTAL_RSS:19848KB
```

자식 프로세스의 CPU 사용률은 약 0.6~0.8% 수준으로 관측되었으며, RSS 역시 17696KB로 일정하게 유지되었다.

이후 대상 프로세스가 종료되면서 다음 로그가 기록되었다.

```text
[2026-10-06 08:36:19] PROCESS:NOT_FOUND
[2026-10-06 08:36:20] PROCESS:NOT_FOUND
[2026-10-06 08:36:21] PROCESS:NOT_FOUND
```

여기서 주의할 점은 CpuWorker의 Current Load와 ps 명령어에서 측정한 CPU 사용률이 일치하지 않는다는 것이다.

ps 명령어의 %CPU는 일반적으로 프로세스 실행 이후의 평균 CPU 사용률을 나타내므로 순간적인 부하 변화가 정확하게 반영되지 않을 수 있다.

그러나 이번 관제 결과에서는 실제 CPU 사용률이 50% 이상으로 급상승한 사실을 확인하지 못하였다.

따라서 CpuWorker가 출력하는 Current Load는 운영체제에서 측정한 CPU 사용률과 구분하여 분석하였다.

## 3. Root Cause Analysis

수집된 로그를 분석한 결과, CpuWorker의 부하 수치가 지속적으로 증가하면서 애플리케이션 내부의 보호 조건을 만족하게 된 것이 프로세스 종료의 직접적인 원인으로 판단된다.

일반적으로 특정 프로세스가 CPU를 장시간 과도하게 점유하면 다른 프로세스가 CPU를 사용할 기회가 줄어들어 시스템 응답 지연이 발생할 수 있다.

이를 방지하기 위해 애플리케이션에서는 CPU 사용량을 감시하고, 일정한 보호 조건을 만족하면 작업을 중단하거나 프로세스를 종료하는 Watchdog을 사용할 수 있다.

이번 프로그램에서도 CpuWorker가 부하를 증가시키다가 보호 조건에 도달하자 Threshold Violation이 발생하였다.

```text
[CRITICAL] [CpuWorker] CPU Threshold Violated! (52.029999999999994%).
```

다만 CPU_MAX_OCCUPY가 80으로 설정되어 있었음에도 52.03%에서 종료 조건이 발생하였다.

이는 CPU_MAX_OCCUPY가 단순한 종료 임계값으로만 사용되는 것이 아니라, 애플리케이션 내부의 부하 제어 방식이나 보호 정책을 결정하는 데 영향을 주는 것으로 추정된다.

Resource Check에서 CPU_MAX_OCCUPY가 50% 미만일 것을 권장하였으며, 서로 다른 두 실행에서 약 50%를 초과한 이후 보호 조건이 발생했다는 점을 고려하면 별도의 보호 기준이 존재할 가능성이 있다.

그러나 바이너리 내부 구현을 확인하지 않았으므로 정확한 임계값이나 판단 조건까지 단정할 수는 없다.

또한 관제 결과에서 실제 OS CPU 사용률의 급상승은 확인되지 않았으므로, 물리적인 CPU 과점유가 직접 발생했다고 확정하기보다는 애플리케이션 내부 부하 지표에 따른 보호 정책이 작동한 것으로 해석하는 것이 적절하다.

이전 실험에서는 다음 종료 로그를 확인하였다.

```text
>>> [SYSTEM] WATCHDOG: INITIATING EMERGENCY ABORT (SIGTERM) <<<

Terminated
```

SIGTERM은 운영체제가 프로세스에 전달하는 종료 요청 신호이다.

Watchdog은 CPU 보호 조건이 발생했을 때 SIGTERM을 통해 실행 중인 프로세스의 종료를 시작한 것으로 판단된다.

장애 발생 과정은 다음과 같이 정리할 수 있다.

1. CpuWorker 실행
2. 내부 CPU 부하 지표 증가
3. 보호 조건에 해당하는 부하 수준 도달
4. CPU Threshold Violation 발생
5. Watchdog의 종료 조치
6. 프로세스 종료

## 4. Workaround & Verification

CPU 보호 설정 변경에 따른 동작 차이를 확인하기 위해 CPU_MAX_OCCUPY를 기존 80에서 10으로 변경하였다.

메모리 제한과 멀티스레드 설정은 동일하게 유지하였다.

### 4.1 Before

기존 환경변수는 다음과 같다.

```text
MEMORY_LIMIT=512
CPU_MAX_OCCUPY=80
MULTI_THREAD_ENABLE=false
```

Resource Check에서 CPU 제한에 대한 경고가 출력되었다.

```text
[ CPU ] Limit: 80% [ WARNING: Recommend Under 50% ]
```

이후 CpuWorker의 부하 수치가 52.03%까지 증가하면서 보호 조건이 발생하였다.

```text
2026-10-06 08:36:12,628 [INFO] [CpuWorker] Current Load: 40.06%
2026-10-06 08:36:15,745 [INFO] [CpuWorker] Current Load: 42.41%
2026-10-06 08:36:18,859 [INFO] [CpuWorker] Current Load: 52.03%
2026-10-06 08:36:18,960 [CRITICAL] [CpuWorker] CPU Threshold Violated! (52.029999999999994%).
```

이후 monitor.sh에서 PID가 더 이상 확인되지 않았다.

### 4.2 After

CPU_MAX_OCCUPY를 10으로 변경한 뒤 애플리케이션을 재실행하였다.

```bash
export CPU_MAX_OCCUPY=10
```

실행 환경은 다음과 같다.

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
2026-10-06 08:27:56,702 [INFO] >>> Scenario Selected: [Healthy System Monitoring]

>>> [SYSTEM] ALL CONFIGURATIONS OPTIMAL. RUNNING STABILITY TEST... <<<
```

CpuWorker는 부하 수치가 10%에 도달하면 Cooldown을 수행하였으며, 일정 수준까지 낮아진 뒤 다시 부하를 증가시키는 동작을 반복하였다.

```text
2026-10-06 08:28:02,703 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
2026-10-06 08:28:03,709 [INFO] [CpuWorker] Current Load: 10.00%
2026-10-06 08:28:06,825 [INFO] [CpuWorker] Current Load: 8.37%
2026-10-06 08:28:08,935 [INFO] [CpuWorker] Cooldown complete (5.00%). Resuming load increase...
2026-10-06 08:28:09,941 [INFO] [CpuWorker] Current Load: 5.00%

2026-10-06 08:28:12,050 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
2026-10-06 08:28:13,054 [INFO] [CpuWorker] Current Load: 10.00%
2026-10-06 08:28:15,166 [INFO] [CpuWorker] Cooldown complete (5.00%). Resuming load increase...
2026-10-06 08:28:16,171 [INFO] [CpuWorker] Current Load: 5.00%
```

이후에도 동일한 동작을 반복하면서 애플리케이션이 정상적으로 실행되었다.

```text
2026-10-06 08:28:49,436 [INFO] [CpuWorker] Peak reached (10.00%). Starting cooldown...
2026-10-06 08:28:50,442 [INFO] [CpuWorker] Current Load: 10.00%
2026-10-06 08:28:52,553 [INFO] [CpuWorker] Cooldown complete (5.00%). Resuming load increase...
2026-10-06 08:28:53,558 [INFO] [CpuWorker] Current Load: 5.00%
```

### 4.3 결과 및 결론

| 항목 | Before | After |
|---|---|---|
| CPU_MAX_OCCUPY | 80% | 10% |
| Resource Check | WARNING | OK |
| 관측된 최대 내부 부하 | 52.03% | 10% |
| 부하 제어 | 지속 증가 | Cooldown 반복 |
| 보호 조건 위반 | 발생 | 관측되지 않음 |
| 프로세스 상태 | 종료 | 실행 지속 |

CPU_MAX_OCCUPY를 80에서 10으로 변경한 결과, CpuWorker는 부하가 10%에 도달하면 Cooldown을 수행하며 안정적인 상태를 유지하였다.

변경 전에는 부하 수치가 지속적으로 증가하다가 Threshold Violation이 발생하였으나, 변경 후에는 부하를 낮추고 다시 증가시키는 동작을 반복하였다.

따라서 CPU_MAX_OCCUPY를 권장 범위로 조정하는 것이 애플리케이션 내부 부하 제어 정책을 안정적으로 동작시키는 데 효과가 있음을 확인하였다.

다만 실제 운영체제의 CPU 사용률은 낮은 수준으로 관측되었으므로, 이번 실험만으로 실제 CPU 과점유 문제가 발생하거나 해결되었다고 단정할 수는 없다.

근본적인 해결을 위해서는 실제 CPU 사용률과 애플리케이션 내부 부하 지표를 구분하여 감시하고, 과도한 연산이 발생하는 경우 해당 작업의 실행 빈도나 처리 방식을 최적화할 필요가 있다.