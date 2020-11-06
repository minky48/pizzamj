
# 개인과제 - 피자주문배달+환불 시스템 

본 프로그램은 피자주문배달에 환불기능이 포함된 시스템이다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [최종조별과제 - 피자주문배달시스템](#---)
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

# 서비스 시나리오 

기능적 요구사항
1. 고객이 피자종류와 수량을 주문한다.
2. 고객이 결제한다
3. 결제가 되면 배달 시작한다
4. 고객이 주문을 취소할 수 있다
5. 주문이 취소되면 결제가 취소된다
6. 고객이 주문상태를 중간중간 조회한다
7. 배송이 완료되면 쿠폰이 지급된다.
8. 고객으로 부터 환불을 처리한다.
9. 환불 처리가 되면 배송 상태를 변경한다.
10. 환불 처리 시 쿠폰은 회수된다.
11. 환불 배송을 취소하면 환불도 취소가 된다.
12. 환불 시 입력한 환불 샤유를 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 주문이 완료되어야 결제가 가능하다.  Sync 호출 
    1. 환불이 안 된 배송상태는 변경되지 않아야 한다. Sync 호출
2. 장애격리
    1. 쿠폰발급기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 주문을 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
    1. 쿠폰시스템이 수행되지 않더라도 환불은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    1. 배송 시스템이 과중되면 주문을 잠시동안 받지 않고 환불을 잠시후에 하도록 유도한다 Circuit breaker, fallback
3. 성능
    1. 고객이 주문에 대한 상태를 시스템에서 확인할 수 있다 CQRS
    1. 배달이 완료되면 쿠폰이 발행된다  Event driven
    1. 고객이 환불상태를 시스템에서 확인할 수 있다 CQRS


# 체크포인트

- 분석 설계

  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계
    
### 이벤트 도출
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/NkR3vaIgD8P8F2y2ub0y45zZLHz2/mine/47c56b46d0d4b9710eeb203d13095a53/-MLLPAsFiPAX5VEAYy-5


![image](https://user-images.githubusercontent.com/70673848/98230891-0b03ed80-1f9f-11eb-9d9e-c22248b3483b.png)


 도메인 서열 분리 
   
    - Core Domain:  order,  delivery : 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 request의 경우 1주일 1회 미만, delivery의 경우 1개월 1회 미만
    
    - Supporting Domain:   statusview, coupon, refund   : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 
                                                          배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
    
    - General Domain:   Payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)



## 헥사고날 아키텍처 다이어그램 도출


![image](https://user-images.githubusercontent.com/70673848/98231718-20c5e280-1fa0-11eb-875f-8004592fc5e7.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8086이다)

```
cd order
mvn spring-boot:run

cd payment
mvn spring-boot:run

cd delivery
mvn spring-boot:run

cd statusview
mvn spring-boot:run

cd coupon
mvn spring-boot:run

cd refund
mvn spring-boot:run 

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 모든 구현에 있어서 영문으로 사용하여 별다른 오류없이 구현하였다.

refund.java

```
package pizzamj;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Refund_table")
public class Refund {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String reason;

    @PostPersist
    public void onPostPersist(){
        Refunded refunded = new Refunded();
        BeanUtils.copyProperties(this, refunded);
        refunded.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        pizzamj.external.Delivery delivery = new pizzamj.external.Delivery();

        delivery.setOrderId(this.getOrderId());
        delivery.setDeliveryStatus("refunded");

        // mappings goes here
        RefundApplication.applicationContext.getBean(pizzamj.external.DeliveryService.class)
            .delivery(delivery);


    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    public String getReason() {
        return reason;
    }

    public void setReason(String reason) {
        this.reason = reason;
    }

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
RefundRepository.java

```
package pizzamj;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface RefundRepository extends PagingAndSortingRepository<Refund, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# 주문처리
http http://localhost:8081/orders pizzaId=10 qty=1

```

![image](https://user-images.githubusercontent.com/70673848/98238682-1c9ec280-1faa-11eb-8522-1ca141843920.png)


```
# 주문 상태 확인
http localhost:8081/orders/2

```
![image](https://user-images.githubusercontent.com/70673848/98238741-33451980-1faa-11eb-8ee8-ad44298a0ac0.png)

## 폴리글랏 퍼시스턴스

H2가 아닌 Derby in-memory DB를 사용함

```
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.4.0</version>
    <scope>runtime</scope>
</dependency>
```
![image](https://user-images.githubusercontent.com/70673848/98238952-8a4aee80-1faa-11eb-8b8f-663d836fec13.png)



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 환불(refund)->배송(delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 배송서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
DeliveryService.java

```
package pizzamj.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="delivery", url="${api.url.delivery}")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.GET, path="/deliveries")
    public void delivery(@RequestBody Delivery delivery);

}
```

- 환불요청을 받은 직후(@PostPersist) 배송을 요청하도록 처리

 refund.java (Entity)
 ```
    @PostPersist
    public void onPostPersist(){
        Refunded refunded = new Refunded();
        BeanUtils.copyProperties(this, refunded);
        refunded.publishAfterCommit();

        pizzamj.external.Delivery delivery = new pizzamj.external.Delivery();

        delivery.setOrderId(this.getOrderId());
        delivery.setDeliveryStatus("refunded");

        RefundApplication.applicationContext.getBean(pizzamj.external.DeliveryService.class)
            .delivery(delivery);


    }
```
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


배송(delivery) 서비스를 잠시 내려놓음 (ctrl+c)
```
# 환불처리

http http://localhost:8086/refunds orderId=1 reason="delivery error"  #Fail

```
![image](https://user-images.githubusercontent.com/70673848/98238015-1fe57e80-1fa9-11eb-9608-72667ceb144b.png)
```
# 배송 서비스 재기동
cd delivery
mvn spring-boot:run

#환불처리
http http://localhost:8086/refunds orderId=1 reason="delivery error" #Success
```

![image](https://user-images.githubusercontent.com/70673848/98238285-8074bb80-1fa9-11eb-8473-8ea5e9d2adba.png)

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

환불이 완료되어진 후에 쿠폰시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 쿠폰 시스템의 처리를 위하여 주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 배달이력에 기록을 남긴 후에 곧바로 쿠폰이 회수 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
```
package pizzamj;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Refund_table")
public class Refund {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String reason;

    @PostPersist
    public void onPostPersist(){
        Refunded refunded = new Refunded();
        BeanUtils.copyProperties(this, refunded);
        refunded.publishAfterCommit();
 ```       

- 환불 이벤트에 대해서 이를 수신하여 자신의 쿠폰 정책을 처리하도록 PolicyHandler 를 구현한다:
 
```
package pizzamj;

import pizzamj.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.Iterator;
import java.util.Optional;

@Service
public class PolicyHandler{

    @Autowired
    CouponRepository couponRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDelivered_PublishCoupon(@Payload Delivered delivered){

        if(delivered.isMe()){
            if(delivered.getDeliveryStatus().equals("Finished")){
                Coupon coupon = new Coupon();
                coupon.setOrderId(delivered.getOrderId());
                coupon.setStatus("published");
                couponRepository.save(coupon);
            }

            System.out.println("##### listener PublishCoupon : " + delivered.toJson());
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRefunded_PublishCoupon(@Payload Refunded refunded){

        if(refunded.isMe()){

            int flag=0;
            Iterator<Coupon> iterator = couponRepository.findAll().iterator();
            while(iterator.hasNext()){
                Coupon pointTmp = iterator.next();
                if(pointTmp.getOrderId() == refunded.getOrderId()){
                    Optional<Coupon> PointOptional = couponRepository.findById(pointTmp.getId());
                    Coupon coupon = PointOptional.get();
                    coupon.setStatus("couponRefunded");
                    couponRepository.save(coupon);
                    flag=1;
                }
            }

            if (flag==0 ){
                Coupon coupon = new Coupon();
                coupon.setOrderId(refunded.getOrderId());
                coupon.setStatus("couponRefunded");
                couponRepository.save(coupon);
            }


            System.out.println("##### listener PublishCoupon : " + refunded.toJson());
        }
    }

}
```

쿠폰 시스템은 환불서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 쿠폰 시스템이 유지보수로 인해 잠시 내려간 상태라도  문제가 없다:

쿠폰 서비스를 잠시 내려놓음 

![image](https://user-images.githubusercontent.com/70673848/98244601-09442500-1fb3-11eb-9b96-1e56e6d5fa1c.png)

쿠폰 서비스 재기동
```
cd coupon
mvn spring-boot:run
```

이벤트 수신 후 환불 정보가 생성된다.

![image](https://user-images.githubusercontent.com/70673848/98244836-650eae00-1fb3-11eb-8539-b0668b288170.png)


## CQRS 적용
```
주문현황및 환불 사유까지  VIEW로 구현
```
![image](https://user-images.githubusercontent.com/70673848/98247596-2aa71000-1fb7-11eb-84ca-bd1682a0ca73.png)



## gateway 적용

application.yaml파일에 소스 적용

![image](https://user-images.githubusercontent.com/70673848/98247432-f7fd1780-1fb6-11eb-8660-5f7fc8cdb4a9.png)

호출확인

![image](https://user-images.githubusercontent.com/70673848/98247834-79ed4080-1fb7-11eb-9f34-7a8ab73b04e9.png)

# 운영

## CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 deployment.yml, service.yml 에 포함되었다.
![image](https://user-images.githubusercontent.com/70673848/98327124-2e2dac00-2036-11eb-9e78-fa92d7a70bc0.png)

![image](https://user-images.githubusercontent.com/70673848/98318588-3c71cd00-2022-11eb-9983-aac3f1cb2d58.png)
![image](https://user-images.githubusercontent.com/70673848/98318592-409dea80-2022-11eb-958c-c47c636a902c.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 환불(refund)-->배송(delivery) 의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 환불 요청이 과도할 경우 CB 를 통하여 장애격리.

- refund의 application.yaml 파일에 Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 500 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 500

```
- 피호출 서비스 pament onPostPersist영역의 부하코드 추가 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# payment.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();

        try {
            Thread.sleep((long) (400 + Math.random() * 300));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
![image](https://user-images.githubusercontent.com/70673848/98318512-19471d80-2022-11eb-96ad-20ca0992d1b9.png)

-운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 


### 오토스케일 아웃

앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 배송서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deployment delivery --cpu-percent=20 --min=1 --max=10
```

![image](https://user-images.githubusercontent.com/70673848/98319585-65935d00-2024-11eb-9545-a4b08b63a219.png)


![image](https://user-images.githubusercontent.com/70673848/98319560-56acaa80-2024-11eb-9118-5c34fe2cdc5c.png)



- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c1 -t10S -r5 -v --content-type "application/json" 'http://refund:8080/refunds POST {"orderId":1, "reason":"test"}'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/70673848/98321302-1d763980-2028-11eb-9ed9-7a7efb5d5f88.png)

```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

```
![image](https://user-images.githubusercontent.com/70673848/98319791-dc305a80-2024-11eb-9cda-e025be9579e7.png)




## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"pizzaId":10, "qty":10}'

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
![image](https://user-images.githubusercontent.com/70673848/98326100-bfe7ea00-2033-11eb-86cc-e6c93fa8a97c.png)



배포기간중 Availability 가 평소 100%에서 90% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:

kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:

![image](https://user-images.githubusercontent.com/70673848/98326277-3258ca00-2034-11eb-913e-d2a0a5af76e8.png)


배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Liveness 구현

- delivery 의 depolyment.yaml 소스 설정
 서비스포트 8080이 아닌 고의로 8081로 포트 변경하여 강제로 재기동 되도록 설정 한다.


![image](https://user-images.githubusercontent.com/70673848/98328001-24a54380-2038-11eb-9671-91094b9bf6d2.png)




