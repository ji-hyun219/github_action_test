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

<br />
<br />
<br />

### 2. Workflow 작성

Repository의 Actions 탭에서 `set up a workflow yourself` 버튼을 눌러 yml 파일을 작성합니다. -> 자동으로 .github/workflows/main.yml 생성됨

1. yml 파일 설명

```
on:
  push:
    branches: [ master ]

env:
  DOCKER_IMAGE: ghcr.io/${{ github.actor }}/trading
  VERSION: ${{ github.sha }}
  NAME: go_cicd
```

여러 브랜치에서 개발 후 master 브랜치에 push or 병합 될 때만 실행되도록 설정하고 Docker image 이름과 version, name을 미리 환경변수로 지정합니다.

- `DOCKER_IMAGE: ghcr.io/${{ github.actor }}/trading`: DOCKER_IMAGE 환경 변수는 Docker 이미지의 이름을 정의합니다. 이미지 이름은 `ghcr.io/${{ github.actor }}/trading`로 설정되어 있습니다. `${{ github.actor }}`는 GitHub Actions에서 현재 실행 중인 워크플로우의 작업을 수행하는 사용자 또는 액터의 이름을 나타냅니다. 따라서, 해당 이미지는 `ghcr.io/<액터 이름>/trading`의 형식으로 지정됩니다.

- `VERSION: ${{ github.sha }}`: VERSION 환경 변수는 Docker 이미지의 버전을 정의합니다. 버전은 `${{ github.sha }}`로 설정되어 있습니다. `${{ github.sha }}`는 GitHub 커밋의 고유 식별자인 커밋 SHA를 나타냅니다. 이렇게 설정된 버전은 커밋의 고유성을 기준으로 한 번의 빌드마다 다른 값을 가지게 됩니다.

- `NAME: go_cicd`: NAME 환경 변수는 Docker 컨테이너의 이름을 정의합니다. 해당 코드에서는 go_cicd로 설정되어 있으며, 컨테이너를 실행할 때 사용됩니다.

<br />
<br />

```
jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
```

Workflow는 다양한 job으로 구성되고 위 yml은 build와 deploy라는 job을 생성합니다. build는 5개의 step이 존재하고 deploy는 2개의 step이 존재한다.
name은 각 단계에 실행할 것을 의미합니다.
runs-on은 어떤 OS에서 실행될지를 지정합니다.

<br />

```
      - name: Check out source code
        uses: actions/checkout@v2
```

워크플로우에서 소스 코드를 사용할 수 있도록 해당 레포지토리의 코드를 가져온다는 역할

<br />

```
      - name: Set up docker buildx
        id: buildx
        uses: docker/setup-buildx-action@v1
```

build의 step 두 번째는 Docker Buildx(이미지 빌드를 위한 도구)를 설정하고 사용할 수 있는 환경을 구성

<br />

```
      - name: Login to ghcr
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}
```

build의 step 세 번째는 생성한 토큰을 이용해 GHCR(도커 허브 같은..)에 로그인하는 역할

<br />

```
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          builder: ${{ steps.buildx.outputs.name }}
          push: true
          tags: ${{ env.DOCKER_IMAGE }}:latest
```

build의 step 네 번째는 도커 이미지를 빌드하고 푸시하는 역할.
이때 앞서 설정한 Docker Buildx 도구를 사용하여 빌드를 수행

<br />

```
 deploy:
    needs: build
    name: Deploy
    runs-on: [ self-hosted, label-go ]
    steps:
      - name: Login to ghcr
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}
      - name: Docker run
        run: |
          docker stop ${{ env.NAME }} && docker rm ${{ env.NAME }} && docker rmi ${{ env.DOCKER_IMAGE }}:latest
          docker run -d -p 8080:5000 --name go_cicd --restart always ${{ env.DOCKER_IMAGE }}:latest
```

- `Deploy`는 GHCR에 로그인 후 저장되어 있는 Docker image를 이용해 `컨테이너를 실행`시키는 역할
- `Docker run`은 실행 중인 도커 컨테이너를 중지하고 이전 버전인 컨테이너와 이미지를 삭제 후 새로운 이미지로 컨테이너를 run하는 방식
- runs-on 중 `self-hosted`는 필수로 써야함(이 값을 설정해야 서버에 등록한 runner가 실행)
- 이 부분에서 `어느 Runner(EC2)`로 실행할지 지정해줘야 함

<br />
<br />
<br />
