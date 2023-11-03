# Chapter 07. 도커 컴포즈로 분산 애플리케이션 실행하기

도커 컴포즈를 사용하면 여러 컨테이너에 걸쳐 실행되는 애플리케이션을 정의하고 관리할 수 있다.

## 7.1 도커 컴포즈 파일의 구조

Dockerfile 스크립트는 애플리케이션을 패키징하기 위한 스크립트이다.

분산 애플리케이션의 기준에서는 Dockerfile 스크립트는 애플리케이션의 한 부분을 패키징하는 수단에 지나지 않는다.

도커 컴포즈 파일에 애플리케이션의 구조를 정의하여 각 컴포넌트에 대한 Dockerfile 스크립트를 자동으로 실행할 수 있다.

도커 컴포즈 파일은 애플리케이션의 `원하는 상태`, 즉 모든 컨테이너들이 실행중일 때 어떤 상태를 가져야 하는지를 정의하는 파일이다.

`docker container run` 명령어로 컨테이너를 실행할 때 지정하는 모든 옵션들을 모아둔 파일이다.

```Docker
# 현재 도커 컴포즈 파일은 3개의 최상위 문(statement)으로 구성된다. 
# version: 도커 컴포즈 파일의 버전을 정의한다.
version: '3.7'

# services: 애플리케이션을 구성하는 모든 컴포넌트를 열거한다. 컨테이너가 아닌 서비스 개념을 단위로 정의한다.
services:
  # todo-web이란 이름으로 서비스를 만든다.
  todo-web:
    # 'diamol/ch06-todo-list' 이미지를 기반으로 컨테이너를 생성한다. 
    image: diamol/ch06-todo-list
    # 호스트의 8020번 포트와 컨테이너의 80번 포트를 연결한다.
    ports:
      - "8020:80"
    # 이 서비스를 app-net이라는 네트워크에 연결한다.
    networks:
      - app-net

# networks: 서비스 컨테이너가 연결된 모든 도커 네트워크를 정의한다.
networks:
  # app-net이라는 네트워크를 정의한다.
  app-net:
    # 외부에 'nat'라는 이름의 네트워크를 사용하도록 연결한다.
    external:
      name: nat
```

서비스 이름(todo-web) 아래로는 속성이 정의된다. 위에서 언급한 `docker container run` 명령어의 옵션들이 여기에 정의된다.

도커 컴포즈를 사용하기 위해서는 `docker-compose` 명령어를 사용해야 한다.

도커 컴포즈 파일은 애플리케이션의 소스 코드, Dockerfile 스크립트와 함께 형상 관리 도구(버전 관리 시스템)에 저장해야 한다.

> README 파일에는 애플리케이션의 이미지 이름이나 공개해야할 포트 등의 문서화가 필요없다.

## 7.2 도커 컴포즈를 활용해 여러 컨테이너로 구성된 애플리케이션 실행하기

```Docker
# 버전 3.7을 정의합니다.
version: '3.7'

# 서비스들을 정의합니다.
services:
  # 'accesslog'라는 이름의 서비스를 'diamol/ch04-access-log' 이미지로 생성하며, 'app-net' 네트워크에 연결됩니다.
  accesslog:
    image: diamol/ch04-access-log
    networks:
      - app-net

  # 'iotd'(Image Of The Day)라는 이름의 서비스를 'diamol/ch04-image-of-the-day' 이미지로 생성하며, 호스트의 80번 포트와 연결되고, 'app-net' 네트워크에 연결됩니다.
  iotd:
    image: diamol/ch04-image-of-the-day
    # 호스트의 포트는 지정하지 않음으로서 호스트 머신의 포트는 스케일업을 통해 컨테이너를 늘려도 충돌이 발생하지 않도록 합니다.
    ports:
      - "80"
    networks:
      - app-net

  # 'image-gallery'라는 이름의 서비스를 'diamol/ch04-image-gallery' 이미지로 생성하며, 호스트의 8010번 포트와 컨테이너의 80번 포트를 연결하고, 'accesslog'과 'iotd' 서비스에 의존성을 가지고 'app-net' 네트워크에 연결됩니다.
  image-gallery:
    image: diamol/ch04-image-gallery
    ports:
      - "8010:80"
    depends_on:
      - accesslog
      - iotd
    networks:
      - app-net

# 네트워크들을 정의합니다.
networks:
  # external이 deprecated 되어서 아래와 같이 수정했습니다.
  app-net:
    name: nat

# 네트워크들을 정의합니다.
# networks:
#   # 'app-net'이라는 이름의 네트워크를 정의하며, 외부에서 'nat'라는 이름의 네트워크를 사용하도록 설정됩니다.
#   app-net:
#     external:
#       name: nat
```

사실 위 명령어는 읽을 수 있으면 중요하지 않고, `detached mode`가 나오는 게 중요해보인다.

`docker-compose up --detach`명령어를 통하여 실행하게 됐을 때랑 비슷한 내용이 출력된다.

도커 컴포즈 파일에는 컨테이너의 설정과 이들이 어떻게 함께 동작하는지 정의되어 있다.

API 서비스는 무상태(stateless)이므로 스케일 아웃이 가능하다. 또한 웹 컨테이너가 API에 데이터를 요청하면 도커가 여러 개의 API 컨테이너 중 하나를 선택해 요청을 전달한다.

--- 

컴포즈로 애플리케이션을 중지할 경우 각 컨테이너의 목록이 표시되지만 애플리케이션을 재시작할 때는 서비스의 이름만 열거된다.

의존관계에 맞춰 서비스의 이름들이 열거되어 출력된다.

또한 새 컨테이너를 만드는 것이 아닌, 중지됐던 기존 컨테이너가 다시 실행되는 것을 알 수 있다.

도커 컴포즈 파일은 애플리케이션 정의에 의존하는 클라이언트 측 도구임을 잊지 말아야 한다.

## 7.3 도커 컨테이너 간의 통신

분산 애플리케이션의 모든 구성 요소는 컴포즈가 도커 컨테이너로 실행한다.

컨테이너는 도커 엔진으로부터 부여받은 자신만의 IP 주소를 가지고, 모두 같은 도커 네트워크로 연결되어 이 IP 주소를 통해 서로 통신할 수 있다.

컨테이너가 교체되면 IP 주소가 변경되지만 도커에서 DNS를 이용하여 서비스 디스커버리 기능을 제공한다.

DNS는 IP 주소를 도메인과 연결하는 기능을 제공하는 시스템이다. 도커에는 이런 DNS 서비스가 내장되어 있다.

컨테이너의 이름을 도메인 삼아 조회하여 해당 컨테이너의 IP 주소를 찾을 수 있다.

`nslookup` 명령어는 인자로 도메인을 지정하면 해당 도메인을 DNS 서비스에서 조회하고 결과를 출력한다.

결론적으로, 도커 컴포즈를 이용하면 컨테이너 간의 트랙픽을 고르게 분산할 수 있다.

> 도커 컴포즈는 로드 밸런싱 기능을 제공하지 않는다.
> 
> Docker Compose 자체는 로드 밸런싱 기능을 직접적으로 제공하지 않습니다. 대신 서비스 또는 컨테이너를 간단하게 스케일링 할 수 있게 해주며, 동일한 이미지를 사용하는 컨테이너들 사이를 자동으로 로드벨런싱하는 Docker 내장된 로드밸런서인 Docker Swarm을 통해 로드밸런싱 기능을 제공합니다.
> 
> 그런데 Docker Swarm의 로드밸런싱은 라운드 로빈 방식을 채택하고 있어, 사용자가 직접 트래픽 분배 비율이나 우선순위를 설정하는 것은 지원하지 않습니다.
> 
> 만약 복잡한 로드 밸런싱 규칙, 세션 유지(Sticky sessions), 또는 사용자 정의 트래픽 분배를 위한 기능이 필요하다면, Nginx나 HAProxy 등의 서드파티 로드밸런서를 컨테이너안에서 구성하여 사용해야 합니다.

## 7.4 도커 컴포즈로 애플리케이션 설정값 지정하기

애플리케이션 컨테이너와 데이터베이스 컨테이너를 따로 실행해보자

```Docker
version: "3.7" # Docker Compose 파일의 버전을 지정합니다.
# 서비스를 정의한다.
services: 
  todo-db: 
    image: diamol/postgres:11.5
    # 호스트(로컬 머신)와 컨테이너 간의 네트워크 포트 매핑을 정의한다.
    ports: 
      # 호스트의 5433 포트가 컨테이너의 5432 포트와 매핑됩니다.
      - "5433:5432" 
    networks:
      - app-net

  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - "8030:80"
    # 환경 변수를 정의한다.
    environment: 
      # 'Database:Provider'라는 이름의 환경 변수에 'Postgres' 값을 설정합니다.
      - Database:Provider=Postgres 
    # 컨테이너 간의 의존성을 선언합니다. 이 컨테이너는 'todo-db' 컨테이너가 먼저 실행된 후에 실행됩니다.
    depends_on: 
      - todo-db
    networks:
      - app-net
    secrets: # Docker 비밀번호를 사용하여 보안 정보를 제공합니다.
      - source: postgres-connection # 'postgres-connection'이라는 Docker비밀을 가져옵니다.
        target: /app/config/secrets.json # 그리고 이를 컨테이너 내에 '/app/config/secrets.json' 파일로 위치시킵니다.

networks: # 사용자 정의 네트워크를 정의하는 섹션입니다.
  app-net: # 'app-net'이라는 사용자 정의 네트워크를 생성하게 됩니다.

secrets: # Docker비밀을 정의하는 섹션입니다.
  postgres-connection: # 이 이동할 'postgres-connection' 비밀을 정의합니다.
    file: ./config/secrets.json # 현 로컬 파일 시스템에 있는 './config/secrets.json' 파일이 해당 비밀을 설정합니다.
```

## 7.5 도커 컴포즈도 만능은 아니다

도커 컴포즈는 분산 어플리케이션의 설정을 짧고 명료한 포맷의 파일로 나타낼 수 있게 해준다.

하지만 도커 컴포즈는 도커 스웜이나 쿠버네티스와 같은 완전한 컨테이너 플랫폼이 아니다.

애플리케이션이 지속적으로 정의된 상태를 유지하도록 하는 기능이 없다.

`docker-compose up`명령을 다시 실행하지 않는 이상 애플리케이션의 상태를 원래대로 되돌릴 수 없다.