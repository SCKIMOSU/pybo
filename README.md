# AWS Lightsail : Pybo + Nginx + Gunicorn 배포

- pybo 저장소: [https://github.com/SCKIMOSU/pybo.git](https://github.com/SCKIMOSU/pybo.git)

---

## ✅ 전체 개요

- 서버: AWS Lightsail (Ubuntu 22.04)
- WSGI 서버: Gunicorn
- Reverse Proxy: Nginx
- Django 앱: pybo (GitHub에서 clone)
- 기타: 가상환경 사용, `ALLOWED_HOSTS`, 정적 파일 설정

---

## ✅ 1. Lightsail 인스턴스 생성 및 접속

1. Lightsail 접속 → Ubuntu 서버 생성
2. 생성 후 "연결 → SSH 터미널 열기" 또는 로컬에서:

```bash
ssh -i LightsailDefaultKey.pem ubuntu@43.201.51.144

```

---

- Lightsail에서 SSH를 이용 서버 접속
    
    ![db.png](db.png)
    

---

## ✅ 1. 접속 방법 요약

### 🔹 A. **Lightsail 웹 콘솔에서 직접 SSH 접속 (간단)**

1. [https://lightsail.aws.amazon.com/](https://lightsail.aws.amazon.com/) 접속
2. 인스턴스 선택
3. **"연결"** 탭 클릭
4. **브라우저 기반 SSH 터미널** 사용 (별도 설정 없이 가능)

---

### 🔹 B. **로컬 PC(리눅스/macOS/WSL)에서 SSH 키로 접속**

### ① 키 파일 다운로드 (예: `LightsailDefaultKey-ap-northeast-2.pem`)

- Lightsail 콘솔 > 계정 메뉴 > SSH 키 > 다운로드
- 해당 키를 안전한 위치에 저장

### ② 키 파일 권한 설정

```bash
chmod 400 LightsailDefaultKey-ap-northeast-2.pem

```

### ③ SSH 접속

```bash
ssh -i LightsailDefaultKey-ap-northeast-2.pem ubuntu@<인스턴스_IP>

```

예시:

```bash
ssh -i LightsailDefaultKey-ap-northeast-2.pem ubuntu@43.201.51.144

```

---

### 🔹 C. **Windows 사용자라면 PuTTY로 접속 (pem → ppk 변환 필요)**

1. `.pem` 파일을 **PuTTYgen**으로 `.ppk` 변환
2. PuTTY에서 IP 입력 + `.ppk` 키 파일 등록
3. 접속

---

## ✅ 접속 안 될 때 점검 체크리스트

| 항목 | 확인 방법 |
| --- | --- |
| 인스턴스 실행 중인지 | Lightsail 콘솔에서 상태 확인 |
| 공인 IP 맞는지 | 콘솔 > 인스턴스 > 네트워킹 탭 |
| 방화벽에서 SSH(포트 22) 허용 중인지 | "네트워킹" > 포트 열림 여부 확인 |
| 키 파일 권한 400인지 | `chmod 400` |
| 키 파일 경로가 정확한지 | `ls -l` 로 경로 확인 |

---

- **접속된 Lightsail 웹 콘솔**

![db.png](db%201.png)

## ✅ 2. 필수 패키지 설치

```bash
sudo apt update
sudo apt install python3-pip python3-venv nginx git -y

```

---

![db.png](db%202.png)

![db.png](db%203.png)

## ✅ 3. 프로젝트 다운로드 및 가상환경 구성

```bash
cd ~
git clone https://github.com/SCKIMOSU/pybo.git
cd pybo
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

```

---

![db.png](db%204.png)

## ✅ 4. Django 설정 변경

**`pybo/config/settings.py`**

```python
DEBUG = False

ALLOWED_HOSTS = ['YOUR_DOMAIN_OR_IP', 'localhost', '127.0.0.1']

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

```

![db.png](db%205.png)

![db.png](db%206.png)

![db.png](db%207.png)

- **추가 권장: SECRET_KEY 환경변수 처리**

```python
import os
SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY', 'insecure_key_for_dev')

```

---

## ✅ 5. 정적 파일 수집 및 DB 마이그레이션

```bash
python manage.py collectstatic
python manage.py migrate

```

---

- pip install django 로 django를 설치하고  python manage.py collectstatic 명령어를 실행함
- python manage.py migrate 실행하여 database를 생성함

![db.png](db%208.png)

## ✅ 6. Gunicorn 설정

```bash
# 테스트 실행
gunicorn --bind 127.0.0.1:8000 config.wsgi

```

- **정상 작동 시 `Ctrl+C` 로 종료 후 systemd 서비스 구성**

---

## ✅ 7. Gunicorn systemd 서비스 파일 생성

```bash
sudo nano /etc/systemd/system/gunicorn.service

```

```
[Unit]
Description=Gunicorn daemon for pybo
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application

[Install]
WantedBy=multi-user.target

```

```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn

```

---

![db.png](db%209.png)

![db.png](db%2010.png)

## ✅ 8. Nginx 설정

```bash
sudo nano /etc/nginx/sites-available/pybo

```

```
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

![db.png](db%2011.png)

```bash
sudo ln -s /etc/nginx/sites-available/pybo /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx

```

---

- 심볼릭 링크가 파일 존재하면 존재한다고 알려 줌

![db.png](db%2012.png)

## ✅ 심볼릭 링크란?

> 심볼릭 링크는 원본 파일의 경로를 담고 있는 얇은 껍데기
> 
> 
> 링크를 열면 실제 원본 파일이 열림
> 
> ## `심볼릭 링크 (symbolic link, symlink)`
> 
> - **다른 파일이나 디렉토리에 대한 "참조(바로가기)" 파일**
>     - **Windows의 "바로가기 아이콘"** 과 비슷

---

### 📌 예시

```bash
ln -s /home/ubuntu/원본.txt 링크.txt

```

- `링크.txt`는 실제로 존재하지 않으며
- `원본.txt`를 가리키는 "가상 파일"
- `cat 링크.txt` 하면 실제 `원본.txt` 내용이 출력됨

---

## ✅ 심볼릭 링크 vs 하드 링크 비교

| 항목 | 심볼릭 링크 (`ln -s`) | 하드 링크 (`ln`) |
| --- | --- | --- |
| 독립성 | 원본 삭제 시 깨짐 ❌ | 원본 삭제해도 유지됨 ✅ |
| 크기 | 아주 작음 (경로만 저장) | 실제 파일과 동일 |
| 디렉토리 링크 가능 여부 | 가능 ✅ | 불가 ❌ (일반적으로) |
| 파일 시스템 경계 넘기기 | 가능 ✅ | 불가 ❌ |
| 용도 | 설정 연결, 바로가기 등 | 동일한 파일 다중 이름 |

---

### 📂 Nginx에서 왜 쓰는가?

- `/etc/nginx/sites-available/` → 설정 파일 저장소
- `/etc/nginx/sites-enabled/` → 실제 적용할 설정 모음

```bash
sudo ln -s /etc/nginx/sites-available/pybo /etc/nginx/sites-enabled/

```

→ `sites-enabled/pybo` 는 원본 파일을 그대로 가리킴

→ `nginx.conf`는 `/etc/nginx/sites-enabled/*` 를 include 하므로, 설정이 반영됨

---

## ✅ 확인 방법

```bash
ls -l

```

출력 예:

```
pybo -> ../sites-available/pybo

```

→ `pybo`는 symlink이며 `../sites-available/pybo`를 가리킴

---

## ✅ 삭제도 안전하게 가능

- 링크만 삭제:

```bash
sudo rm /etc/nginx/sites-enabled/pybo

```

→ 원본 파일은 **절대 삭제되지 않음**

---

## ✅ 9. 방화벽 설정 (Lightsail)

- Lightsail 콘솔 → 네트워킹 → 포트 80, 443, 22 열기
    - Lightsail 네트워킹 화면

![db.png](db%2013.png)

---

## ✅ 10. 도메인 연동 (선택)

- 도메인이 있다면 A 레코드 → Lightsail IP로 연결
- `server_name` 항목에 도메인 입력

---

## ✅ 11. HTTPS 설정 (선택, Let’s Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d YOUR_DOMAIN

```

---

## ✅ 12. 검증

브라우저 접속:

```
http://<YOUR_PUBLIC_IP>
또는
http://<YOUR_DOMAIN>

```

---

## ✅ 자주 발생되는 문제

- `502 Bad Gateway`: Gunicorn이 제대로 작동 중인지 확인
- `Permission denied`: `pybo.sock`의 권한 (`www-data` 그룹 포함) 확인
- 정적 파일 안 뜰 때: `STATIC_ROOT`, `collectstatic`, `root` 경로 재확인

---

## ✅ 완성된 서비스 구조 요약

```
[ Client ] → HTTP → [ Nginx (80) ]
                        ↓ proxy_pass
                [ Gunicorn (UNIX Socket) ]
                        ↓
                [ Django app (pybo) ]

```

---

- 실행 화면
    - 파이보 서비스가 AWS **43.201.51.144 (외부 서버)에서 동작중**

![db.png](db%2014.png)

## AWS **43.201.51.144 (외부 서버)에서 동작중**

- `43.201.51.144`는 AWS Lightsail에서 발급받은 공인 IP 주소(Public IP address)이므로, AWS **43.201.51.144 (외부 서버)에서 동작중**

---

## ✅ 정리

| 표현 | 설명 |
| --- | --- |
| 외부 서버 (external server) ✅ | **당신이 아닌 제3자 또는 인터넷 사용자들이 접속할 수 있는 서버**라는 의미에서 올바른 표현 |
| 공인 IP (public IP) ✅ | 인터넷 상에서 유일하며, **어디서든 접근 가능한 IP 주소** |
| 원격 서버 (remote server) ✅ | **내 로컬이 아닌, 네트워크를 통해 접근하는 서버**라는 의미에서 정확한 표현 |
| 퍼블릭 호스트 ✅ | 퍼블릭 IP를 가진 호스트, Nginx나 Django 웹앱이 서비스되는 대상 |
| 클라우드 서버 ✅ | AWS Lightsail은 클라우드 환경에서 운영되므로 정확한 표현 |

---

## ❗주의 (내부 서버와의 구분)

| 용어 | 예시 IP | 의미 |
| --- | --- | --- |
| 외부 서버 | `43.201.51.144` | 퍼블릭 인터넷에서 접근 가능 |
| 내부 서버 | `192.168.x.x`, `172.16.x.x`, `10.x.x.x` | 로컬 네트워크 내 전용 IP (외부 접속 불가) |

---

- 43.201.51.144는 외부 서버, 공인 IP를 가진 원격 서버, AWS Lightsail 클라우드 인스턴스
    - → 따라서 `"외부 서버에 pybo를 배포했다"` 또는 `"공인 IP 43.201.51.144를 통해 서비스 중이다"` 라고 말할 수 있음

---

## ssh -i LightsailDefaultKey-ap-northeast-2.pem [ubuntu@43.201.51.144](mailto:ubuntu@43.201.51.144) 접속 안 될 때

- `ssh -i LightsailDefaultKey-ap-northeast-2.pem ubuntu@43.201.51.144` 명령어로 AWS Lightsail 인스턴스에 접속이 안 되는 경우,
    - 우분투 터미널에서 Lightsail ssh 접속

---

## ✅ 1. 키 파일 경로 및 권한 확인

```bash
ls -l LightsailDefaultKey-ap-northeast-2.pem

```

- 파일이 현재 디렉터리에 없거나 이름이 틀리면 "No such file" 오류 발생
- 권한 확인

```bash
chmod 400 LightsailDefaultKey-ap-northeast-2.pem

```

> 🔒 퍼미션이 400이 아니면 SSH에서 거부됨 (보안상 제한)
> 

---

## ✅ 2. IP 주소 확인

- Lightsail 콘솔에서 해당 인스턴스의 **공인 IP (Public IP)** 가 `43.201.51.144`가 맞는지 다시 확인
    - 콘솔에서 해당 인스턴스를 선택 → "연결" → "공용 IP" 확인
    - IP 주소가 바뀔 수 있으니 재확인 필수

---

## ✅ 3. SSH 포트가 열려 있는지 확인 (Lightsail 방화벽 설정)

- Lightsail > 네트워킹 > 인스턴스 > 방화벽 탭에서 아래 설정이 있는지 확인:

| 프로토콜 | 포트 | 허용 대상 |
| --- | --- | --- |
| SSH | 22 | 0.0.0.0/0 |

---

## ✅ 4. 인스턴스가 켜져 있는지 확인

- Lightsail 콘솔에서 인스턴스 상태가 **"정상 실행 중"** 인지 확인

---

## ✅ 5. 연결 예시 전체 (현재 디렉터리에 pem 파일 있을 경우)

```bash
ssh -i ./LightsailDefaultKey-ap-northeast-2.pem ubuntu@43.201.51.144

```

---

## ✅ 6. 오류 메시지 분석 (필요시 전체 출력 공유)

접속 시 나오는 **오류 메시지 전체**를 복사해서 보내주시면, 어떤 단계에서 막히는지 정확히 분석 가능합니다.

예:

- `Permission denied (publickey)`
- `Connection timed out`
- `No such file or directory`
- `WARNING: UNPROTECTED PRIVATE KEY FILE!`

---

## nginx는 `requirements.txt`에 **포함하지 않음**

- `requirements.txt`: `pip install`로 설치하는 **Python 라이브러리 목록**
- `nginx`: Python과 관련 없는 **시스템 웹서버 소프트웨어** → `apt`로 설치
- `requirements.txt`는 Python 패키지를 설치하는 용도로 사용되며, `nginx`는 **Python 패키지가 아니라 리눅스 시스템 패키지**

---

### ✅ Ubuntu에서 nginx 설치:

```bash
sudo apt update
sudo apt install nginx -y

```

---

## ✅ pybo 프로젝트의 requirements.txt 예시 (nginx 제외)

```
Django>=4.0,<5.0
gunicorn>=20.1.0        # Python WSGI 서버
markdown                # 질문/답변 포맷팅에 사용될 수 있음
django-crispy-forms     # 폼 스타일링 (선택)
python-dotenv           # 환경변수 관리 (선택)

```

- nginx는 여기에 절대 포함되지 않음.

---

## ✅ 추가로 필요한 시스템 패키지 정리 (nginx 관련)

| 역할 | 설치 명령어 |
| --- | --- |
| Nginx 설치 | `sudo apt install nginx -y` |
| Gunicorn 서비스 구성 | `pip install gunicorn` |
| 가상환경 준비 | `python3 -m venv venv` |

---

## ✅ 결론

- `nginx`는 `pip`로 설치 ❌
- `requirements.txt`에 포함 ❌
- 대신 `apt install nginx`로 설치 ✅

---

## sudo systemctl start gunicorn 명령어 에러시

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo systemctl start gunicorn
**Failed** to start gunicorn.service: **Unit gunicorn.service has a bad unit file setting.**
See system logs and 'systemctl status gunicorn.service' for details.
```

- `gunicorn.service` 파일 설정 오류로 인해 `sudo systemctl start gunicorn` 명령이 실패.
- 오류 메시지: `Unit gunicorn.service has a bad unit file setting.`

---

## ✅ 해결 절차

### 1️⃣ 현재 서비스 파일 내용 확인

- 아래 명령어로 설정 내용을 확인

```bash
sudo cat /etc/systemd/system/gunicorn.service

```

---

### 2️⃣ 올바른 예시: gunicorn.service

```
[Unit]
Description=Gunicorn daemon for pybo
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application

[Install]
WantedBy=multi-user.target

```

- ⚠️ WorkingDirectory, ExecStart 경로가 실제 존재하는지 꼭 확인하세요.

---

### 3️⃣ 수정 후 작업 순서

1. 파일 수정:
    
    ```bash
    sudo nano /etc/systemd/system/gunicorn.service
    
    ```
    
2. systemd 리로드:
    
    ```bash
    sudo systemctl daemon-reexec
    sudo systemctl daemon-reload
    
    ```
    
3. Gunicorn 실행:
    
    ```bash
    sudo systemctl start gunicorn
    sudo systemctl status gunicorn
    
    ```
    

---

### ✅ 로그 확인 (에러 원인 자세히 보기)

```bash
journalctl -xe

```

→ 해당 로그에 `gunicorn.service` 관련 오류 메시지가 나옵니다.

---

## 🛠️ 자주 발생하는 오류 예시

| 원인 | 메시지 예시 |
| --- | --- |
| 잘못된 경로 (`ExecStart`) | `No such file or directory` |
| 오타 (예: `ExceStart`) | `bad unit file setting` |
| `User=ubuntu`인데 존재 안함 | `Failed to determine user credentials` |
| `WorkingDirectory` 경로 없음 | `chdir() to /home/... failed: No such` |

---

---

## 🧙‍♂️ 데몬(daemon)이란?

> 데몬(daemon) 은 백그라운드에서 지속적으로 실행되며, 특정 요청이나 작업을 처리하는 프로그램.
> 
- 사용자가 직접 실행하거나 조작하지 않아도
- 시스템이 부팅될 때 자동으로 시작되어
- **지속적으로 서비스 또는 이벤트를 대기하고 처리**.
- 리눅스/유닉스 시스템에서 매우 중요한 개념

---

## ✅ 데몬의 특징

| 특징 | 설명 |
| --- | --- |
| **백그라운드 실행** | 화면에 표시되지 않음 (터미널 차지 X) |
| **자동 시작** | 시스템 부팅 시 자동으로 실행 |
| **지속 실행** | 사용자가 종료하지 않는 한 계속 작동 |
| **서비스 제공** | 네트워크, 로그, 웹서버, 데이터베이스 등 |

---

## ✅ 데몬 이름 규칙

대부분 데몬의 이름은 **`d`로 끝남**:

| 데몬 이름 | 기능 |
| --- | --- |
| `ssh**d**` | SSH 접속을 처리 |
| `nginx` | 웹 서버 요청 처리 |
| `mysql**d**` | MySQL 데이터베이스 데몬 |
| `gunicorn` | Django 앱을 실행하는 WSGI 서버 데몬 |
| `system**d**` | 모든 시스템 서비스 관리 총괄 데몬 |

---

## ✅ 데몬이 하는 일

- **웹서버 데몬(Nginx)**
    
    → 브라우저에서 접속 요청이 오면 HTML 페이지 반환
    
- **데이터베이스 데몬(MySQLd)**
    
    → 앱이 DB 조회 요청을 하면 결과 반환
    
- **SSH 데몬(sshd)**
    
    → 원격에서 SSH 접속이 오면 로그인 처리
    
- **Gunicorn 데몬**
    
    → Django 앱을 WSGI 방식으로 처리해 웹 응답 전송
    

---

## ✅ 데몬 vs 일반 프로그램

| 항목 | 일반 프로그램 | 데몬 |
| --- | --- | --- |
| 실행 방식 | 사용자 수동 실행 | 시스템 자동 실행 |
| 터미널 점유 | 예 (`foreground`) | 아니오 (`background`) |
| 종료 방식 | 사용자 또는 Ctrl+C | `systemctl stop`, 자동종료 없음 |
| 예시 | `vim`, `python` | `sshd`, `nginx`, `gunicorn` |

---

## ✅ 데몬 실행 확인

```bash
ps aux | grep sshd

```

또는

```bash
systemctl status nginx

```

---

## 🧾 어원

- `daemon`은 고대 그리스어 **δαίμων (daimon)** 에서 유래
- 의미: **보이지 않지만 일을 수행하는 "중간 존재"**
- 컴퓨터에서는: **백그라운드에서 사용자 대신 일을 처리하는 프로그램**

---

## `systemctl`은 리눅스의 서비스 관리자 **systemd를 제어하는 명령어**

- `systemctl`은 systemd를 통해 **서버의 모든 서비스(데몬)를 시작, 중지, 재시작, 상태확인**하는 데 사용.

---

## ✅ 정의

> systemctl은 systemd를 사용하는 시스템에서 서비스 및 프로세스를 관리하는 명령어.
> 

---

## ✅ 주요 역할

| 기능 | 설명 |
| --- | --- |
| 서비스 시작 | `systemctl start` |
| 서비스 중지 | `systemctl stop` |
| 서비스 재시작 | `systemctl restart` |
| 서비스 상태 확인 | `systemctl status` |
| 부팅 시 자동실행 설정 | `systemctl enable` |
| 부팅 시 자동실행 해제 | `systemctl disable` |
| 설정 반영 (리로드) | `systemctl daemon-reload` |

---

## ✅ 자주 쓰는 명령어 예시

```bash
# 서비스 시작
sudo systemctl start gunicorn

# 서비스 중지
sudo systemctl stop nginx

# 서비스 재시작
sudo systemctl restart gunicorn

# 서비스 상태 확인
sudo systemctl status nginx

# 부팅 시 자동 실행 설정
sudo systemctl enable gunicorn

# 부팅 시 자동 실행 해제
sudo systemctl disable gunicorn

# 서비스 설정 변경 후 systemd 재로드
sudo systemctl daemon-reload

```

---

## ✅ systemctl이 관리하는 대상

- `systemctl`은 단순히 서비스만 관리하는 것이 아니라, 다음도 관리

| 대상 | 예시 |
| --- | --- |
| **서비스** (.service) | `nginx.service`, `gunicorn.service` |
| **소켓** (.socket) | `docker.socket`, `cups.socket` |
| **타이머** (.timer) | `apt-daily.timer`, `logrotate.timer` |
| **디바이스** | `dev-sda.device` |
| **마운트 포인트** (.mount) | `/mnt/backup.mount` |
| **타겟** (.target) | `multi-user.target`, `graphical.target` |

---

## ✅ 상태 확인 예시

```bash
sudo systemctl status gunicorn

```

출력 예:

```
● gunicorn.service - Gunicorn daemon for pybo
   Loaded: loaded (/etc/systemd/system/gunicorn.service; enabled)
   Active: active (running)
   Main PID: 1234 (gunicorn)

```

---

## 🔍 `systemd` vs `systemctl`

| 항목 | 설명 |
| --- | --- |
| `systemd` | 실제 서비스들을 관리하는 **백그라운드 데몬** |
| `systemctl` | systemd를 제어하는 **명령어 도구** |

---

## ✅ gunicorn.service 파일 오타 주의

```jsx
[Unit]
Description=Gunicorn daemon for pybo
After=network.target

[Services]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/
home/ubuntu/pybo/pybo.sock config.wsgi:application

[Install]
WanredBy=multi-user.target
```

---

## ❌ 잘못된 부분 요약

1. `[Services]` → **오타**, 올바른 섹션 이름은 `[Service]`
2. `WanredBy` → **오타**, 올바른 키는 `WantedBy`

---

## ✅ 올바르게 수정된 gunicorn.service 파일:

```
[Unit]
Description=Gunicorn daemon for pybo
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application

[Install]
WantedBy=multi-user.target

```

---

## 🔧 수정 방법

```bash
sudo nano /etc/systemd/system/gunicorn.service

```

---

## 🔄 수정 후 재로드 및 실행

```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo systemctl status gunicorn

```

---

## ✅ 결과 확인

정상이라면 `status`에서 다음과 같은 메시지가 확인

```
Active: active (running)
Main PID: 12345 (gunicorn)

```

---

✅ **정상** 

- Gunicorn이 `pybo` 프로젝트에 대해 성공적으로 실행 중

```jsx
● gunicorn.service - Gunicorn daemon for pybo
     Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
     Active: active (running) since Sat 2025-05-31 07:25:18 UTC; 6s ago
   Main PID: 1474 (gunicorn)
      Tasks: 4 (limit: 476)
     Memory: 91.9M
        CPU: 645ms
     CGroup: /system.slice/gunicorn.service
             ├─1474 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             ├─1476 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             ├─1477 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             └─1478 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>

May 31 07:25:18 ip-172-26-0-96 systemd[1]: Started Gunicorn daemon for pybo.
May 31 07:25:18 ip-172-26-0-96 gunicorn[1474]: [2025-05-31 07:25:18 +0000] [1474] [INFO] Starting gunicorn 23.0.0
May 31 07:25:18 ip-172-26-0-96 gunicorn[1474]: [2025-05-31 07:25:18 +0000] [1474] [INFO] Listening at: unix:/home/u>
May 31 07:25:18 ip-172-26-0-96 gunicorn[1474]: [2025-05-31 07:25:18 +0000] [1474] [INFO] Using worker: sync
May 31 07:25:18 ip-172-26-0-96 gunicorn[1476]: [2025-05-31 07:25:18 +0000] [1476] [INFO] Booting worker with pid: 1>
May 31 07:25:18 ip-172-26-0-96 gunicorn[1477]: [2025-05-31 07:25:18 +0000] [1477] [INFO] Booting worker with pid: 1>
May 31 07:25:18 ip-172-26-0-96 gunicorn[1478]: [2025-05-31 07:25:18 +0000] [1478] [INFO] Booting worker with pid: 1>
lines 1-20/20 (END)
```

```
Active: active (running)

```

- 소켓 바인딩도 잘 되어 있음

```
Listening at: unix:/home/ubuntu/pybo/pybo.sock

```

---

## 🔜 다음 단계: Nginx 설정

- Gunicorn은 백엔드에서 잘 작동하므로, 프론트엔드 요청을 처리할 Nginx를 설정

---

## ✅ 1. Nginx 설정 파일 생성

```bash
sudo nano /etc/nginx/sites-available/pybo

```

### ⬇️ 아래 내용 입력:

```
server {
    listen 80;
    server_name <YOUR_PUBLIC_IP>;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

- 주의: <YOUR_PUBLIC_IP>에 실제 Lightsail 공인 IP 입력 (예: 43.201.51.144)

---

## ✅ 2. 심볼릭 링크로 설정 활성화

```bash
sudo ln -s /etc/nginx/sites-available/pybo /etc/nginx/sites-enabled

```

---

## ✅ 3. 설정 문법 체크 및 재시작

```bash
sudo nginx -t
sudo systemctl restart nginx

```

---

## ✅ 4. 방화벽 확인 (Lightsail 콘솔)

- Lightsail 인스턴스 → "네트워킹" → 아래 포트가 열려 있어야 함

| 프로토콜 | 포트 | 허용 대상 |
| --- | --- | --- |
| HTTP | 80 | 0.0.0.0/0 |
| SSH | 22 | 0.0.0.0/0 |

---

## ✅ 5. 접속 테스트

브라우저 또는 curl로 접속:

```bash
http://<YOUR_PUBLIC_IP>

```

---

## `502 Bad Gateway` 웹사이트 오류

- Nginx가 백엔드(Gunicorn)와 연결하려 했지만 응답을 받지 못했을 때 발생
- 현재 Gunicorn은 정상적으로 실행 중이므로, **Nginx 설정에서 Gunicorn의 소켓 경로**가 잘못되었거나 접근 권한 문제일 가능성 높음 .

```jsx
502 Bad Gateway
nginx/1.18.0 (Ubuntu)
```

---

## ✅ 순서대로 점검

---

### 1️⃣ Gunicorn 소켓 파일이 존재하는지 확인

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

**결과 예시:**

```bash
srw-rw-r-- 1 ubuntu www-data 0 May 31 07:25 /home/ubuntu/pybo/pybo.sock

```

**확인할 것:**

- `pybo.sock` 파일이 실제로 존재해야 함
- `Group`이 `www-data` 이어야 함
- 권한이 `rw` 로 되어 있어야 함

> ❗ 없거나 권한이 이상하면 다음 명령 실행:
> 

```bash
sudo chown ubuntu:www-data /home/ubuntu/pybo/pybo.sock
sudo chmod 660 /home/ubuntu/pybo/pybo.sock

```

---

### 2️⃣ Nginx 설정 확인

```bash
sudo nano /etc/nginx/sites-available/pybo

```

- `proxy_pass`가 아래처럼 정확한지 확인:

```
proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;

```

> 🔴 오타 예: unix:/\nhome/..., proxy_pass unix: (앞에 http:// 누락)
> 

---

### 3️⃣ Nginx 재시작

```bash
sudo nginx -t    # 문법 검사
sudo systemctl restart nginx

```

---

### 4️⃣ Gunicorn이 제대로 실행 중인지 재확인

```bash
sudo systemctl status gunicorn

```

→ `active (running)` 이면 OK.

---

### 5️⃣ Nginx 로그 확인 (문제 계속될 경우)

```bash
sudo tail -n 100 /var/log/nginx/error.log

```

→ 여기에 실제로 어떤 문제로 502가 발생했는지 상세 로그가 찍혀 있음. 예시:

- `connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied)`
- `connect() failed (2: No such file or directory)`

---

## ✅ 502 오류의 주요 원인 정리

| 원인 | 해결 방법 |
| --- | --- |
| `.sock` 파일이 없음 | Gunicorn이 제대로 안 켜짐 또는 경로 오타 |
| 권한 문제 (`Permission denied`) | `chown ubuntu:www-data` 및 `chmod 660` 적용 |
| Nginx 경로 오타 | `proxy_pass http://unix:/home/...` 확인 |
| Gunicorn 죽음 | `systemctl status gunicorn`로 확인 후 재시작 |

---

## 🔄 필요한 경우 Gunicorn 재시작

```bash
sudo systemctl restart gunicorn

```

---

## Gunicorn 측에 문제가 없을 때

- `pybo.sock` 파일이 다음과 같이 **정상적으로 존재하고 권한도 충분**

```jsx
srwxrwxrwx 1 ubuntu www-data 0 May 31 07:25 /home/ubuntu/pybo/pybo.sock
```

```
srwxrwxrwx 1 ubuntu www-data 0 May 31 07:25 /home/ubuntu/pybo/pybo.sock

```

---

## ✅ 다음 점검: Nginx 설정 확인

- 아래 2가지를 꼭 확인

---

### 1️⃣ Nginx 설정 파일 `/etc/nginx/sites-available/pybo`

```bash
sudo nano /etc/nginx/sites-available/pybo

```

- 다음처럼 정확히 작성되어 있는지 확인:

```
server {
    listen 80;
    server_name <YOUR_PUBLIC_IP>;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

> ❗ <YOUR_PUBLIC_IP>에는 실제 Lightsail IP (예: 43.201.51.144) 입력해야 합니다.
> 

---

### 2️⃣ 심볼릭 링크 연결 확인

```bash
ls -l /etc/nginx/sites-enabled/pybo

```

→ `/etc/nginx/sites-available/pybo`로 연결돼 있어야 함.

- 없다면:

```bash
sudo ln -s /etc/nginx/sites-available/pybo /etc/nginx/sites-enabled/

```

---

### 3️⃣ 설정 테스트 및 재시작

```bash
sudo nginx -t
sudo systemctl restart nginx

```

---

### 4️⃣ 여전히 502면: 로그 확인

```bash
sudo tail -n 50 /var/log/nginx/error.log

```

→ -여기에 `connect() failed`, `permission`, `file not found` 등이 나오면 원인을 파악 가능.

---

## 🔍 요약 체크리스트

| 항목 | 체크 |
| --- | --- |
| Gunicorn 실행 중 (`.sock` 있음) | ✅ |
| `.sock` 권한 (`ubuntu:www-data`) | ✅ |
| Nginx `proxy_pass` 경로 정확함 | ❓ |
| Nginx 설정 테스트 통과 | ❓ |
| `/static/` root 설정 확인 | ❓ |
| `/sites-enabled` 링크 확인 | ❓ |
| Nginx 재시작 후 상태 확인 | ❓ |

---

## 에러 포함된 Nginx 설정 예시

```jsx
server {
listen 80;
server_name 43.201.51.144;

location = /favicon.ico { access_log off; log_not_found off; }

location /static/ {
root /home/ubuntu/pybo;
}

location / {
include proxy_params;
proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
}
}
```

- 502 Bad Gateway 문제를 유발할 수 있는 **아주 작은 실수 또는 빠진 항목**이 있음

---

## ✅ 점검 요약

### 🔍 1. 설정 내용 확인 (정상)

```
server {
    listen 80;
    server_name 43.201.51.144;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

> ✅ 문법적으로 OK.
> 
- 하지만 실무적으로 proxy_params 파일이 없거나 누락된 변수가 있을 수 있음.

---

## ✅ 권장 수정: `proxy_params` 경로 명확히 지정

```
include /etc/nginx/proxy_params;

```

→ 기존의 상대경로 `proxy_params`는 종종 못 찾는 경우가 있음

🔧 수정된 설정:

```
server {
    listen 80;
    server_name 43.201.51.144;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include **/etc/nginx/proxy_params**;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

---

## ✅ 저장 후 실행 순서

```bash
sudo nginx -t
sudo systemctl restart nginx

```

---

## ✅ 추가 확인: error.log

- 아직 502가 나오면 에러 메시지 확인:

```bash
sudo tail -n 50 /var/log/nginx/error.log

```

예를 들어:

- `permission denied` → `.sock` 파일 권한 문제
- `connect() failed` → Gunicorn이 꺼져 있음 또는 잘못된 경로

---

## ✅ 정리

| 체크리스트 | 상태 |
| --- | --- |
| `.sock` 존재 및 권한 (`srwxrwxrwx`) | ✅ |
| Nginx `proxy_pass` 경로 정확 (`http://unix:` 포함) | ✅ |
| `include /etc/nginx/proxy_params;` | 🔧 **수정 필요** |
| `nginx -t` 테스트 후 재시작 | ⏱️ |
| `error.log`로 디버깅 | ⏱️ |

---

## sudo tail -n 50 /var/log/nginx/error.log 로그 분석

```jsx
connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied
) while connecting to upstream,
```

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo tail -n 50 /var/log/nginx/error.log
2025/05/31 01:58:17 [notice] 14178#14178: using inherited sockets from "6;7;"
2025/05/31 07:30:10 [crit] 1504#1504: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied
) while connecting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: 
"http://unix:/home/ubuntu/pybo/pybo.sock:/", host: "43.201.51.144"
2025/05/31 07:31:01 [crit] 1504#1504: *4 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied
) while connecting to upstream, client: 104.167.221.114, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream:
 "http://unix:/home/ubuntu/pybo/pybo.sock:/", host: "43.201.51.144:80"
2025/05/31 07:32:37 [crit] 1521#1521: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied
) while connecting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: 
"http://unix:/home/ubuntu/pybo/pybo.sock:/", host: "43.201.51.144"
```

- 로그 분석 결과:

```
connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied)

```

→ **Nginx가 Gunicorn 소켓 파일에 접근할 권한이 없어서 502 오류가 발생.**

---

## ✅ 원인

- Nginx는 기본적으로 **`www-data`** 사용자로 실행됨

### Gunicorn이 생성한 소켓 파일 `/home/ubuntu/pybo/pybo.sock`에 대해 Nginx(`www-data`)가 **읽기/쓰기 권한이 없음**

---

## ✅ 해결 방법

### 🔧 1. Gunicorn 소켓 권한 수정

```bash
sudo chmod 660 /home/ubuntu/pybo/pybo.sock
sudo chown ubuntu:www-data /home/ubuntu/pybo/pybo.sock

```

→ `.sock` 파일을 `www-data` 그룹이 읽고 쓸 수 있게 설정

---

### 🔁 2. Gunicorn 서비스 파일에 소켓 권한 강제 설정 추가 (추천)

`/etc/systemd/system/gunicorn.service` 열기:

```bash
sudo nano /etc/systemd/system/gunicorn.service

```

- **기존 [Service] 섹션에 아래 줄 추가:**

```
[Service]
...
UMask=0007

```

🔍 전체 예시:

```
[Unit]
Description=Gunicorn daemon for pybo
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application
UMask=007

[Install]
WantedBy=multi-user.target

```

---

### 🔄 3. 서비스 리로드 및 재시작

```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo systemctl restart nginx

```

---

### ✅ 4. 소켓 권한 재확인

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

- → 아래와 같이 되어 있어야 함

```bash
srw-rw---- 1 ubuntu www-data ... /home/ubuntu/pybo/pybo.sock

```

---

## ✅ 접속 확인

브라우저에서:

```
http://43.201.51.144

```

---

## 아래 소켓 권한은 **정상적이며 보안적으로도 안전한 상태**

```jsx
srw-rw---- 1 ubuntu www-data 0 May 31 07:35 /home/ubuntu/pybo/pybo.sock
```

```
srw-rw---- 1 ubuntu www-data ...

```

| 사용자 | 권한 |
| --- | --- |
| ubuntu | 읽기/쓰기 |
| www-data (Nginx) | 읽기/쓰기 |
| others | ❌ 없음 |

---

## ✅ 최종 확인

### 🔁 Nginx 재시작

- 변경 사항이 제대로 반영되었는지 다시 한 번 확인:

```bash
sudo systemctl restart nginx

```

---

### 🌐 웹브라우저 접속

- 접속 주소:

```
http://43.201.51.144

```

- 정상이라면 `pybo`의 질문 목록 페이지 또는 기본 화면이 떠야 함

---

## ✅ 관리자 페이지도 확인해 보세요

```
http://43.201.51.144/admin

```

→ `createsuperuser`로 만든 계정으로 로그인 가능

---

## 📌 요약: 서비스 상태 확인 명령들

| 목적 | 명령어 |
| --- | --- |
| Gunicorn 상태 확인 | `sudo systemctl status gunicorn` |
| Nginx 상태 확인 | `sudo systemctl status nginx` |
| Nginx 로그 확인 | `sudo tail -n 50 /var/log/nginx/error.log` |
| Gunicorn 재시작 | `sudo systemctl restart gunicorn` |

---

## `pybo.sock` 소켓 권한이 완벽한 상태인데도 여전히 **502 Bad Gateway** 오류가 발생한다면,

- **Gunicorn이 꺼져 있거나, .sock 파일이 Nginx가 기대하는 시점에 존재하지 않기 때문**
- 에러 메시지 확인

```jsx
502 Bad Gateway
nginx/1.18.0 (Ubuntu)
```

---

## ✅ 1. Gunicorn 서비스 상태 확인

- 먼저 Gunicorn이 **정상 실행 중인지** 확인:

```bash
sudo systemctl status gunicorn

```

- `active (running)` 이면 OK
- `inactive`, `failed`, 또는 `dead`이면 ❌ 문제 있음

---

## ✅ 2. Gunicorn 로그 확인

```bash
journalctl -u gunicorn -n 50 --no-pager

```

- → 에러 메시지 확인 (예: `ImportError`, `ModuleNotFoundError`, `Permission denied` 등)

---

## ✅ 3. Gunicorn 수동 재시작

- 다시 시작:

```bash
sudo systemctl restart gunicorn

```

---

## ✅ 4. Gunicorn 수동 실행 (테스트용)

- 만약 오류가 있다면 수동으로 Gunicorn 실행해서 디버그:

```bash
source ~/pybo/venv/bin/activate
cd ~/pybo
gunicorn --bind 127.0.0.1:8000 config.wsgi

```

- → 브라우저에서 http://<서버IP>:8000 으로 접속.
- (보안 그룹에서 포트 8000이 열려 있어야 함)

---

## ✅ 5. `.sock` 삭제 후 다시 생성 시도

- 가끔 `.sock` 파일이 고장날 수 있음. 아래 순서로 초기화:

```bash
sudo systemctl stop gunicorn
sudo rm /home/ubuntu/pybo/pybo.sock
sudo systemctl start gunicorn

```

- 이후 다시 권한 확인:

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

- → `srw-rw---- ubuntu www-data` 이어야 함.

---

## ✅ 6. 마지막 확인: Nginx 로그

```bash
sudo tail -n 50 /var/log/nginx/error.log

```

- → 최신 로그 기준으로 어떤 이유로 502가 뜨는지 확인 (예: 연결 실패, 권한 오류, 경로 오류)

---

## ⛳ 체크리스트

| 항목 | 확인 완료 여부 |
| --- | --- |
| Gunicorn 실행 중 (`systemctl status gunicorn`) | ☐ |
| `.sock` 존재 + 권한 (`srw-rw----`) | ✅ |
| `.sock` 경로 정확 (`proxy_pass http://unix:/...`) | ✅ |
| Nginx `include /etc/nginx/proxy_params;` | ✅ |
| Nginx 문법 확인 `nginx -t` 및 재시작 | ☐ |
| Nginx 로그 확인 | ☐ |

---

## ⛳ 체크리스트 : Gunicorn 실행 중 (`systemctl status gunicorn`)

```jsx
    Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
     Active: active (running) since Sat 2025-05-31 07:35:31 UTC; 1min 38s ago
   Main PID: 1576 (gunicorn)
      Tasks: 4 (limit: 476)
     Memory: 91.0M
        CPU: 725ms
     CGroup: /system.slice/gunicorn.service
             ├─1576 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             ├─1578 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             ├─1579 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>
             └─1580 /home/ubuntu/pybo/venv/bin/python3 /home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --wor>

May 31 07:35:31 ip-172-26-0-96 systemd[1]: Started Gunicorn daemon for pybo.
May 31 07:35:32 ip-172-26-0-96 gunicorn[1576]: [2025-05-31 07:35:32 +0000] [1576] [INFO] Starting gunicorn 23.0.0
May 31 07:35:32 ip-172-26-0-96 gunicorn[1576]: [2025-05-31 07:35:32 +0000] [1576] [INFO] Listening at: unix:/home/u>
May 31 07:35:32 ip-172-26-0-96 gunicorn[1576]: [2025-05-31 07:35:32 +0000] [1576] [INFO] Using worker: sync
May 31 07:35:32 ip-172-26-0-96 gunicorn[1578]: [2025-05-31 07:35:32 +0000] [1578] [INFO] Booting worker with pid: 1>
May 31 07:35:32 ip-172-26-0-96 gunicorn[1579]: [2025-05-31 07:35:32 +0000] [1579] [INFO] Booting worker with pid: 1>
```

- 로그를 보면 **Gunicorn은 정상적으로 실행 중이며 `.sock` 소켓도 잘 바인딩되어 있음**

```
Active: active (running)
...
Listening at: unix:/home/ubuntu/pybo/pybo.sock

```

- 문제는 **Nginx 설정이 Gunicorn 소켓과의 연결을 제대로 하지 못하고 있다는 것**

---

## ✅ 다음 조치: Nginx 설정 최종 점검

### 🔍 `/etc/nginx/sites-available/pybo` 파일을 **정확히** 아래와 같이 수정

```
server {
    listen 80;
    server_name 43.201.51.144;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        root /home/ubuntu/pybo;
    }

    location / {
        include /etc/nginx/proxy_params;
        proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
    }
}

```

> ❗ 핵심 수정: include proxy_params; → include **/etc/nginx/proxy_params;** (절대 경로)
> 

---

## ✅ 이후 작업

```bash
# 문법 확인
sudo nginx -t

# 문제 없으면 재시작
sudo systemctl restart nginx

```

---

## ✅ 확인 URL

- 브라우저에서:

```
http://43.201.51.144

```

> 정상이면 pybo 화면이 출력.
> 
> 
> 여전히 502이면 **에러 로그** 다시 확인:
> 

```bash
sudo tail -n 50 /var/log/nginx/error.log

```

---

## ✅ 마지막 요약 체크리스트

| 항목 | 상태 |
| --- | --- |
| Gunicorn 실행 중 (`.sock` 생성) | ✅ |
| `.sock` 권한 OK (`ubuntu:www-data`, 660) | ✅ |
| Nginx 설정 파일 `proxy_pass` 정확 | 🔄 절대경로로 재확인 필요 |
| Nginx include 절대경로 사용 | 🔄 반드시 `/etc/nginx/proxy_params` |
| Nginx 재시작 (`nginx -t`, restart) | 🔄 실행 필요 |
| 502 시 `error.log` 마지막 줄 확인 | ⏱️ |

---

## `502 Bad Gateway` 문제를 디버깅할 때 핵심 중 하나가 **Nginx에서 지정한 `.sock` 경로와 Gunicorn에서 실제로 생성한 `.sock` 경로가 정확히 일치하는가임**.

 

---

## ✅ 1. Gunicorn에서 사용하는 `.sock` 경로

- Gunicorn은 `.service` 파일에서 지정된 `--bind` 옵션을 사용하여 `.sock` 파일을 생성:

### 🔍 `/etc/systemd/system/gunicorn.service` 예시:

```
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 \
  --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application

```

→ Gunicorn은 **`/home/ubuntu/pybo/pybo.sock`** 파일을 생성

---

## ✅ 2. Nginx에서 참조하는 `.sock` 경로

- Nginx는 `proxy_pass`에서 이 소켓을 참조함

### 🔍 `/etc/nginx/sites-available/pybo` 예시:

```
location / {
    include /etc/nginx/proxy_params;
    proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
}

```

→ Nginx도 **`/home/ubuntu/pybo/pybo.sock`** 에 연결을 시도

---

## ✅ 두 경로가 정확히 같아야 함

| 비교 대상 | 경로 |
| --- | --- |
| Gunicorn socket bind | `/home/ubuntu/pybo/pybo.sock` |
| Nginx proxy_pass | `http://unix:/home/ubuntu/pybo/pybo.sock` |

✔️ 이 둘이 **완벽히 동일해야 연결됨**

---

## 🔍 실수 예시 (502 오류 발생):

| 실수 유형 | 설명 |
| --- | --- |
| Gunicorn: `/home/ubuntu/pybo/pybo.sock` |  |
| Nginx: `/tmp/pybo.sock` | ❌ 경로 불일치 |
| Gunicorn은 `.sock`을 만들었지만 Nginx가 잘못된 권한으로 접근 | ❌ Permission denied |
| Gunicorn 실행 전에 Nginx가 먼저 접근 시도 | ❌ 연결 실패 |

---

## ✅ 권장 디버깅 순서

1. `.sock` 파일 존재 확인
    
    ```bash
    ls -l /home/ubuntu/pybo/pybo.sock
    
    ```
    
2. Nginx 설정에서 정확히 해당 경로를 참조 중인지 확인
    
    ```bash
    sudo grep pybo.sock /etc/nginx/sites-available/*
    
    ```
    
3. Gunicorn `.service` 파일에서 `-bind unix:` 경로가 위와 일치하는지 확인

---

## 🔚 결론

- **Gunicorn과 Nginx 모두 정확히 같은 `.sock` 경로를 사용해야 하며**
- `.sock` 파일은 `ubuntu:www-data` 소유, `660` 권한이 있어야 하고
- Nginx 설정에선 반드시 `proxy_pass http://unix:/절대경로.sock;` 형식을 써야 함

---

## `sock` 파일의 **실제 생성 위치**를 찾는 3가지 방법

- Gunicorn이나 다른 서비스가 `.sock` 파일을 어디에 만들었는지 **확실하게 추적**.

---

## ✅ 1. Gunicorn 서비스 파일에서 `-bind` 경로 확인

```bash
sudo cat /etc/systemd/system/gunicorn.service

```

🔍 예시 출력:

```
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --bind unix:/home/ubuntu/pybo/pybo.sock ...

```

→ 이 줄이 **Gunicorn이 소켓 파일을 생성할 정확한 위치**

---

## ✅ 2. 실제 파일 존재 여부 확인 (`find` 명령)

- 전체 시스템에서 `.sock` 파일을 검색하려면:

```bash
sudo find / -type s -name "*.sock" 2>/dev/null

```

→ `.sock` 확장자를 가진 **모든 유닉스 도메인 소켓 파일**을 찾아줍니다.

예시 출력:

```
/home/ubuntu/pybo/pybo.sock
/var/run/docker.sock
...

```

> -type s는 유닉스 소켓만 찾겠다는 의미
> 

---

## ✅ 3. 프로세스별 `.sock` 열람 (`lsof` 명령)

- 실행 중인 Gunicorn 프로세스가 어떤 소켓을 사용하는지 확인하려면:

```bash
sudo lsof -U | grep gunicorn

```

- 또는 전체 유닉스 도메인 소켓 열람:

```bash
sudo lsof -U

```

- → 아래와 같이 Gunicorn이 열고 있는 `.sock` 경로를 확인

```
gunicorn  1576 ubuntu    5u  unix 0xffff880...   /home/ubuntu/pybo/pybo.sock

```

---

## 📌 요약

| 목적 | 명령어 |
| --- | --- |
| 서비스에서 지정한 경로 확인 | `sudo cat /etc/systemd/system/gunicorn.service` |
| 실제 파일 검색 | `sudo find / -type s -name "*.sock"` |
| 현재 열려 있는 소켓 확인 | `sudo lsof -U | grep gunicorn` |

---

## 확인 결과

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo find / -type s -name "*.sock" 2>/dev/null
/home/ubuntu/pybo/pybo.sock
```

```
/home/ubuntu/pybo/pybo.sock

```

- 이것이 **현재 Gunicorn이 생성한 실제 `.sock` 파일 경로**
- 이 경로를 기준으로 Nginx와 Gunicorn 설정이 정확히 일치해야 Django 서비스가 정상적으로 동작

---

## ✅ 최종 점검: 설정 일치 여부

### 🔹 Gunicorn `.service` 파일에서:

```
ExecStart=... --bind unix:/home/ubuntu/pybo/pybo.sock ...

```

### 🔹 Nginx 설정 파일 `/etc/nginx/sites-available/pybo` 에서:

```
location / {
    include /etc/nginx/proxy_params;
    proxy_pass http://unix:/home/ubuntu/pybo/pybo.sock;
}

```

- → 이 두 위치가 **완벽히 같아야 502 오류 없이 연결됩니다.**

---

## ✅ 권한 및 그룹 점검

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

→ 아래와 같아야 합니다:

```
srw-rw---- 1 ubuntu www-data ...

```

---

## ✅ Nginx 재시작

- 설정이 정확하다면 아래 명령어로 다시 시작:

```bash
sudo nginx -t
sudo systemctl restart nginx

```

---

## ✅ 브라우저 접속

```
http://43.201.51.144

```

→ 정상 출력되어야 함.

아직 502라면:

```bash
sudo tail -n 30 /var/log/nginx/error.log

```

→ 에러 로그를 보내주시면 남은 원인을 정밀 분석해 드릴 수 있습니다.

---

### 🎯  설정 경로가 정확하므로, 권한/시작 순서/설정 적용 문제만 정리되면 바로 접속 가능.

## 2025/05/31 07:30:10 [crit] 1504#1504: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied)

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo systemctl restart nginx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ sudo tail -n 30 /var/log/nginx/error.log
2025/05/31 01:58:17 [notice] 14178#14178: using inherited sockets from "6;7;"
2025/05/31 07:30:10 [crit] 1504#1504: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu/
pybo/pybo.sock:/", host: "43.201.51.144"
2025/05/31 07:31:01 [crit] 1504#1504: *4 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 104.167.221.114, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu
/pybo/pybo.sock:/", host: "43.201.51.144:80"
2025/05/31 07:32:37 [crit] 1521#1521: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu/
pybo/pybo.sock:/", host: "43.201.51.144"
2025/05/31 07:36:46 [crit] 1604#1604: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu/
pybo/pybo.sock:/", host: "43.201.51.144"
2025/05/31 07:36:47 [crit] 1604#1604: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu/
pybo/pybo.sock:/", host: "43.201.51.144"
2025/05/31 07:43:08 [crit] 1639#1639: *1 connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied) while conne
cting to upstream, client: 203.246.112.82, server: 43.201.51.144, request: "GET / HTTP/1.1", upstream: "http://unix:/home/ubuntu/
pybo/pybo.sock:/", host: "43.201.51.144"
```

- `nginx: configuration file ... test is successful` 확인에도 불구하고 여전히 `502 Bad Gateway`이며, `error.log`에 다음과 같은 에러가 반복되고 있음:

```
connect() to unix:/home/ubuntu/pybo/pybo.sock failed (13: Permission denied)

```

- 이는 **Gunicorn이 만든 `.sock` 파일에 대해 Nginx가 접근 권한이 없어서 발생하는 문제**

---

## ✅ 핵심 원인: `Permission denied` (권한 문제)

- `.sock` 파일 자체 권한은 이미 `srw-rw----`이고 소유자: ubuntu / 그룹: www-data 이므로, 다음 두 가지를 최종적으로 점검

---

## ✅ 1. Gunicorn이 생성한 `.sock` 파일 권한 다시 확인

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

- → 결과가 정확히 아래와 같아야 함

```
srw-rw---- 1 ubuntu www-data ...

```

✅ 이 상태면 통과

---

## ✅ 2. **Nginx가 `www-data` 유저로 실행 중인지 확인**

```bash
ps aux | grep nginx

```

- → 아래와 같은 줄이 있어야 함

```
www-data  ... nginx: worker process

```

✅ `www-data` 로 실행 중이면 통과

---

## ✅ 3. Gunicorn이 `.sock` 파일을 만들 때 **UMask**를 `007`로 설정했는지 확인

- `/etc/systemd/system/gunicorn.service` 파일을 열어 다음 줄이 있는지 확인:

```
UMask=0007

```

- 예시 전체:

```
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/pybo
ExecStart=/home/ubuntu/pybo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/pybo/pybo.sock config.wsgi:application
UMask=0007

```

- 변경했으면 다음 수행:

```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn

```

---

## ✅ 4. `/home/ubuntu/pybo` 폴더 접근 권한 확인

- 폴더 자체가 `www-data`가 접근 가능한지 확인:

```bash
namei -l /home/ubuntu/pybo/pybo.sock

```

출력 예:

```
f: /home/ubuntu/pybo/pybo.sock
drwxr-xr-x root   root   /
drwxr-xr-x ubuntu ubuntu home
drwxr-xr-x ubuntu ubuntu ubuntu
drwxr-xr-x ubuntu ubuntu pybo     ← 여기!
srw-rw---- ubuntu www-data pybo.sock

```

- → `pybo` 폴더가 `drwxr-xr-x`처럼 `others`에게도 `x`(실행/접근) 권한이 있어야 함.
- **만약 폴더 권한이 `drwx------` 등이라면 `www-data`가 `.sock` 파일까지 접근 못 함 → 502**

🔧 해결:

```bash
chmod o+x /home/ubuntu
chmod o+x /home/ubuntu/pybo

```

---

## ✅ 5. 다시 Nginx 재시작

```bash
sudo systemctl restart nginx

```

---

## ✅ 마지막으로 다시 접속 확인:

```bash
http://43.201.51.144

```

---

## ✅ 결론

`Permission denied`가 발생하는 가장 흔한 이유는:

| 원인 | 해결 방법 |
| --- | --- |
| `.sock` 권한 불충분 | `UMask=0007`, `chmod 660`, `chown ubuntu:www-data` |
| `.sock` 폴더 접근 권한 없음 | `chmod o+x /home/ubuntu`, `chmod o+x /home/ubuntu/pybo` |
| Gunicorn 그룹 미지정 | `Group=www-data` 설정 |
| Nginx가 다른 사용자로 실행 | `ps aux |

---

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ ls -l /home/ubuntu/pybo/pybo.sock
srw-rw---- 1 ubuntu www-data 0 May 31 07:35 /home/ubuntu/pybo/pybo.sock
(venv) ubuntu@ip-172-26-0-96:~/pybo$ ps aux | grep nginx
root        1638  0.0  0.5  55228  2200 ?        Ss   07:43   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_pro
cess on;
www-data    1639  0.0  1.3  55896  5912 ?        S    07:43   0:00 nginx: worker process
www-data    1640  0.0  1.2  55896  5144 ?        S    07:43   0:00 nginx: worker process
ubuntu      1731  0.0  0.5   7008  2304 pts/0    S+   07:44   0:00 grep --color=auto nginx
```

- ✅ `.sock` 권한: `srw-rw---- ubuntu www-data` → **정상**
- ✅ Nginx 실행 유저: `www-data` → **정상**
- ✅ Gunicorn 실행 중: 이전에 확인 완료
- 그런데도 `502 Bad Gateway`가 뜨는 이유는  **확실히 디렉터리 접근 권한 문제**

---

## 🔥 최종 핵심 원인: **폴더 접근 권한 부족**

### ✅ 확인: `.sock` 파일까지 접근 가능한지

```bash
namei -l /home/ubuntu/pybo/pybo.sock

```

- 예시 결과가 아래와 같아야 함

```
f: /home/ubuntu/pybo/pybo.sock
drwxr-xr-x root   root   /
drwxr-xr-x ubuntu ubuntu home
drwxr-xr-x ubuntu ubuntu ubuntu
drwxr-xr-x ubuntu ubuntu pybo     ← 여기가 중요!
srw-rw---- ubuntu www-data pybo.sock

```

### ❌ 만약 아래처럼 되어 있다면:

```
drwx------ ubuntu ubuntu pybo

```

- → `www-data`는 `.sock` 파일을 **접근조차 못 함** ⇒ `Permission denied` ⇒ 502 Bad Gateway

---

## ✅ 해결: 폴더 접근 권한 부여

```bash
chmod o+x /home/ubuntu
chmod o+x /home/ubuntu/pybo

```

- → 이렇게 하면 `www-data`가 해당 경로로 "접근" 가능해집니다. (`x` = 실행/진입 권한)

---

## 🔁 이후 재시작

```bash
sudo systemctl restart gunicorn
sudo systemctl restart nginx

```

---

## ✅ 다시 접속 확인

브라우저에서 접속:

```
http://43.201.51.144

```

---

## 🎯 정리

| 항목 | 상태 |
| --- | --- |
| `.sock` 경로와 설정 일치 | ✅ |
| `.sock` 권한 및 그룹 | ✅ |
| Nginx 유저 = www-data | ✅ |
| `UMask=0007` 적용 | ✅ |
| **폴더 접근 권한 (`o+x`)** | ❗ **지금 해야 함** |

---

```jsx
(venv) ubuntu@ip-172-26-0-96:~/pybo$ namei -l /home/ubuntu/pybo/pybo.sock
f: /home/ubuntu/pybo/pybo.sock
drwxr-xr-x root   root     /
drwxr-xr-x root   root     home
drwxr-x--- ubuntu ubuntu   ubuntu
drwxrwxr-x ubuntu ubuntu   pybo
srw-rw---- ubuntu www-data pybo.sock
```

- **문제가 명확히 드러남.** 🔍

```
drwxr-x--- ubuntu ubuntu ubuntu  ← ❗ 여기가 문제!

```

- `/home/ubuntu` 디렉토리의 권한이 **`750` (`rwxr-x---`)**
- 즉, **others (www-data 포함)** 는 `x` 접근권한이 없어 `.sock` 파일에 접근하지 못함
- 그래서 Nginx에서 **`Permission denied`** 로 502 오류 발생 중입니다.

---

## ✅ 해결 방법: `/home/ubuntu` 디렉토리에 접근 권한 부여

```bash
chmod o+x /home/ubuntu

```

이 명령은:

- others (`o`) 에게 **진입 가능(x)** 권한만 부여
- 읽기/쓰기 권한은 부여하지 않음 → 안전합니다

---

## 🔁 이후 서비스 재시작

```bash
sudo systemctl restart gunicorn
sudo systemctl restart nginx

```

---

## ✅ 브라우저 확인

```
http://43.201.51.144

```

→  `pybo` Django 웹페이지가 나타나야 함

---

## 📌 보안 참고

| 디렉토리 | 권장 권한 |
| --- | --- |
| `/home/ubuntu` | `751` 또는 `755` |
| `/home/ubuntu/pybo` | `755` 또는 그대로 `775` |
| `.sock` 파일 | `srw-rw----` (ubuntu:www-data) |

---

## Lightsail SSH에 접속한 후, Django `pybo` 서비스를 Gunicorn과 Nginx를 통해 다시 시작

---

## ✅ 1. Lightsail SSH 접속

```bash
ssh -i ~/LightsailDefaultKey-ap-northeast-2.pem ubuntu@<Public-IP>

```

예:

```bash
ssh -i ~/LightsailDefaultKey-ap-northeast-2.pem ubuntu@43.201.51.144

```

---

## ✅ 2. Gunicorn 서비스 다시 시작

```bash
sudo systemctl restart gunicorn

```

Gunicorn 서비스가 정상 작동하는지 확인:

```bash
sudo systemctl status gunicorn

```

→ `Active: active (running)`이면 정상 작동 중입니다.

---

## ✅ 3. Nginx 다시 시작

```bash
sudo systemctl restart nginx

```

또는 설정 확인 후 재시작:

```bash
sudo nginx -t && sudo systemctl restart nginx

```

---

## ✅ 4. 소켓 파일 권한 확인 (필수)

```bash
ls -l /home/ubuntu/pybo/pybo.sock

```

정상 예시:

```
srw-rw---- 1 ubuntu www-data ... pybo.sock

```

만약 권한이 부족하다면:

```bash
sudo chown ubuntu:www-data /home/ubuntu/pybo/pybo.sock
sudo chmod 660 /home/ubuntu/pybo/pybo.sock

```

---

## ✅ 5. 서비스 정상 확인

브라우저에서 접속:

```
http://43.201.51.144

```

정상 작동하지 않으면 에러 로그 확인:

- Gunicorn:
    
    ```bash
    journalctl -u gunicorn --no-pager --since "5 minutes ago"
    
    ```
    
- Nginx:
    
    ```bash
    sudo tail -n 30 /var/log/nginx/error.log
    
    ```
    

---

## ✅ Tip: 서버 재부팅 시 자동 시작 설정 (한 번만 수행)

```bash
sudo systemctl enable gunicorn
sudo systemctl enable nginx

```

---
