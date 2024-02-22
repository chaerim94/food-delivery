![image](https://github.com/chaerim94/labshoppubsub/assets/39048893/d780e53e-b820-48e9-b5de-bf66a5bd6b3d)

# 숙박예약



# Table of contents

- [예제 - 숙박예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
    - [클라우드 아키텍처 구성, MSA 아키텍처 구성도](#클라우드아키텍처구성,MSA아키텍처구성도)
    - [도메인분석 - 이벤트스토밍](#도메인분석-이벤트스토밍)
  - [구현:](#구현-)
    - [분산트랜잭션 - Saga](#분산트랜잭션-Saga)
    - [단일 진입점 - Gateway](#단일진입점-Gateway)
    - [보상처리 - Compensation ](#보상처리-Compensation)
    - [분산 데이터 프로젝션 - CQRS ](#분산데이터프로젝션-CQRS)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [컨테이너 자동확장 - HPA ](#컨테이너자동확장-HPA)
    - [컨테이너로부터 환경분리 - CofigMap](#컨테이너로부터환경분리-CofigMap)
    - [클라우드스토리지 활용 - PVC ](#클라우드스토리지활용-PVC)
    - [셀프 힐링/무정지배포 - Liveness/Rediness Probe](#셀프힐링/무정지배포-Liveness)
    - [서비스 메쉬 응용 - Mesh](#서비스메쉬응용-Mesh)
    - [통합 모니터링 - Loggregation/Monitoring](#통합모니터링-Loggregation/Monitoring)

# 서비스 시나리오

아고다 커버하기 - [https://1sung.tistory.com/106](https://brunch.co.kr/@soons/89)

기능적 요구사항
1. 고객이 숙소를 예약한다
1. 고객이 결제한다
1. 결제가 되면 예약내역이 숙박업소주인에게 전달된다
1. 업소주인이 확인하여 예약확정을 한다
1. 고객이 예약을 취소할 수 있다
1. 고객이 예약상태를 중간중간 조회한다

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 예약건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 예약관리 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 예약관리에서 확인할 수 있는 예약상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
       

# 분석/설계


## 클라우드 아키텍처 구성, MSA 아키텍처 구성도
![image](https://github.com/chaerim94/food-delivery/assets/39048893/d7fbef75-f26f-4580-9994-267dd5a2bd82)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  https://www.msaez.io/#/storming/reserveplace
![KakaoTalk_20240222_172406171_10](https://github.com/chaerim94/food-delivery/assets/39048893/cd234a49-f1a7-4f2e-b17c-70124bb44101)

    - 고객이 숙소를 선택하여 주문한다 (ok)
    - 고객이 결제한다 (ok)
    - 주문이 되면 주문 내역이 숙소주인에게 전달된다 (ok)
    - 숙소 예약이 완료되면 프론트단에서 업데이트된 상태를 확인한다 (ok)

    - 고객이 주문을 취소할 수 있다 (ok)
    - 고객이 주문상태를 중간중간 조회한다 (View-green sticker 의 추가로 ok) 


### 비기능 요구사항에 대한 검증

![image](https://github.com/chaerim94/food-delivery/assets/39048893/d0baa82a-5f7b-4484-9748-edffbb90aee1)


    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
    - 고객 주문시 결제처리:  결제가 완료되지 않은 예약은 절대 받지 않는다에 따라, ACID 트랜잭션 적용. 예약완료시 결제처리에 대해서는 Request-Response 방식 처리




# 구현:

분석/설계 단계에서 도출된 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8080 ~ 808n 이다)

```
cd place
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd management
mvn spring-boot:run  

cd customer
mvn spring-boot:run

cd gateway
mvn spring-boot:run
```

## [분산트랜잭션 - Saga, 단일 진입점 - Gateway]

```
//재고생성
gitpod /workspace/reserveplace-v3 (main) $ http POST localhost:8080/reservationManagements placeId=1 stock=1

HTTP/1.1 201 Created
Content-Type: application/json
Date: Thu, 22 Feb 2024 08:14:51 GMT
Location: http://localhost:8084/reservationManagements/1
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
transfer-encoding: chunked

{
    "_links": {
        "reservationManagement": {
            "href": "http://localhost:8084/reservationManagements/1"
        },
        "reservationcancelprocessing": {
            "href": "http://localhost:8084/reservationManagements/1/reservationcancelprocessing"
        },
        "reservationinform": {
            "href": "http://localhost:8084/reservationManagements/1/reservationinform"
        },
        "self": {
            "href": "http://localhost:8084/reservationManagements/1"
        }
    },
    "orderId": null,
    "stock": 1
}

//숙소예약
gitpod /workspace/reserveplace-v3 (main) $ http POST localhost:8080/accommodations placeNm=test01 status=예약처리 usrId=2 strDt=20240221 endDt=20240222 qty=1 amount=5000 placeId=1
HTTP/1.1 201 Created
Content-Type: application/json
Date: Thu, 22 Feb 2024 08:15:36 GMT
Location: http://localhost:8082/accommodations/1
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
transfer-encoding: chunked

{
    "_links": {
        "accommodation": {
            "href": "http://localhost:8082/accommodations/1"
        },
        "reservationstatusupdate": {
            "href": "http://localhost:8082/accommodations/1/reservationstatusupdate"
        },
        "self": {
            "href": "http://localhost:8082/accommodations/1"
        }
    },
    "amount": 5000.0,
    "endDt": "1970-01-01T05:37:20.222+00:00",
    "placeId": 1,
    "placeNm": "test01",
    "qty": 1,
    "status": "예약처리",
    "strDt": "1970-01-01T05:37:20.221+00:00",
    "usrId": "2"
}
```
```
// 토픽확인
[appuser@fcd4fde9e121 bin]$ ./kafka-console-consumer --bootstrap-server localhost:9092 --topic reserveplace  --from-beginning


{"eventType":"ReservationPlaced","timestamp":1708589735956,"orderId":1,"placeNm":"test01","status":"예약처리","usrId":"2","strDt":"1970-01-01T05:37:20.221+00:00","endDt":"1970-01-01T05:37:20.222+00:00","qty":1,"amount":5000.0,"placeId":1}
{"eventType":"PaymentApproved","timestamp":1708589736229,"payId":null,"orderId":1,"usrId":"2","status":"결제완료","amount":5000.0,"placeId":1,"qty":1}
{"eventType":"ReservationConfirmed","timestamp":1708589736448,"placeId":1,"stock":0,"orderId":1}
{"eventType":"ReservationStatusChanged","timestamp":1708589736551,"orderId":1,"placeNm":"test01","status":"예약완료","usrId":"2","strDt":"1970-01-01T05:37:20.221+00:00","endDt":"1970-01-01T05:37:20.222+00:00","qty":1,"amount":5000.0,"placeId":1}
```

## [보상트랜젝션 확인]
```
gitpod /workspace/reserveplace-v3 (main) $ http POST localhost:8080/accommodations placeNm=test01 status=예약처리 usrId=33 strDt=20240221 endDt=20240222 qty=1 amount=5000 placeId=1
HTTP/1.1 201 Created
Content-Type: application/json
Date: Thu, 22 Feb 2024 08:16:41 GMT
Location: http://localhost:8082/accommodations/2
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
transfer-encoding: chunked

{
    "_links": {
        "accommodation": {
            "href": "http://localhost:8082/accommodations/2"
        },
        "reservationstatusupdate": {
            "href": "http://localhost:8082/accommodations/2/reservationstatusupdate"
        },
        "self": {
            "href": "http://localhost:8082/accommodations/2"
        }
    },
    "amount": 5000.0,
    "endDt": "1970-01-01T05:37:20.222+00:00",
    "placeId": 1,
    "placeNm": "test01",
    "qty": 1,
    "status": "예약처리",
    "strDt": "1970-01-01T05:37:20.221+00:00",
    "usrId": "33"
}
```

```
{"eventType":"ReservationPlaced","timestamp":1708589801055,"orderId":2,"placeNm":"test01","status":"예약처리","usrId":"33","strDt":"1970-01-01T05:37:20.221+00:00","endDt":"1970-01-01T05:37:20.222+00:00","qty":1,"amount":5000.0,"placeId":1}
{"eventType":"PaymentApproved","timestamp":1708589801064,"payId":null,"orderId":2,"usrId":"33","status":"결제완료","amount":5000.0,"placeId":1,"qty":1}
{"eventType":"ReservationCancelConfirmed","timestamp":1708589801094,"placeId":1,"stock":null,"orderId":2}
{"eventType":"PaymentCancelApproved","timestamp":1708589801186,"payId":null,"orderId":2,"usrId":null,"status":"결제취소","amount":null,"placeId":1,"qty":null}
{"eventType":"ReservationCanceled","timestamp":1708589801201,"orderId":2,"placeNm":null,"status":"예약취소","usrId":null,"strDt":null,"endDt":null,"qty":null,"amount":null,"placeId":null}
```

## [CQRS - mypage]
```
gitpod /workspace/reserveplace-v3 (main) $ http :8086/mypages/1
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Thu, 22 Feb 2024 08:17:29 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8086/mypages/1"
        },
        "self": {
            "href": "http://localhost:8086/mypages/1"
        }
    },
    "amount": 5000.0,
    "placeId": 1,
    "placeNm": "test01",
    "status": "예약완료",
    "usrId": "2"
}
```

# 운영

## CI/CD 설정


aws CodeBuild를 활용한 CI/CD 처리, pipeline build script 는 buildspec.yml 에 포함되었다.
```
version: 0.2

env:
  variables:
    IMAGE_REPO_NAME: "user20-gateway"

phases:
  install:
    runtime-versions:
      java: corretto17
      docker: 20
  pre_build:
    commands:
      - cd gateway
      - echo Logging in to Amazon ECR...
      - echo $IMAGE_REPO_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION  .
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION

#cache:
#  paths:
#    - '/root/.m2/**/*'
```

- 프로젝트 빌드 결과
![KakaoTalk_20240222_172420432_01](https://github.com/chaerim94/food-delivery/assets/39048893/dd8b4f16-d326-4143-920f-c7b4c42798f1)

- 프라이빗 ECR 결과
![KakaoTalk_20240222_172420432](https://github.com/chaerim94/food-delivery/assets/39048893/c8a97f83-42e5-4c0e-b311-fd9bb769cd6f)



## 컨테이너 자동확장 - HPA 

- Kubernetes 클러스터에 Metrics Server를 배포하여 리소스 모니터링을 활성화
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get deployment metrics-server -n kube-system
```

- deployment.yaml 아래 추가
```
containers:
        - name: admin
          image: khsh5592/admin:240221
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "200m"
```

- Kubernetes 클러스터에서 place 디플로이먼트를 자동으로 확장
```
//cpu-percent=50: CPU 사용률이 50%를 넘으면 자동으로 스케일링을 시작
//min=1: 최소 파드 수를 1로 설정합니다. 따라서 최소 1개의 파드가 항상 실행
//max=3: 최대 파드 수를 3으로 설정합니다. 따라서 파드 수는 최대 3개까지 확장
kubectl autoscale deployment place --cpu-percent=50 --min=1 --max=3
```

- 부하 테스트 Pod 설치 후 부하 발생
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF

$ kubectl exec -it siege -- /bin/bash
$ siege -c20 -t40S -v http://10.100.106.43:8080/

HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.06 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.43 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.06 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.30 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.49 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.48 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.14 secs:     361 bytes ==> GET  /
HTTP/1.1 200     0.53 secs:     361 bytes ==> GET  /

Lifting the server siege...
Transactions:                  25600 hits
Availability:                 100.00 %
Elapsed time:                  39.25 secs
Data transferred:               8.81 MB
Response time:                  0.03 secs
Transaction rate:             652.23 trans/sec
Throughput:                     0.22 MB/sec
Concurrency:                   17.49
Successful transactions:       25601
Failed transactions:               0
Longest transaction:            0.91
Shortest transaction:           0.00
 
HTTP/1.1 200     0.30 secs:     361 bytes ==> GET  /
```

- autoscale 결과
```
gitpod /workspace/reserveplace-v3 (main) $ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/customer-dbddf74c7-vf5mf        1/1     Running   0          91m
pod/gateway-55b7667485-9gvdk        1/1     Running   0          91m
pod/my-kafka-0                      1/1     Running   0          98m
pod/notification-66cc75c68d-lknhw   1/1     Running   0          91m
pod/payment-7c65bb8db5-j94cd        1/1     Running   0          92m
pod/place-5c5487ff9d-72rsb          1/1     Running   0          9m43s
pod/place-5c5487ff9d-7w4mz          0/1     Running   0          35s
pod/place-5c5487ff9d-kp7q9          0/1     Running   0          35s
pod/siege                           1/1     Running   0          2m58s

NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                      AGE
service/customer            ClusterIP      10.100.237.171   <none>                                                                   8080/TCP                     91m
service/gateway             LoadBalancer   10.100.203.15    a97bdf07dceef4c81a91fe5dfc486a93-940952681.eu-west-2.elb.amazonaws.com   8080:31929/TCP               91m
service/kubernetes          ClusterIP      10.100.0.1       <none>                                                                   443/TCP                      98m
service/my-kafka            ClusterIP      10.100.52.181    <none>                                                                   9092/TCP                     98m
service/my-kafka-headless   ClusterIP      None             <none>                                                                   9092/TCP,9094/TCP,9093/TCP   98m
service/notification        ClusterIP      10.100.140.110   <none>                                                                   8080/TCP                     91m
service/payment             ClusterIP      10.100.54.158    <none>                                                                   8080/TCP                     92m
service/place               ClusterIP      10.100.106.43    <none>                                                                   8080/TCP                     9m34s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/customer       1/1     1            1           91m
deployment.apps/gateway        1/1     1            1           91m
deployment.apps/notification   1/1     1            1           91m
deployment.apps/payment        1/1     1            1           92m
deployment.apps/place          1/3     3            1           9m43s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/customer-dbddf74c7        1         1         1       91m
replicaset.apps/gateway-55b7667485        1         1         1       91m
replicaset.apps/notification-66cc75c68d   1         1         1       92m
replicaset.apps/payment-7c65bb8db5        1         1         1       92m
replicaset.apps/place-5c5487ff9d          3         3         1       9m44s

NAME                        READY   AGE
statefulset.apps/my-kafka   1/1     98m

NAME                                        REFERENCE          TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/place   Deployment/place   792%/50%   1         3         3          8m22s
```



## 컨테이너로부터 환경분리 - CofigMap

- 진행 전 application-resource.yaml 파일에 logging 이 추가된 docker image를 사용
- 데이터베이스 연결 정보와 로그 레벨을 ConfigMap에 저장하여 관리
```
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-dev
  namespace: default
data:
  ORDER_DB_URL: jdbc:mysql://mysql:3306/connectdb1?serverTimezone=Asia/Seoul&useSSL=false
  ORDER_DB_USER: myuser
  ORDER_DB_PASS: mypass
  ORDER_LOG_LEVEL: DEBUG
EOF
```

```
gitpod /workspace/reserveplace-v3/place (main) $ kubectl get configmap
NAME               DATA   AGE
config-dev         4      12s
kube-root-ca.crt   1      4h54m
my-kafka-scripts   1      119m

gitpod /workspace/reserveplace-v3/place (main) $ kubectl get configmap config-dev -o yaml
apiVersion: v1
data:
  ORDER_DB_PASS: mypass
  ORDER_DB_URL: jdbc:mysql://mysql:3306/connectdb1?serverTimezone=Asia/Seoul&useSSL=false
  ORDER_DB_USER: myuser
  ORDER_LOG_LEVEL: DEBUG
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"ORDER_DB_PASS":"mypass","ORDER_DB_URL":"jdbc:mysql://mysql:3306/connectdb1?serverTimezone=Asia/Seoul\u0026useSSL=false","ORDER_DB_USER":"myuser","ORDER_LOG_LEVEL":"DEBUG"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"config-dev","namespace":"default"}}
  creationTimestamp: "2024-02-21T10:11:30Z"
  name: config-dev
  namespace: default
  resourceVersion: "65163"
  uid: 43da75be-815e-4101-b54b-a22d8bdcfc8c
```


- place 재배포 후 로그 레벨 INFO에서 DEBUG 변경 확인 >>> kubectl logs -l app=place
``` 
gitpod /workspace/reserveplace-v3/place (main) $ kubectl logs -l app=place
2024-02-21 19:54:59.047 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.kafka.clients.FetchSessionHandler    : [Consumer clientId=consumer-place-2, groupId=place] Node 0 sent an incremental fetch response with throttleTimeMs = 0 for session 1598538034 with 0 response partition(s), 1 implied partition(s)
2024-02-21 19:54:59.048 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.k.c.consumer.internals.Fetcher       : [Consumer clientId=consumer-place-2, groupId=place] Added READ_UNCOMMITTED fetch request for partition reserveplace-0 at position FetchPosition{offset=2, offsetEpoch=Optional.empty, currentLeader=LeaderAndEpoch{leader=Optional[my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 0 rack: null)], epoch=0}} to node my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 0 rack: null)
2024-02-21 19:54:59.048 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.kafka.clients.FetchSessionHandler    : [Consumer clientId=consumer-place-2, groupId=place] Built incremental fetch (sessionId=1598538034, epoch=41) for node 0. Added 0 partition(s), altered 0 partition(s), removed 0 partition(s) out of 1 partition(s)
2024-02-21 19:54:59.048 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.k.c.consumer.internals.Fetcher       : [Consumer clientId=consumer-place-2, groupId=place] Sending READ_UNCOMMITTED IncrementalFetchRequest(toSend=(), toForget=(), implied=(reserveplace-0)) to broker my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 0 rack: null)
2024-02-21 19:54:59.310 DEBUG [place,,,] 1 --- [-thread | place] o.a.k.c.c.internals.AbstractCoordinator  : [Consumer clientId=consumer-place-2, groupId=place] Sending Heartbeat request with generation 12 and member id consumer-place-2-69ac14f5-d696-4d3b-bf32-c1a501286d65 to coordinator my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 2147483647 rack: null)
2024-02-21 19:54:59.312 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.k.c.c.internals.AbstractCoordinator  : [Consumer clientId=consumer-place-2, groupId=place] Received successful Heartbeat response
2024-02-21 19:54:59.551 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.kafka.clients.FetchSessionHandler    : [Consumer clientId=consumer-place-2, groupId=place] Node 0 sent an incremental fetch response with throttleTimeMs = 0 for session 1598538034 with 0 response partition(s), 1 implied partition(s)
2024-02-21 19:54:59.551 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.k.c.consumer.internals.Fetcher       : [Consumer clientId=consumer-place-2, groupId=place] Added READ_UNCOMMITTED fetch request for partition reserveplace-0 at position FetchPosition{offset=2, offsetEpoch=Optional.empty, currentLeader=LeaderAndEpoch{leader=Optional[my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 0 rack: null)], epoch=0}} to node my-kafka-0.my-kafka-headless.default.svc.cluster.local:9092 (id: 0 rack: null)
2024-02-21 19:54:59.551 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.kafka.clients.FetchSessionHandler    : [Consumer clientId=consumer-place-2, groupId=place] Built incremental fetch (sessionId=1598538034, epoch=42) for node 0. Added 0 partition(s), altered 0 partition(s), removed 0 partition(s) out of 1 partition(s)
2024-02-21 19:54:59.551 DEBUG [place,,,] 1 --- [container-0-C-1] o.a.k.c.consumer.internals.Fetcher       : [Consumer clientId=consumer-place-2, groupId=place] Sending READ_UNCOMMITTED IncrementalFetchReque

```


## 클라우드스토리지 활용 - PVC 

```
// Kubernetes 클러스터에 PersistentVolumeClaim 생성
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
  labels:
    app: ebs-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
  storageClassName: ebs-sc
EOF

gitpod /workspace/reserveplace-v3/place (main) $ kubectl get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-my-kafka-0   Bound    pvc-596060f8-96c1-4bbc-b56c-4b14160ea88e   8Gi        RWO            ebs-sc         5h56m
ebs-pvc           Bound    pvc-d55fe6ea-c007-4edc-a074-8ecbb3fcb16f   1Gi        RWO            ebs-sc         4m32s
gitpod /workspace/reserveplace-v3/place (main) $ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
pvc-596060f8-96c1-4bbc-b56c-4b14160ea88e   8Gi        RWO            Delete           Bound    default/data-my-kafka-0   ebs-sc                  5h56m
pvc-d55fe6ea-c007-4edc-a074-8ecbb3fcb16f   1Gi        RWO            Delete           Bound    default/ebs-pvc           ebs-sc                  11s
```

```
// place서비스를 배포할 pvcconfig.yaml 생성
apiVersion: "apps/v1"
kind: "Deployment"
metadata:
  name: place
  labels:
    app: "place"
spec:
  selector:
    matchLabels:
      app: "place"
  replicas: 1
  template:
    metadata:
      labels:
        app: "place"
    spec:
      containers:
      - name: "place"
        image: chather/place:0222
        ports:
          - containerPort: 8080
        volumeMounts: # 컨테이너에 마운트할 볼륨을 정의
          - mountPath: "/mnt/data" # 볼륨을 마운트할 경로를 지정
            name: volume
      volumes: # Pod에 사용될 볼륨을 정의
      - name: volume
        persistentVolumeClaim:
           claimName: ebs-pvc 
```

```
gitpod /workspace/reserveplace-v3/place (main) $ kubectl delete -f pvcconfig.yaml 
deployment.apps "place" deleted
gitpod /workspace/reserveplace-v3/place (main) $ kubectl apply -f pvcconfig.yaml 
deployment.apps/place created

gitpod /workspace/reserveplace-v3/place (main) $ kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
customer-dbddf74c7-vf5mf        1/1     Running   0          4h29m
gateway-55b7667485-9gvdk        1/1     Running   0          4h30m
my-kafka-0                      1/1     Running   0          4h36m
notification-66cc75c68d-lknhw   1/1     Running   0          4h30m
payment-7c65bb8db5-j94cd        1/1     Running   0          4h31m
place-79bdb44547-j4wrd          1/1     Running   0          22s
siege                           1/1     Running   0          3h1m                   
```
- place Pod에 대해 쉘을 실행하고, /mnt/data 경로에 MOUNT_TEST.TEST 테스트 파일 생성
```         
gitpod /workspace/reserveplace-v3/place (main) $ kubectl exec -it pod/place-79bdb44547-j4wrd -- /bin/sh
/ # cd /mnt/data
/mnt/data # ls
lost+found
/mnt/data #  touch MOUNT_TEST.TEST
/mnt/data # ls
MOUNT_TEST.TEST  lost+found
/mnt/data # exit
```
- place 서비스 중지 후 다시 배포하였을때 클라우드스토리지에 생성한 MOUNT_TEST.TEST 테스트 파일이 조회되어야한다.
```
gitpod /workspace/reserveplace-v3/place (main) $ kubectl delete -f pvcconfig.yaml 
deployment.apps "place" deleted
gitpod /workspace/reserveplace-v3/place (main) $ kubectl apply -f pvcconfig.yaml 
deployment.apps/place created
gitpod /workspace/reserveplace-v3/place (main) $ kubectl get pod
NAME                            READY   STATUS              RESTARTS   AGE
customer-dbddf74c7-vf5mf        1/1     Running             0          4h32m
gateway-55b7667485-9gvdk        1/1     Running             0          4h32m
my-kafka-0                      1/1     Running             0          4h39m
notification-66cc75c68d-lknhw   1/1     Running             0          4h33m
payment-7c65bb8db5-j94cd        1/1     Running             0          4h34m
place-79bdb44547-vzcsx          0/1     ContainerCreating   0          7s
siege                           1/1     Running             0          3h4m
gitpod /workspace/reserveplace-v3/place (main) $ kubectl exec -it pod/place-79bdb44547-vzcsx -- /bin/sh
/ # ls /mnt/data
MOUNT_TEST.TEST  lost+found
/ # 
```


# 셀프 힐링/무정지배포 - Liveness/Rediness Probe 

- HttpGet type의 Probe Action이 설정된 place 서비스 배포
``` 
$ vi deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: place
  labels:
    app: place
spec:
  replicas: 1
  selector:
    matchLabels:
      app: place
  template:
    metadata:
      labels:
        app: place
    spec:
      containers:
        - name: place
          image: chather/place:0222
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 2
            successThreshold: 1
            periodSeconds: 5
            failureThreshold: 3

$ kubectl apply -f kubernetes/deployment.yaml


$ kubectl expose deploy place --type=LoadBalancer --port=8080 //배포된 place 서비스에 대해 라우터를 생성한다.
$ kubectl get all
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                      AGE
service/customer            ClusterIP      10.100.237.171   <none>                                                                   8080/TCP                     20h
service/gateway             LoadBalancer   10.100.203.15    a97bdf07dceef4c81a91fe5dfc486a93-940952681.eu-west-2.elb.amazonaws.com   8080:31929/TCP               20h
service/kubernetes          ClusterIP      10.100.0.1       <none>                                                                   443/TCP                      20h
service/my-kafka            ClusterIP      10.100.52.181    <none>                                                                   9092/TCP                     20h
service/my-kafka-headless   ClusterIP      None             <none>                                                                   9092/TCP,9094/TCP,9093/TCP   20h
service/notification        ClusterIP      10.100.140.110   <none>                                                                   8080/TCP                     20h
service/payment             ClusterIP      10.100.54.158    <none>                                                                   8080/TCP                     20h
service/place               LoadBalancer   10.100.12.221    a17ef1fc8481448e5afef33f66dcd77d-646506338.eu-west-2.elb.amazonaws.com   8080:30731/TCP               7s


$ http a17ef1fc8481448e5afef33f66dcd77d-646506338.eu-west-2.elb.amazonaws.com:8080/accommodations // Liveness Probe 확인

HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/vnd.spring-boot.actuator.v3+json
Date: Thu, 22 Feb 2024 02:21:32 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked

{
    "groups": [
        "liveness",
        "readiness"
    ],
    "status": "UP"
}
```

- place CrashLoopBackOff에 따른 서버복구 이력
```
$ gitpod /workspace/reserveplace-v3 (main) $ kubectl get po -w
NAME                       READY   STATUS             RESTARTS        AGE
gateway-55b7667485-v6nqs   1/1     Running            0               19m
my-kafka-0                 0/1     Running            0               10s
payment-7c65bb8db5-47skp   0/1     Running            1 (112s ago)    4m18s
place-b49b5c96d-hncl5      0/1     CrashLoopBackOff   9 (4m21s ago)   19m
my-kafka-0                 1/1     Running            0               25s
payment-7c65bb8db5-47skp   1/1     Running            1 (2m10s ago)   4m36s
place-b49b5c96d-hncl5      0/1     Running            10 (5m6s ago)   19m
place-b49b5c96d-hncl5      1/1     Running            10 (5m30s ago)   20m
place-b49b5c96d-xgj7p      0/1     Pending            0                1s
place-b49b5c96d-xgj7p      0/1     Pending            0                1s
place-b49b5c96d-ncqts      0/1     Pending            0                0s
place-b49b5c96d-xgj7p      0/1     ContainerCreating   0                1s
place-b49b5c96d-ncqts      0/1     Pending             0                0s
place-b49b5c96d-ncqts      0/1     ContainerCreating   0                0s
place-b49b5c96d-xgj7p      0/1     Running             0                3s
place-b49b5c96d-ncqts      0/1     Running             0                6s
place-b49b5c96d-xgj7p      1/1     Running             0                31s
place-b49b5c96d-ncqts      1/1     Running             0                30s


$ gitpod /workspace/reserveplace-v3 (main) $ kubectl get po
NAME                       READY   STATUS    RESTARTS         AGE
gateway-55b7667485-v6nqs   1/1     Running   0                25m
my-kafka-0                 1/1     Running   0                5m23s
payment-7c65bb8db5-47skp   1/1     Running   1 (7m5s ago)     9m31s
place-b49b5c96d-hncl5      1/1     Running   10 (9m34s ago)   24m
place-b49b5c96d-ncqts      1/1     Running   0                3m52s
place-b49b5c96d-xgj7p      1/1     Running   0                3m53s


Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  26m                   default-scheduler  Successfully assigned default/place-b49b5c96d-hncl5 to ip-192-168-7-119.eu-west-2.compute.internal
  Normal   Pulled     26m                   kubelet            Successfully pulled image "chather/place:0227" in 4.286742258s (4.286765048s including waiting)
  Normal   Created    26m (x2 over 26m)     kubelet            Created container place
  Normal   Started    26m (x2 over 26m)     kubelet            Started container place
  Normal   Pulled     26m                   kubelet            Successfully pulled image "chather/place:0227" in 640.76274ms (640.775742ms including waiting)
  Warning  Unhealthy  25m (x4 over 26m)     kubelet            Liveness probe failed: Get "http://192.168.7.154:8080/actuator/health": dial tcp 192.168.7.154:8080: connect: connection refused
  Warning  Unhealthy  25m (x3 over 26m)     kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  25m (x2 over 26m)     kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    25m (x2 over 26m)     kubelet            Container place failed liveness probe, will be restarted
  Normal   Pulling    21m (x7 over 26m)     kubelet            Pulling image "chather/place:0227"
  Warning  BackOff    11m (x52 over 24m)    kubelet            Back-off restarting failed container place in pod place-b49b5c96d-hncl5_default(e2bf1311-698f-4517-8d6c-cd870e052f51)
  Warning  Unhealthy  6m40s (x31 over 26m)  kubelet            Readiness probe failed: Get "http://192.168.7.154:8080/actuator/health": dial tcp 192.168.7.154:8080: connect: connection refused
```
- comandtype yaml 배포하여 확인
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: place
  name: place-exec
spec:
  containers:
  - name: place
    image: chather/place:0227
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

$ kubectl get po -w
gitpod /workspace/reserveplace-v3/place (main) $ kubectl get po -w
NAME                       READY   STATUS    RESTARTS      AGE
gateway-55b7667485-v6nqs   1/1     Running   0             40m
my-kafka-0                 1/1     Running   0             20m
payment-7c65bb8db5-47skp   1/1     Running   1 (22m ago)   24m
place-exec                 1/1     Running   0             6s
place-exec                 1/1     Running   1 (1s ago)    16s
place-exec                 1/1     Running   2 (1s ago)    36s
place-exec                 1/1     Running   3 (1s ago)    51s
place-exec                 0/1     CrashLoopBackOff   3 (1s ago)    66s
place-exec                 1/1     Running            4 (31s ago)   96s

```


# 서비스 메쉬 응용 - Istio

-  Istio 설치
```
export ISTIO_VERSION=1.18.1
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=$ISTIO_VERSION TARGET_ARCH=x86_64 sh -
export PATH=$PWD/bin:$PATH // 경로세팅
istioctl install --set profile=demo --set hub=gcr.io/istio-release // demo를 기반으로 core모듈 설치
kubectl get ns // 네임스페이스 생성확인
kubectl get all -n istio-system // 객체생성확인
```
```
// Istio add-on Dashboard 설치
mv samples/addons/loki.yaml samples/addons/loki.yaml.old
curl -o samples/addons/loki.yaml https://raw.githubusercontent.com/msa-school/Lab-required-Materials/main/Ops/loki.yaml
kubectl apply -f samples/addons
kubectl get svc -n istio-system // istio-ingressgateway 서비스 생성 확인, (참고,ClusterIP 타입은 외부서 접속 불가)
```

- 브라우저로 접속
  > aabe5631736f94b76a286ba7133a6608-998086793.eu-west-2.elb.amazonaws.com  // istio-ingressgateway
  > aca2804fff55a41aea0b8bc125a7f3e3-195376226.eu-west-2.elb.amazonaws.com:20001/kiali/  //Kiali  
  > a98bed2bd6bb5407a8fa39036d778df7-687685025.eu-west-2.elb.amazonaws.com //Jaeger
```
$ gitpod /workspace/reserveplace-v3/istio-1.18.1 (main) $ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)                                                                      AGE
grafana                ClusterIP      10.100.102.116   <none>                                                                   3000/TCP                                                                     11m
istio-egressgateway    ClusterIP      10.100.201.11    <none>                                                                   80/TCP,443/TCP                                                               13m
istio-ingressgateway   LoadBalancer   10.100.12.8      aabe5631736f94b76a286ba7133a6608-998086793.eu-west-2.elb.amazonaws.com   15021:31473/TCP,80:30803/TCP,443:32221/TCP,31400:30908/TCP,15443:30239/TCP   13m
istiod                 ClusterIP      10.100.93.181    <none>                                                                   15010/TCP,15012/TCP,443/TCP,15014/TCP                                        13m
jaeger-collector       ClusterIP      10.100.139.59    <none>                                                                   14268/TCP,14250/TCP,9411/TCP                                                 11m
kiali                  LoadBalancer   10.100.147.251   aca2804fff55a41aea0b8bc125a7f3e3-195376226.eu-west-2.elb.amazonaws.com   20001:30939/TCP,9090:32515/TCP                                               11m
loki                   ClusterIP      10.100.112.195   <none>                                                                   3100/TCP,9095/TCP                                                            11m
loki-headless          ClusterIP      None             <none>                                                                   3100/TCP                                                                     11m
loki-memberlist        ClusterIP      None             <none>                                                                   7946/TCP                                                                     11m
prometheus             ClusterIP      10.100.252.117   <none>                                                                   9090/TCP                                                                     10m
tracing                LoadBalancer   10.100.23.194    a98bed2bd6bb5407a8fa39036d778df7-687685025.eu-west-2.elb.amazonaws.com   80:30234/TCP,16685:30658/TCP                                                 11m
zipkin                 ClusterIP      10.100.162.254   <none>                                                                   9411/TCP                                                                     11m                                                                 9411/TCP                                                                     80s
```



- Istio의 기능을 사용하여 네트워크 관리 및 보안을 강화하기 위해 Kubernetes 클러스터에 배포된 애플리케이션에 Sidecar 프록시를 주입
- Sidecar 프록시는 애플리케이션과 통신하여 네트워크 정책, 트래픽 관리 및 보안 기능을 제공
```
//inject Sidecar on Istio environment : 아래 YAML을 deployment.yaml로 저장
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-nginx
  template:
    metadata:
      labels:
        app: hello-nginx
    spec:
      containers:
        - name: hello-nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```
```
istioctl kube-inject -f deployment.yaml > output.yaml //저장후 실행할 커맨드
```
![KakaoTalk_20240222_172406171_09](https://github.com/chaerim94/food-delivery/assets/39048893/1d857ca9-9b14-4b01-aa9b-0e32c0e0eafa)
![KakaoTalk_20240222_172406171_07](https://github.com/chaerim94/food-delivery/assets/39048893/67356844-1058-464b-bfd5-eaccb7f568d1)
![KakaoTalk_20240222_172406171_08](https://github.com/chaerim94/food-delivery/assets/39048893/8a03b22b-3a77-41df-9c4c-148a120ffea8)

![KakaoTalk_20240222_172406171_04](https://github.com/chaerim94/food-delivery/assets/39048893/938325c8-7e6c-406e-88d7-c02e4f1b2c53)
![KakaoTalk_20240222_172406171_05](https://github.com/chaerim94/food-delivery/assets/39048893/ba5e75ae-4c8b-4163-bf93-880d76b25059)



# 통합 모니터링 - Loggregation/Monitoring

- Prometheus/Grafana기반 K8s 통합 모니터링
```
// 통합로깅대상 서비스 설치
kubectl create ns hotel
kubectl apply -f kubernetes/deployment.yaml -n hotel
kubectl expose deploy place --port=8080 -n hotel
kubectl apply -f kubernetes/deployment.yaml -n hotel
kubectl expose deploy payment --port=8080 -n hotel
kubectl apply -f kubernetes/deployment.yaml -n hotel
kubectl expose deploy gateway --port=8080 -n hotel
# hotel에 kafka 설치
helm install my-kafka bitnami/kafka --version 23.0.5 --namespace hotel
# Client Pod deploy
kubectl apply -f https://raw.githubusercontent.com/acmexii/demo/master/edu/siege-pod.yaml -n shop

// Prometheus UI 사용을 위해 Service Scope을 LoadBalancer Type으로 수정
kubectl patch service/prometheus -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'
```
![KakaoTalk_20240222_172406171_12](https://github.com/chaerim94/food-delivery/assets/39048893/73949c96-06b9-4e61-bd85-b8de2043a1aa)

- 계속하기 전 place 서비스 엔드 포인트를 조회한다
``` 
kubectl exec -it pod/siege -n shop -- /bin/bash
root@siege:/# http GET http://place:8080
HTTP/1.1 200 OK
content-type: application/hal+json
date: Thu, 22 Feb 2024 07:07:42 GMT
server: envoy
transfer-encoding: chunked
vary: Origin,Access-Control-Request-Method,Access-Control-Request-Headers
x-envoy-upstream-service-time: 82

{
    "_links": {
        "accommodations": {
            "href": "http://place/accommodations{?page,size,sort}",
            "templated": true
        },
        "placeStatuses": {
            "href": "http://place/placeStatuses{?page,size,sort}",
            "templated": true
        },
        "profile": {
            "href": "http://place/profile"
        }
    }
}
```

- 부하테스트
```
siege -c30 -t40S -v http://place:8080

HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.06 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.10 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.09 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.12 secs:     322 bytes ==> GET  /

Lifting the server siege...
Transactions:                  21440 hits
Availability:                 100.00 %
Elapsed time:                  39.57 secs
Data transferred:               6.58 MB
Response time:                  0.05 secs
Transaction rate:             541.82 trans/sec
Throughput:                     0.17 MB/sec
Concurrency:                   28.76
Successful transactions:       21440
Failed transactions:               0
Longest transaction:            0.41
Shortest transaction:           0.00
```

- Expression Browser에 아래 쿼리로 모니터링
```
rate(istio_requests_total{app="place",destination_service="place.hotel.svc.cluster.local",response_code="200"}[5m])
```
![KakaoTalk_20240222_172406171_13](https://github.com/chaerim94/food-delivery/assets/39048893/64c89711-ec7a-40a6-9bbf-4d75bfdf9f59)


- Grafana 서비스 Open
```
- kubectl patch service/grafana -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'
- aeb5dda45fa96460c913ea4cd793d6dc-1305265694.eu-west-2.elb.amazonaws.com/3000
```
![KakaoTalk_20240222_172406171_15](https://github.com/chaerim94/food-delivery/assets/39048893/77f59da0-507d-47ea-8e50-b0fec5a33139)
![KakaoTalk_20240222_172406171_17](https://github.com/chaerim94/food-delivery/assets/39048893/89f2490d-22a8-441a-aa89-c1521fc2a593)


- 부하테스트
```
siege -c30 -t40S -v http://place:8080
HTTP/1.1 200     0.05 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.03 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /
HTTP/1.1 200     0.04 secs:     322 bytes ==> GET  /

Lifting the server siege...
Transactions:                  33324 hits
Availability:                 100.00 %
Elapsed time:                  39.05 secs
Data transferred:              10.23 MB
Response time:                  0.03 secs
Transaction rate:             853.37 trans/sec
Throughput:                     0.26 MB/sec
Concurrency:                   26.65
Successful transactions:       33329
Failed transactions:               0
Longest transaction:            0.17
Shortest transaction:           0.00

```
![KakaoTalk_20240222_172406171_01](https://github.com/chaerim94/food-delivery/assets/39048893/8b62d651-2af2-4079-8b79-9fca7fa8c208)
![KakaoTalk_20240222_172406171](https://github.com/chaerim94/food-delivery/assets/39048893/f9b4d389-293d-4542-88e3-e5c8e841dc27)

