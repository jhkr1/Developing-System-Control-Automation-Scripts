# Linux Server Operation Mission Book

> 이 문서는 서버 운영 미션을 처음부터 끝까지 수행하기 위한 실습형 안내서다.  
> 목표는 단순히 명령어를 따라 치는 것이 아니라, “왜 이 설정이 필요한지” 이해하고, 최종 제출물인 수행 내역서와 자동화 스크립트를 스스로 완성하는 것이다.

---

## 0장. 미션 전체 그림

서버 장애가 발생했을 때 로그가 없다면 원인 분석은 감에 의존하게 된다.  
현업에서는 이런 상황이 복구 시간을 늘리고, 같은 장애를 반복시키는 직접적인 원인이 된다.

이번 미션에서는 다음 흐름을 직접 구성한다.

1. SSH 기본 보안 설정
2. 필요한 포트만 허용하는 방화벽 정책
3. 역할 기반 계정, 그룹, 디렉토리 권한 설계
4. 제공 애플리케이션 실행 환경 구성
5. 시스템 상태를 수집하고 로그로 남기는 Bash 자동화
6. cron을 이용한 주기 실행
7. 로그 보존 정책 설계

최종 산출물은 다음 2개다.

- `README.md` 또는 별도 문서 형태의 요구사항 수행 내역서
- `$AGENT_HOME/bin/monitor.sh` 자동화 스크립트

---

## 1장. 실습 환경과 주의사항

### 권장 환경

- Ubuntu 22.04 LTS 또는 동등한 Linux 환경
- VM 또는 컨테이너 실습 환경 권장
- sudo 권한이 있는 일반 사용자

### 내 공부용 컴퓨터에서 바로 하면 안 되는 이유

이 미션은 SSH 포트, 방화벽, 사용자 계정, 시스템 디렉토리, cron 같은 운영체제 설정을 직접 바꾼다.  
따라서 macOS나 Windows 같은 내 실제 공부용 컴퓨터에서 그대로 실행하는 것이 아니라, 별도의 Linux 실습 환경에서 진행해야 한다.

특히 다음 명령들은 실습 Linux 안에서 실행되어야 한다.

```bash
sudo useradd ...
sudo groupadd ...
sudo ufw ...
sudo systemctl ...
sudo crontab ...
```

여기서 `sudo`는 “내 맥북 관리자 권한”이 아니라 “실습 Linux 안의 관리자 권한”을 뜻한다.  
즉, 내 공부용 컴퓨터에서 `sudo`가 안 되더라도, 별도로 만든 Ubuntu VM이나 Linux 컨테이너 안에서는 root 또는 sudo 권한을 사용할 수 있어야 한다.

### OrbStack을 사용해도 되는가

결론부터 말하면, OrbStack의 Linux 환경은 이 미션의 일부를 연습하기에 좋다.  
하지만 전체 요구사항을 가장 깔끔하게 만족하려면 Ubuntu VM을 더 권장한다.

OrbStack으로 연습하기 좋은 부분:

- Bash 명령어 연습
- 계정과 그룹 생성
- 디렉토리 권한 설정
- ACL 확인
- 제공 앱 실행
- `monitor.sh` 작성과 실행
- `monitor.log` 누적 기록
- cron 연습

OrbStack에서 애매할 수 있는 부분:

- SSH 서버 자체를 운영하는 실습
- SSH 포트를 `20022`로 바꾸고 외부 접속 검증
- UFW/firewalld로 실제 서버 방화벽 정책을 검증
- `systemctl`로 서비스를 제어하는 과정

컨테이너 계열 Linux는 호스트와 커널 또는 네트워크 계층을 공유하는 경우가 많다.  
그래서 `ufw status`가 실행되더라도, 실제 독립 서버의 방화벽처럼 의미 있게 동작하지 않을 수 있다.

제출 증거까지 안정적으로 만들려면 다음 순서를 추천한다.

1. 가능하면 Ubuntu 22.04 VM에서 전체 미션 수행
2. VM이 어렵다면 OrbStack에서 앱/권한/스크립트/cron 위주로 수행
3. SSH와 방화벽 항목은 실습 환경 한계를 수행 내역서에 명확히 기록

### 가장 추천하는 실습 환경

Mac을 사용한다면 다음 중 하나가 좋다.

| 환경 | 추천도 | 이유 |
|---|---:|---|
| Ubuntu 22.04 VM | 높음 | SSH, 방화벽, systemd, cron 검증이 가장 자연스럽다 |
| OrbStack Linux | 중간 | 빠르고 편하지만 방화벽/SSH 검증이 제한될 수 있다 |
| 로컬 macOS 터미널 | 낮음 | Linux 미션 요구사항과 다르고 시스템 변경 위험이 있다 |

이 문서의 명령어는 Ubuntu 22.04 VM 기준으로 작성한다.  
OrbStack을 사용할 때는 SSH와 방화벽 부분에서 환경 차이가 있을 수 있음을 기억한다.

### 매우 중요한 주의사항

SSH 포트를 바꾸거나 방화벽을 켜면 원격 접속이 끊길 수 있다.  
반드시 다음 중 하나를 확보한 뒤 진행한다.

- VM 콘솔 접근
- 클라우드 콘솔 접근
- 현재 SSH 세션을 유지한 상태에서 새 포트 접속 테스트

실습에서는 SSH 포트를 `20022`, 앱 포트를 `15034`로 사용한다.

### 제공 파일 확인

현재 미션 폴더의 `agent-app.zip` 안에는 Linux 실행 파일 `agent-app` 하나가 들어 있다.

```bash
unzip -l agent-app.zip
file agent-app
```

확인된 구조:

```text
agent-app.zip
└── agent-app
```

macOS나 Windows 호스트에서는 이 파일을 직접 실행할 수 없을 수 있다.  
앱 실행 검증은 Ubuntu 22.04 LTS 또는 동등한 Linux 실습 환경에서 수행한다.

### sudo가 안 될 때 판단 기준

다음 명령으로 현재 계정이 sudo를 사용할 수 있는지 확인한다.

```bash
whoami
id
sudo -v
```

`sudo -v`가 실패한다면 선택지는 2가지다.

첫 번째는 root 계정으로 실습하는 방법이다.  
컨테이너나 VM에서 `whoami` 결과가 `root`라면, 명령어 앞의 `sudo`를 빼고 실행하면 된다.

예:

```bash
groupadd agent-common
useradd -m -s /bin/bash agent-admin
mkdir -p /var/log/agent-app
```

두 번째는 sudo 가능한 일반 계정을 만드는 방법이다.  
Ubuntu VM에서 root 접근이 가능하다면 다음처럼 만들 수 있다.

```bash
adduser student
usermod -aG sudo student
```

그 뒤 `student` 계정으로 로그인해서 이 문서의 `sudo` 명령을 실행한다.

정리하면 다음과 같다.

| 현재 상태 | 진행 방법 |
|---|---|
| 일반 계정이고 sudo 가능 | 문서 명령어 그대로 실행 |
| root 계정 | `sudo`를 빼고 실행 |
| 일반 계정이고 sudo 불가 | root 접근 가능한 VM/컨테이너를 새로 준비 |

---

## 2장. 제출 체크리스트

아래 항목을 수행 내역서에 증거와 함께 기록한다.

- SSH 포트 `20022` 변경 확인
- Root 원격 접속 차단 확인
- 방화벽 활성화 확인
- TCP `20022`, TCP `15034`만 허용 확인
- `agent-admin`, `agent-dev`, `agent-test` 계정 생성 확인
- `agent-common`, `agent-core` 그룹 생성 확인
- `$AGENT_HOME`, `upload_files`, `api_keys`, `/var/log/agent-app` 권한 확인
- 앱 Boot Sequence 5단계 `[OK]` 및 `Agent READY` 확인
- 앱이 `0.0.0.0:15034`에서 LISTEN 중임을 확인
- `monitor.sh` 실행 결과 확인
- `/var/log/agent-app/monitor.log` 누적 확인
- `agent-admin` crontab 매분 실행 등록 확인
- 1분 뒤 로그 라인 증가 확인

---

## 3장. SSH 기본 보안 설정

### 3.1 SSH 설정 파일 백업

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

### 3.2 SSH 포트와 Root 로그인 설정

`/etc/ssh/sshd_config` 파일을 연다.

```bash
sudo vi /etc/ssh/sshd_config
```

다음 설정을 추가하거나 수정한다.

```text
Port 20022
PermitRootLogin no
```

### 3.3 SSH 설정 문법 검사

```bash
sudo sshd -t
```

아무 출력이 없으면 설정 문법이 정상이다.

### 3.4 SSH 서비스 재시작

Ubuntu 22.04에서는 보통 다음 명령을 사용한다.

```bash
sudo systemctl restart ssh
```

환경에 따라 서비스명이 `sshd`일 수도 있다.

```bash
sudo systemctl restart sshd
```

### 3.5 확인 명령

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
ss -tulnp | grep ':20022'
```

기대 결과:

```text
port 20022
permitrootlogin no
```

---

## 4장. 방화벽 설정

이 문서는 Ubuntu 기본 환경을 기준으로 UFW를 사용한다.  
방화벽 정책은 “필요한 포트만 허용”하는 것이 핵심이다.

### 4.1 UFW 설치 확인

```bash
sudo apt update
sudo apt install -y ufw
```

### 4.2 기본 정책 설정

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 4.3 필요한 포트만 허용

```bash
sudo ufw allow 20022/tcp
sudo ufw allow 15034/tcp
```

### 4.4 UFW 활성화

```bash
sudo ufw enable
```

### 4.5 확인 명령

```bash
sudo ufw status verbose
```

기대 결과에는 다음 항목이 포함되어야 한다.

```text
Status: active
20022/tcp ALLOW IN Anywhere
15034/tcp ALLOW IN Anywhere
```

---

## 5장. 계정, 그룹, 권한 설계

### 5.1 계정과 그룹의 의미

이번 미션에서는 역할을 다음처럼 나눈다.

| 계정 | 역할 |
|---|---|
| `agent-admin` | 운영/관리, cron 실행 |
| `agent-dev` | 개발/운영, `monitor.sh` 작성 |
| `agent-test` | QA/테스트 |

| 그룹 | 포함 계정 | 목적 |
|---|---|---|
| `agent-common` | admin, dev, test | 공유 업로드 디렉토리 접근 |
| `agent-core` | admin, dev | API 키와 로그 접근 |

핵심은 `agent-test`가 공유 파일에는 접근할 수 있지만, API 키와 운영 로그에는 접근하지 못하게 하는 것이다.

### 5.2 그룹 생성

```bash
sudo groupadd agent-common
sudo groupadd agent-core
```

이미 존재한다면 오류가 날 수 있다. 이 경우 `getent group`으로 확인한다.

```bash
getent group agent-common
getent group agent-core
```

### 5.3 계정 생성

```bash
sudo useradd -m -s /bin/bash agent-admin
sudo useradd -m -s /bin/bash agent-dev
sudo useradd -m -s /bin/bash agent-test
```

필요하면 비밀번호를 설정한다.

```bash
sudo passwd agent-admin
sudo passwd agent-dev
sudo passwd agent-test
```

### 5.4 그룹에 계정 추가

```bash
sudo usermod -aG agent-common,agent-core agent-admin
sudo usermod -aG agent-common,agent-core agent-dev
sudo usermod -aG agent-common agent-test
```

### 5.5 확인 명령

```bash
id agent-admin
id agent-dev
id agent-test
```

기대 관계:

- `agent-admin`: `agent-common`, `agent-core`
- `agent-dev`: `agent-common`, `agent-core`
- `agent-test`: `agent-common`

---

## 6장. 디렉토리 구조와 권한

### 6.1 환경 변수 기준 경로

이번 문서에서는 다음 경로를 기준으로 한다.

```bash
export AGENT_HOME=/home/agent-admin/agent-app
```

### 6.2 디렉토리 생성

```bash
sudo mkdir -p /home/agent-admin/agent-app/upload_files
sudo mkdir -p /home/agent-admin/agent-app/api_keys
sudo mkdir -p /home/agent-admin/agent-app/bin
sudo mkdir -p /var/log/agent-app
```

### 6.3 소유자와 그룹 설정

```bash
sudo chown -R agent-admin:agent-common /home/agent-admin/agent-app
sudo chown agent-admin:agent-common /home/agent-admin/agent-app/upload_files
sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys
sudo chown agent-admin:agent-core /var/log/agent-app
```

### 6.4 기본 권한 설정

```bash
sudo chmod 750 /home/agent-admin/agent-app
sudo chmod 770 /home/agent-admin/agent-app/upload_files
sudo chmod 770 /home/agent-admin/agent-app/api_keys
sudo chmod 770 /var/log/agent-app
```

### 6.5 ACL을 이용한 기본 권한 유지

새로 생성되는 파일에도 그룹 권한이 유지되도록 기본 ACL을 설정한다.

```bash
sudo setfacl -m g:agent-common:rwx /home/agent-admin/agent-app/upload_files
sudo setfacl -d -m g:agent-common:rwx /home/agent-admin/agent-app/upload_files

sudo setfacl -m g:agent-core:rwx /home/agent-admin/agent-app/api_keys
sudo setfacl -d -m g:agent-core:rwx /home/agent-admin/agent-app/api_keys

sudo setfacl -m g:agent-core:rwx /var/log/agent-app
sudo setfacl -d -m g:agent-core:rwx /var/log/agent-app
```

### 6.6 확인 명령

```bash
ls -ld /home/agent-admin/agent-app
ls -ld /home/agent-admin/agent-app/upload_files
ls -ld /home/agent-admin/agent-app/api_keys
ls -ld /var/log/agent-app

getfacl /home/agent-admin/agent-app/upload_files
getfacl /home/agent-admin/agent-app/api_keys
getfacl /var/log/agent-app
```

---

## 7장. 애플리케이션 실행 환경 구성

### 7.1 환경 변수 설정

`agent-admin` 계정의 `~/.bashrc` 또는 앱 실행 스크립트에 다음 값을 설정한다.

```bash
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app
```

현재 쉘에 즉시 적용하려면 다음을 실행한다.

```bash
source ~/.bashrc
```

### 7.2 키 파일 생성

```bash
sudo -u agent-admin bash -c 'echo agent_api_key_test > /home/agent-admin/agent-app/api_keys/t_secret.key'
sudo chown agent-admin:agent-core /home/agent-admin/agent-app/api_keys/t_secret.key
sudo chmod 660 /home/agent-admin/agent-app/api_keys/t_secret.key
```

### 7.3 제공 앱 배치

현재 제공 파일은 `agent-app.zip`이며, 압축을 풀면 Linux 실행 파일 `agent-app`이 나온다.

```bash
unzip agent-app.zip
```

실습 서버의 현재 작업 디렉토리에서 앱을 `$AGENT_HOME`으로 복사한다.

```bash
sudo cp agent-app /home/agent-admin/agent-app/
sudo chown agent-admin:agent-core /home/agent-admin/agent-app/agent-app
sudo chmod 750 /home/agent-admin/agent-app/agent-app
```

확인:

```bash
ls -l /home/agent-admin/agent-app/agent-app
file /home/agent-admin/agent-app/agent-app
```

### 7.4 앱 실행

루트가 아닌 `agent-admin` 계정으로 실행한다.

```bash
sudo -iu agent-admin
export AGENT_HOME=/home/agent-admin/agent-app
export AGENT_PORT=15034
export AGENT_UPLOAD_DIR=$AGENT_HOME/upload_files
export AGENT_KEY_PATH=$AGENT_HOME/api_keys/t_secret.key
export AGENT_LOG_DIR=/var/log/agent-app
$AGENT_HOME/agent-app
```

성공 기준:

```text
[1/5] Checking User Account               [OK]
[2/5] Verifying Environment Variables     [OK]
[3/5] Checking Required Files             [OK]
[4/5] Checking Port Availability          [OK]
[5/5] Verifying Log Permission            [OK]
Agent READY
```

### 7.5 포트 확인

다른 터미널에서 확인한다.

```bash
ss -tulnp | grep ':15034'
```

기대 결과:

```text
LISTEN ... 0.0.0.0:15034 ...
```

---

## 8장. monitor.sh 설계

### 8.1 스크립트 요구사항 정리

`monitor.sh`는 다음 일을 수행해야 한다.

- 앱 프로세스 실행 여부 확인
- TCP `15034` LISTEN 여부 확인
- 방화벽 활성화 여부 확인
- CPU, 메모리, 루트 디스크 사용률 수집
- 임계값 초과 시 경고 출력
- `/var/log/agent-app/monitor.log`에 한 줄씩 기록
- `monitor.log`가 10MB를 넘으면 최대 10개까지 회전 보관

Health Check 실패 조건:

- 앱 프로세스가 없으면 `exit 1`
- 포트가 LISTEN 상태가 아니면 `exit 1`

경고만 출력하는 조건:

- 방화벽 비활성
- CPU 사용률 `20%` 초과
- 메모리 사용률 `10%` 초과
- 루트 디스크 사용률 `80%` 초과

### 8.2 monitor.sh 예시 코드

파일 경로:

```text
/home/agent-admin/agent-app/bin/monitor.sh
```

```bash
#!/usr/bin/env bash
set -u

APP_PATTERN="${APP_PATTERN:-agent-app}"
APP_PORT="${AGENT_PORT:-15034}"
LOG_FILE="${AGENT_LOG_DIR:-/var/log/agent-app}/monitor.log"
MAX_LOG_SIZE=$((10 * 1024 * 1024))
MAX_ROTATED_FILES=10

CPU_THRESHOLD="${CPU_THRESHOLD:-20}"
MEM_THRESHOLD="${MEM_THRESHOLD:-10}"
DISK_THRESHOLD="${DISK_THRESHOLD:-80}"

print_header() {
  printf '\n====== SYSTEM MONITOR RESULT ======\n\n'
}

rotate_log_if_needed() {
  local log_file="$1"
  local log_dir
  local log_size
  log_dir="$(dirname "$log_file")"

  mkdir -p "$log_dir"
  touch "$log_file"

  log_size="$(stat -c '%s' "$log_file" 2>/dev/null || echo 0)"
  if [ "$log_size" -lt "$MAX_LOG_SIZE" ]; then
    return 0
  fi

  local i
  for ((i = MAX_ROTATED_FILES - 1; i >= 1; i--)); do
    if [ -f "${log_file}.${i}" ]; then
      mv "${log_file}.${i}" "${log_file}.$((i + 1))"
    fi
  done

  mv "$log_file" "${log_file}.1"
  : > "$log_file"

  if [ -f "${log_file}.$((MAX_ROTATED_FILES + 1))" ]; then
    rm -f "${log_file}.$((MAX_ROTATED_FILES + 1))"
  fi
}

find_app_pid() {
  pgrep -f "$APP_PATTERN" | head -n 1
}

check_process() {
  local pid="$1"
  printf "Checking process '%s'... " "$APP_PATTERN"

  if [ -z "$pid" ]; then
    printf '[FAIL]\n'
    return 1
  fi

  printf '[OK] (PID: %s)\n' "$pid"
}

check_port() {
  printf 'Checking port %s... ' "$APP_PORT"

  if ss -tuln | awk '{print $5}' | grep -Eq "(:|\\.)${APP_PORT}$"; then
    printf '[OK]\n'
    return 0
  fi

  printf '[FAIL]\n'
  return 1
}

check_firewall() {
  if command -v ufw >/dev/null 2>&1; then
    if ufw status 2>/dev/null | grep -q 'Status: active'; then
      printf 'Firewall status... [OK] UFW active\n'
    else
      printf '[WARNING] Firewall is not active\n'
    fi
    return 0
  fi

  if command -v firewall-cmd >/dev/null 2>&1; then
    if firewall-cmd --state 2>/dev/null | grep -q 'running'; then
      printf 'Firewall status... [OK] firewalld running\n'
    else
      printf '[WARNING] Firewall is not active\n'
    fi
    return 0
  fi

  printf '[WARNING] No supported firewall command found\n'
}

get_cpu_usage() {
  top -bn1 | awk -F'id,' '/Cpu\(s\)|%Cpu/ {
    split($1, parts, ",")
    idle=parts[length(parts)]
    gsub(/[^0-9.]/, "", idle)
    if (idle != "") {
      printf "%.1f", 100 - idle
      exit
    }
  }'
}

get_mem_usage() {
  free | awk '/Mem:/ { printf "%.1f", ($3 / $2) * 100 }'
}

get_disk_usage() {
  df / | awk 'NR==2 { gsub(/%/, "", $5); print $5 }'
}

warn_if_exceeded() {
  local name="$1"
  local value="$2"
  local threshold="$3"
  local unit="$4"

  if awk "BEGIN { exit !($value > $threshold) }"; then
    printf '[WARNING] %s threshold exceeded (%s%s > %s%s)\n' "$name" "$value" "$unit" "$threshold" "$unit"
  fi
}

main() {
  local pid
  local cpu_usage
  local mem_usage
  local disk_usage
  local timestamp

  print_header

  pid="$(find_app_pid)"

  printf '[HEALTH CHECK]\n'
  check_process "$pid" || exit 1
  check_port || exit 1
  check_firewall

  cpu_usage="$(get_cpu_usage)"
  mem_usage="$(get_mem_usage)"
  disk_usage="$(get_disk_usage)"

  printf '\n[RESOURCE MONITORING]\n'
  printf 'CPU Usage : %s%%\n' "$cpu_usage"
  printf 'MEM Usage : %s%%\n' "$mem_usage"
  printf 'DISK Used : %s%%\n\n' "$disk_usage"

  warn_if_exceeded CPU "$cpu_usage" "$CPU_THRESHOLD" "%"
  warn_if_exceeded MEM "$mem_usage" "$MEM_THRESHOLD" "%"
  warn_if_exceeded DISK_USED "$disk_usage" "$DISK_THRESHOLD" "%"

  rotate_log_if_needed "$LOG_FILE"

  timestamp="$(date '+%Y-%m-%d %H:%M:%S')"
  printf '[%s] PID:%s CPU:%s%% MEM:%s%% DISK_USED:%s%%\n' \
    "$timestamp" "$pid" "$cpu_usage" "$mem_usage" "$disk_usage" >> "$LOG_FILE"

  printf '\n[INFO] Log appended: %s\n' "$LOG_FILE"
}

main "$@"
```

### 8.3 파일 생성과 권한 설정

```bash
sudo vi /home/agent-admin/agent-app/bin/monitor.sh
sudo chown agent-dev:agent-core /home/agent-admin/agent-app/bin/monitor.sh
sudo chmod 750 /home/agent-admin/agent-app/bin/monitor.sh
```

확인:

```bash
ls -l /home/agent-admin/agent-app/bin/monitor.sh
```

기대 결과:

```text
-rwxr-x--- 1 agent-dev agent-core ... monitor.sh
```

---

## 9장. monitor.sh 실행과 검증

### 9.1 직접 실행

`agent-admin`은 `agent-core` 그룹에 포함되어 있으므로 실행 가능해야 한다.

```bash
sudo -iu agent-admin
/home/agent-admin/agent-app/bin/monitor.sh
```

기대 출력:

```text
====== SYSTEM MONITOR RESULT ======

[HEALTH CHECK]
Checking process 'agent-app'... [OK] (PID: 48291)
Checking port 15034... [OK]
Firewall status... [OK] UFW active

[RESOURCE MONITORING]
CPU Usage : 25.3%
MEM Usage : 5.2%
DISK Used : 23%

[WARNING] CPU threshold exceeded (25.3% > 20%)

[INFO] Log appended: /var/log/agent-app/monitor.log
```

### 9.2 로그 확인

```bash
tail -n 5 /var/log/agent-app/monitor.log
```

기대 결과:

```text
[2026-02-25 13:58:01] PID:48291 CPU:10.2% MEM:3.2% DISK_USED:23%
[2026-02-25 13:59:01] PID:48291 CPU:18.7% MEM:5.0% DISK_USED:23%
```

---

## 10장. cron 자동 실행

### 10.1 agent-admin crontab 편집

```bash
sudo crontab -u agent-admin -e
```

다음 라인을 추가한다.

```cron
* * * * * /home/agent-admin/agent-app/bin/monitor.sh >> /var/log/agent-app/monitor-cron.out 2>&1
```

### 10.2 등록 확인

```bash
sudo crontab -u agent-admin -l
```

### 10.3 자동 실행 확인

현재 로그 라인 수를 확인한다.

```bash
wc -l /var/log/agent-app/monitor.log
```

1분 이상 기다린 뒤 다시 확인한다.

```bash
sleep 70
wc -l /var/log/agent-app/monitor.log
tail -n 5 /var/log/agent-app/monitor.log
```

라인 수가 증가했다면 cron 자동 실행이 정상이다.

---

## 11장. 제출용 수행 내역서 작성법

수행 내역서는 “무엇을 했는지”보다 “검증했다는 증거”가 중요하다.  
아래 형식으로 작성하면 채점자가 빠르게 확인할 수 있다.

### 11.1 기본 정보

```text
실습 환경: Ubuntu 22.04 LTS
방화벽: UFW
SSH 포트: 20022
APP 포트: 15034
AGENT_HOME: /home/agent-admin/agent-app
```

### 11.2 SSH 설정 기록

기록할 명령:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
ss -tulnp | grep ':20022'
```

붙여 넣을 증거:

```text
port 20022
permitrootlogin no
```

### 11.3 방화벽 기록

기록할 명령:

```bash
sudo ufw status verbose
```

붙여 넣을 증거:

```text
Status: active
20022/tcp ALLOW IN Anywhere
15034/tcp ALLOW IN Anywhere
```

### 11.4 계정과 그룹 기록

기록할 명령:

```bash
id agent-admin
id agent-dev
id agent-test
```

### 11.5 디렉토리와 ACL 기록

기록할 명령:

```bash
ls -ld /home/agent-admin/agent-app /home/agent-admin/agent-app/upload_files /home/agent-admin/agent-app/api_keys /var/log/agent-app
getfacl /home/agent-admin/agent-app/upload_files
getfacl /home/agent-admin/agent-app/api_keys
getfacl /var/log/agent-app
```

### 11.6 앱 실행 기록

기록할 내용:

- Boot Sequence 5단계 `[OK]`
- `Agent READY`
- `ss -tulnp | grep ':15034'` 결과

### 11.7 monitor.sh 실행 기록

기록할 명령:

```bash
/home/agent-admin/agent-app/bin/monitor.sh
tail -n 5 /var/log/agent-app/monitor.log
```

### 11.8 cron 기록

기록할 명령:

```bash
sudo crontab -u agent-admin -l
wc -l /var/log/agent-app/monitor.log
sleep 70
wc -l /var/log/agent-app/monitor.log
tail -n 5 /var/log/agent-app/monitor.log
```

---

## 12장. 보너스 1: report.sh 요약 리포트

`monitor.log`를 분석해 CPU, MEM, DISK 사용률의 평균/최대/최소를 출력한다.

파일 경로 예시:

```text
/home/agent-admin/agent-app/bin/report.sh
```

```bash
#!/usr/bin/env bash
set -u

LOG_FILE="${1:-/var/log/agent-app/monitor.log}"

if [ ! -f "$LOG_FILE" ]; then
  echo "[ERROR] Log file not found: $LOG_FILE" >&2
  exit 1
fi

awk '
{
  timestamp = substr($0, 2, 19)

  for (i = 1; i <= NF; i++) {
    if ($i ~ /^CPU:/) {
      cpu = $i
      gsub(/^CPU:|%$/, "", cpu)
    }
    if ($i ~ /^MEM:/) {
      mem = $i
      gsub(/^MEM:|%$/, "", mem)
    }
    if ($i ~ /^DISK_USED:/) {
      disk = $i
      gsub(/^DISK_USED:|%$/, "", disk)
    }
  }

  count++

  cpu_sum += cpu
  mem_sum += mem
  disk_sum += disk

  if (count == 1 || cpu > cpu_max) { cpu_max = cpu; cpu_max_time = timestamp }
  if (count == 1 || cpu < cpu_min) { cpu_min = cpu; cpu_min_time = timestamp }

  if (count == 1 || mem > mem_max) { mem_max = mem; mem_max_time = timestamp }
  if (count == 1 || mem < mem_min) { mem_min = mem; mem_min_time = timestamp }

  if (count == 1 || disk > disk_max) { disk_max = disk; disk_max_time = timestamp }
  if (count == 1 || disk < disk_min) { disk_min = disk; disk_min_time = timestamp }
}
END {
  if (count == 0) {
    print "[WARNING] No samples found"
    exit 0
  }

  print "====== STATISTICS REPORT ======"
  print "[CPU]"
  printf "Average : %.1f%%\n", cpu_sum / count
  printf "Maximum : %.1f%% at %s\n", cpu_max, cpu_max_time
  printf "Minimum : %.1f%% at %s\n", cpu_min, cpu_min_time

  print "[Memory]"
  printf "Average : %.1f%%\n", mem_sum / count
  printf "Maximum : %.1f%% at %s\n", mem_max, mem_max_time
  printf "Minimum : %.1f%% at %s\n", mem_min, mem_min_time

  print "[Disk]"
  printf "Average : %.1f%%\n", disk_sum / count
  printf "Maximum : %.1f%% at %s\n", disk_max, disk_max_time
  printf "Minimum : %.1f%% at %s\n", disk_min, disk_min_time

  print "[Samples]"
  printf "Data Points: %d samples\n", count
}
' "$LOG_FILE"
```

권한:

```bash
sudo chown agent-dev:agent-core /home/agent-admin/agent-app/bin/report.sh
sudo chmod 750 /home/agent-admin/agent-app/bin/report.sh
```

---

## 13장. 보너스 2: 시간 기반 로그 보존 정책

### 13.1 정책

- 7일 이상 지난 `/var/log/agent-app/*.log` 파일 압축
- 압축 파일을 `/var/log/monitor/agent-app/archive/`로 이동
- 30일 이상 지난 `.gz` 아카이브 삭제

### 13.2 archive_logs.sh 예시

```bash
#!/usr/bin/env bash
set -u

SOURCE_DIR="/var/log/agent-app"
ARCHIVE_DIR="/var/log/monitor/agent-app/archive"

if [ ! -d "$SOURCE_DIR" ]; then
  echo "[WARNING] Source directory not found: $SOURCE_DIR"
  exit 0
fi

if ! mkdir -p "$ARCHIVE_DIR"; then
  echo "[ERROR] Cannot create archive directory: $ARCHIVE_DIR" >&2
  exit 1
fi

if [ ! -w "$ARCHIVE_DIR" ]; then
  echo "[ERROR] Archive directory is not writable: $ARCHIVE_DIR" >&2
  exit 1
fi

found_old_logs=0

while IFS= read -r -d '' log_file; do
  found_old_logs=1
  gzip -c "$log_file" > "$ARCHIVE_DIR/$(basename "$log_file").$(date '+%Y%m%d%H%M%S').gz"
  : > "$log_file"
  echo "[INFO] Archived and truncated: $log_file"
done < <(find "$SOURCE_DIR" -maxdepth 1 -type f -name '*.log' -mtime +7 -print0)

if [ "$found_old_logs" -eq 0 ]; then
  echo "[INFO] No log files older than 7 days"
fi

find "$ARCHIVE_DIR" -type f -name '*.gz' -mtime +30 -print -delete
```

---

## 14장. 장애 상황별 점검표

### 앱이 실행되지 않을 때

확인할 것:

```bash
id
echo "$AGENT_HOME"
echo "$AGENT_PORT"
echo "$AGENT_UPLOAD_DIR"
echo "$AGENT_KEY_PATH"
echo "$AGENT_LOG_DIR"
ls -l "$AGENT_KEY_PATH"
ls -ld "$AGENT_LOG_DIR"
```

자주 발생하는 원인:

- 루트 계정으로 앱 실행
- 환경 변수 누락
- 키 파일 내용 불일치
- 로그 디렉토리 쓰기 권한 없음
- 이미 `15034` 포트를 다른 프로세스가 사용 중

### monitor.sh가 실패할 때

확인할 것:

```bash
pgrep -af 'agent-app'
ss -tulnp | grep ':15034'
ls -l /home/agent-admin/agent-app/bin/monitor.sh
ls -ld /var/log/agent-app
```

자주 발생하는 원인:

- 앱 프로세스가 실행 중이 아님
- 앱이 `15034`로 LISTEN하지 않음
- `agent-admin`이 `agent-core` 그룹에 없음
- `/var/log/agent-app`에 쓰기 권한 없음

### cron이 실행되지 않을 때

확인할 것:

```bash
sudo crontab -u agent-admin -l
systemctl status cron
tail -n 20 /var/log/agent-app/monitor-cron.out
```

자주 발생하는 원인:

- cron 서비스가 중지됨
- crontab 경로 오타
- cron 환경에 환경 변수가 없음
- 스크립트 실행 권한 없음

---

## 15장. 최종 제출 전 점검

최종 제출 전에는 아래 순서로 한 번 더 확인한다.

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
sudo ufw status verbose
id agent-admin
id agent-dev
id agent-test
ls -ld /home/agent-admin/agent-app /home/agent-admin/agent-app/upload_files /home/agent-admin/agent-app/api_keys /var/log/agent-app
getfacl /home/agent-admin/agent-app/upload_files
getfacl /home/agent-admin/agent-app/api_keys
getfacl /var/log/agent-app
ss -tulnp | grep ':15034'
/home/agent-admin/agent-app/bin/monitor.sh
tail -n 5 /var/log/agent-app/monitor.log
sudo crontab -u agent-admin -l
```

이 명령들의 출력이 수행 내역서의 핵심 증거가 된다.

---

## 16장. 이 미션에서 설명할 수 있어야 하는 것

### SSH 포트 변경과 Root 원격 접속 차단

SSH 기본 포트인 `22`는 자동 스캔과 무차별 대입 공격의 주요 대상이다.  
포트를 변경한다고 보안이 완성되는 것은 아니지만, 불필요한 자동 공격 노출을 줄일 수 있다.

Root 원격 접속을 차단하는 이유는 가장 강력한 계정이 직접 공격 대상이 되는 것을 막기 위해서다.  
일반 계정으로 접속한 뒤 필요한 작업만 `sudo`로 수행하면 작업 기록과 권한 통제가 더 명확해진다.

### 방화벽에서 필요한 포트만 허용하는 이유

서버는 외부에 열려 있는 포트가 적을수록 공격 표면이 줄어든다.  
이번 미션에서는 관리용 SSH `20022`와 앱 서비스 `15034`만 필요하므로 나머지 인바운드 연결은 차단한다.

### 역할 기반 권한과 ACL의 의미

`agent-common`은 협업이 필요한 공유 영역을 위한 그룹이다.  
`agent-core`는 API 키와 운영 로그처럼 더 민감한 영역을 위한 그룹이다.

ACL을 사용하면 디렉토리의 기본 권한을 더 세밀하게 제어할 수 있다.  
특히 새로 생성되는 파일에도 그룹 접근 정책을 유지할 수 있어 운영 중 권한 꼬임을 줄인다.

### 환경 변수를 사용하는 이유

환경 변수는 앱 실행 환경을 고정하는 계약이다.  
코드나 스크립트에 경로와 포트를 흩뿌리지 않고 `AGENT_HOME`, `AGENT_PORT` 같은 값으로 통일하면 배포와 점검이 쉬워진다.

### 로그 자동화가 중요한 이유

장애가 난 뒤에 상태를 확인하면 이미 원인이 사라졌을 수 있다.  
`monitor.sh`와 cron은 시스템 상태를 주기적으로 기록해 나중에 장애 시점의 근거를 남긴다.

### 로그 보존 정책이 필요한 이유

로그는 많을수록 좋지만 무한히 쌓이면 디스크 장애를 만든다.  
따라서 용량 기반 회전, 시간 기반 압축, 오래된 아카이브 삭제 같은 보존 정책이 필요하다.
