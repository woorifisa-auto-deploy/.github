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
