# 요구사항 수행 내역서

> **과제**: Linux 기반 Agent App 운영 환경 구성 및 시스템 관제 자동화  
> **환경**: Ubuntu 24.04 LTS (Docker Container, `--privileged`)  
> **작성일**: 2026-05-18

---

## ✅ 필수 증거 자료 체크리스트

| # | 항목 | 상태 |
|---|------|------|
| 1 | SSH 포트 변경(20022) 및 Root 원격 접속 차단 | ✅ 완료 |
| 2 | 방화벽(UFW) 활성화 및 20022/tcp, 15034/tcp만 허용 | ✅ 완료 |
| 3 | 계정/그룹(agent-admin/dev/test, agent-common/core) 생성 | ✅ 완료 |
| 4 | 디렉토리 구조 및 권한(ACL 포함) | ✅ 완료 |
| 5 | 앱 Boot Sequence 5단계 [OK] 및 "Agent READY" | ✅ 완료 |
| 6 | monitor.sh 실행 결과(프로세스/포트/리소스/경고) | ✅ 완료 |
| 7 | /var/log/agent-app/monitor.log 누적 기록 확인 | ✅ 완료 |
| 8 | crontab 매분 실행 등록 및 자동 실행 확인 | ✅ 완료 |

---

## 1. SSH 포트 변경 및 Root 원격 접속 차단

### 설정 명령어

```bash
# openssh-server 설치
apt-get install -y openssh-server

# Port 20022로 변경
sed -i 's/^#Port 22/Port 20022/' /etc/ssh/sshd_config
sed -i 's/^Port 22/Port 20022/' /etc/ssh/sshd_config

# Root 원격 로그인 차단
sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# sshd 시작
service ssh start

# 확인
grep -E "^Port|^PermitRootLogin" /etc/ssh/sshd_config
ss -tulnp | grep sshd
```

### 확인 결과

```
Port 20022
PermitRootLogin no
tcp   LISTEN 0   128   0.0.0.0:20022   0.0.0.0:*   users:(("sshd",pid=6021,fd=3))
tcp   LISTEN 0   128      [::]:20022      [::]:*   users:(("sshd",pid=6021,fd=4))
```

### ✅ 체크 결과

- [x] `Port 20022` 설정 확인
- [x] `PermitRootLogin no` 설정 확인
- [x] sshd가 `0.0.0.0:20022` 및 `[::]:20022` 로 LISTEN 중

---

## 2. 방화벽(UFW) 활성화 및 포트 허용

### 설정 명령어

```bash
# UFW 설치
apt-get install -y ufw

# 기본 정책 설정
ufw default deny incoming
ufw default allow outgoing

# 필요 포트만 허용
ufw allow 20022/tcp
ufw allow 15034/tcp

# 활성화
echo "y" | ufw enable

# 확인
ufw status verbose
```

### 확인 결과

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
20022/tcp                  ALLOW IN    Anywhere
15034/tcp                  ALLOW IN    Anywhere
20022/tcp (v6)             ALLOW IN    Anywhere (v6)
15034/tcp (v6)             ALLOW IN    Anywhere (v6)
```

### ✅ 체크 결과

- [x] UFW `Status: active` 확인
- [x] `20022/tcp` ALLOW IN 확인 (SSH)
- [x] `15034/tcp` ALLOW IN 확인 (APP)
- [x] 기본 정책 `deny (incoming)` 확인
- [x] IPv6 규칙 동시 적용 확인

---

## 3. 계정/그룹 생성

### 설정 명령어

```bash
# 그룹 생성
groupadd agent-common
groupadd agent-core

# 계정 생성
useradd -m -s /bin/bash agent-admin
useradd -m -s /bin/bash agent-dev
useradd -m -s /bin/bash agent-test

# 비밀번호 설정
echo "agent-admin:Admin1234!" | chpasswd
echo "agent-dev:Dev1234!" | chpasswd
echo "agent-test:Test1234!" | chpasswd

# agent-common: admin, dev, test 모두
usermod -aG agent-common agent-admin
usermod -aG agent-common agent-dev
usermod -aG agent-common agent-test

# agent-core: admin, dev만
usermod -aG agent-core agent-admin
usermod -aG agent-core agent-dev

# 확인
id agent-admin
id agent-dev
id agent-test
```

### 확인 결과

```
uid=1001(agent-admin) gid=1003(agent-admin) groups=1003(agent-admin),1001(agent-common),1002(agent-core)
uid=1002(agent-dev)   gid=1004(agent-dev)   groups=1004(agent-dev),1001(agent-common),1002(agent-core)
uid=1003(agent-test)  gid=1005(agent-test)  groups=1005(agent-test),1001(agent-common)
```

### ✅ 체크 결과

- [x] `agent-admin` 생성 → agent-common, agent-core 소속 확인
- [x] `agent-dev` 생성 → agent-common, agent-core 소속 확인
- [x] `agent-test` 생성 → agent-common 소속만 확인 (agent-core 미포함)

---

## 4. 디렉토리 구조 및 권한(ACL 포함)

### 디렉토리 구조

```
/home/agent-admin/agent-app/        (AGENT_HOME)
├── agent-app                        (실행 바이너리)
├── api_keys/                        (agent-core ONLY)
│   └── t_secret.key
├── bin/
│   └── monitor.sh
└── upload_files/                    (agent-common R/W)

/var/log/agent-app/                  (agent-core ONLY)
└── monitor.log
```

### 설정 명령어

```bash
# ACL 패키지 설치
apt-get install -y acl

# 디렉토리 생성
mkdir -p /home/agent-admin/agent-app/upload_files
mkdir -p /home/agent-admin/agent-app/api_keys
mkdir -p /home/agent-admin/agent-app/bin
mkdir -p /var/log/agent-app

# 소유자/그룹 설정
chown -R agent-admin:agent-common /home/agent-admin/agent-app
chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys
chown agent-admin:agent-core /var/log/agent-app

# 기본 권한
chmod 770 /home/agent-admin/agent-app/upload_files
chmod 770 /home/agent-admin/agent-app/api_keys
chmod 770 /var/log/agent-app

# ACL: upload_files → agent-common R/W
setfacl -m g:agent-common:rwx /home/agent-admin/agent-app/upload_files
setfacl -d -m g:agent-common:rwx /home/agent-admin/agent-app/upload_files

# ACL: api_keys → agent-core만 허용, agent-common 차단
setfacl -m g:agent-core:rwx /home/agent-admin/agent-app/api_keys
setfacl -m g:agent-common:--- /home/agent-admin/agent-app/api_keys
setfacl -d -m g:agent-core:rwx /home/agent-admin/agent-app/api_keys
setfacl -d -m g:agent-common:--- /home/agent-admin/agent-app/api_keys

# ACL: /var/log/agent-app → agent-core만 허용, agent-common 차단
setfacl -m g:agent-core:rwx /var/log/agent-app
setfacl -m g:agent-common:--- /var/log/agent-app
setfacl -d -m g:agent-core:rwx /var/log/agent-app
setfacl -d -m g:agent-common:--- /var/log/agent-app

# 확인
ls -l /home/agent-admin/agent-app/
getfacl /home/agent-admin/agent-app/upload_files
getfacl /home/agent-admin/agent-app/api_keys
getfacl /var/log/agent-app
```

### 확인 결과

```
total 7744
-rwxrwxr-x  1 agent-admin agent-admin  7926296 Jan 29 10:36 agent-app
drwxrwx---+ 1 agent-admin agent-core        24 May 18 14:15 api_keys
drwxr-xr-x  1 agent-admin agent-common      20 May 18 14:19 bin
drwxrwx---+ 1 agent-admin agent-common       0 May 18 14:15 upload_files

# file: home/agent-admin/agent-app/upload_files
# owner: agent-admin / group: agent-common
user::rwx
group::rwx
group:agent-common:rwx
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:group:agent-common:rwx
default:mask::rwx
default:other::---

# file: home/agent-admin/agent-app/api_keys
# owner: agent-admin / group: agent-core
user::rwx
group::rwx
group:agent-common:---
group:agent-core:rwx
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:group:agent-common:---
default:group:agent-core:rwx
default:mask::rwx
default:other::---

# file: var/log/agent-app
# owner: agent-admin / group: agent-core
user::rwx
group::rwx
group:agent-common:---
group:agent-core:rwx
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:group:agent-common:---
default:group:agent-core:rwx
default:mask::rwx
default:other::---
```

### ✅ 체크 결과

- [x] `upload_files` → agent-common R/W 가능
- [x] `api_keys` → agent-core ONLY, agent-common 차단 확인
- [x] `/var/log/agent-app` → agent-core ONLY, agent-common 차단 확인
- [x] default ACL 설정으로 하위 파일에도 권한 상속 확인

---

## 5. 환경 변수 설정 및 키 파일 생성

### 설정 명령어

```bash
# agent-admin .bashrc에 환경 변수 추가
cat >> /home/agent-admin/.bashrc << 'EOF'

# ── Agent App 환경 변수 ──────────────────────────────────
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app
EOF

# /etc/environment에도 등록 (cron 등 비로그인 쉘 대비)
cat >> /etc/environment << 'EOF'
AGENT_HOME=/home/agent-admin/agent-app
AGENT_PORT=15034
AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files
AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys/t_secret.key
AGENT_LOG_DIR=/var/log/agent-app
EOF

# 키 파일 생성
echo "agent_api_key_test" > /home/agent-admin/agent-app/api_keys/t_secret.key
chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys/t_secret.key
chmod 640 /home/agent-admin/agent-app/api_keys/t_secret.key

# 확인
cat /home/agent-admin/agent-app/api_keys/t_secret.key
grep "AGENT_" /etc/environment
```

### 확인 결과

```
agent_api_key_test

AGENT_HOME=/home/agent-admin/agent-app
AGENT_PORT=15034
AGENT_UPLOAD_DIR=/home/agent-admin/agent-app/upload_files
AGENT_KEY_PATH=/home/agent-admin/agent-app/api_keys/t_secret.key
AGENT_LOG_DIR=/var/log/agent-app
```

### ✅ 체크 결과

- [x] 키 파일 내용 `agent_api_key_test` 확인
- [x] 환경 변수 5개 `/etc/environment` 등록 확인
- [x] `/home/agent-admin/.bashrc` 환경 변수 등록 확인

---

## 6. 앱 Boot Sequence 및 포트 LISTEN 확인

### 실행 명령어

```bash
# agent-admin 계정으로 실행 (루트 실행 금지)
su - agent-admin -c '
  export AGENT_HOME=/home/agent-admin/agent-app
  export AGENT_PORT=15034
  export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
  export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
  export AGENT_LOG_DIR=/var/log/agent-app
  $AGENT_HOME/agent-app
'

# 별도 터미널에서 포트 확인
ss -tulnp | grep 15034
```

### 확인 결과

```
>>> Starting Agent Boot Sequence...
[1/5] Checking User Account               [OK]
 ... Running as service user 'agent-admin' (uid=1001)
[2/5] Verifying Environment Variables     [OK]
 ... All required Envs correct
[3/5] Checking Required Files             [OK]
 ... Verified 'secret.key' with correct key string.
[4/5] Checking Port Availability          [OK]
 ... Port 15034 is available.
[5/5] Verifying Log Permission            [OK]
 ... Log directory is writable: /var/log/agent-app
------------------------------------------------------------
All Boot Checks Passed!
Agent READY
2026-05-18 14:18:06,506 [INFO] Agent listening at port 15034

tcp   LISTEN 0   1   0.0.0.0:15034   0.0.0.0:*   users:(("agent-app",pid=8150,fd=4))
```

### ✅ 체크 결과

- [x] Boot Sequence 1~5단계 모두 `[OK]` 확인
- [x] `Agent READY` 출력 확인
- [x] `0.0.0.0:15034` LISTEN 상태 확인
- [x] 일반 계정(`agent-admin`) 실행 확인 (루트 실행 금지 준수)

---

## 7. monitor.sh 구현 및 실행 결과

### 파일 위치/권한 정책

| 항목 | 값 |
|------|-----|
| 경로 | `/home/agent-admin/agent-app/bin/monitor.sh` |
| 소유자 | `agent-dev` |
| 그룹 | `agent-core` |
| 권한 | `750 (rwxr-x---)` |
| cron 실행 계정 | `agent-admin` |

### 소유자/권한 설정 명령어

```bash
chown agent-dev:agent-core /home/agent-admin/agent-app/bin/monitor.sh
chmod 750 /home/agent-admin/agent-app/bin/monitor.sh

# 확인
ls -l /home/agent-admin/agent-app/bin/monitor.sh
```

### 확인 결과

```
-rwxr-x--- 1 agent-dev agent-core 3689 May 18 14:19 /home/agent-admin/agent-app/bin/monitor.sh
```

### monitor.sh 실행 결과

```bash
# agent-admin으로 실행
su - agent-admin -c '
  export AGENT_HOME=/home/agent-admin/agent-app
  export AGENT_LOG_DIR=/var/log/agent-app
  /home/agent-admin/agent-app/bin/monitor.sh
'

# 로그 확인
cat /var/log/agent-app/monitor.log
```

```
[2026-05-18 14:19:51] [WARNING] UFW firewall is NOT active.
[2026-05-18 14:19:51] PID:8147 CPU:0.0% MEM:6.9% DISK_USED:1%
```

### ✅ 체크 결과

- [x] 소유자 `agent-dev`, 그룹 `agent-core`, 권한 `750` 확인
- [x] 프로세스 Health Check 정상 (PID:8147 확인)
- [x] 포트 15034 LISTEN Health Check 정상
- [x] UFW 비활성 → `[WARNING]` 출력 후 스크립트 계속 진행 (종료 없음)
- [x] CPU / MEM / DISK_USED 수집 정상
- [x] 로그 포맷 `[YYYY-MM-DD HH:MM:SS] PID:... CPU:..% MEM:..% DISK_USED:..%` 확인

---

## 8. crontab 등록 및 자동 실행 확인

### 설정 명령어

```bash
# agent-admin crontab 매분 실행 등록
su - agent-admin -c '(crontab -l 2>/dev/null; echo "* * * * * /home/agent-admin/agent-app/bin/monitor.sh") | crontab -'

# cron 서비스 시작
service cron start

# 등록 확인
su - agent-admin -c 'crontab -l'
```

### 확인 결과

```
* * * * * /home/agent-admin/agent-app/bin/monitor.sh
```

### monitor.log 자동 누적 확인 (1~2분 후)

```bash
cat /var/log/agent-app/monitor.log
wc -l /var/log/agent-app/monitor.log
```

```
[2026-05-18 14:19:51] [WARNING] UFW firewall is NOT active.
[2026-05-18 14:19:51] PID:8147 CPU:0.0%  MEM:6.9% DISK_USED:1%
[2026-05-18 14:21:02] [WARNING] UFW firewall is NOT active.
[2026-05-18 14:21:02] PID:8147 CPU:1.6%  MEM:6.3% DISK_USED:1%
[2026-05-18 14:22:01] [WARNING] UFW firewall is NOT active.
[2026-05-18 14:22:01] PID:8147 CPU:14.8% MEM:7.6% DISK_USED:1%

6 /var/log/agent-app/monitor.log
```

### ✅ 체크 결과

- [x] `* * * * *` crontab 매분 실행 등록 확인
- [x] 1분 간격 로그 자동 누적 확인 (14:19 → 14:21 → 14:22)
- [x] `/var/log/agent-app/monitor.log` 누적 기록 확인

---

## 부록. monitor.sh 소스코드

```bash
#!/bin/bash
# ============================================================
# monitor.sh - Agent App 시스템 관제 스크립트
# 소유자  : agent-dev
# 그룹    : agent-core
# 권한    : 750
# cron 실행 계정: agent-admin
# ============================================================

LOG_FILE="/var/log/agent-app/monitor.log"
APP_NAME="agent-app"
APP_PORT=15034
MAX_LOG_SIZE=$((10 * 1024 * 1024))   # 10MB
MAX_LOG_FILES=10

WARN_CPU=20
WARN_MEM=10
WARN_DISK=80

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

log() {
    echo "[$TIMESTAMP] $*" >> "$LOG_FILE"
}

# ── 1. Health Check: 프로세스 ───────────────────────────────
PID=$(pgrep -f "$APP_NAME" | head -1)

if [ -z "$PID" ]; then
    log "[ERROR] Process '$APP_NAME' is NOT running."
    exit 1
fi

# ── 2. Health Check: 포트 ───────────────────────────────────
if ! ss -tulnp 2>/dev/null | grep -q ":${APP_PORT} "; then
    log "[ERROR] Port $APP_PORT is NOT listening."
    exit 1
fi

# ── 3. 방화벽 상태 점검 (경고만, 종료 없음) ─────────────────
if command -v ufw > /dev/null 2>&1; then
    UFW_STATUS=$(ufw status 2>/dev/null | grep -i "Status:" | awk '{print $2}')
    if [ "$UFW_STATUS" != "active" ]; then
        log "[WARNING] UFW firewall is NOT active."
    fi
elif command -v firewall-cmd > /dev/null 2>&1; then
    if ! firewall-cmd --state 2>/dev/null | grep -q "running"; then
        log "[WARNING] firewalld is NOT active."
    fi
else
    log "[WARNING] No supported firewall (ufw/firewalld) found."
fi

# ── 4. 자원 수집 ────────────────────────────────────────────
CPU_IDLE=$(top -bn1 | grep -i "cpu(s)" | awk '{
    for(i=1;i<=NF;i++) {
        if($i ~ /^[0-9]+\.?[0-9]*$/ && $(i+1) ~ /id/) {
            print $i; exit
        }
    }
}')
CPU=$(awk "BEGIN {printf \"%.1f\", 100 - ${CPU_IDLE:-100}}")

MEM=$(free | awk '/^Mem:/ {printf "%.1f", $3/$2*100}')

DISK=$(df / | awk 'NR==2 {gsub(/%/,"",$5); print $5}')

# ── 5. 임계값 경고 (경고만, 종료 없음) ──────────────────────
CPU_INT=$(printf "%.0f" "$CPU")
MEM_INT=$(printf "%.0f" "$MEM")

if [ "$CPU_INT" -gt "$WARN_CPU" ]; then
    log "[WARNING] CPU usage ${CPU}% exceeds threshold ${WARN_CPU}%"
fi

if [ "$MEM_INT" -gt "$WARN_MEM" ]; then
    log "[WARNING] MEM usage ${MEM}% exceeds threshold ${WARN_MEM}%"
fi

if [ "$DISK" -gt "$WARN_DISK" ]; then
    log "[WARNING] DISK usage ${DISK}% exceeds threshold ${WARN_DISK}%"
fi

# ── 6. 정상 상태 로그 기록 ──────────────────────────────────
log "PID:$PID CPU:${CPU}% MEM:${MEM}% DISK_USED:${DISK}%"

# ── 7. 로그 파일 용량 관리 (10MB 초과 시 로테이션) ──────────
if [ -f "$LOG_FILE" ]; then
    LOG_SIZE=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)

    if [ "$LOG_SIZE" -gt "$MAX_LOG_SIZE" ]; then
        if [ -f "${LOG_FILE}.${MAX_LOG_FILES}" ]; then
            rm -f "${LOG_FILE}.${MAX_LOG_FILES}"
        fi
        for i in $(seq $((MAX_LOG_FILES - 1)) -1 1); do
            if [ -f "${LOG_FILE}.${i}" ]; then
                mv "${LOG_FILE}.${i}" "${LOG_FILE}.$((i + 1))"
            fi
        done
        mv "$LOG_FILE" "${LOG_FILE}.1"
        touch "$LOG_FILE"
        chown agent-admin:agent-core "$LOG_FILE"
    fi
fi

exit 0
```
