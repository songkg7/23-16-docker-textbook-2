# 4.1 Dockerfile이 있는데 빌드 서버가 필요할까?

- 팀 협업 시 Git 코드가 변경되면 빌드를 해주는 빌드 서버
    - 빌드가 실패하면 금방 알 수 있지만, 빌드 서버 비용이 발생
- 빌드를 하기 위해 다양한 도구가 필요함
    - 링커
    - 컴파일러
    - 패키지 관리자
    - 런타임
    - 이걸 각자 관리하면 여러 모로 복잡
- 빌드 툴체인을 한꺼번에 패키징해서 공유하면 편리할 것
    - 개발에 필요한 모든 도구를 배포하는 Dockerfile 스크립트를 작성, 이를 이미지로 만듬
    - 어플리케이션 패키징을 위한 Dockerfile 스크립트에서 이 이미지를 사용해 소스 코드를 컴파일함으로서 어플리케이션을 패키징
- 멀티 스테이지 빌드
    - Dockerfile 스크립트에 작성된 순서대로 빌드가 진행
    - build-stage → test-stage(build-stage 산출물 받아서 진행) → build(test-stage 산출물 받아서 진행)
- 이런 방법으로 어플리케이션의 진정한 이식성을 확보
    - 도커만 갖춰진다면 컨테이너를 통해 어떤 환경에서든 어플리케이션을 빌드하거나 실행할 수 있다.

Q1. 멀티 스테이지 빌드 시 앞선 단계의 파일 시스템에 있는 파일을 가져오고 싶을 때는 __ 인자를 사용할 수 있다.

# 4.3 어플리케이션 빌드 실전 예제 : Node.js 소스 코드

- 언어마다 빌드 방식이 조금씩 다름
    - JAVA : 빌드 단계에서 소스 코드 패키징 → JAR 파일 생성
        - JAR : 컴파일된 어플리케이션 → 다시 최종 어플리케이션 이미지에 복사
    - .NET : JAVA와 비슷, 컴파일된 바이너리는 DLL 포맷
    - Node.js : JS는 인터프리터 언어이기 때문에 별도의 컴파일 절차가 필요 없다.
        - 컨테이너화 된 Node.js 어플리케이션을 실행하려면 Node.js 런타임과 소스 코드가 어플리케이션 이미지에 포함되어야 한다.
- npm을 사용해 Node.js 어플리케이션을 빌드하는 Dockerfile 스크립트

    ```docker
    FROM diamol/node AS builder
    
    WORKDIR /src
    COPY src/package.json .
    
    RUN npm install
    
    # app
    FROM diamol/node
    
    EXPOSE 80
    CMD ["node", "server.js"]
    
    WORKDIR /app
    COPY --from=builder /src/node_modules/ /app/node_modules/
    COPY src/.
    ```


Q2. Node.js는 JAVA, .NET 등 다른 언어와 다르게 ___ 이기 때문에 컴파일 절차가 필요하지 않다.

# 4.5 멀티 스테이지 Dockerfile 스크립트 이해하기

도커를 통한 어플리케이션 빌드의 장점

1. 표준화
    - 어떤 운영체제를 사용하든, 로컬에 어떤 도구를 설치했든 상관 없이 모든 빌드는 도커 내부에서 이뤄진다.
2. 성능 향상
    - 멀티 스테이지 빌드는 각 단계마다 캐시를 가지고, 캐시 재사용을 하면서 빌드할 수 있다.
3. 최종 산출물인 이미지를 작게 유지 가능
    - 예를 들어 curl은 중요하나, 파일 다운로드를 빌드 초기에 모아 놓으면, 최종 이미지에는 curl을 빼도 된다.

A1. —from (e.g. COPY —from=build-stage /build.txt /build.txt

A2. 인터프리터 언어
