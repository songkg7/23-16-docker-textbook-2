# Chapter 08. 헬스 체크와 디펜던시 체크로 애플리케이션의 신뢰성 확보하기

컨테이너에서 실행 중인 애플리케이션을 운영 환경에 맞게 다듬기를 시작한다.

애플리케이션을 패키징하고, 컨테이너에서 실행하고, 도커 컴포즈로 여러 컨테이너에 걸쳐 애플리케이션을 실행해봤다.

운영 환경에서는 도커 컴포즈가 아닌, 도커 스웜이나 쿠버네티스 같은 컨테이너 플랫폼 상에서 애플리케이션을 실행하게 된다.

7장 마지막에서 이러한 컨테이너 플랫폼은 애플리케이션이 스스로 이상 상태에서 회복할 수 있도록 해 주는 기능을 제공한다고 했다.

또한 플랫폼이 컨테이너에서 실행 중인 애플리케이션 상태가 정상인지 확인할 수 있는 정보를 이미지에 함께 패키징 할 수도 있다.

이를 통하여 새 컨테이너로 비정상 상태인 컨테이너를 대체할 수 있다.

## 8.1 헬스 체크를 지원하는 도커 이미지 빌드하기

도커는 컨테이너를 시작할 때마다 특정한 프로세스가 실행되는데, 해당 프로세스가 제대로 실행중인지 상태를 체크한다.

단순히 프로세스가 실행 상태인지만을 체크하는 것을 넘어서 애플리케이션이 정상 상태인지를 체크할 수도 있다.

Dockerfile 스크립트에 상태 확인을 위한 로직을 추가하면 된다.

Dockerfile v2 에서는 app image 부분에 `HEALTHCHECK CMD curl --fail http://localhost/health`을 추가한다.

헬스 체크 시에는 엔드포인트 `/health`로 요청을 보낸다. `--fail` 옵션을 붙이면 curl이 전달 받은 상태 코드를 도커에 전달한다.

`Dockerfile`이라는 스크립트 파일명이 다를 땐, build 부 명령에 사용하려는 파일명을 `-f` 옵션으로 지정해야 한다.

헬스 체크 결과가 실패로 나오더라도 컨테이너는 여전히 실행 중(running) 상태이다. 

재시작 하거나 다른 컨테이너로 교체하지 않는 이유는, 도커는 이런 작업을 안전하게 처리할 수 없기 때문이다.

다른 컨테이너 플랫폼을 사용하면, 도커의 헬스 체크 기능을 활용하여 컨테이너의 이상 상태를 감지하고, 새 컨테이너로 대체할 수 있다.

## 8.2 디펜던시 체크가 적용된 컨테이너 실행하기

의존 관계를 점검하는 디펜던시 체크 기능을 도커 이미지에 추가할 수 있다.

실행 시점이 헬스 체크와는 다르다.

모든 요구 사항을 확인하는 기능으로, 만족하지 못하는 요구 사항이 있다면 디펜던시 체크가 실패하고 애플리케이션이 실행되지 않는다.

`CMD curl --fail http://numbers-api/rng && \dotnet Numbers.Web.dll` 

위 명령어 처럼, `&&` 연산자를 사용하여 앞의 명령어가 성공하면 뒤의 명령어를 실행하도록 한다.

외부 도구에 의존할 필요 없이 애플리케이션 자체에서 디펜던시 체크를 수행할 수 있다.

## 8.3 애플리케이션 체크를 위한 커스텀 유틸리티 만들기

curl은 API를 테스트하는데 유용하지만 보안 정책상의 이유로 이미지에 curl을 포함시킬 수 없을 수 있다.

```Docker
# 이 이미지는 .NET SDK를 포함한 diamol/dotnet-sdk 이미지를 기반으로 합니다. 
# 'builder'라는 별칭을 사용하여 이 단계를 참조할 수 있습니다.
FROM diamol/dotnet-sdk AS builder 

# 작업 디렉토리를 /src로 설정합니다.
WORKDIR /src 

# Numbers.Web 프로젝트 파일을 이미지에 복사합니다.
COPY src/Numbers.Web/Numbers.Web.csproj . 

# .NET 의존성 복원을 수행합니다.
RUN dotnet restore 

# 프로젝트의 모든 파일을 이미지에 복사합니다.
COPY src/Numbers.Web/ . 

# Release 구성으로 .NET 프로젝트를 빌드하고 출시합니다. 
# 빌드 결과물은 /out 디렉토리에 위치합니다.
RUN dotnet publish -c Release -o /out Numbers.Web.csproj 

# 두 번째 빌드 단계를 정의합니다. 이 단계는 http-check-builder라는 별칭을 가집니다.
# 이 단계는 HTTP 체크 유틸리티를 빌드하는 데 사용됩니다.
FROM diamol/dotnet-sdk AS http-check-builder 

WORKDIR /src
COPY src/Utilities.HttpCheck/Utilities.HttpCheck.csproj .
RUN dotnet restore
COPY src/Utilities.HttpCheck/ .
RUN dotnet publish -c Release -o /out Utilities.HttpCheck.csproj

# 최종 어플리케이션 이미지를 정의합니다. 이 이미지는 diamol/dotnet-aspnet로 시작합니다.
FROM diamol/dotnet-aspnet 

# 환경 변수를 설정하여 숫자 API의 URL을 정의합니다.
ENV RngApi__Url=http://numbers-api/rng 

# 컨테이너가 시작할 때 실행할 명령을 정의합니다. 
# HTTP 체크 유틸리티를 사용하여 API가 사용 가능한지 확인하고, 그 후에 웹 애플리케이션을 실행합니다.
# -t 옵션은 HTTP 체크 유틸리티가 API가 사용 가능한지 확인하는 데 사용할 시간을 초 단위로 지정합니다.
# -c 옵션은 애플리케이션과 같은 설정 파일을 읽어 대상 URL을 지정합니다.
CMD dotnet Utilities.HttpCheck.dll -c RngApi:Url -t 900 && \dotnet Numbers.Web.dll

# 작업 디렉토리를 /app로 설정합니다.
WORKDIR /app 

# 위의 빌드 단계에서 생성된 결과물을 /app 디렉토리로 복사합니다.
COPY --from=http-check-builder /out/ .
COPY --from=builder /out/ .
```

커스텀 테스트 유틸리티를 만드는 것은 이미지의 이식성을 높이는 데 도움이 된다.

## 8.4 도커 컴포즈에 헬스 체크와 디펜던시 체크 정의하기

## 8.5 헬스 체크와 디펜던시 체크로 복원력 있는 애플리케이션을 만들 수 있는 이유

디펜던시 체크와 헬스 체크를 도입하면 처음부터 플랫폼이 실행 순서를 보장하게 할 필요가 없다.

플랫폼이 애플리케이션의 일시적인 오류를 해소한다.


