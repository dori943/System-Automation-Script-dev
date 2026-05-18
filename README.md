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

### SSH 포트 변경과 Root 원격 접속 차단이 왜 기본 보안에 해당하는가?
SSH 기본 포트 22는 해커들이 자동화 봇으로 24시간 공격하는 대상임. 따라서, 2만번대의 비표준 포트로 바꾸는 것만으로도 대부분 차단 가능. 또한 Root 게정은 시스템 전체 권한을 가지기 때문에 원격에서 직접 접근 가능하면 탈취 시 서버 전체가 위험함.

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

### UFW로 "필요 포트만 허용"하는 방화벽 정책

방화벽 없이 서브를 운영하면 모든 포트가 외부에 노출됨. 서비스에 필요한 포트만 열고 나머지는 전부 차단하는 것이 "최소 권한 원칙"의 테크워크 버전임.
따라서, 들어오는 건 전부 차단하고 필요한 포트만 명시적으로 허용함.


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

### 역할 기반 계정/그룹과 ACL로 공유/보안 디렉토리를 분리하는 이유

모든 사람이 모든 파일에 접근 가능하면 실수나 악의적 행위로 중요 파일이 유출될 수 있음.
setfacl이라는 명령어로 각 계정이 어디 디렉토리에 rwx를 가능하게 할 건지 설정 가능.

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

### 환경 변수로 실행 환경을 고정하는 이유와 검증 방법

경로를 코드나 스크립트에 하드코딩하면 서버 환경이 바뀔때마다 코드를 수정해야함.
환경 변수로 경로를 고정하면 환경이 바뀌어도 변수 값만 수정하면 되고, 실수로 잘못된 경로를 참조하는 것도 방지할 수 있음.

.bashrc → 사람이 로그인할 때 사용
/etc/environment → cron처럼 로그인 없이 실행될 때도 변수 참조 가능
환경 변수가 없으면 앱이 아예 실행 거부 → Boot Sequence [2/5]에서 확인됨

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

### 쉘 스크립트로 프로세스/포트/리소스를 수집하고 로그로 남기는 흐름

서버 문제는 "언제 어떤 상태였는지" 몰르면 원인을 찾기 어려움. 따라서 주기적으로 상태를 수집해서 로그로 남기면 문제 발생 전후로 추적이 가능.
Health Check 실패 → exit 1 로 즉시 중단, 이후 로직 실행 안 함
WARNING은 종료 없이 기록만 → 운영자가 나중에 확인 가능
타임스탬프 포함 로그 → 언제 문제가 생겼는지 정확히 추적 가능

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

### crontab 주기 실행과 로그 보존 정책이 왜 필요한가

모니터링은 사람이 수동으로 실행하면 의미가 없음. 자동으로 주기 실행해야 문제를 놓치지 않기 때문. 또한 로그를 무한개로 쌓으면 디스크 한계때문에 서버 멈출수도 있으므로 로그 보존 정책으로 디스크를 관리해야함.

```
crontab 표현식 의미:
* * * * * 명령어
│ │ │ │ │
│ │ │ │ └── 요일 (0-7)
│ │ │ └──── 월 (1-12)
│ │ └────── 일 (1-31)
│ └──────── 시 (0-23)
└────────── 분 (0-59)
* = 매번 → 매분 실행
```

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
