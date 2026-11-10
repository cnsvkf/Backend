---
title: "Ready GSM - CI/CD 파이프 라인"
date: 2026-09-23
tags: []
velog_url: https://velog.io/@da2un/Ready-GSM-CICD-%ED%8C%8C%EC%9D%B4%ED%94%84-%EB%9D%BC%EC%9D%B8
---

Ready GSM에 `.github` 패키지 내에 있는 CI/CD 워크 플로우 코드와 흐름을 파악해서 정리해보겠습니다.

+ secret처럼 환경변수도 어떻게 관리되는지 정리하겠습니다.


`github/workflows`를 보면
- readygsm-prod-cd.yml()
- readygsm-stage-cd.yml()
두개로 나누어져있다
## readygsm-prod-cd.yml
#### 1. 빌드 준비 단계
`readygsm-prod-cd.yml()`을 보면 `Checkout`, `Set up JDK 21`, `Setup Node.js`, `Setup Gradle`, `Build JAR` 등으로 필요한 코드를 가져오고 도구를 설치하고 컴퓨터가 알아들을수있는 .jar 파일로 빌드한다
- master 브랜치에 올리면 자동으로 빌드
- 자바 21버전
- Node는 24 버전


#### 2. 서버(인프라) 준비
`Install Pulumi CLI`을 보면 Pulumi 명령어로 github 러너 컴퓨터에 설치합니다.
`Install infra dependencies`로 `infra` 폴더에 들어가서 `npm ci`를 실행합니다. Pulumi 코드가 TypeScript로 짜여있어서 그걸 돌리는데 필요한 패키지들을 설치해주는 과정입니다.

`Deploy infrastructure`에서 진짜로 서버를 만듭니다.
`infra/index.ts`에 코드로 적어놓은 EC2 서버 설정을 `pulumi up`으로 실제 AWS에 그대로 만들어줍니다. 마지막 줄에서 `pulumi stack output ec2InstanceId`로 방금 만든 EC2의 ID를 꺼내서 저장해두는데, 뒤에 나오는 `Deploy to EC2 via SSM` 단계에서 이 ID로 "어느 서버한테" 명령을 보낼지 정하는 것 같습니다.

#### 3. Docker 이미지 -> ECR 업로드
`Configure AWS credentials`, `Login to Amazon ECR`, `Build and push Docker image to ECR` 순서로 진행됩니다. 먼저 AWS 계정 인증(access key)을 하고, 이어서 Docker가 ECR 저장소에 접근할 수 있도록 로그인합니다. 그 다음 `docker/stage.Dockerfile`을 바탕으로 이미지를 빌드하는데, 이때 커밋 해시(github.sha)와 latest 두 가지 태그를 함께 붙입니다. 완성된 이미지는 두 태그 모두 ECR에 업로드(push)됩니다.

#### 4. Secret 동기화
`Sync secrets to SSM Parameter Store`에서 GitHub Secrets에 등록해둔 값들(DB 비밀번호, API 키 등)을 AWS SSM Parameter Store에 SecureString(암호화) 형태로 저장합니다. SSM으로 명령을 실행하면 명령어 내용이 로그에 그대로 남기 때문에, 비밀번호를 명령어에 직접 쓰지 않고 이렇게 따로 저장해두고 나중에 꺼내쓰는 방식을 쓰는 것 같습니다.

#### 5. 실제 배포
`Deploy to EC2 via SSM`에서 SSM으로 EC2한테 명령을 보냅니다. compose 파일이랑 nginx 설정 같은 것들을 EC2로 전달하고, EC2는 아까 SSM에 저장해둔 secret 값들을 다시 꺼내와서 `.env` 파일을 만든 다음, ECR에서 이미지를 받아와서 `docker compose up -d`로 컨테이너를 실행합니다.

## readygsm-stage-cd.yml
- develop 브랜치에 push되면 실행
- prod처럼 서버를 새로 만들지 않고, 이미 떠있는 고정 서버에 SSH로 직접 붙어서 파일 복사 + 재기동하는 더 단순한 구조

## secret 같은 환경 변수 관리하는법
secret같은 환경변수는 남에게 보여지면 안되는 것들입니다
예를 들어 api 키 같은것들이요.
저희 readygsm-prod-cd.yml에도 있습니다
```
with:

aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
aws-region: ${{ env.AWS_REGION }}
```
이런 유출이 되면 안되는 것들은 환경변수 처리를 하는데요.
> 그럼 실제로 저장되는 곳은 어딜까요?

코드 파일 안에는 ${{ secrets.AWS_ACCESS_KEY_ID }}처럼 이름표(변수명)만 적혀있고, 진짜 값(실제 access key 문자열)은
GitHub Secrets는 레포 저장소의 `Settings → Secrets and variables → Actions` 메뉴에 값을 저장해둔다.

# docker, deploy

`docker`, `deploy` 패키지 내에 있는 모든 파일을 조사하고, 어떤 흐름으로 작동하는지 정리해보겠습니다.

`docker`를 보면
- nginx-stage.conf
- nginx.conf
- stage.Dockerfile

`deploy`를 보면
- .certbot-env.example
- backup-mysql.sh
- compose.prod.yml
- compose.stage.yml
- renew-cert.sh

로 나누어져있다

## docker

#### stage.Dockerfile
자바 21이 설치된 기본 이미지 위에 빌드된 jar 파일 하나만 넣고 실행시키는 단순한 구조입니다. 이름은 stage지만 prod CD 워크플로우에서도 이 파일을 그대로 씁니다. jar를 실행시키는 작업 자체는 환경에 따라 달라질 이유가 없어서 파일을 따로 안 만들고 공유하는 것 같습니다.

#### nginx.conf / nginx-stage.conf
둘 다 nginx가 요청을 앱(`app:8080`)으로 전달(proxy_pass)해주는 설정입니다. 차이는
- `server_name`이 다릅니다 (`api.ready.hellogsm.kr` vs `api.stage.ready.hellogsm.kr`) - 도메인이 다르기 때문
- prod는 80번 포트로 들어오면 443(https)으로 강제 리다이렉트시키는데, stage는 80번도 그냥 http로 처리합니다
- 둘 다 `/api/v1/chat` 경로는 따로 location 블록을 나눠서 `proxy_buffering off`를 하고 있습니다. SSE(실시간 스트리밍) 응답이 버퍼에 모였다가 나가면 안 되기 때문인 것 같습니다.

## deploy

#### compose.prod.yml / compose.stage.yml
둘 다 nginx, certbot, app, mysql, redis 5개 컨테이너로 구성되어 있고 구조는 거의 같습니다. `depends_on`에 `condition: service_healthy`가 걸려있어서, mysql/redis가 헬스체크를 통과한 뒤에 app이, app이 뜬 뒤에 nginx가 뜨도록 순서가 강제되어 있습니다.

가장 큰 차이는 app을 만드는 방식입니다.
- prod: `image: ${PROD_ECR_IMAGE}` → CI/CD에서 이미 빌드해서 ECR에 올려둔 이미지를 그대로 받아와서 씁니다.
- stage: `build: dockerfile: docker/stage.Dockerfile` → 서버에서 코드로 직접 이미지를 빌드합니다.

이건 CI/CD 워크플로우에서 본 것과도 이어집니다. prod 워크플로우엔 ECR에 이미지를 push하는 단계가 있었지만, stage는 그런 단계 없이 파일만 복사해서 서버에서 바로 빌드하는 구조였습니다.

각 서비스의 비밀번호 값들은 `${PROD_MYSQL_PASSWORD}`처럼 환경변수에서 읽어오게 되어있는데, prod는 CI/CD에서 SSM으로 값을 꺼내와 `.env` 파일을 만들어 쓰고, stage는 오히려 `.env` 파일을 `rm -f .env`로 지운 다음 SSH 세션에서 export해둔 환경변수를 그대로 참조합니다.

#### .certbot-env.example
`renew-cert.sh`가 필요로 하는 환경변수(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)의 형식만 보여주는 예시 파일입니다. 진짜 값이 든 `.certbot-env`는 커밋되지 않고 서버에만 따로 존재합니다.

#### renew-cert.sh
stage 서버에서 cron으로 정기 실행되는 SSL 인증서 자동 갱신 스크립트입니다. certbot으로 인증서를 갱신하고, 끝나면 nginx를 reload해서 새 인증서를 적용합니다.

#### backup-mysql.sh
운영 DB를 mysqldump로 덤프해서 압축한 뒤 AWS S3에 백업으로 올리는 스크립트입니다.

## 전체 흐름
CI/CD 워크플로우가 서버에 compose 파일 + nginx 설정 + .env(secret 값들, stage는 제외)를 전달하면, 서버에서 `docker compose up -d`로 mysql/redis -> app -> nginx 순서로 컨테이너들이 뜨면서 서비스가 시작됩니다. prod는 미리 빌드된 이미지를 받아서 실행하고, stage는 서버에서 직접 빌드해서 실행한다는 점이 가장 큰 차이입니다.

### 이해에 도움이 되는 용어들
GitHub Actions 워크플로우란?
- 배포를 정해놓은 순서대로 자동으로 실행해주는 것
Ready GSM Github Actions 워크플로우에서는 13단계를 거쳐서 배포해준다


ECR은?
Docker image를 저장해놓는 창고
ECR에 Docker 이미지를 올려놓고 서버에서 "받아와서" 실행시킨다.

Pulumi
서버를 코드로 적어서 만드는 방식 -> IaC
(Pulumi는 그 방식중 하나)

SSM은?
- 일반적으로 서버에 접속해서 명령어를 실행하려면 보통 SSH를 쓰는데
Ready GSM은 컴퓨터에서 SSH포트로 막혀있는 문제가 있었다고 합니다.
그래서 AWS SSM RunCommand라는 걸로 SSH 접속 없이 AWS로 원격 접속해 서버 배포

