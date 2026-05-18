# System-Automation-Script-dev

# 요구사항 수행 내역서

> **과제**: Linux 기반 Agent App 운영 환경 구성 및 시스템 관제 자동화  
> **환경**: Ubuntu 22.04 LTS (Docker Container, `--privileged`)  
> **작성일**: 2026-05-18

---

## ✅ 필수 증거 자료 체크리스트

| # | 항목 | 상태 |
|---|------|------|
| 1 | SSH 포트 변경(20022) 및 Root 원격 접속 차단 | ✅ 완료 |
| 2 | 방화벽(UFW) 활성화 및 20022/tcp, 15034/tcp만 허용 | ✅ 완료 |
| 3 | 계정/그룹(agent-admin/dev/test, agent-common/core) 생성 | 🔲 진행 예정 |
| 4 | 디렉토리 구조 및 권한(ACL 포함) | 🔲 진행 예정 |
| 5 | 앱 Boot Sequence 5단계 [OK] 및 "Agent READY" | 🔲 진행 예정 |
| 6 | monitor.sh 실행 결과(프로세스/포트/리소스/경고) | 🔲 진행 예정 |
| 7 | /var/log/agent-app/monitor.log 누적 기록 확인 | 🔲 진행 예정 |
| 8 | crontab 매분 실행 등록 및 자동 실행 확인 | 🔲 진행 예정 |

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
tcp   LISTEN 0   128   0.0.0.0:20022   0.0.0.0:*   users:(("sshd",pid=8099,fd=3))
tcp   LISTEN 0   128      [::]:20022      [::]:*   users:(("sshd",pid=8099,fd=4))
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

To                         Action      From
--                         ------      ----
20022/tcp                  ALLOW       Anywhere
15034/tcp                  ALLOW       Anywhere
20022/tcp (v6)             ALLOW       Anywhere (v6)
15034/tcp (v6)             ALLOW       Anywhere (v6)
```

### ✅ 체크 결과

- [x] UFW `Status: active` 확인
- [x] `20022/tcp` ALLOW 확인 (SSH)
- [x] `15034/tcp` ALLOW 확인 (APP)
- [x] IPv6 규칙 동시 적용 확인

---

## 3. 계정/그룹 생성

> 🔲 진행 예정

---

## 4. 디렉토리 구조 및 권한(ACL 포함)

> 🔲 진행 예정

---

## 5. 앱 Boot Sequence

> 🔲 진행 예정

---

## 6. monitor.sh 실행 결과

> 🔲 진행 예정

---

## 7. monitor.log 누적 기록 확인

> 🔲 진행 예정

---

## 8. crontab 등록 및 자동 실행 확인

> 🔲 진행 예정

---

## 부록. 자동화 스크립트 소스코드

> 🔲 monitor.sh 완성 후 추가 예정
