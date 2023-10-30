# Chapter 7, 8

## Docker Compose

### 들어가기 전에

- [[Docker Compose]] 가 V2 로 업데이트됨[^1]에 따라서 deprecated 된 `docker-compose` 명령은 `docker compose` 로 대체하여 설명합니다.

### Docker Compose?

- 여러 컨테이너를 묶어 하나로 관리하기 용이하게 하는 방법
- 모든 컴포넌트가 어떤 '상태'로 동작해야 하는지를 설명해주는 파일
- 필요한 모든 도커 객체를 한 번에 생성할 수 있다
- 컨테이너 간 통신이 용이하다

### Docker Compose 의 관리

- 도커 컴포즈는 컨테이너를 관리하는 별도의 명령이지만 내부적으로는 도커 API 를 사용
- 컴포즈가 실행된 이후, `docker-compose.yml` 파일을 수정하더라도 컴포즈는 이를 인지하지 못함
- 컴포즈는 yaml 파일에 강하게 의존하는 클라이언트 측 도구로, 컨테이너 리소스는 컴포즈 파일을 통해 관리해야 한다

### 컨테이너 간의 통신

- 기본적으로 컨테이너는 생성시에 부여되는 가상 IP 로 통신할 수 있다.
- 하지만 IP 주소는 컨테이너의 라이프사이클에 따라 계속 변화하기 때문에 활용이 어렵다.
- 도커는 같은 네트워크에 포함되어 있다면 DNS 를 사용할 수 있다
- 도커는 DNS 조회 순서를 변화시키는 방법으로 로드밸런싱을 구현한다

#### 실습

 ```bash
docker compose up -d --scale iotd=3
```

![[Pasted image 20231026162715.png]]

```bash
$ docker exec -it image-of-the-day-image-gallery-1 sh
/web # nslookup accesslog
nslookup: can't resolve '(null)': Name does not resolve

Name:      accesslog
Address 1: 192.168.228.2 image-of-the-day-accesslog-1.nat
```

컨테이너 이름으로 `nslookup` 기능이 동작하는 것을 볼 수 있다. 

> [!warning] DNS 가 항상 가능한 것은 아니다
> 컴포즈로 인하여 이미 같은 도커 네트워크(nat)에 존재하기 때문에 이 기능이 동작하는 것이고, 만약 도커 네트워크를 사용하지 않는다면 DNS 는 동작하지 않는다.

#### 로드밸런싱

하나의 도메인에 대해 DNS 결과에 여러 개의 IP 주소가 나올 수 있다. 도커의 DNS 시스템은 조회 결과의 순서를 매번 변화시키는 방식으로 트래픽을 분산시킨다.

```bash
$ docker exec -it image-of-the-day-image-gallery-1 sh
/web # nslookup iotd
nslookup: can't resolve '(null)': Name does not resolve

Name:      iotd
Address 1: 192.168.228.4 image-of-the-day-iotd-1.nat
Address 2: 192.168.228.5 image-of-the-day-iotd-2.nat
Address 3: 192.168.228.3 image-of-the-day-iotd-3.nat

/web # nslookup iotd
nslookup: can't resolve '(null)': Name does not resolve

Name:      iotd
Address 1: 192.168.228.5 image-of-the-day-iotd-2.nat
Address 2: 192.168.228.3 image-of-the-day-iotd-3.nat
Address 3: 192.168.228.4 image-of-the-day-iotd-1.nat
```

### 도커 컴포즈로 설정값 지정하기

- `environment` : 컨테이너 안에서 사용될 환경 변수 값 정의
- `secrets`: 컨테이너 내부의 파일에 기록될 비밀값 정의

개발 환경과 운영 환경의 컴포즈 파일을 다르게 정의하는 식으로 애플리케이션의 기능을 선택적으로 활성화 할 수 있다.

이 외에도 다양한 설정값이 존재하니 필요할 때 찾아보면 되겠다.

### 도커 컴포즈도 만능은 아니다

컴포즈는 정의된대로 실행만 해줄 수 있다. 상태 관리나 내결함성을 유지하는 부분까지는 해줄 수 없다. 서로 다른 서버를 관리하는 환경에서도 사용하기 어렵다. 로컬 머신에서만 리소스를 관리할 수 있기 때문이다.

```bash
docker compose up -d
docker compose stop
docker compose start
docker compose down
```

## 헬스 체크와 디펜던시 체크

### 헬스 체크

`HEALTHCHECK` 인스트럭션을 통해 애플리케이션의 상태를 도커에 전달할 수 있다.

```dockerfile
HEALTHCHECK CMD curl --fail http://localhost/health
```

`--fail` 옵션을 붙이면 curl 이 전달받은 상태코드를 도커에 전달한다.

- 성공: 0
- 실패: 0 이외의 값

> [!tip] Spring Actuator
> SpringBoot 를 사용한다면 [[Spring Actuator]] 를 통해서 health 체크 기능을 사용할 수 있다.

기본으로 30초 간격으로 연속 3회 이상 실패하면 애플리케이션이 이상 상태로 간주된다.

```bash
docker run -d -p 8081:80 diamol/ch08-numbers-api:v2
```

![[Pasted image 20231028173613.png]]

```bash
http localhost:8081/rng
http localhost:8081/rng
http localhost:8081/rng
http localhost:8081/rng # 실패
```

![[Pasted image 20231028173848.png]]

잠시 기다리면 상태가 변한 것을 확인할 수 있다.

`docker inspect` 명령을 통해 헬스 체크 관련 정보를 확인할 수도 있다.

```bash
docker container inspect $(docker container ls --last 1 --format '{{.ID}}')
```

명령어는 다양한 방식으로 축약할 수 있습니다. 명령어에 익숙해지면 축약형을 사용하여 더 빠르게 원하는 정보를 확인할 수 있습니다.

```bash
# 위와 같은 명령, 옵션의 의미에 익숙하다면 사용해보자
docker inspect $(docker ls -lq)
```

도커는 컨테이너의 이상을 감지할 수는 있지만 이후 작업을 처리해주지는 않는데, 이런 작업을 안전하게 처리할 수 없기 때문이다. 도커 스웜이나 쿠버네티스가 관리하는 클러스터 환경에서는 문제가 발생한 컨테이너를 종료해도 다른 컨테이너가 역할을 수행할 수 있으므로 안전하게 처리할 수 있다. 이런 상황에서는 헬스 체크가 매우 유용하다.

### 디펜던시 체크

여러 개의 컨테이너가 서로에게 복잡한 의존관계를 가지고 있다면 이 의존관계를 안전하게 해결해줄 수 있는 방법이 필요하다.

디펜던시 체크는 헬스 체크처럼 별도의 인스트럭션으로 도커에 구현된 것은 아니고 애플리케이션 실행 명령에 로직을 추가하는 방법으로 구현한다. 디펜던시를 만족하지 못했을 경우 컨테이너가 실행되지 않게 하여 개발자가 문제의 원인을 빠르게 파악할 수 있도록 한다.

### 도커 컴포즈에 헬스 체크와 디펜던시 체크 정의하기

```yaml
numbers-api:
  image: diamol/ch08-numbers-api
  port:
    - "8087:80"
  healthcheck:
    interval: 5s
    timeout: 1s
    retries: 2
    start_period: 5s
  networks:
    - app-net
```

- interval: 헬스체크 실시 간격
- timeout: 실패로 간주하는 제한시간
- retries: 재시도 횟수, 초과하면 컨테이너가 이상상태로 간주된다.
- start_period: 컨테이너 실행 후 첫 헬스 체크를 실시하는 시간 간격, 애플리케이션을 시작하는 데 시간이 오래 걸리는 경우 필요하다.

헬스 체크를 실시하는데도 CPU 와 메모리 자원이 필요하므로 실험을 통해 적절한 수치를 찾아가야 한다.

> [!NOTE] `depends_on` 설정을 사용하지 않은 이유
> 도커 컴포즈가 디펜던시 체크를 할 수 있는 범위가 단일 서버로 제한되기 때문이다. 운영 환경에서 애플리케이션이 실제 시작할 때 일어나는 상황은 훨씬 예측하기 어렵다.

### 헬스 체크와 디펜던시 체크로 복원력있는 애플리케이션을 만들 수 있는 이유

구성요소 간의 복잡한 의존관계를 미리 정의해두고 모델링하여 시작할 수도 있다. 하지만 이 방법은 많은 서버를 관리하는 환경에서는 그리 바람직한 상황이 아니다.

1. 20여개의 API 컨테이너와 50여개의 웹 애플리케이션 컨테이너를 실행한다.
2. 1개의 API 컨테이너와 1개의 웹 애플리케이션 컨테이너라도 멀쩡하게 동작한다면 서비스를 제공할 수 있다.
3. 의존관계를 미리 정의해두고 실행한다면, 20개의 API 컨테이너가 모두 실행될 때까지 서비스를 제공할 수 없다.

디펜던시 체크와 헬스 체크를 도입하면 처음부터 실행 순서를 보장할 필요가 없고, 가능한 한 빨리 컨테이너를 모두 실행하는 것에 초점을 둔다. 일부 컨테이너가 의존관계를 완성하지 못하여 재실행될 수 있지만, 일부 컨테이너는 의존관계를 완성하고 빠르게 서비스를 제공할 수 있다. 결과적으로 의존관계를 완성하지 못했던 컨테이너도 곧 정상화될 것이다.

[^1]: https://docs.docker.com/compose/migrate/
