
## 코로나 백신 접종 신청 및 증명서 발급
![image](https://user-images.githubusercontent.com/82795860/123352333-0a55b100-d59a-11eb-8ea6-9aec60b5214e.png)
### Repositories

*전체 소스 받기*
```
git clone https://github.com/winner59/anticorona.git
```

### Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [기능적 요구사항](#기능적-요구사항)
  - [비기능적 요구사항](#비기능적-요구사항)
- [분석/설계](#분석설계)
  - [AS-IS 조직 (Horizontally-Aligned)](#AS-IS-조직-(Horizontally-Aligned))
  - [TO-BE 조직 (Vertically-Aligned)](#TO-BE-조직-(Vertically-Aligned))
  - [Event 도출](#Event-도출)
  - [부적격 이벤트 제거](#부적격-이벤트-제거)
  - [액터, 커맨드 부착](#액터,-커맨드-부착)
  - [어그리게잇으로 묶기](#어그리게잇으로-묶기)
  - [바운디드 컨텍스트로 묶기](#바운디드-컨텍스트로-묶기)
  - [폴리시 부착/이동 및 컨텍스트 매핑](#폴리시-부착/이동-및-컨텍스트-매핑)
  - [Event Storming 최종 결과](#Event-Storming-최종-결과)
  - [기능 요구사항 Coverage](#기능-요구사항-Coverage)
  - [헥사고날 아키텍처 다이어그램 도출](#헥사고날-아키텍처-다이어그램-도출)
  - [System Architecture](#System-Architecture)
- [구현](#구현)
  - [DDD(Domain Driven Design)의 적용](#DDD(Domain-Driven-Design)의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [CQRS](#CQRS)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
- [운영](#운영)
  - [Deploy/ Pipeline](#Deploy/Pipeline)
  - [Config Map](#Config-Map)
  - [Persistence Volume](#Persistence-Volume)
  - [Autoscale (HPA)](#Autoscale-(HPA))
  - [Circuit Breaker](#Circuit-Breaker)
  - [Zero-Downtime deploy (Readiness Probe)](#Zero-Downtime-deploy-(Readiness-Probe))
  - [Self-healing (Liveness Probe)](#Self-healing-(Liveness-Probe))


# 서비스 시나리오

## 기능적 요구사항

* 백신 관리자는 백신정보 및 재고를 등록한다.
* 백신 관리자는 백신 재고를 추가한다.
* 고객은 접종을 예약한다.
* 고객은 접종 예약을 취소 할 수 있다.
* 고객은 증명서 발급을 신청한다.
* 고객은 증명서 발급신청을 취소 할 수 있다.
* 접종 예약수량은 백신 재고수량을 초과 할 수 없다.
* 고객이 접종 완료 하면, 예약 수량과 재고 수량이 감소한다.
* 고객이 방문하여 접종하면 접종 관리자에 의해 접종완료된다.
* 고객은 예약정보를 확인 할 수 있다. 
     * 고객은 증명서를 발급 받을 수 있다.
     * 접종 예약 및 증명서 발급 서비스는 게이트웨이를 통해 고객과 통신한다.


## 비기능적 요구사항
* 트랜잭션
    * 예약 수량은 재고 수량을 초과하여 예약 할 수 없다. (Sync 호출)
      * 증명서 발급은 접종 완료에 한하여 신청 할 수 있다. (Sync 호출)
* 장애격리
    * 백신접종 기능이 수행되지 않더라도 백신예약은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    * 예약시스템이 과중 되면 사용자를 잠시동안 받지 않고 예약을 잠시후에 하도록 유도한다. Circuit breaker, fallback
       * 증명서 발급 신청 기능이 수행되지 않더라도 신청은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
       * 증명서 발급 시스템이 과중 되면 사용자를 잠시동안 받지 않고 예약을 잠시후에 하도록 유도한다. Circuit breaker, fallback
* 성능
       * 고객은 MyPage에서 본인 예약 상태 를 확인하고 증명서를 발급받을 수 있어야 한다. (CQRS)
    
# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
![Horizontally-Aligned](https://user-images.githubusercontent.com/2360083/119254418-278d0d80-bbf1-11eb-83d1-494ba83aeaf1.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/82795860/123352803-1130f380-d59b-11eb-8179-d117dfdd4b0f.png)

## Event 도출
![image](https://user-images.githubusercontent.com/82795860/123352835-23129680-d59b-11eb-81a8-ad3a2d5f9a8b.png)

## 부적격 이벤트 제거
![image](https://user-images.githubusercontent.com/82795860/123352867-3160b280-d59b-11eb-8ff4-e58eb685a222.png)

```
- 이벤트를 식별하여 타임라인으로 배치하고 중복되거나 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
- 현업이 사용하는 용어를 그대로 사용(Ubiquitous Language) 
```
## 액터, 커맨드 부착
```
- Event를 발생시키는 Command와 Command를 발생시키는주체, 담당자 또는 시스템을 식별함 
- Command : 백신등록,백신수량 추가,접종 예약,접종예약 취소,접종,체크 및 예약수량 변경
            ,증명서 발급신청,증명서 발급 신청 취소,증명서 발급
- Actor : 백신관리자,접종자, 접종관리자, 시스템 , 증명서 발급 관리자

```
## 어그리게잇으로 묶기
```
- 연관있는 도메인 이벤트들을 Aggregate 로 묶었음 
- Aggregate : 백신정보,예약정보,접종정보,증명서신청 정보,증명서발급 정보

```
## 바운디드 컨텍스트로 묶기
## 폴리시 부착/이동 및 컨텍스트 매핑
```
- Policy의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res)
```

## Event Storming 최종 결과
![image](https://user-images.githubusercontent.com/82795860/123355732-404a6380-d5a1-11eb-8ef9-d889095b1e5b.png)

![image](https://user-images.githubusercontent.com/82795860/123354727-2d369400-d59f-11eb-957d-9ba3fdf9d13e.png)



## 기능 요구사항 Coverage

![image](https://user-images.githubusercontent.com/82795860/123356984-be0f6e80-d5a3-11eb-8b69-d44483845d98.png)
```
 1.백신관리자는 백신정보(백신이름,재고수량)을 등록한다
 2.고객은 접종을 예약한다(Async)
 3.접종 예약수량은 백신 재고 수량을 초과할 수 없다. (Sync)
 4.예약을 하면 예약 수량과 재고수량이 감소한다
 5.접종관리자는 접종을 한다(Async)
 6.고객은 접종 증명서 발급을 신청할 수 있다.(Sync)
 7.발급관리자는 고객이 신청한 증명서를 발급한다(Async)
```

## 헥사고날 아키텍처 다이어그램 도출
![image](https://user-images.githubusercontent.com/82795860/123352956-6240e780-d59b-11eb-9fa3-9184f80850c1.png)

## System Architecture
![image](https://user-images.githubusercontent.com/82795860/123356761-43deea00-d5a3-11eb-8dbe-28b4f8cc00a4.png)


# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다
(각자의 포트넘버는 8081 ~ 8086, 8088 이다)
```shell
cd vaccine
mvn spring-boot:run

cd booking
mvn spring-boot:run 

cd mypage 
mvn spring-boot:run 

cd injection 
mvn spring-boot:run

cd gateway
mvn spring-boot:run

cd applying
mvn spring-boot:run 

cd issue
mvn spring-boot:run 
```
## DDD(Domain-Driven-Design)의 적용
msaez.io 를 통해 구현한 Aggregate 단위로 Entity 를 선언 후, 구현을 진행하였다.
Entity Pattern 과 Repository Pattern을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

Appylying 서비스의 appying.java

![image](https://user-images.githubusercontent.com/82795860/123354847-6f5fd580-d59f-11eb-8cb3-4012150a6e9a.png)

 Appylying 서비스의 PolicyHandler.java

![image](https://user-images.githubusercontent.com/82795860/123354879-80104b80-d59f-11eb-8239-96d7df85b65f.png)

 Appylying 서비스의 ApplyingRepository.java

![image](https://user-images.githubusercontent.com/82795860/123354917-90c0c180-d59f-11eb-9760-5a6f31814742.png)

DDD 적용 후 REST API의 테스트를 통하여 정상적으로 동작하는 것을 확인할 수 있었다.

## Gateway 적용

API GateWay를 통하여 마이크로 서비스들의 진입점을 통일할 수 있다. 
다음과 같이 GateWay를 적용하였다.

```yaml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: vaccine
          uri: http://localhost:8081
          predicates:
            - Path=/vaccines/** 
        - id: booking
          uri: http://localhost:8082
          predicates:
            - Path=/bookings/** 
        - id: mypage
          uri: http://localhost:8083
          predicates:
            - Path= /mypages/**
        - id: injection
          uri: http://localhost:8084
          predicates:
            - Path=/injections/**,/cancellations/**
        - id: applying
          uri: http://localhost:8085
          predicates:
            - Path= /applyings/**
        - id: issue
          uri: http://localhost:8086
          predicates:
            - Path=/issues/**  
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: vaccine
          uri: http://vaccine:8080
          predicates:
            - Path=/vaccines/** 
        - id: booking
          uri: http://booking:8080
          predicates:
            - Path=/bookings/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
        - id: injection
          uri: http://injection:8080
          predicates:
            - Path=/injections/**,/cancellations/**
        - id: applying
          uri: http://applying:8080
          predicates:
            - Path= /applyings/**
        - id: issue
          uri: http://issue:8080
          predicates:
            - Path= /issues/** 
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```  
mypage 서비스의 GateWay 적용


![image](https://user-images.githubusercontent.com/82795860/120988904-f0eeef80-c7b9-11eb-92e3-ed97ecc2b047.png)

## CQRS
Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능하게 구현해 두었다.
본 프로젝트에서 View 역할은 mypage 서비스가 수행한다.

접종 신청(Applied) 실행 후 myPage 화면
 
![image](https://user-images.githubusercontent.com/82795860/121005958-526b8a00-c7cb-11eb-9bae-ad4bd70ef2eb.png)



![image](https://user-images.githubusercontent.com/82795860/121006311-bb530200-c7cb-11eb-9d85-a7b22d1a2729.png)
  
## 폴리글랏 퍼시스턴스
mypage 서비스의 DB와 Booking/injection/vaccine/applying/issue 서비스의 DB를 다른 DB를 사용하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 
동작하여 다형성을 만족하는지 확인하였다.(폴리글랏을 만족)

|서비스|DB|pom.xml|
| :--: | :--: | :--: |
|vaccine| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|booking| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|injection| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|mypage| HSQL |![image](https://user-images.githubusercontent.com/2360083/120982836-1842be00-c7b4-11eb-91de-ab01170133fd.png)|
|applying| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|issuen| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|

## 동기식 호출과 Fallback 처리
분석단계에서의 조건 중 하나로  증명서 발급 신청은 접종 완료 상태인 경우에만 신청할 수 있으며
신청(applying)->접종(injection) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.



Applying 서비스 내 external.InjectionService

![image](https://user-images.githubusercontent.com/82795860/123353483-9668d800-d59c-11eb-9939-039db5370c1e.png)

Applying 서비스 내 Req/Resp

![image](https://user-images.githubusercontent.com/82795860/123353801-4b02f980-d59d-11eb-8df3-2c293f4dea03.png)

Injection 서비스 내 Applying 서비스 Feign Client 요청 대상

![image](https://user-images.githubusercontent.com/82795860/123353731-23139600-d59d-11eb-9f82-ad89c0783d0b.png)

동작 확인

접종 신청하기 시도 시  접종 완료 여부를 체크함

![image](https://user-images.githubusercontent.com/82795860/120994076-1e8a6780-c7bf-11eb-8374-53f7a4336a1a.png)


접종완료 시 증명서 발급 신청 가능

![image](https://user-images.githubusercontent.com/82795860/120997798-78406100-c7c2-11eb-90fa-b8ff71f53c77.png)


접종완료가 아닐경우 증명서 발급 신청  

![image](https://user-images.githubusercontent.com/82795860/123369429-b6a69000-d5b8-11eb-80e3-4551479347d2.png)

  
# 운영
  
## Deploy/ Pipeline
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.

- git에서 소스 가져오기

```
git clone https://github.com/winner59/anticorona.git
```

- Build 하기

```bash
cd /anticorona
cd gateway
mvn package

cd ..
cd booking
mvn package

cd ..
cd vaccine
mvn package

cd ..
cd injection
mvn package

cd ..
cd mypage
mvn package

cd ..
cd applying
mvn package

cd ..
cd issue
mvn package

```

- Docker Image Push/deploy/서비스생성(yml이용)

```sh
-- 기본 namespace 설정
kubectl config set-context --current --namespace=anticorona

-- namespace 생성
kubectl create ns anticorona

cd gateway
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/gateway:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd booking
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/booking:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd vaccine
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/vaccine:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd injection
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/injection:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd mypage
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/mypage:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd applying
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/applying:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd issue
az acr build --registry skccanticorona --image skccanticorona.azurecr.io/issue:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

```

- anticorona/applying/kubernetes/deployment.yml 파일 
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: applying
  namespace: anticorona
  labels:
    app: applying
spec:
  replicas: 1
  selector:
    matchLabels:
      app: applying
  template:
    metadata:
      labels:
        app: applying
    spec:
      containers:
        - name: applying
          image: skccanticorona.azurecr.io/applying:latest
          ports:
            - containerPort: 8080
          env:
            - name: injection-url
              valueFrom:
                configMapKeyRef:
                  name: apiurl
                  key: url
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
          volumeMounts:
            - name: volume
              mountPath: "/mnt/azure"
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "500Mi"
              cpu: "500m"
      volumes:
      - name: volume
        persistentVolumeClaim:
          claimName: applying-disk
```	
- anticorona/applying/kubernetes/service.yaml 파일 

```yml
apiVersion: v1
kind: Service
metadata:
  name: applying
  namespace: anticorona
  labels:
    app: applying
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: applying
```	  

- deploy 완료
![image](https://user-images.githubusercontent.com/82795860/123371395-ae505400-d5bc-11eb-8e66-fffc5e8ab1e4.png)


***

## Config Map

- 변경 가능성이 있는 설정을 ConfigMap을 사용하여 관리  
  - applying 서비스에서 바라보는 injection 서비스 url 일부분을 ConfigMap 사용하여 구현​  

- in applying src (booking/src/main/java/anticorona/external/VaccineService.java)  
    ![image](https://user-images.githubusercontent.com/82795860/123354105-e6946a00-d59d-11eb-8d9b-df1bf74e3ee3.png)

- applying application.yml (booking/src/main/resources/application.yml)​  
  ![image](https://user-images.githubusercontent.com/82795860/123354126-f4e28600-d59d-11eb-8db6-7d4e60b2a299.png)

- applying deploy yml (booking/kubernetes/deployment.yml)  
  ![image](https://user-images.githubusercontent.com/82795860/123354155-0330a200-d59e-11eb-8f10-0c5198b150a3.png)

- configmap 생성 후 조회

    ```sh
    kubectl create configmap apiurl --from-literal=url=injection -n anticorona
    ```

    ![configmap-configmap조회](https://user-images.githubusercontent.com/18115456/120985042-2eea1480-c7b6-11eb-9dbc-e73d696c003b.PNG)

- configmap 삭제 후, 에러 확인  

    ```sh
    kubectl delete configmap apiurl
    ```

    ![configmap-오류1](https://user-images.githubusercontent.com/18115456/120985205-5b9e2c00-c7b6-11eb-8ede-df74eff7f344.png)

    ![configmap-오류2](https://user-images.githubusercontent.com/18115456/120985213-5ccf5900-c7b6-11eb-9c06-5402942329a3.png)  

## Persistence Volume
  
PVC 생성 파일

<code>applying-pvc.yml</code>
- AccessModes: **ReadWriteMany**
- storeageClass: **azurefile**
![image](https://user-images.githubusercontent.com/82795860/123357735-13984b00-d5a5-11eb-9475-86d023eac0a4.png)

<code>deployment.yml</code>

- Container에 Volumn Mount

![image](https://user-images.githubusercontent.com/2360083/120983890-175e5c00-c7b5-11eb-9332-04033438cea1.png)

<code>application.yml</code>
- profile: **docker**
- logging.file: PVC Mount 경로

![image](https://user-images.githubusercontent.com/2360083/120983856-10374e00-c7b5-11eb-93d5-42e1178912a8.png)

마운트 경로에 logging file 생성 확인

```sh
$ kubectl exec -it issue -n anticorona -- /bin/sh
$ cd /mnt/azure/logs
$ tail -f issue.log
```

![image](https://user-images.githubusercontent.com/82795860/123354338-6589a280-d59e-11eb-8b60-f9f4667764f3.png)

## Autoscale (HPA)

  앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 증명서 발급 서비스에 리소스에 대한 사용량을 정의한다.

<code>applying/kubernetes/deployment.yml</code>

```yml
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "500Mi"
      cpu: "500m"
```

- 증명서 신청 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:

```sh
$ kubectl autoscale deploy booking --min=1 --max=10 --cpu-percent=15
```

![image](https://user-images.githubusercontent.com/82795806/120987663-c51f3a00-c7b8-11eb-8cc3-59d725ca2f69.png)


- CB 에서 했던 방식대로 워크로드를 걸어준다.

```sh
$ siege -c200 -t10S -v --content-type "application/json" 'http://booking:8080/bookings POST {"vaccineId":1, "vcName":"FIZER", "userId":5, "status":"BOOKED"}'
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:

```sh
$ watch kubectl get all
```

- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다:

* siege 부하테스트 - 전

![image](https://user-images.githubusercontent.com/82795806/120990254-51caf780-c7bb-11eb-98a6-243b69344f12.png)

* siege 부하테스트 - 후

![image](https://user-images.githubusercontent.com/82795806/120989337-66f35680-c7ba-11eb-9b4e-b1425d4a3c2f.png)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![image](https://user-images.githubusercontent.com/82795806/120990490-93f43900-c7bb-11eb-9295-c3a0a8165ff6.png)

## Circuit Breaker

  * 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Istio를 설치하여, anticorona namespace에 주입하여 구현함

시나리오는 증명서발급 신청(applying)-->증명서 발급(issue) 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 예약 요청이 과도할 경우 CB 를 통하여 장애격리.

- Istio 다운로드 및 PATH 추가, 설치, namespace에 istio주입

```sh
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.7.1 TARGET_ARCH=x86_64 sh -
※ istio v1.7.1은 Kubernetes 1.16이상에서만 동작
```

- istio PATH 추가

```sh
$ cd istio-1.7.1
$ export PATH=$PWD/bin:$PATH
```

- istio 설치

```sh
$ istioctl install --set profile=demo --set hub=gcr.io/istio-release
※ Docker Hub Rate Limiting 우회 설정
```

- namespace에 istio주입

```sh
$ kubectl label anticorona tutorial istio-issue=enabled
```

- Virsual Service 생성 (Timeout 3초 설정)
- anticorona/applying/kubernetes/applying-istio.yaml 파일 

![image](https://user-images.githubusercontent.com/82795860/125012611-701b6000-e0a5-11eb-9155-da808d750007.png)
```	  




- Applying 서비스 재배포 후 Pod에 CB 부착 확인

![image](https://user-images.githubusercontent.com/82795806/120985804-ed0d9e00-c7b6-11eb-9f13-8a961c73adc0.png)


- 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
  - 동시사용자 100명, 60초 동안 실시

```sh
$ siege -c100 -t10S -v --content-type "application/json" 'http://booking:8080/bookings POST {"vaccineId":1, "vcName":"FIZER", "userId":5, "status":"BOOKED"}'
```
![image](https://user-images.githubusercontent.com/82795806/120986972-1549cc80-c7b8-11eb-83e1-7bac5a0e80ed.png)


- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 
- 약 84%정도 정상적으로 처리되었음.

***

## Zero-Downtime deploy (Readiness Probe)

- deployment.yml에 정상 적용되어 있는 readinessProbe  
```yml
readinessProbe:
  httpGet:
    path: '/actuator/health'
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
```

- deployment.yml에서 readiness 설정 제거 후, 배포중 siege 테스트 진행  
    - hpa 설정에 의해 target 지수 초과하여 booking scale-out 진행됨  
        ![readiness-배포중](https://user-images.githubusercontent.com/18115456/120991348-7ecbda00-c7bc-11eb-8b4d-bdb6dacad1cf.png)

    - booking이 배포되는 중,  
    정상 실행중인 booking으로의 요청은 성공(201),  
    배포중인 booking으로의 요청은 실패(503 - Service Unavailable) 확인
        ![readiness2](https://user-images.githubusercontent.com/18115456/120987386-81c4cb80-c7b8-11eb-84e7-5c00a9b1a2ff.PNG)  

- 다시 readiness 정상 적용 후, Availability 100% 확인  
![readiness4](https://user-images.githubusercontent.com/18115456/120987393-825d6200-c7b8-11eb-887e-d01519123d42.PNG)

    
## Self-healing (Liveness Probe)

- deployment.yml에 정상 적용되어 있는 livenessProbe  

```yml
livenessProbe:
  httpGet:
    path: '/actuator/health'
    port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```

- port 및 path 잘못된 값으로 변경 후, retry 시도 확인 (in booking 서비스)  
    - booking deploy yml 수정  
        ![selfhealing(liveness)-세팅변경](https://user-images.githubusercontent.com/18115456/120985806-ed0d9e00-c7b6-11eb-834f-ffd2c627ecf0.png)

    - retry 시도 확인  
        ![selfhealing(liveness)-restarts수](https://user-images.githubusercontent.com/18115456/120985797-ebdc7100-c7b6-11eb-8b29-fed32d4a15a3.png)  
