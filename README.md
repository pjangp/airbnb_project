![image](https://user-images.githubusercontent.com/15603058/119284989-fefe2580-bc7b-11eb-99ca-7a9e4183c16f.jpg)

# 숙소예약(AirBnB)

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 숙소예약](#---)
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

AirBnB 커버하기

기능적 요구사항
1. 호스트가 임대할 숙소를 등록/수정/삭제한다.
2. 고객이 숙소를 선택하여 예약한다.
3. 예약과 동시에 결제가 진행된다.
4. 예약이 되면 예약 내역(Message)이 전달된다.
5. 고객이 예약을 취소할 수 있다.
6. 예약 사항이 취소될 경우 취소 내역(Message)이 전달된다.
7. 숙소에 후기(review)를 남길 수 있다.
8. 전체적인 숙소에 대한 정보 및 예약 상태 등을 한 화면에서 확인 할 수 있다.(viewpage)
9. 결제 승인/취소가 발생하면 손익계산서에 기록한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 예약 건은 성립되지 않아야 한다.  (동기)
    2. 손익 매뉴얼 등록시 PayID가 없으면 손익계산서에 기록하지 않는다. (동기)
    3. 결제가 승인되면 손익계산서에 등록한다.(비동기)
1. 장애격리
    1. 숙소 등록 및 메시지 전송 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시 후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 모든 방에 대한 정보 및 예약 상태 등을 한번에 확인할 수 있어야 한다  (CQRS)
    1. 예약의 상태가 바뀔 때마다 메시지로 알림을 줄 수 있어야 한다  (Event driven)

## 이벤트 스토밍
![image](https://user-images.githubusercontent.com/80744273/120888196-f28dab80-c631-11eb-869e-57c339c0e81c.png)


![image](https://user-images.githubusercontent.com/80744273/121295627-5235d080-c92a-11eb-9c91-e18a45c44f04.png)


## 헥사고날 아키텍처 다이어그램 도출

- Before
	
![image](https://user-images.githubusercontent.com/80744273/119319091-fc6bf200-bcb4-11eb-9dac-0995c84a82e0.png)

- After
	
![image](https://user-images.githubusercontent.com/80744273/120888309-8f504900-c632-11eb-8634-0f442c54399f.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
   mvn spring-boot:run
```

## CQRS

손익조회 조회
- Table 모델링 (profit view)
  ![image](https://user-images.githubusercontent.com/80744273/121295320-c623a900-c929-11eb-96f0-b5b60dbc411d.png)
-  ProfitIssued 이벤트 수신

```
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_AddProfit(@Payload ProfitIssued profitIssued){

        if(profitIssued.isMe()){

                // Sample Logic //
                ProfitView profitView = new ProfitView();
                profitView.setPayId(profitIssued.getPayId());
                profitView.setFlag(profitIssued.getFlag());
                profitView.setAmount(profitIssued.getAmount());
                profitViewRepository.save(profitView);


        }
            
    }
```


## API 게이트웨이
1. gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 gateway 서버의 포트를 8080 으로 설정함
       
          - application.yaml 예시
            ```
		spring:
		  profiles: docker
		  cloud:
		    gateway:
		      routes:
			- id: payment
			  uri: http://payment:8080
			  predicates:
			    - Path=/payments/**, /chk/** 
			- id: room
			  uri: http://room:8080
			  predicates:
			    - Path=/rooms/**, /reviews/**, /check/**
			- id: reservation
			  uri: http://reservation:8080
			  predicates:
			    - Path=/reservations/**
			- id: message
			  uri: http://message:8080
			  predicates:
			    - Path=/messages/** 
			- id: viewpage
			  uri: http://viewpage:8080
			  predicates:
			    - Path= /roomviews/**
			- id: profit
			  uri: http://profit:8080
			  predicates:
			    - Path= /profits/**            
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

2. Kubernetes용 Deployment.yaml 을 작성하고 Kubernetes에 Deploy를 생성함
          - Deployment.yaml 예시
          

            ```
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: gateway
              namespace: airbnb
              labels:
                app: gateway
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: gateway
              template:
                metadata:
                  labels:
                    app: gateway
                spec:
                  containers:
                    - name: gateway
                      image: 247785678011.dkr.ecr.us-east-2.amazonaws.com/gateway:1.0
                      ports:
                        - containerPort: 8080
            ```               
            

            ```
            Deploy 생성
            kubectl apply -f deployment.yaml
            ```     
          - Kubernetes에 생성된 Deploy. 확인
            
![image](https://user-images.githubusercontent.com/80744273/121028268-a46bda00-c7e2-11eb-82ba-ca571edd9a21.png)

            
3. Kubernetes용 Service.yaml을 작성하고 Kubernetes에 Service/LoadBalancer을 생성하여 Gateway 엔드포인트를 확인함. 
          - Service.yaml 예시
          
            ```
            apiVersion: v1
              kind: Service
              metadata:
                name: gateway
                namespace: airbnb
                labels:
                  app: gateway
              spec:
                ports:
                  - port: 8080
                    targetPort: 8080
                selector:
                  app: gateway
                type:
                  LoadBalancer           
            ```             

           
            ```
            Service 생성
            kubectl apply -f service.yaml            
            ```             
            
            
          - API Gateay 엔드포인트 확인
           
            ```
            Service  및 엔드포인트 확인 
            kubectl get svc -n airbnb           
            ```                 
![image](https://user-images.githubusercontent.com/80744273/121028351-b8174080-c7e2-11eb-9b15-ae228d7c573e.png)


# Correlation

결제 서비스와 손익계산서의 연관 프로세스가 구현되어 결제 승인/취소 이벤트가 발생하면 Profit PolicyHandler를 통해 
손익에 반영하고 있습니다.


예약 등록/취소 후 손익계산서 조회
![image](https://user-images.githubusercontent.com/80744273/121029887-f103e500-c7e3-11eb-97ce-3fe0e435b0e8.png)




## 동기식 호출(Sync)

분석 단계에서의 조건 중 하나로 예약 시 숙소(room) 간의 예약 가능 상태 확인 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 또한 예약(reservation) -> 결제(payment) , 결제(payment) -> 손익 (Profit) 서비스도 동기식으로 처리하기로 하였다.


# PaymentController.java - Payment 서비스
```
 @RestController
 public class PaymentController {

    @Autowired
    PaymentRepository paymentRepository;

    @RequestMapping(value = "/chk/chkPayId",
                    method = RequestMethod.GET,
                    produces = "application/json;charset=UTF-8")

    public boolean chkPayId(HttpServletRequest request, HttpServletResponse response) throws Exception {
            System.out.println("##### /chk/chkPayId  called #####");

            // Parameter로 받은 payId 추출
            long payId = Long.valueOf(request.getParameter("payId"));
            System.out.println("######################## chkPayId payId : " + payId);

            // payId 데이터 조회
            Optional<Payment> res = paymentRepository.findById(payId);
            Payment payment = res.get(); // 조회한 payId 데이터
            System.out.println("######################## chkPayId payment.payId() : " + payment.getPayId());

            // payId 가 존재하면 true
            boolean result = false;
            if(payment.getPayId() != null) {
                    result = true;
            } 

            System.out.println("######################## chkPayId Return : " + result);
            return result;
    }    

 }
```

# PaymentService.java - Profit 서비스
```
package airbnb.external;

<import문 생략>

@FeignClient(name="Payment", url="${prop.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.GET, path="/chk/chkPayId")
    public boolean chkPayId(@RequestParam("payId") long payId);

}

```


```
- 손익등록 요청을 받은 직후(@PostPersist) 가능상태 확인 및 결제번호확인을 동기(Sync)로 요청하도록 처리
```
# Profit.java (Entity)
    @PostPersist	
    public void onPostPersist(){
        ////////////////////////////
        // 손익등록시 결제번호 유효성 검증
        ////////////////////////////

        //airbnb.external.Payment payment = new airbnb.external.Payment();
        // mappings goes here
        boolean result = ProfitApplication.applicationContext.getBean(airbnb.external.PaymentService.class)
            .chkPayId(this.getPayId());

            System.out.println("######## Check Result : " + result);

            if(result) {  
				// 이벤트 발행 -> ProfitIssued
				ProfitIssued profitIssued = new ProfitIssued();
				BeanUtils.copyProperties(this, profitIssued);
				profitIssued.publishAfterCommit();				
            }

    }

```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```


## 비동기식
```
결제가 이루어진 후에 손익 시스템과의 통신 행위는 비동기식으로 처리한다.
 
- 이를 위하여 결제가 승인되면 결제가 승인 되었다는 이벤트를 카프카로 송출한다. (Publish)
 
```
# Payment.java - 결제 승인완료 이벤트 발생.

```
package airbnb;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Payment_table")
public class Payment {

    ....

    @PostPersist
    public void onPostPersist(){
        ////////////////////////////
        // 결제 승인 된 경우
        ////////////////////////////

        // 이벤트 발행 -> PaymentApproved
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }
 
}
```

- 손익 시스템에서는 결제 승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:


# PROFIT/PolicyHandler.java - 결제 승인완료 이벤트 수신.

```
public class PolicyHandler{
    @Autowired ProfitRepository profitRepository;


    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_AddProfit(@Payload PaymentApproved paymentApproved){

        if(paymentApproved.isMe()){

                // Sample Logic //
                Profit profit = new Profit();
                profit.setPayId(payId);
                profit.setFlag("P");
                profit.setAmount((long)0);
                profitRepository.save(profit);
        }
            
    }

```
이벤트 수신에 따라 처리되기 때문에, profit 서비스가 유지보수로 인해 잠시 내려간 상태 라도 예약/결제를 받는데 문제가 없다.


# 손익 서비스 (PROFIT) 를 잠시 내려놓음 (ctrl+c)

# 예약 요청
```
http POST http://localhost:8088/reservations roomId=1 status=reqReserve   #Success
```
# 예약 상태 확인
```
http GET localhost:8088/reservations    #메시지 서비스와 상관없이 예약 상태는 정상 확인

```

# 운영
## Deploy
```
   - 도커 이미지 생성
   	docker build -t 075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-airbnb-profit:2.0 .
   - ECR 에 도커 이미지 등록
   	docker push 075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-airbnb-profit:2.0
   - Deployment 생성
   	kubectl apply -f kubernetes/deployment.yml
   - Service 생성 
   	kubectl apply -f kubernetes/service.yaml
```

![image](https://user-images.githubusercontent.com/80744273/121303804-75667d00-c936-11eb-8f39-7082a7164af0.png)


## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD는 buildspec.yml을 이용한 AWS codebuild를 사용하였습니다.

- CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅을 한다

![image](https://user-images.githubusercontent.com/80744273/121305814-0179a400-c939-11eb-96e6-2ff25d388cf2.png)


buildspec.yml 파일 
```
version: 0.2

env:
  variables:
    _PROJECT_NAME: "test01-profit"
    _IMAGE_VERSION: "1.0"
    _AWS_DEFAULT_REGION: "ap-northeast-2"
    

phases:
  install:
    runtime-versions:
      java: openjdk8
      docker: 18
    commands:
      - echo install kubectl
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
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
#      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$_IMAGE_VERSION  . 
      - docker build -t 075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-profit:2.0 .
  post_build:
    commands:
      - echo Pushing the Docker image...
#      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$_IMAGE_VERSION
      - docker push 075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-profit:2.0
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - kubectl replace -f kubernetes/deployment.yml --force
      - kubectl replace -f kubernetes/service.yaml --force
#      - kubectl replace  -f kubernetes/destination-rule.yml --force

cache:
  paths:
    - '/root/.m2/**/*'
```

- codebuild 실행
codebuild 프로젝트 및 빌드 이력

![image](https://user-images.githubusercontent.com/80744273/121308605-34716700-c93c-11eb-9640-f3cd35d7b019.png)





### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- Metric Server 설치
	kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
	kubectl get deployment metrics-server -n kube-system

- profit deployment.yml 파일에 resources 설정을 추가한다

```
    spec:
      containers:
        - name: profit
          image: 075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-airbnb-profit:1.9
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "1000m"
              memory: "256Mi"
            limits:
              cpu: "2500m"
              memory: "512Mi"
```	
- profit 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 1프로를 넘어서면 replica 를 10개까지 늘려준다:

```
kubectl autoscale deployment profit -n airbnb --cpu-percent=1 --min=1 --max=10
```

- 부하를 동시사용자 100명, 1분 동안 걸어준다.
```
siege -c100 -t60S -v --content-type "application/json" 'http://profit:8080/profits POST {"pay_id":"1", "amounT":"1000", "flag":"P"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다
```
kubectl get deploy profit -w -n airbnb 
```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/80744273/121148686-f4977a80-c87c-11eb-91f4-fae80e27c9e8.png)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Lifting the server siege...
Transactions:                  15615 hits
Availability:                 100.00 %
Elapsed time:                  59.44 secs
Data transferred:               3.90 MB
Response time:                  0.32 secs
Transaction rate:             262.70 trans/sec
Throughput:                     0.07 MB/sec
Concurrency:                   85.04
Successful transactions:       15675
Failed transactions:               0
Longest transaction:            2.55
Shortest transaction:           0.01
```

## 무정지 재배포

- 새버전으로의 배포 시작
```
kubectl set image deployment profit profit=075134174875.dkr.ecr.ap-northeast-2.amazonaws.com/test01-airbnb-profit:2.0  -n airbnb 
```

![image](https://user-images.githubusercontent.com/80744273/121157546-9ff7fd80-c884-11eb-8714-91c6d22bf9b9.png)


- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

```
siege -c1000 -t60S -r10 -v --content-type "application/json" 'http://profit:8080/profits POST {"pay_id":"1", "amounT":"1000", "flag":"P"}'


Transactions:                   3464 hits
Availability:                  73.05 %
Elapsed time:                  22.24 secs
Data transferred:               0.80 MB
Response time:                  1.27 secs
Transaction rate:             155.76 trans/sec
Throughput:                     0.04 MB/sec
Concurrency:                  198.54
Successful transactions:        3464
Failed transactions:            1278
Longest transaction:            8.55
Shortest transaction:           0.08
 


```
- 배포기간중 Availability 가 평소 100%에서 73.05% 대로 떨어지는 것을 확인. 이를 막기위해 Readiness Probe 를 설정함

```
# deployment.yaml 의 readiness probe 의 설정:
```

```
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
```


```
kubectl apply -f kubernetes/deployment.yml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Lifting the server siege...
Transactions:                   6692 hits
Availability:                 100.00 %
Elapsed time:                  59.60 secs
Data transferred:               1.54 MB
Response time:                  2.21 secs
Transaction rate:             112.28 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                  248.60
Successful transactions:        6692
Failed transactions:               0
Longest transaction:           19.05
Shortest transaction:           0.04

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


# Self-healing (Liveness Probe)
- profit deployment.yml 파일 수정 
```
콘테이너 실행 후 /tmp/healthy 파일을 만들고 
90초 후 삭제
livenessProbe에 'cat /tmp/healthy'으로 검증하도록 함
```


![image](https://user-images.githubusercontent.com/80744273/121273844-2142a500-c904-11eb-8e9f-03b74fcbb510.png)


- kubectl describe pod profit -n airbnb 실행으로 확인
```
컨테이너 실행 후 90초 동인은 정상이나 이후 /tmp/healthy 파일이 삭제되어 livenessProbe에서 실패를 리턴하게 됨
pod 정상 상태 일때 pod 진입하여 /tmp/healthy 파일 생성해주면 정상 상태 유지됨
```
kubectl exec -ti pod/profit-6ff99989bc-8bjhg -n airbnb -- touch /tmp/healthy

![image](https://user-images.githubusercontent.com/80744273/121279492-98316b00-c90f-11eb-833c-024f05dfc622.png)
![image](https://user-images.githubusercontent.com/80744273/121279598-c6af4600-c90f-11eb-944d-0377c2e8c65f.png)

# Config Map/ Persistence Volume
- Persistence Volume

1: EFS 생성
```
EFS 생성 시 클러스터의 VPC를 선택해야함
```

![클러스터의 VPC를 선택해야함]![image](https://user-images.githubusercontent.com/80744273/121289114-704a0380-c91f-11eb-939a-7fe45d87160b.png)
![EFS생성]![image](https://user-images.githubusercontent.com/80744273/121289010-44c71900-c91f-11eb-8eee-326ed6fee2d7.png)

![image](https://user-images.githubusercontent.com/80744273/121292414-e0a75380-c924-11eb-84a7-329bd6856d47.png)




2. EFS 계정 생성 및 ROLE 바인딩
```
kubectl apply -f efs-sa.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
  namespace: airbnb


kubectl get ServiceAccount efs-provisioner -n airbnb
NAME              SECRETS   AGE
efs-provisioner   1         9m1s  
  
  
  
kubectl apply -f efs-rbac.yaml

namespace를 반듯이 수정해야함

  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: efs-provisioner-runner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
     # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: Role
  name: leader-locking-efs-provisioner
  apiGroup: rbac.authorization.k8s.io


```

3. EFS Provisioner 배포
```
kubectl apply -f efs-provisioner-deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-provisioner
  namespace: airbnb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: efs-provisioner
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccount: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:latest
          env:
            - name: FILE_SYSTEM_ID
              value: fs-562f9c36
            - name: AWS_REGION
              value: ap-northeast-2
            - name: PROVISIONER_NAME
              value: my-aws.com/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-562f9c36.efs.ap-northeast-2.amazonaws.com
            path: /


kubectl get Deployment efs-provisioner -n airbnb
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
efs-provisioner   1/1     1            1           16s

```

4. 설치한 Provisioner를 storageclass에 등록
```
kubectl apply -f efs-storageclass.yml


kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aws-efs
  namespace: airbnb
provisioner: my-aws.com/aws-efs


kubectl get sc aws-efs -n airbnb
NAME      PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
aws-efs   my-aws.com/aws-efs   Delete          Immediate           false                  14d

```

5. PVC(PersistentVolumeClaim) 생성
```
kubectl apply -f volume-pvc.yml


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aws-efs
  namespace: airbnb
  labels:
    app: test-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 6Ki
  storageClassName: aws-efs
  
  
kubectl get pvc aws-efs -n airbnb
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
aws-efs   Bound    pvc-78729e12-18ba-483d-9bca-649886984670   6Ki        RWX            aws-efs        15s


```

6. profit/payment pod 적용
```
kubectl apply -f deployment.yml
```
![pod with pvc](https://user-images.githubusercontent.com/38099203/119349966-bd9c6300-bcd9-11eb-9f6d-08e4a3ec82f0.PNG)


7. A pod에서 마운트된 경로에 파일을 생성하고 B pod에서 파일을 확인함
```
![image](https://user-images.githubusercontent.com/80744273/121293819-3a108200-c927-11eb-8fc5-eaf659199ca7.png)


kubectl exec -it pod/profit-fb8bc5484-6pzlz profit -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # touch common_code


```
![profit pod에서 파일확인](https://user-images.githubusercontent.com/80744273/121293721-0d5c6a80-c927-11eb-82ce-e6cec09f1a1c.png)

```
kubectl exec -it pod/payment-6d9f8858fd-4s8v5 payment -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # ls -al
total 8
drwxrws--x    2 root     2000          6144 May 24 15:44 .
drwxr-xr-x    1 root     root            17 May 24 15:42 ..
-rw-r--r--    1 root     2000             0 May 24 15:44 intensive_course_work
```
![payment pod에서 파일확인](https://user-images.githubusercontent.com/80744273/121293748-19e0c300-c927-11eb-8f92-564925e6a885.png)


- Config Map

1: cofingmap.yml 파일 생성
```
kubectl apply -f cofingmap.yml


apiVersion: v1
kind: ConfigMap
metadata:
  name: airbnb-config
  namespace: airbnb
data:
  # 속성과 비슷한 키; 각 키는 간단한 값으로 매핑됨
  max_reservation_per_person: "10"
  db-host: xxxxxxxxxxxxxxxxx.ap-northeast-2.rds.amazonaws.com
  ui_properties_file_name: "user-interface.properties"
```

2. deployment.yml에 적용하기

```
kubectl apply -f deployment.yml


.......
          env:
			# cofingmap에 있는 단일 key-value
            - name: MAX_RESERVATION_PER_PERSION
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: max_reservation_per_person
           - name: UI_PROPERTIES_FILE_NAME
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: ui_properties_file_name
            - name: DB-HOST
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: db-host    		  
          volumeMounts:
          - mountPath: "/mnt/aws"
            name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: aws-efs
```
profit 서비스의 application.yml 변경
```
  datasource:
    # jndi-name: jdbc/sqlserver/skupv
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    url: jdbc:sqlserver://${DB-HOST}:1433;DatabaseName=eco
```




