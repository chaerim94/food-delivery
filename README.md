![image](https://github.com/chaerim94/labshoppubsub/assets/39048893/d780e53e-b820-48e9-b5de-bf66a5bd6b3d)

# 숙박예약



# Table of contents

- [예제 - 숙박예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

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












* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) 결제이력.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/orders

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/orders   
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           17s
pay     1         2         1            1           45s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5078 hits
Availability:		       92.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


# 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## 마케팅팀의 추가
    - KPI: 신규 고객의 유입률 증대와 기존 고객의 충성도 향상
    - 구현계획 마이크로 서비스: 기존 customer 마이크로 서비스를 인수하며, 고객에 음식 및 맛집 추천 서비스 등을 제공할 예정

## 이벤트 스토밍 
    ![image](https://user-images.githubusercontent.com/487999/79685356-2b729180-8273-11ea-9361-a434065f2249.png)


## 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/487999/79685243-1d704100-8272-11ea-8ef6-f4869c509996.png)

## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 마이크로서비스의 경우 API 변화나 Retire 시에 app(주문) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

                --> 

        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제2(pay);

    }
```

예) Retire 시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        /**
        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

        **/
    }
```
