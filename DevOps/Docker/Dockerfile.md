# Dockerfile

## Dockerfile이란
Docker 이미지는 Dockerhub을 통해 다운받아서 사용할 수 있다. Dockerhub에 있는 이미지들도 누군가 만들어서 업로드 해놓은 것이다.  
이미지는 Dockerfile을 활용해서 만들 수 있다.

예를 들어서 내가 만든 Spring boot 프로젝트를 Docker 이미지로 만들고 싶다면 Dockerfile 을 활용하면 된다.

즉, Dockerfile은 **Docker 이미지를 만들게 해주는 파일**이다.


## Dockerfile에 쓰이는 명령어

### FROM: 베이스 이미지 생성

#### 의미
**FROM**은 베이스 이미지를 생성하는 역할을 한다. 즉 Dockerfile로 이미지를 만들 때 기반이 되는 이미지를 세팅하는 것이다.  

예를 들어서 Spring boot 프로젝트를 배포하기 위한 이미지를 만든다면 컨테이너에는 jdk가 이미 깔려 있어야 프로젝트를 실행시킬 수 있다.  
그래서 베이스 이미지로 jdk를 선택하는 것이다.

#### 문법
```bash
FROM [이미지명]
FROM [이미지명]:[태그명]
```
- 태그명을 명시하지 않으면 최신(latest) 버전을 사용한다.

#### 예시

1. **Dockerfile**
```bash
# JDK 17
FROM openjdk:17-jdk
```

2. **Dockerfile 기반으로 이미지 만들기**

```bash
# docker build -t [이미지명]:[태그명] [Dockerfile이 존재하는 디렉터리 경로]

docker build -t my-jdk17-server .
```
- `-t`는 태그명을 의미하는 것으로 이미지명과 태그명을 생성해준다.
- 이미지명만 작성하면 태그명은 **latest**로 표시된다.
- `-t`를 생략하면 이미지명과 태그명이 `<none>`으로 표시된다.

3. 컨테이너 내부에서 jdk가 잘 깔렸는지 확인하는 방법
```bash
FROM openjdk:17-jdk

ENTRYPOINT ["/bin/bash", "-c", "sleep 500"] 
```
- `FROM`만 작성할 경우 컨테이너 실행 즉시 종료돼서 컨테이너 내부를 확인할 수 없다.
- 컨테이너 실행 후 시작되는 명령어를 정의하는 ENTRYPOINT를 사용해서 해당 명령어를 작성해주면 500초 동안 서버가 정지된다.
  그래서 500초 동안은 컨테이너 내부로 들어가 수 있다.
- 컨테이너 내부로 들어가서 `java -version`을 입력하면 올바르게 설치되어 있음을 확인할 수 있다.

> ubuntu같은 것도 Dockerfile로 만들어졌을텐데 태초의 이미지?느낌이라 FROM에 무엇을 쓸 지 궁금했는데 이러한 이미지들은 `FROM scratch`로 시작했을 가능성이 높다고 한다.  
> `scratch`는 빈 상태라는 것을 의미한다.  
> 솔직히 Dockerfile을 까보기 전까진 알 수 없지만, `FROM`을 비워둘 수 없다는 점에서 `scratch`로 시작했을 것 같다. 확실하진 않다.

### COPY: 파일 복사
`COPY`는 호스트 컴퓨터에 있는 파일을 복사해서 컨테이너로 전달한다.

#### 사용법
```bash
# 문법
COPY [호스트 컴퓨터에 있는 복사할 파일의 경로] [컨테이너에서 파일이 위치할 경로]

# 예시
COPY aaa.txt /bbb.txt # 현재 경로에 있는 aaa.txt파일을 컨테이너 루트 경로에 bbb.txt라는 이름으로 복사

COPY my-app /container-app # 컨테이너안에 container-app이라는 폴더를 만들어서 my-app 폴더 내의 폴더와 파일들을 옮긴다. 

COPY *.txt /text-files/ # 현재 경로의 모든 txt확장자를 가진 파일을 text-files폴더로 이동, text-files 뒤에 "/" 필수
```

#### .dockerignore: 특정 파일 제외하고 COPY
1. .dockerignore 파일 만들기
```bash
readme.txt
```
2. Dockerfile 만들어서 이미지 생성 및 컨테이너 실행
```bash
FROM ubuntu

COPY ./ /  # 현재 경로의 모든 폴더와 파일을 컨테이너로 복사

ENTRYPOINT ["/bin/bash", "-c", "sleep 500"] # 디버깅용 코드 
```
- readme.txt 빼고 복사된다.

### ENTRYPOINT: 컨테이너가 시작할 때 실행 되는 명령어
`ENTRYPOINT`는 컨테이너가 생성되고 최초로 실행할 때 수행되는 명령어를 뜻한다.

```bash
FROM ubuntu

ENTRYPOINT ["/bin/bash", "-c", "echo hello"]
```

### RUN: 이미지를 생성하는 과정에서 사용할 명령문 실행
이미지 생성 과정에서 명령어를 실행시켜야 할 때 사용한다.

#### `RUN` vs `ENTRYPOINT`

`RUN` 명령어와 `ENTRYPOINT` 명령어가 헷갈릴 때가 있다. 둘 다 같이 명령어를 실행시키기 때문이다.  
하지만 엄연히 둘의 사용 용도는 다르다. `RUN`은 ‘**이미지 생성 과정**’에서 필요한 명령어를 실행시킬 때 사용하고, `ENTRYPOINT`는 생성된 이미지를 기반으로 **컨테이너를 생성한 직후에** 명령어를 실행시킬 때 사용한다.
**즉, `RUN`은 빌드 시 실행 되고, `ENTRYPOINT`는 컨테이너가 실행될 때 명령이 실행된다.**

#### 예제
```bash
FROM ubuntu

RUN apt update && apt install -y git

ENTRYPOINT ["/bin/bash", "-c", "sleep 500"]
```
- ubuntu 베이스 이미지 위에서 git을 설치한다. 그리고 실행되면 500초동안 중지된다.

### WORKDIR: 작업 디렉토리 지정
`WORKDIR`로 작업 디렉토리를 지정하면 그 이후에 등장하는 모든 `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD` 명령문은 해당 디렉터리를 기준으로 실행된다

#### 사용법
```bash
# 문법
WORKDIR [작업 디렉토리로 사용할 절대 경로]

# 예시
WORKDIR /usr/src/app
```
- "/"이 아니라 /usr/src/app 경로를 기준으로 작업이 발생한다.

### EXPOSE: 컨테이너 내부에서 사용 중인 포트를 문서화하기
`EXPOSE`는 컨테이너 내부에서 어떤 포트에 프로그램이 실행되는 지를 문서화하는 역할만 한다.  
어떠한 기능을 하는 것이 아니라 주석의 역할을 한다.

#### 사용법
```bash
# 문법
EXPOSE [포트 번호]

# 예시
EXPOSE 3000
```