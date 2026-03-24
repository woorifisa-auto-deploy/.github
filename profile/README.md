# Spring Boot 자동 배포 환경 구축 (inotify-tools 활용)


## 팀 Two-Face
|<img src="https://github.com/Federico-15.png" width="120"/> | <img src="https://github.com/Sungjun24s.png" width="120"/>|
|:--:|:--:|
| [**류승환**](https://github.com/Federico-15) | [**박성준**](https://github.com/Sungjun24s) |

로컬(Windows) 환경에서 개발 및 빌드한 Spring Boot 애플리케이션(`app.jar`)을 운영 서버(Ubuntu)에 전송하면, 서버가 파일 변경을 자동 감지하여 기존 프로세스를 종료하고 새 버전을 재실행하는 배포 파이프라인입니다.


<br/>

## 🛠 환경 및 사전 요구사항
- **Local OS:** Windows (IDE: SpringToolsForEclipse (STS))
- **Server OS:** Ubuntu Linux
- **Java Version:** Java 17
- **필수 패키지 (Ubuntu):** `inotify-tools` (파일 변경 감지용)


<br/>

### 필수 패키지 설치 명령어 (Ubuntu)
```bash
sudo apt-get update
sudo apt-get install inotify-tools
```

<br/>

## 📁 디렉토리 구조
서버의 작업 디렉토리는 /home/ubuntu/을 기준으로 합니다.
```
/home/ubuntu/
├── app.jar
├── watch_app.sh
├── app.log
├── watch_app.log
├── watch_runner.log
└── app.pid
```

## ⚙️ 환경설정


## 🏗️ 아키텍처
```text
[Windows Host]
   │
   │  (MobaXterm로 app.jar 업로드)
   ▼
[Ubuntu Server : /home/ubuntu/app.jar]
   │
   │  inotifywait 가 디렉터리 변경 감지
   ▼
[watch_app.sh]
   │
   ├── 기존 app.jar 프로세스 종료
   ├── 7072 포트 해제 대기
   └── 수정된 app.jar 재실행
   ▼
[Spring Boot App Running on Port 7072]
```
## 🧾 구현 세부 사항
```
#!/bin/bash

APP_DIR="/home/ubuntu"
APP_JAR="$APP_DIR/app.jar"
LOG_FILE="$APP_DIR/app.log"
PID_FILE="$APP_DIR/app.pid"
WATCH_LOG="$APP_DIR/watch_app.log"
PORT=7072

start_app() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] starting app" | tee -a "$WATCH_LOG"

    nohup java -jar "$APP_JAR" >> "$LOG_FILE" 2>&1 &
    APP_PID=$!
    echo "$APP_PID" > "$PID_FILE"

    echo "[$(date '+%Y-%m-%d %H:%M:%S')] started app pid=$APP_PID" | tee -a "$WATCH_LOG"
}

stop_app() {
    if [ -f "$PID_FILE" ]; then
        PID=$(cat "$PID_FILE")

        if ps -p "$PID" > /dev/null 2>&1; then
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] stopping app pid=$PID" | tee -a "$WATCH_LOG"
            kill "$PID"

            for i in {1..20}; do
                if ps -p "$PID" > /dev/null 2>&1; then
                    sleep 1
                else
                    break
                fi
            done

            if ps -p "$PID" > /dev/null 2>&1; then
                echo "[$(date '+%Y-%m-%d %H:%M:%S')] force killing app pid=$PID" | tee -a "$WATCH_LOG"
                kill -9 "$PID"
            fi

            while ps -p "$PID" > /dev/null 2>&1; do
                sleep 1
            done
        fi

        rm -f "$PID_FILE"
    else
        OLD_PID=$(pgrep -f "java -jar $APP_JAR")
        if [ -n "$OLD_PID" ]; then
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] stopping existing app pid=$OLD_PID" | tee -a "$WATCH_LOG"
            kill "$OLD_PID"
            sleep 2

            if ps -p "$OLD_PID" > /dev/null 2>&1; then
                kill -9 "$OLD_PID"
            fi
        fi
    fi
}

wait_for_port_release() {
    while ss -ltnp 2>/dev/null | grep -q ":$PORT "; do
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] waiting for port $PORT to be released..." | tee -a "$WATCH_LOG"
        sleep 1
    done
}

restart_app() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] restarting app" | tee -a "$WATCH_LOG"
    stop_app
    wait_for_port_release
    sleep 1
    start_app
}

if [ ! -f "$APP_JAR" ]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $APP_JAR not found" | tee -a "$WATCH_LOG"
    exit 1
fi

if ! pgrep -f "java -jar $APP_JAR" > /dev/null 2>&1; then
    start_app
else
    RUNNING_PID=$(pgrep -f "java -jar $APP_JAR" | head -n 1)
    echo "$RUNNING_PID" > "$PID_FILE"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] app already running pid=$RUNNING_PID" | tee -a "$WATCH_LOG"
fi

echo "[$(date '+%Y-%m-%d %H:%M:%S')] watching directory: $APP_DIR" | tee -a "$WATCH_LOG"

while true; do
    CHANGED_FILE=$(inotifywait -e close_write,modify,create,moved_to --format '%f' "$APP_DIR")

    if [ "$CHANGED_FILE" = "app.jar" ]; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] detected change on app.jar" | tee -a "$WATCH_LOG"
        sleep 1
        restart_app
    fi
done
```


