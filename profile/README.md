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

```bash
+--------------------------------------------------+
|                 Windows Host                     |
+--------------------------------------------------+
| 1) Gradle Build                                  |
|    ./gradlew clean bootJar                       |
|                                                  |
| 2) JAR 생성                                      |
|    build/libs/app.jar                            |
|                                                  |
| 3) SCP 전송 (MobaXterm)                          |
|    scp build/libs/app.jar                        |
|        ubuntu@SERVER_IP:/home/ubuntu/app.jar     |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
|                 Ubuntu Server                    |
+--------------------------------------------------+
| /home/ubuntu/watch_app.sh                        |
|                                                  |
| - inotifywait 로 /home/ubuntu 감시               |
| - app.jar 변경 감지                              |
| - 기존 app.jar 프로세스 종료                     |
| - Port 7072 해제 대기                            |
| - 새 app.jar 재실행                              |
+--------------------------------------------------+
```

## 🧾 구현 세부 사항

```bash
#!/bin/bash

# ==========================================
# 기본 경로 및 파일 설정
# ==========================================
APP_DIR="/home/ubuntu"
APP_JAR="$APP_DIR/app.jar"
LOG_FILE="$APP_DIR/app.log"
PID_FILE="$APP_DIR/app.pid"
WATCH_LOG="$APP_DIR/watch_app.log"
PORT=7072

# ==========================================
# 애플리케이션 시작
# - app.jar를 백그라운드에서 실행
# - 실행 PID를 app.pid 파일에 저장
# - 실행 로그는 app.log에 누적
# - 동작 로그는 watch_app.log에 기록
# ==========================================
start_app() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] starting app" | tee -a "$WATCH_LOG"

    nohup java -jar "$APP_JAR" >> "$LOG_FILE" 2>&1 &
    APP_PID=$!
    echo "$APP_PID" > "$PID_FILE"

    echo "[$(date '+%Y-%m-%d %H:%M:%S')] started app pid=$APP_PID" | tee -a "$WATCH_LOG"
}

# ==========================================
# 애플리케이션 종료
# - PID 파일이 있으면 해당 PID를 읽어서 종료
# - 먼저 정상 종료 시도
# - 일정 시간 내 종료되지 않으면 강제 종료
# - 종료 후 PID 파일 삭제
# - PID 파일이 없더라도 실행 중인 app.jar 프로세스를 찾아 종료 시도
# ==========================================
stop_app() {
    if [ -f "$PID_FILE" ]; then
        PID=$(cat "$PID_FILE")

        if ps -p "$PID" > /dev/null 2>&1; then
            echo "[$(date '+%Y-%m-%d %H:%M:%S')] stopping app pid=$PID" | tee -a "$WATCH_LOG"
            kill "$PID"

            # 정상 종료 대기 (최대 20초)
            for i in {1..20}; do
                if ps -p "$PID" > /dev/null 2>&1; then
                    sleep 1
                else
                    break
                fi
            done

            # 종료되지 않으면 강제 종료
            if ps -p "$PID" > /dev/null 2>&1; then
                echo "[$(date '+%Y-%m-%d %H:%M:%S')] force killing app pid=$PID" | tee -a "$WATCH_LOG"
                kill -9 "$PID"
            fi

            # 프로세스가 완전히 사라질 때까지 대기
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

# ==========================================
# 포트 해제 대기
# - 기존 애플리케이션이 사용하던 7072 포트가
#   완전히 해제될 때까지 대기
# - 포트 충돌 방지를 위한 단계
# ==========================================
wait_for_port_release() {
    while ss -ltnp 2>/dev/null | grep -q ":$PORT "; do
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] waiting for port $PORT to be released..." | tee -a "$WATCH_LOG"
        sleep 1
    done
}

# ==========================================
# 애플리케이션 재시작
# - 기존 앱 종료
# - 포트 해제 대기
# - 잠시 대기 후 새 app.jar 실행
# ==========================================
restart_app() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] restarting app" | tee -a "$WATCH_LOG"
    stop_app
    wait_for_port_release
    sleep 1
    start_app
}

# ==========================================
# app.jar 존재 여부 확인
# - 파일이 없으면 에러 로그를 남기고 종료
# ==========================================
if [ ! -f "$APP_JAR" ]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $APP_JAR not found" | tee -a "$WATCH_LOG"
    exit 1
fi

# ==========================================
# 초기 실행 상태 확인
# - 이미 app.jar가 실행 중이면 해당 PID를 기록
# - 실행 중이 아니면 새로 시작
# ==========================================
if ! pgrep -f "java -jar $APP_JAR" > /dev/null 2>&1; then
    start_app
else
    RUNNING_PID=$(pgrep -f "java -jar $APP_JAR" | head -n 1)
    echo "$RUNNING_PID" > "$PID_FILE"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] app already running pid=$RUNNING_PID" | tee -a "$WATCH_LOG"
fi

# ==========================================
# 디렉토리 감시 시작 로그
# - /home/ubuntu 디렉토리를 감시 대상으로 설정
# ==========================================
echo "[$(date '+%Y-%m-%d %H:%M:%S')] watching directory: $APP_DIR" | tee -a "$WATCH_LOG"

# ==========================================
# app.jar 변경 감지
# - inotifywait 로 /home/ubuntu 디렉토리를 실시간 감시
# - close_write, modify, create, moved_to 이벤트 감지
# - 변경된 파일명이 app.jar 인 경우에만 재시작 수행
# ==========================================
while true; do
    CHANGED_FILE=$(inotifywait -e close_write,modify,create,moved_to --format '%f' "$APP_DIR")

    if [ "$CHANGED_FILE" = "app.jar" ]; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] detected change on app.jar" | tee -a "$WATCH_LOG"
        sleep 1
        restart_app
    fi
done
```


