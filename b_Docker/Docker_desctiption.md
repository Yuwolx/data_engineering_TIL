# 🐳 Docker 학습 총정리 — SSAFY 실습 기반 완전판

---

## 📘 0. Docker란 무엇인가?

> **Docker(도커)** 는 애플리케이션을 실행하기 위한 환경을
> **하나의 격리된 단위(컨테이너)** 로 패키징하고 배포할 수 있게 해주는 플랫폼입니다.

쉽게 말해,

> “📦 내가 만든 프로그램이 어디서든 똑같이 동작하도록 보장해주는 도구”입니다.

### 핵심 개념

| 개념                   | 설명                       |
| -------------------- | ------------------------ |
| **Image (이미지)**      | 실행 환경의 ‘설계도’, 일종의 템플릿    |
| **Container (컨테이너)** | 이미지를 실제로 실행한 인스턴스        |
| **Dockerfile**       | 이미지를 만드는 레시피 (환경 설정서)    |
| **Docker Compose**   | 여러 컨테이너를 한 번에 관리하는 설정 도구 |

---

## ⚙️ 1. Docker의 기본 구조

```
[ Dockerfile ] ─▶ [ Image ] ─▶ [ Container ]
   (레시피)        (설계도)        (실행 중인 앱)
```

* **하나의 이미지로 여러 컨테이너를 실행**할 수 있습니다.
* 컨테이너는 격리된 환경이지만, **호스트(WSL)와 네트워크·파일을 공유**할 수 있습니다.

---

## 🧩 2. 도커 정상 작동 확인 — “Hello World”

```bash
docker run hello-world
```

* `hello-world` 이미지를 Docker Hub에서 자동으로 가져와 실행합니다.
* 도커 엔진이 정상적으로 설치되었는지 확인하는 기본 테스트입니다.

---

## 🧱 3. Python 환경 이미지 만들기

### 📄 Dockerfile

```dockerfile
FROM python:3.10-slim
WORKDIR /workspace
COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

### 🧩 명령어

```bash
docker build -t python-lab .
docker run -it --name pycheck python-lab
```

* `build`: Dockerfile을 기반으로 이미지 생성
* `run`: 해당 이미지를 기반으로 컨테이너 실행
* `-it`: 터미널 상호작용 모드로 실행

✅ **결과**
컨테이너 내부에서 `print("Hello from Docker Python environment")`가 출력됩니다.

---

## 🧠 4. 컨테이너 명령어 정리

| 명령어                 | 설명                   |
| ------------------- | -------------------- |
| `docker ps`         | 현재 실행 중인 컨테이너 목록 확인  |
| `docker ps -a`      | 중지된 컨테이너 포함 전체 목록 확인 |
| `docker start <이름>` | 중지된 컨테이너 실행          |
| `docker stop <이름>`  | 실행 중인 컨테이너 중지        |
| `docker rm <이름>`    | 컨테이너 삭제              |
| `docker images`     | 보유 중인 이미지 목록         |
| `docker rmi <이미지명>` | 이미지 삭제               |

---

## 🌐 5. Docker Compose로 여러 컨테이너 관리

### 📄 docker-compose.yml

```yaml
version: "3"
services:
  web:
    image: nginx
    container_name: web
    ports:
      - "8080:80"

  api:
    image: alpine
    container_name: api
    command: ["sleep", "infinity"]
```

### 🧩 명령어

```bash
docker compose up -d
docker compose ps
docker compose down
```

✅ **결과**

* `web`(nginx)과 `api`(alpine)가 같은 네트워크에 연결됩니다.
* `ping web` 또는 `curl http://web` 명령으로 통신이 가능합니다.

💡 **핵심 개념**

> Compose는 여러 컨테이너를 하나의 네트워크에 연결해
> “하나의 서비스처럼” 관리하게 해주는 도구입니다.

---

## 🌍 6. Nginx 웹 서버 구성

### 📁 구조

```
compose-lab/
├── index.html
└── docker-compose.yml
```

### 📄 index.html

```html
<h1>Hello Docker Compose</h1>
```

### 📄 docker-compose.yml

```yaml
version: "3"
services:
  web:
    image: nginx:alpine
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html
    ports:
      - "8080:80"
```

### 🧩 실행

```bash
docker compose up -d
curl http://localhost:8080
```

✅ **결과**
→ HTML 페이지 내용이 출력되며, 도커로 웹 서버가 완성됩니다.

---

## 🚀 7. Flask 웹 애플리케이션 배포

### 📄 app.py

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Dockerized Flask!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 📄 requirements.txt

```
flask
```

### 📄 Dockerfile

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

### 🧩 실행 명령

```bash
docker build -t flask-docker-app .
docker run -d -p 5000:5000 --name flask-container flask-docker-app
```

✅ **결과**
브라우저에서 `http://localhost:5000` 접속 시
“Hello, Dockerized Flask!” 문구가 출력됩니다.

📘 **핵심**

> 컨테이너 내부 5000번 포트를 호스트의 5000번 포트로 연결하여
> 외부에서 Flask 서버에 접근할 수 있도록 설정합니다.

---

## 📂 8. 볼륨 마운트를 통한 로그 모니터링 (Log Watcher)

### 📁 구조

```
log-watcher/
├── app.log
├── watch_log.py
└── Dockerfile
```

### 📄 watch_log.py

```python
import time, sys

log_path = "/log/app.log"

with open(log_path, "r") as f:
    f.seek(0, 2)
    print("Started watching the log file...")
    while True:
        line = f.readline()
        if line:
            print("[LOG] " + line.strip())
            sys.stdout.flush()
        else:
            time.sleep(1)
```

### 📄 Dockerfile

```dockerfile
FROM python:3.10-slim
COPY watch_log.py /app/
WORKDIR /app
CMD ["python", "watch_log.py"]
```

### 🧩 명령어

```bash
docker build -t log-watcher .
docker run -v /home/ssafy/hw/data_engineering1_hw_2_4/log-watcher/app.log:/log/app.log log-watcher
```

### 🧪 로그 추가 테스트

```bash
echo "New log entry" >> app.log
```

✅ **결과**

```
Started watching the log file...
[LOG] New log entry
```

💡 **핵심 개념**

> 호스트의 파일(`app.log`)을 컨테이너 내부로 연결(-v 옵션)해
> 파일 변경을 컨테이너가 실시간으로 감시합니다.

📘 실제 **로그 수집기(Fluentd, Filebeat)** 와 동일한 방식입니다.

---

## 🧹 9. 컨테이너 및 이미지 관리

| 명령어                   | 설명           |
| --------------------- | ------------ |
| `docker ps`           | 실행 중 컨테이너 확인 |
| `docker stop <ID>`    | 실행 중 컨테이너 중지 |
| `docker rm <ID>`      | 중지된 컨테이너 삭제  |
| `docker images`       | 이미지 목록 확인    |
| `docker rmi <IMAGE>`  | 이미지 삭제       |
| `docker system prune` | 불필요한 리소스 정리  |

---

## 💾 10. WSL에서 윈도우로 파일 복사

```bash
cp -r /home/ssafy/hw/data_engineering1_hw_2_4/log-watcher /mnt/c/Users/SSAFY/Desktop/
```

✅ **결과**
→ 윈도우 바탕화면(Desktop)에 `log-watcher` 폴더가 복사됩니다.

---

## 🎯 정리 — 지금까지 배운 핵심 흐름

| 주제           | 핵심 개념             | 실습 예시            |
| ------------ | ----------------- | ---------------- |
| Docker 기본    | Image, Container  | hello-world      |
| Python 환경 구성 | Dockerfile 빌드     | python-lab       |
| 컨테이너 관리      | run, ps, stop, rm | pycheck          |
| Compose 네트워크 | 여러 컨테이너 연결        | nginx + alpine   |
| 웹 서버 배포      | 포트 매핑, HTML 노출    | nginx 8080       |
| Flask 서버     | Python 웹 실행       | flask-docker-app |
| Volume 마운트   | 파일 공유 / 실시간 로그    | log-watcher      |

---

## 🧠 한 문장으로 요약하자면

> “도커는 내가 만든 프로그램과 그 실행 환경을
> 한 박스(컨테이너) 안에 넣어 어디서나 똑같이 실행하게 하는 기술이며,
> Docker Compose는 그 박스 여러 개를 하나의 시스템처럼 다루는 도구입니다.”
