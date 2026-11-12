# ubuntu-server-hardening
Ubuntu 24.04 서버에 SSH 키 인증, ufw, fail2ban, SSH 하드닝을 적용하고 검증한 실습

VirtualBox 가상 서버(Ubuntu 24.04 LTS)에 **SSH 키 인증, 방화벽(ufw), 침입 차단(fail2ban), SSH 하드닝**을 직접 적용하고, 각 설정이 실제로 동작하는지 확인한 기록입니다.

## 목표

- 서버를 처음 열었을 때의 상태를 확인하고, 접속 경로를 단계적으로 좁혀 간다.
- 설정만 하는 데서 끝내지 않고 **차단과 거부가 실제로 일어나는지** 로그와 접속 결과로 검증한다.
- 막혔던 문제와 원인을 함께 기록한다.

## 환경

| 항목 | 내용 |
|---|---|
| 가상화 | VirtualBox (버전:7.2.22 ) |
| OS | Ubuntu 24.04.5 LTS (Ubuntu Desktop 22.04.4 ISO로 설치하는 중 24.04로 업그레이드) |
| VM 사양 | 메모리 4096MB, CPU 2개, 디스크 25GB |
| 접속 | 윈도우 PowerShell에서 SSH |
| 네트워크 | 처음에는 브리지 어댑터, 이후 카페 네트워크에서 작업하며 NAT + 포트 포워딩으로 변경 |

> **네트워크를 바꾼 이유**: 작업 장소가 집에서 카페로 바뀌자 브리지 방식에서는 VM이 주소를 받지 못했습니다. 공용 네트워크에 VM을 그대로 노출하는 것도 보안 실습에는 좋지 않다고 판단해 NAT로 전환하고, 호스트 2222번 포트를 게스트 22번 포트로 연결했습니다. 이 때문에 앞부분 기록은 `172.30.1.x`, 이후 기록은 `127.0.0.1`(포트 2222)과 `10.0.2.2`로 표기가 다릅니다.

## 1. 적용 전 상태

OS 버전, 네트워크 주소, 열린 포트, 방화벽 상태를 먼저 확인했습니다. 방화벽(ufw)은 꺼져 있는 상태(`inactive`)였습니다.

```bash
cat /etc/os-release
ip a
ss -tulnp
sudo ufw status
```

![적용 전 상태](images/(1)ip-a.png)

## 2. SSH 서버 설치와 접속

데스크톱 버전에는 SSH 서버가 기본으로 없어서 직접 설치했습니다.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server -y
sudo systemctl status ssh
```

![SSH 서비스 실행 확인](images/02_ssh설치.png)

윈도우에서 처음 접속할 때는 서버의 지문(fingerprint)을 확인하라는 질문이 나옵니다. 이 서버가 맞는지 처음 한 번 확인하는 단계이고, 이후 지문이 바뀌면 경고가 뜹니다.

![비밀번호로 SSH 접속](images/03_ssh접속.png)

## 3. SSH 키 인증

비밀번호 대신 키로 로그인하도록 ed25519 키를 만들고 공개 키를 서버에 등록했습니다. 개인 키는 PC 안에만 두고 서버에는 공개 키만 올립니다.

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh ubuntu-sec@<서버IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

![키 생성](images/04_키생성.png)

![공개 키 등록](images/05_키등록.png)

키를 등록한 뒤에는 서버 비밀번호 대신 **키 암호(passphrase)** 를 묻습니다. 키 암호는 내 PC 안에서만 쓰이고 서버로 전송되지 않습니다.

![키로 접속](images/06_키로접속.png)

## 4. 방화벽(ufw)

들어오는 연결은 기본 차단하고 SSH(22번)만 허용했습니다. **22번 허용을 먼저, 방화벽 활성화는 나중에** 해야 현재 접속이 끊기지 않습니다.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status verbose
```

![ufw 설정](images/08_ufw설정.png)

## 5. fail2ban

SSH 로그인 실패가 반복되면 해당 IP를 자동으로 차단하도록 설정했습니다.

`/etc/fail2ban/jail.local`

```ini
[sshd]
enabled = true
backend = systemd
maxretry = 3
findtime = 10m
bantime = 10m
```

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

![fail2ban 설정 직후 상태](images/10_fail2ban설정.png)

### 차단 테스트

키 로그인을 끈 상태(`PubkeyAuthentication=no`)로 일부러 비밀번호를 틀려 로그인에 실패해 보았습니다.

```powershell
ssh -o PubkeyAuthentication=no ubuntu-sec@<서버IP>
```

![로그인 실패](images/11_로그인실패.png)

잠시 뒤 같은 접속이 `timed out`으로 막혔습니다.

![접속 차단됨](images/12_차단됨.png)

서버에서 차단 이력을 확인했습니다.

![차단 이력 요약](images/13_차단목록.png)

로그(`/var/log/fail2ban.log`)를 시간순으로 읽으면 다음과 같습니다.

```bash
sudo grep -E "Found|Ban|Unban" /var/log/fail2ban.log
```

![fail2ban 로그](images/13-2_차단흔적.png)

| 시각 | 내용 |
|---|---|
| 15:28:27 ~ 15:28:38 | 로그인 실패 3회 감지 (`Found`) |
| 15:28:56, 15:29:02 | 실패 2회 추가 감지 |
| 15:29:02 | 5번째 실패 직후 차단 (`Ban`) |
| 15:39:02 | 정확히 10분 뒤 차단 해제 (`Unban`) |

### 설정이 적용되지 않았던 문제

설정은 `maxretry = 3`이었는데 차단은 5번째 실패에서 걸렸습니다. 확인해 보니 실제 적용값이 5(기본값)였습니다.

```bash
sudo fail2ban-client get sshd maxretry
```

![적용되지 않은 설정](images/13-3_설정미적용.png)

`apt install` 직후 fail2ban은 **기본 설정으로 이미 실행 중**이었고, 그 뒤에 `jail.local`을 만들었지만 서비스를 재시작하지 않아 새 설정을 읽지 못한 것이었습니다. 재시작 뒤 값이 3으로 바뀐 것을 확인했습니다.

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client get sshd maxretry
```

![재시작 후 적용된 설정](images/13-4_재시작후적용.png)

**배운 점**: 설정 파일을 고치는 것과 서비스에 적용되는 것은 별개이고, 적용 여부는 `fail2ban-client get`으로 직접 확인해야 합니다.

## 6. SSH 하드닝

루트 로그인과 비밀번호 로그인을 막아, 키가 없으면 비밀번호를 대입해도 들어올 수 없게 했습니다. 우분투는 `/etc/ssh/sshd_config.d/` 안의 파일을 이름순으로 읽고 먼저 읽은 값이 우선하므로, 다른 설정이 덮어쓰지 못하게 파일 이름을 `01-`로 시작하게 만들었습니다.

`/etc/ssh/sshd_config.d/01-hardening.conf`

```
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo sshd -t
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication"
sudo systemctl restart ssh
```

적용 전:

![하드닝 전](images/14_하드닝전.png)

설정 후 적용값 확인:

![하드닝 설정](images/15_하드닝설정.png)

### 검증

기존 SSH 창은 열어 둔 채 새 창에서 두 가지를 확인했습니다. 설정을 잘못 바꿔도 되돌릴 수 있게 하기 위해서입니다.

- 키로 접속하면 정상 접속된다.
- 비밀번호 로그인을 시도하면 비밀번호를 묻지도 않고 거부된다.

```powershell
ssh -o PubkeyAuthentication=no ubuntu-sec@<서버IP>
```

거부 메시지가 `(publickey,password)`에서 `(publickey)`로 바뀐 것을 확인했습니다. 비밀번호 방식이 더 이상 허용되지 않는다는 뜻입니다.

![비밀번호 로그인 거부](images/16_비밀번호거부.png)

## 막혔던 점과 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| `sudo`를 인식하지 못한다는 오류 | 우분투 명령을 윈도우 PowerShell에 입력함 | 프롬프트로 창을 구분 (`PS C:\...>`는 윈도우, `ubuntu-sec@...$`는 우분투) |
| 접속이 안 됨 | `ip a`에서 `127.0.0.1`을 서버 주소로 착각 | `enp0s3` 항목의 `inet`이 접속용 주소 |
| 장소를 옮긴 뒤 VM이 주소를 받지 못함 | 브리지 방식은 PC가 연결된 네트워크에 의존 | NAT + 포트 포워딩으로 변경 |
| fail2ban 로그가 `journalctl`에 없음 | 차단 기록은 `/var/log/fail2ban.log`에 저장됨 | `fail2ban-client get logtarget`으로 로그 위치 확인 |
| `maxretry` 설정이 적용되지 않음 | 설정 변경 후 서비스를 재시작하지 않음 | 재시작 후 `get` 명령으로 적용값 확인 |

## 정리

| 항목 | 적용 내용 | 확인 방법 |
|---|---|---|
| SSH 키 인증 | ed25519 키 등록 | 서버 비밀번호 대신 키 암호를 묻는 것 확인 |
| 방화벽 | 기본 차단, 22번만 허용 | `ufw status verbose` |
| fail2ban | 10분 내 3회 실패 시 10분 차단 | 로그에서 감지·차단·해제 확인 |
| SSH 하드닝 | 루트·비밀번호 로그인 차단 | 비밀번호 로그인 거부 확인 |

## 다음 계획

- Docker로 DVWA 등 취약한 웹 환경을 띄워 SQL 인젝션, XSS를 재현하고 방어 방법을 정리하기
- 예약 시스템 프로젝트에 Spring Security 적용하기
