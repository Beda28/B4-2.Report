# 1. 문서 개요
> 본 문서는 4-2 과정을 진행하기 위한 환경을 설정하기 위한 세팅 명령어 모음집입니다.

# 2. 환경설정
## 2-1. 도커 up
```bash
docker pull ubuntu:22.04  # 생략 가능
docker run -dit --name agent-lab -p 15034:15034 -v ./bind:/usr/bind ubuntu:22.04 sleep infinity
docker exec -it agent-lab bash
```

## 2-2. 폴더 / 권한 설정
```bash
apt update
apt install -y procps psmisc iproute2

useradd -m -s /bin/bash agent-admin

mkdir -p /home/agent-admin/agent-app/api_keys
mkdir -p /home/agent-admin/agent-app/upload_files
mkdir -p /var/log/agent-app

cp /usr/bind/agent-leak-app-x86 /home/agent-admin/agent-app/agent-leak-app

echo "agent_api_key_test" > /home/agent-admin/agent-app/api_keys/secret.key

chmod +x /home/agent-admin/agent-app/agent-leak-app
chmod 660 /home/agent-admin/agent-app/api_keys/secret.key

chown -R agent-admin:agent-admin /home/agent-admin
chown -R agent-admin:agent-admin /var/log/agent-app

su - agent-admin
```

## 2-3. 권한 이동
```bash
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys
export AGENT_LOG_DIR=/var/log/agent-app

export MEMORY_LIMIT=256
export CPU_MAX_OCCUPY=80
export MULTI_THREAD_ENABLE=true

cd $AGENT_HOME
```

```bash
# 적용 확인
whoami

echo $AGENT_HOME
echo $AGENT_PORT
echo $AGENT_UPLOAD_DIR
echo $AGENT_KEY_PATH
echo $AGENT_LOG_DIR
echo $MEMORY_LIMIT
echo $CPU_MAX_OCCUPY
echo $MULTI_THREAD_ENABLE

ls -l $AGENT_HOME
ls -l $AGENT_KEY_PATH
ls -ld $AGENT_UPLOAD_DIR
ls -ld $AGENT_LOG_DIR

cat $AGENT_KEY_PATH/secret.key
```

## 2-4. 실행
```bash
./agent-leak-app
```

## 2-5. monitor.sh 작성
```bash
cd /home/agent-admin/agent-app
rm -f monitor.sh

cat > monitor.sh <<'EOF'
#!/bin/bash

LOG_FILE="/var/log/agent-app/monitor.log"

while true
do
    RESULT=$(ps -C agent-leak-app -o pid=,ppid=,%cpu=,%mem=,rss=,vsz=)

    if [ -n "$RESULT" ]; then
        echo "$RESULT" | \
        awk -v date="$(date '+%Y-%m-%d %H:%M:%S')" \
        '{printf "[%s] PID:%s PPID:%s CPU:%s%% MEM:%s%% RSS:%sKB VSZ:%sKB\n", date,$1,$2,$3,$4,$5,$6}' \
        >> "$LOG_FILE"

        TOTAL_RSS=$(echo "$RESULT" | awk '{sum += $5} END {print sum}')

        echo "[$(date '+%Y-%m-%d %H:%M:%S')] TOTAL_RSS:${TOTAL_RSS}KB" >> "$LOG_FILE"
    else
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] PROCESS:NOT_FOUND" >> "$LOG_FILE"
    fi

    sleep 1
done
EOF

chmod +x monitor.sh
./monitor.sh
```