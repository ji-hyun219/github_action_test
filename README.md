# github_action_test

Github Actions/Nuxtjs/Docker/EC2

https://www.youtube.com/watch?v=E3i9qt0SS-I&list=LL&index=1

<br />

## 전체적인 Flow

1. 로컬 PC 에서 개발을 한 후 Github 에 푸시
2. 특정 브랜치에 푸시가 되면 (=event trigger) 깃헙 액션이 동작되도록
3. 깃헙 액션이 Github Container Registry 에 소스를 받은 후 도커 이미지로 빌드
4. 빌드된 이미지를 EC2 에 등록된 Runner 가 복사
5. 기존 이미지를 삭제하고 새로운 이미지로 실행

<br />
<br />

### 1. 도커 파일 생성

도커 파일 생성 -> touch Dockerfile

<br />

`Dockerfile`

```
FROM node:14.19.0 : nodejs 14.19.0 버전의 이미지를 받아서 docker image를 생성합니다.

RUN mkdir -p /app : app 폴더를 생성합니다.
WORKDIR /app : app 폴더를 work directory 로 설정
ADD . /app/ : 현재 폴더의 모든것을 생성된 app 폴더에 복사합니다.

RUN rm ... || true : 파일 삭제. 뒤의 "|| true" 는 파일이 없을 경우 오류로 인해 실행이 중단되지 않게 하기 위해 추가합니다.
RUN yarn build : nuxt 프로젝트를 빌드합니다.

ENV HOST 0.0.0.0 : 모든 IP를 개방합니다.
EXPOSE 3000 : 3000번 포트를 개방합니다.

CMD ["yarn", "start"] : yarn start 명령을 실행합니다.
```

<br />
<br />

`.dockerignore`

Docker 이미지 생성시 불필요한 파일을 제외하도록 루트 폴더에 .dockerignore 파일을 생성합니다.

```
node_modules/
dist/
```

<br />
<br />

1. Dockerfile을 저장하고 다음 명령으로 이미지를 생성 합니다.

```
docker build --tag nuxt-auto-deploy:0.0.1 .
```

뒤의 마지막 . 은 이 프로젝트 전체를 의미합니다.

<br />

2. Docker 이미지를 실행합니다.

```
docker run --name nuxt-auto-deploy -d -p 3000:3000 nuxt-auto-deploy:0.0.1
```

- --name 옵션을 사용하여 컨테이너 이름을 `nuxt-auto-deploy` 로 지정합니다.
- 뒤의 nuxt-auto-deploy:0.0.1은 docker image명과 tag명입니다.
- -d 옵션 : 컨테이너를 `백그라운드 모드로 실행하도록 지시`하는 옵션입니다. 컨테이너를 백그라운드에서 실행하면 컨테이너의 출력이 터미널에 표시되지 않고 백그라운드에서 실행됩니다.

```
docker run -d `이미지명`
```

- -p 옵션은 호스트와 컨테이너 사이의 `포트 매핑을 설정`하는 옵션입니다. 예를 들어, -p 3000:3000 옵션을 사용하여 호스트의 3000번 포트와 컨테이너의 3000번 포트를 매핑하면 호스트의 3000번 포트로 접근하면 컨테이너의 3000번 포트로 연결됩니다.

```
docker run -p 3000:3000 `이미지명`
```

<br />

3. 도커 실행되는지 확인

```
docker ps -a
```
