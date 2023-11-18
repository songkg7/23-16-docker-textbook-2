---
marp: true
---

# 11장. 도커와 도커 컴포즈를 이용한 애플리케이션 빌드 및 테스트

---

## 11.1 도커를 이용한 지속적 통합 절차

CI 절차
- 최신 상태의 코드를 받아오기
- 도커 이미지 빌드 (빌드 단계)
- 컨테이너 실행 (테스트 단계)
- 레지스트리에 이미지 푸시 (배포 단계)

실제 과정은 컨테이너 내부에서 진행
- 컴파일러나 SDK를 서버에 직접 설치할 필요가 없음

---

## 11.2 도커를 이용한 빌드 인프라스트럭처 구축하기

실습
- 형상 관리 기능 (Gogs)
- 이미지 배포 (도커 레지스트리)
- 자동화 서버 (Jenkins)

---

ch11/exercises/infrastructure/docker-compose-linux.yml

~~~
version: "3.7"

services:
  jenkins:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
~~~

---

ch11/exercises/infrastructure/docker-compose.yml

~~~
version: "3.7"

services:
  gogs:
    image: diamol/gogs
    ports:
      - "3000:3000"
    networks:
      - infrastructure

  registry.local:
    image: diamol/registry
    ports:
      - "5000:5000"
    networks:
      - infrastructure

  jenkins:
    image: diamol/jenkins
    ports:
      - "8080:8080"
    networks:
      - infrastructure

networks:
  infrastructure:
    name: build-infrastructure
~~~

--- 

git 코드 수정 -> git push -> (Jenkins) Build Now

전체 CI 파이프라인이 도커 컨테이너를 통해 실행
- 도커에서 실행된 컨테이너는 도커 API, 같은 도커 엔진 내 컨테이너와 연결
- 젠킨스 이미지에는 도커 CLI와 젠킨스 컨테이너 설정을 위한 컴포즈 파일을 포함
  - 도커 명령을 실행하면 호스트 컴퓨터에서 실행 중인 도커 엔진으로 전달
- 컨테이너에서 실행 중인 앱 -> 호스트에서 동작 중인 도커의 모든 기능에 접근이 가능 -> 보안 문제가 생길 수 있음
  - 신뢰할 수 있는 이미지에만 적용해야 함

---

![width:600px](/ch11/wonjong/11_1.jpeg)

---

## 11.3 도커 컴포즈를 이용한 빌드 설정 

도커 컴포즈 파일에 환경변수를 사용
- 예를 들어 ${REGISTRY:-docker.io} 와 같이 사용하면 환경 변수 REGISTRY의 값을 치환하되, 해당 변수가 정의되어 있지 않으면 docker.io 를 기본값으로 사용한다는 의미

도커 컴포즈 파일을 통해 web과 api의 이미지를 빌드

컨테이너, 이미지, 네트워크 볼륨 등 대부분의 도커 리소스에는 레이블을 부여할 수 있음
- 레이블은 이미지에 포함시킬 수 있다는 점에서 이미지와 함께 사용될 때 특히 유용
- CI 파이프라인을 통한 앱 빌드에서 빌드 진행 중, 사후 추적이 중요한데 레이블이 여기서 도움이 됨

---

~~~
# 애플리케이션 이미지
FROM diamol/dotnet-aspnet

ARG BUILD_NUMBER=0
ARG BUILD_TAG=local

LABEL version="3.0"
LABEL build_number=${BUILD_NUMBER}
LABEL build_tag=${BUILD_TAG}

ENTRYPOINT ["dotnet", "Numbers.Api.dll"]
~~~

LABEL 인스트럭션 : Dockerfile 스크립트에 정의된 키-값 쌍을 빌드되는 이미지에 적용
ARG 인스트럭션 : 이미지를 빌드하는 시점에서만 유효한 환경변수를 설정

---

## 11.4 도커 외의 의존 모듈이 불필요한 CI 작업 만들기

의존 모듈이 뱔도로 필요하지 않다는 점은 컨테이너에서 수행하는 CI의 주된 장점
- 도커 허브, 깃허브 액션스, 애저 데브옵스 등 주요 메니지드 빌드 서비스가 이를 지원

---
ch11/exercises/Jenkinsfile
~~~
pipeline {
    agent any
    environment {
       REGISTRY = "registry.local:5000"
    }
    stages {
        stage('Verify') {
            steps {
                dir('ch11/exercises') {
                    sh 'chmod +x ./ci/00-verify.bat'
                    sh './ci/00-verify.bat'
                }
            }
        }
        stage('Build') {
            steps {
                dir('ch11/exercises') {
                    sh 'chmod +x ./ci/01-build.bat'
                    sh './ci/01-build.bat'
                }
            }
        }
        stage('Test') {
            steps {
                dir('ch11/exercises') {
                    sh 'chmod +x ./ci/02-test.bat'
                    sh './ci/02-test.bat'
                }
            }
        }
        stage('Push') {
            steps {
                dir('ch11/exercises') {
                    sh 'chmod +x ./ci/03-push.bat'
                    sh './ci/03-push.bat'
                    echo "Pushed web to http://$REGISTRY/v2/diamol/ch11-numbers-web/tags/list"
                    echo "Pushed api to http://$REGISTRY/v2/diamol/ch11-numbers-api/tags/list"
                }
            }
        }
    }
}
~~~

---

젠킨스 빌드의 단계

1. 검증 단계: 00-verify.bat 스크립트
- 도커 및 도커 컴포즈의 버전을 출력
2. 빌드 단계: 01-build.bat 스크립트
- 도커 컴포즈를 실행해 이미지를 빌드하는 역할
3. 테스트 단계: 02-test.bat 스크립트
- 도커 컴포즈로 빌드된 앱을 실행하고 컨테이너 목록을 출력한 뒤 앱을 다시 종료
4. 푸시 단계: 03-push.bat 스크립트 
- 도커 컴포즈를 실행해 빌드된 이미지를 레지스트리에 푸시

---

## 11.5 CI 파이프라인에 관계된 컨테이너

![width:600px](/ch11/wonjong/11_2.jpeg)