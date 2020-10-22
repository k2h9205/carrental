# Table of contents

- [렌트카 예약](#---)
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
    - [Configmap](#configmap)

# 서비스 시나리오



기능적 요구사항
1.	매니저는 렌트카를 등록할 수 있다.
2.	고객이 렌트카를 선택해 예약하면 결제가 진행된다.
3.	예약이 결제되면 렌트카의 예약 가능 여부가 변경된다.
4.	렌트카가 예약 불가 상태로 변경되면 예약이 확정된다.
5.	고객은 예약을 취소할 수 있다.
6.	예약이 취소되면 결제가 취소되고, 렌트카의 예약 가능 여부가 변경된다.
7.	고객은 렌트카 예약가능여부를 확인할 수 있다.




비기능적 요구사항
1. 트랜잭션
    1. 결제가 완료 되지 않은 예약 건은 예약이 성립되지 않는다.  Sync 호출
    2. 예약과 결제는 동시에 진행된다.  Sync 호출
    3. 예약 취소와 결제 취소는 동시에 진행된다.  Sync 호출
2. 장애격리
    1. 렌트카관리 시스템이 수행되지 않더라도 예약 / 결제는 365일 24시간 받을 수 있어야 한다.  Async 호출 (event-driven)
    2. 렌트카관리 시스템이 과중되면 예약 / 결제를 받지 않고 결제 취소를 잠시 후에 하도록 유도한다.  Circuit breaker, fallback
    3. 결제가 취소되면 렌트카의 예약 취소가 확정되고, 렌트카의 예약 가능 여부가 변경된다.  Circuit breaker, fallback
3. 성능
    1. 고객은 렌트카 예약 가능 여부를 확인할 수 있다.  CQRS
    2. 예약/결제 취소 정보가 변경 될 때마다 렌트카 예약 가능 여부가 변경 될 수 있어야 한다.  Event Driven





# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/qmad35C2LBcAbGuuvevOqnvnzOp1/local/e52331bbb6b4ba54020a9896ff4aaac2/init


### 이벤트 도출
![1](https://user-images.githubusercontent.com/54618778/96805071-af3c4f00-144b-11eb-971b-4a71ba7fc504.png)



### 어그리게잇으로 묶기 / 액터, 커맨드 부착하여 읽기 좋게
![2](https://user-images.githubusercontent.com/54618778/96805078-b2cfd600-144b-11eb-9f1f-3fc83f95136e.png)

    - 렌트카 예약, 결제, 렌트카 관리 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어준다.

### 바운디드 컨텍스트로 묶기

![3](https://user-images.githubusercontent.com/54618778/96805076-b2cfd600-144b-11eb-8dbb-ac5188c57a43.png)

    - 도메인 서열 분리 
        - 예약 : 고객 예약 오류를 최소화 한다. (Core)
        - 결제 : 결제 오류를 최소화 한다. (Supporting)
        - 렌트카관리 : 렌트카 예약 상태 오류를 최소화 한다. (Supporting)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![121212](https://user-images.githubusercontent.com/54618778/96825728-c63f6900-146c-11eb-8199-adf51a2903ee.png)


### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![4](https://user-images.githubusercontent.com/54618778/96805075-b19ea900-144b-11eb-8d20-66ca132c9d36.png)

### 완성된 1차 모형

![4](https://user-images.githubusercontent.com/54618778/96805075-b19ea900-144b-11eb-8d20-66ca132c9d36.png)



### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![555](https://user-images.githubusercontent.com/54618778/96828062-db6ac680-1471-11eb-8306-3e6bfe9c8f4e.png)

    - 고객이 렌트카 예약 가능 여부를 확인한다.(?)
    - 고객이 렌트카를 선택해 예약을 진행한다. (OK)
    - 예약 시 자동으로 결제가 진행된다. (OK)
    - 결제가 성공하면 렌트카가 예약불가 상태가 된다. (OK)
    - 렌트카 상태 변경 시 예약이 확정상태가 된다. (OK)    
    

![777](https://user-images.githubusercontent.com/54618778/96828477-b88ce200-1472-11eb-962a-6629e996f183.png)

    - 고객이 예약/결제를 취소한다. (OK)
    - 예약/결제 취소 시 자동 예약/결제 취소된다. (OK)
    - 결제가 취소되면 렌트카는 예약가능 상태가 된다. (OK)






### 모델 수정

![666](https://user-images.githubusercontent.com/54618778/96828417-9b581380-1472-11eb-9a6c-44ed995fdf09.png)

    
    - View Model 추가
    - 수정된 모델은 모든 요구사항을 커버함.


### 비기능 요구사항에 대한 검증

![666](https://user-images.githubusercontent.com/54618778/96828417-9b581380-1472-11eb-9a6c-44ed995fdf09.png)



    - 렌트카 등록 서비스를 예약/결제 서비스와 격리하여 렌트카 등록 서비스 장애 시에도 예약이 가능
    - 렌트카가 예약 불가 상태일 경우 예약 확정이 불가함
    - 먼저 결제가 이루어진 렌트카에 대해서는 예약을 불가 하도록 함.    









## 헥사고날 아키텍처 다이어그램 도출
    
![헥사고날11](https://user-images.githubusercontent.com/54618778/96803579-b19caa00-1447-11eb-8d2e-dd63b8e84a13.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 단계별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd reservation
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd carmanagement
mvn spring-boot:run  

cd mypage
mvn spring-boot:run  
```

게이트웨이 내부에서 spring, docker 환경에 따른 각 서비스 uri를 설정해주고 있다.
```
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://localhost:8081
          predicates:
            - Path=/reservations/** 
        - id: payment
          uri: http://localhost:8082
          predicates:
            - Path=/payments/** 
        - id: carManagement
          uri: http://localhost:8083
          predicates:
            - Path=/carManagements/** 
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
.....생략

---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://reservation:8080
          predicates:
            - Path=/reservations/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: carmanagement
          uri: http://carmanagement:8080
          predicates:
            - Path=/carmanagements/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
...
```




## DDD 의 적용

- 각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언
- 각 서비스는 연동 서비스의 key id를 가지고 있어 어떤 건에 대한 요청인지 구별 가능하다. (Correlation-key)
```
package carrental;

import org.springframework.beans.BeanUtils;

import javax.persistence.*;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long carNo;
    .../... 중략  .../...
    private Double carPrice;

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    
    public Long getCarNo() {
        return carNo;
    }
    public void setCarNo(Long carNo) {
        this.carNo = carNo;
    }
    .../... 중략  .../...

}
```



- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록    
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용
```
package carrental;
import org.springframework.data.repository.PagingAndSortingRepository;
    public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{

}
```
   
   
---
#### 적용 후 REST API 의 테스트

1. 렌트카1 등록(local)
``` http POST http://localhost:8083/carManagements carNo=1111 status=WAITING carPrice=30000000 ```

![렌트카등록1](https://user-images.githubusercontent.com/54618778/96792995-76de4600-1436-11eb-93a1-1a95edd8c2c4.png)


2. 렌트카2 등록(local)
``` http POST http://localhost:8083/carManagements carNo=2222 status=WAITING carPrice=50000000 ```

![렌트카등록2](https://user-images.githubusercontent.com/54618778/96793000-78a80980-1436-11eb-8b8a-d1a98795842e.png)


3. 등록된 렌트카 조회(local)
``` http localhost:8083/carManagements ```

![렌트카조회](https://user-images.githubusercontent.com/54618778/96793001-7940a000-1436-11eb-8648-af61e3bd5cf9.png)


4. 렌트카1 예약 (local)
``` http POST http://localhost:8081/reservations carNo=1111 reserveDate=20201020 status=RESERVED carPrice=30000000 ```

![예약1](https://user-images.githubusercontent.com/54618778/96793005-7a71cd00-1436-11eb-8828-e5424ee16580.png)


5. 렌트카2 예약(local)
``` http POST http://localhost:8081/reservations carNo=2222 reserveDate=20201021 status=RESERVED carPrice=50000000 ```

![예약2](https://user-images.githubusercontent.com/54618778/96793007-7b0a6380-1436-11eb-96cc-0693155cb173.png)


6. 렌트카1 예약 취소(local)
``` http http://localhost:8081/reservations carNo=1111 reserveCancelDate=20201020 status=RESERVATION_CANCELED ```

![렌트카예약취소](https://user-images.githubusercontent.com/54618778/96793501-7db98880-1437-11eb-824e-832619559993.png)


7. 예약 보기(local)
```http localhost:8081/books ```

![예약보기](https://user-images.githubusercontent.com/54618778/96793009-7ba2fa00-1436-11eb-8dc5-34bfbff2a7b4.png)



- 결제 요청 (aws)-> gateway 통한 api호출 확인!

![결제(운영)](https://user-images.githubusercontent.com/54618778/96828969-934ca380-1473-11eb-8c0a-58961ea07eb8.png)

- 뷰(mypage) 조회 (aws)

![뷰(운영)](https://user-images.githubusercontent.com/54618778/96828977-98115780-1473-11eb-8c4e-d483a361757d.png)

---


## 폴리글랏 퍼시스턴스

각 마이크로서비스는 별도의 H2 DB를 가지고 있으며 CQRS를 위한 Mypage에서는 H2가 아닌 HSQLDB를 적용하였다.

```
# Mypage의 pom.xml에 dependency 추가
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>


```





## 동기식 호출과 Fallback 처리
Reservation → Payment 간 호출은 동기식 일관성 유지하는 트랜잭션으로 처리.     
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출.     

```
BookApplication.java.
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableBinding(KafkaProcessor.class)
@EnableFeignClients
public class ReservationApplication {
    protected static ApplicationContext applicationContext;
    public static void main(String[] args) {
        applicationContext = SpringApplication.run(ReservationApplication.class, args);
    }
}
```

FeignClient 방식을 통해서 Request-Response 처리.     
Feign 방식은 넷플릭스에서 만든 Http Client로 Http call을 할 때, 도메인의 변화를 최소화 하기 위하여 interface 로 구현체를 추상화.    
→ 실제 Request/Response 에러 시 Fegin Error 나는 것 확인   




- 예약 받은 직후(@PostPersist) 결제 요청함
```
-- Reservation.java
    @PostPersist
    public void onPostPersist(){
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        carrental.external.Payment payment = new carrental.external.Payment();
        // mappings goes here
        
        payment.setCarNo(this.getCarNo());
        payment.setStatus(reserved.getStatus());
        ...// 중략 //...

        ReservationApplication.applicationContext.getBean(carrental.external.PaymentService.class)
            .paymentRequest(payment);

    }
```



- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인함.   
```
Reservation -- (http request/response) --> Payment

# Payment 서비스 종료

# Reservation 등록
http POST http://localhost:8081/reservations carNo=1111 reserveDate=20201020 status=RESERVED carPrice=30000000    #Fail!!!!
```
Payment를 종료한 시점에서 상기 Reservation 등록 Script 실행 시, 500 Error 발생.
("Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction")   

![500error](https://user-images.githubusercontent.com/54618778/96793715-d8eb7b00-1437-11eb-8df1-2a9ff7f34e0c.png)

---
## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

Payment가 이루어진 후에(PAID) CarManagement시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리.   
CarManagement 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리.   
이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish).   

- CarManagement 서비스에서는 PAID 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:   
```
@Service
public class PolicyHandler{

    @Autowired
    CarManagementRepository carManagementRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Rental(@Payload Paid paid){

        if(paid.isMe()){
            System.out.println("##### listener Rental : " + paid.toJson());

            List<CarManagement> optional = carManagementRepository.findByCarNo(paid.getCarNo());
            for(CarManagement carManagement : optional) {
                carManagement.setStatus("RENTED");
                carManagementRepository.save(carManagement);
            }
        }
    }
```

- CarManagement 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, CarManagement 시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# CarManagement Service 를 잠시 내려놓음 (ctrl+c)

#PAID 처리
http http://localhost:8082/payments status=PAID carNo=1111 houseId=1 paymentDate=20201021  #Success!!

#결제상태 확인
http http://localhost:8082/payments  #제대로 Data 들어옴   

#CarManagement 서비스 기동
cd carmanagement
mvn spring-boot:run

#CarManagement 상태 확인
http http://localhost:8083/carManagements     # 제대로 kafka로 부터 data 수신 함을 확인
```


---



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

Webhook으로 연결되어 github에서 수정 시 혹은 codebuild에서 곧바로 빌드가 가능하다.

![빌드1](https://user-images.githubusercontent.com/54618778/96833416-5c7a8b80-147b-11eb-8766-9bca5745a898.JPG)

## 동기식 호출 / 서킷 브레이킹 / 장애격리

서킷 브레이킹은 istio 데스티네이션 룰을 이용하여 구현하였다.
시나리오는 reservation -> payment 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.



- 피호출 서비스(결제:payment) 의 추가로 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게 
```
# Payment.java (Entity)
    @PostPersist
    public void onPostPersist(){
        System.out.println("##### onPostPersist status = " + this.getStatus());
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
            System.out.println("##### SLEEP");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        if(this.getStatus().equals("RESERVED")) {
            Paid paid = new Paid();
            BeanUtils.copyProperties(this, paid);
            paid.setStatus("PAID");
            paid.publishAfterCommit();
        } 
    }
```

일단 서킷브레이커 미적용 시, 모든 요청이 성공했음을 확인

![미적용100](https://user-images.githubusercontent.com/54618778/96838656-188b8480-1483-11eb-84f8-3a35bc86eb3a.JPG)


데스티네이션 룰 적용
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-payment
  namespace: istio-cb-ns
spec:
  host: payment
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 5
        maxRequestsPerConnection: 5
    outlierDetection:
      interval: 1s
      consecutiveErrors: 1
      baseEjectionTime: 10m
      maxEjectionPercent: 100
EOF
```
적용 후 부하테스트 시 서킷브레이커의 동작으로 미연결된 결과가 보임
80%정도의 요청 성공률 확인

![적용84](https://user-images.githubusercontent.com/54618778/96838659-19bcb180-1483-11eb-8840-af748eabdd48.JPG)

키알리 화면 캡쳐
![image](https://user-images.githubusercontent.com/70302894/96579638-1eae2380-1312-11eb-8b9e-c3e21aec7d75.JPG)

예거 화면 캡쳐
![예거](https://user-images.githubusercontent.com/70302894/96673034-8743e180-13a0-11eb-9617-d9ab5149590f.JPG)


- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 더 과부하를 걸면 반 이상이 실패 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.




### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy skccuser04-payment --min=1 --max=10 --cpu-percent=10 -n istio-cb-ns

kubectl autoscale deployment.apps/skccuser04-payment --cpu-percent=10 --min=1 --max=10 -n istio-cb-ns
```

오토스케일을 위한 metrics-server를 설치하고 배포한다. 적용한 istrio injection을 해제한다.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

kubectl get deployment metrics-server -n kube-system

kubectl label namespace istio-cb-ns istio-injection=disabled --overwrite

```
- CB 에서 했던 방식대로 워크로드를 120초 동안 걸어준다.
```
siege -c20 -t120S -v  --content-type "application/json" 'http://skccuser04-payment:8080/payments POST {"id":"1","houseId":"1","bookId":"1","status":"BOOKED"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
watch kubectl get deploy skccuser04-payment -n istio-cb-ns
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:


![오토스케일링](https://user-images.githubusercontent.com/54618778/96845620-d7e43900-148b-11eb-9ddd-dd12b1546c17.JPG)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![오토스케일 적용 90](https://user-images.githubusercontent.com/54618778/96840372-51c4f400-1485-11eb-867c-e7946d9b4077.JPG)




## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.


![무정지 readiness 제거](https://user-images.githubusercontent.com/54618778/96843984-ea5d7300-1489-11eb-8b5e-23aa204cb6a6.JPG)



배포기간중 Availability 가 평소 100%에서 80% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 설정을 다시 추가:

```
# deployment.yaml 에 readiness probe 설정
      containers:
        - name: payment
          image: username/payment:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10

```
- git commit 이후 자동배포 시 siege 돌리고 Availability 확인:
배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.



![무정지 추가](https://user-images.githubusercontent.com/54618778/96843987-eb8ea000-1489-11eb-8368-cdaf01430569.JPG)



# configmap
carmanagement 서비스의 경우, 국가와 지역에 따라 설정이 변할 수도 있음을 가정할 수 있다.   
configmap에 설정된 국가와 지역 설정을 carmanagement 서비스에서 받아 사용 할 수 있도록 한다.   
   
아래와 같이 configmap을 생성한다.   
data 필드에 보면 country와 region정보가 설정 되어있다. 
##### configmap 생성
```
kubectl apply -f - <<EOF 
apiVersion: v1
kind: ConfigMap
metadata:
  name: carmanagement-region
  namespace: istio-cb-ns
data:
  country: "korea"
  region: "seoul"
EOF
```
 
carmanagement deployment를 위에서 생성한 carmanagement-region(cm)의 값을 사용 할 수 있도록 수정한다.
###### configmap내용을 deployment에 적용 
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carmanagement
  labels:
    app: carmanagement
...
    spec:
      containers:
        - name: carmanagement
          env:                                                 ##### 컨테이너에서 사용할 환경 변수 설정
            - name: COUNTRY
              valueFrom:
                configMapKeyRef:
                  name: carmanagement-region
                  key: country
            - name: REGION
              valueFrom:
                configMapKeyRef:
                  name: carmanagement-region
                  key: region
          volumeMounts:                                                 ##### CM볼륨을 바인딩
          - name: config
            mountPath: "/config"
            readOnly: true
...
      volumes:                                                 ##### CM 볼륨 
      - name: config
        configMap:
          name: carmanagement-region
```
configmap describe 시 확인 가능

![컨픽맵2](https://user-images.githubusercontent.com/70302894/96668946-37145180-1397-11eb-8465-0a34c4e271f4.JPG)
