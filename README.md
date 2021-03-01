# meetingroom

# 서비스 시나리오
## 기능적 요구사항
1. 기본적으로 선착순으로 회의실에 대한 예약을 선점하도록 시나리오를 작성했다.
2. 해당 회의실이 예약 중이거나 회의 중이라면 예약을 할 수 없고, 예약 취소나 회의 종료가 일어났을 때 예약을 할 수 있도록 생각했다. 
3. 회원이 회의실에 대한 예약을 한다.
4. 회원이 예약한 회의실에 대해 회의 시작을 요청한다.
5. 예약한 사람이면 해당 회의를 시작할 수 있도록 한다.
6. 예약한 사람이 아니면 회의를 시작할 수 없다.
7. 예약한 회의에 대해 예약 취소를 요청할 수 있다.
8. 시작했던 회의를 종료한다.

## 비기능적 요구사항
1. 트랜잭션
  - 회의 시작을 요청할 때, 회의를 예약한 사람이 아니라면 회의를 시작하지 못하게 한다.(Sync 호출)
2. 장애격리
  - 회의실 시스템이 수행되지 않더라도 예약취소, 사용자 확인, 회의 종료는 365일 24시간 받을 수 있어야 한다. (Async 호출)
3. 성능
  - 회의실 현황에 대해 예약 상황을 별도로 확인할 수 있어야 한다.(CQRS)

# 체크포인트
https://workflowy.com/s/assessment/qJn45fBdVZn4atl3

## EventStorming 결과
### 완성된 1차 모형
<img width="1086" alt="스크린샷 2021-02-25 오후 10 59 46" src="https://user-images.githubusercontent.com/43164924/109471758-8fc9c880-7ab4-11eb-8855-5e2e7d31328b.png">

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
<img width="1040" alt="스크린샷 2021-03-01 오후 5 41 29" src="https://user-images.githubusercontent.com/43164924/109472783-e71c6880-7ab5-11eb-834c-32bdbaef4639.png">
    
    - 회의실이 등록이 된다. (7)
    - 회원이 회의실을 예약을 한다. (3 -> 6)
    - 회원이 예약한 회의실에 대해 회의 시작을 요청한다.
      - 예약한 사람이면 회의를 시작한다. (1 -> 5 -> 6)
      - 예약한 사람이 아니면 회의 시작을 못한다. (1)
    - 회원이 예약한 회의실을 예약 취소한다. (4 -> 6)
    - 시작했던 회의를 종료한다. (2 -> 6)

### 헥사고날 아키텍쳐 다이어그램 도출 (Polyglot)
<img width="1200" alt="스크린샷 2021-03-01 오후 6 14 04" src="https://user-images.githubusercontent.com/43164924/109476149-ea195800-7ab9-11eb-88c3-4e231a205c70.png">

# 구현
도출해낸 헥사고날 아키텍처에 맞게, 로컬에서 SpringBoot를 이용해 Maven 빌드 하였다. 각각의 포트넘버는 8081 ~ 8084, 8088 이다.

    cd conference
    mvn spring-boot:run
    
    cd gateway
    mvn spring-boot:run
    
    cd reserve
    mvn spring-boot:run
    
    cd schedule
    mvn spring-boot:run
  
## DDD의 적용
**Room 서비스의 Reserve.java**

```java
package meetingroom;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Room_table")
public class Room {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String status;
    private Integer floor;

    @PostPersist
    public void onPostPersist(){
        Added added = new Added();
        BeanUtils.copyProperties(this, added);
        added.publishAfterCommit();
    }
    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getStatus() {
        return status;
    }
    public void setStatus(String status) {
        this.status = status;
    }
    public Integer getFloor() {
        return floor;
    }
    public void setFloor(Integer floor) {
        this.floor = floor;
    }
}
```

**Room 서비스의 PolicyHandler.java**
```java
package meetingroom;

import meetingroom.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
public class PolicyHandler{
    @Autowired
    RoomRepository roomRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverCanceled_(@Payload Canceled canceled){

        if(canceled.isMe()){
            Optional<Room> room = roomRepository.findById(canceled.getRoomId());
            System.out.println("##### listener  : " + canceled.toJson());
            if (room.isPresent()){
                room.get().setStatus("Available");//회의실 예약이 취소되어 예약이 가능해짐.
                roomRepository.save(room.get());
            }
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReserved_(@Payload Reserved reserved){

        if(reserved.isMe()){
            Optional<Room> room = roomRepository.findById(reserved.getRoomId());
            System.out.println("##### listener  : " + reserved.toJson());
            if (room.isPresent()){
                room.get().setStatus("Reserved");//회의실이 예약됨.
                roomRepository.save(room.get());
            }
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverUserChecked_(@Payload UserChecked userChecked){

        if(userChecked.isMe()){
            Optional<Room> room = roomRepository.findById(userChecked.getRoomId());
            System.out.println("##### listener  : " + userChecked.toJson());
            if(room.isPresent()){
                room.get().setStatus("Started");//회의가 시작됨.
                roomRepository.save(room.get());
            }
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverEnded_(@Payload Ended ended){

        if(ended.isMe()){
            Optional<Room> room = roomRepository.findById(ended.getRoomId());
            System.out.println("##### listener  : " + ended.toJson());
            if(room.isPresent()){
                room.get().setStatus("Available");//회의가 종료됨.
                roomRepository.save(room.get());
            }
        }
    }

}

```


- 적용 후 REST API의 테스트를 통해 정상적으로 작동함을 알 수 있었다.
- 회의실 등록(Added) 후 결과

<img width="1116" alt="스크린샷 2021-03-01 오후 6 38 24" src="https://user-images.githubusercontent.com/43164924/109479041-5184d700-7abd-11eb-84d6-782c4b94779e.png">


- 회의 예약(Reserved) 후 결과

<img width="1116" alt="스크린샷 2021-03-01 오후 6 37 36" src="https://user-images.githubusercontent.com/43164924/109478970-3ade8000-7abd-11eb-836a-e07a7b3dec80.png">

## Gateway 적용
API Gateway를 통해 마이크로 서비스들의 진입점을 하나로 진행하였다.
```yml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: conference
          uri: http://localhost:8081
          predicates:
            - Path=/conferences/** 
        - id: reserve
          uri: http://localhost:8082
          predicates:
            - Path=/reserves/** 
        - id: room
          uri: http://localhost:8083
          predicates:
            - Path=/rooms/** 
        - id: schedule
          uri: http://localhost:8084
          predicates:
            - Path= /reserveTables/**
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
        - id: conference
          uri: http://conference:8080
          predicates:
            - Path=/conferences/** 
        - id: reserve
          uri: http://reserve:8080
          predicates:
            - Path=/reserves/** 
        - id: room
          uri: http://room:8080
          predicates:
            - Path=/rooms/** 
        - id: schedule
          uri: http://schedule:8080
          predicates:
            - Path= /reserveTables/**
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

## Polyglot Persistence
- Conference 서비스의 경우, 다른 서비스들이 h2 저장소를 이용한 것과는 다르게 hsql을 이용하였다. 
- 이 작업을 통해 서비스들이 각각 다른 데이터베이스를 사용하더라도 전체적인 기능엔 문제가 없음을, 즉 Polyglot Persistence를 충족하였다.

<img width="446" alt="스크린샷 2021-03-01 오후 6 43 02" src="https://user-images.githubusercontent.com/43164924/109479663-f6071900-7abd-11eb-8c22-eda690cadea4.png">



# 운영
## CI/CD 설정
- git에서 소스 가져오기
```
https://github.com/dngur6344/meetingroom
```
- Build 하기
```
cd /meetingroom
cd conference
mvn package

cd ..
cd gateway
mvn package

cd ..
cd reserve
mvn package

cd ..
cd room
mvn package

cd ..
cd schedule
mvn package
```
- Dockerlizing, ACR(Azure Container Registry에 Docker Image Push하기
```
cd /meetingroom
cd rental
az acr build --registry meetingroomacr --image meetingroomacr.azurecr.io/conference:latest .

cd ..
cd gateway
az acr build --registry meetingroomacr --image meetingroomacr.azurecr.io/gateway:latest .

cd ..
cd reserve
az acr build --registry meetingroomacr --image meetingroomacr.azurecr.io/reserve:latest .

cd ..
cd room
az acr build --registry meetingroomacr --image meetingroomacr.azurecr.io/room:latest .

cd ..
cd schedule
az acr build --registry meetingroomacr --image meetingroomacr.azurecr.io/schedule:latest .
```
- ACR에서 이미지 가져와서 Kubernetes에서 Deploy하기
```
kubectl create deploy gateway --image=meetingroomacr.azurecr.io/gateway:latest
kubectl create deploy conference --image=meetingroomacr.azurecr.io/conference:latest
kubectl create deploy reserve --image=meetingroomacr.azurecr.io/reserve:latest
kubectl create deploy room --image=meetingroomacr.azurecr.io/room:latest
kubectl create deploy schedule --image=meetingroomacr.azurecr.io/schedule:latest
kubectl get all
```
- Kubectl Deploy 결과 확인  
  <img width="556" alt="스크린샷 2021-02-28 오후 12 47 12" src="https://user-images.githubusercontent.com/33116855/109407331-52394280-79c3-11eb-8283-ba98b2899f69.png">

- Kubernetes에서 서비스 생성하기 (Docker 생성이기에 Port는 8080이며, Gateway는 LoadBalancer로 생성)
```
kubectl expose deploy conference --type="ClusterIP" --port=8080
kubectl expose deploy reserve --type="ClusterIP" --port=8080
kubectl expose deploy room --type="ClusterIP" --port=8080
kubectl expose deploy schedule --type="ClusterIP" --port=8080
kubectl expose deploy gateway --type="LoadBalancer" --port=8080
kubectl get all
```
- Kubectl Expose 결과 확인  
  <img width="646" alt="스크린샷 2021-02-28 오후 12 47 50" src="https://user-images.githubusercontent.com/33116855/109407339-5feec800-79c3-11eb-9f3f-18d9d2b812f0.png">


  
## 무정지 재배포
- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
- siege 로 배포작업 직전에 워크로드를 모니터링 함
```
siege -c10 -t60S -r10 -v --content-type "application/json" 'http://52.231.13.109:8080/reserves POST {"userId":1, "roomId":"3"}'
```
- Readiness가 설정되지 않은 yml 파일로 배포 진행  
  <img width="871" alt="스크린샷 2021-02-28 오후 1 52 52" src="https://user-images.githubusercontent.com/33116855/109408363-4b62fd80-79cc-11eb-9014-638a09b545c1.png">

```
kubectl apply -f deployment.yml
```
- 아래 그림과 같이, Kubernetes가 준비가 되지 않은 delivery pod에 요청을 보내서 siege의 Availability 가 100% 미만으로 떨어짐 
  <img width="480" alt="스크린샷 2021-02-28 오후 2 30 37" src="https://user-images.githubusercontent.com/33116855/109408933-97fd0780-79d1-11eb-8ec6-f17d44161eb5.png">

- Readiness가 설정된 yml 파일로 배포 진행  
  <img width="779" alt="스크린샷 2021-02-28 오후 2 32 51" src="https://user-images.githubusercontent.com/33116855/109408971-e4484780-79d1-11eb-8989-cd680e962eff.png">

```
kubectl apply -f deployment.yml
```
- 배포 중 pod가 2개가 뜨고, 새롭게 띄운 pod가 준비될 때까지, 기존 pod가 유지됨을 확인  
  <img width="764" alt="스크린샷 2021-02-28 오후 2 34 54" src="https://user-images.githubusercontent.com/33116855/109408992-2b363d00-79d2-11eb-8024-07aeade9e928.png">
  
- siege 가 중단되지 않고, Availability가 높아졌음을 확인하여 무정지 재배포가 됨을 확인함  
  <img width="507" alt="스크린샷 2021-02-28 오후 2 48 28" src="https://user-images.githubusercontent.com/33116855/109409209-093dba00-79d4-11eb-9793-d1a7cdbe55f0.png">


## 오토스케일 아웃
- 서킷 브레이커는 시스템을 안정되게 운영할 수 있게 해줬지만, 사용자의 요청이 급증하는 경우, 오토스케일 아웃이 필요하다.

  - 단, 부하가 제대로 걸리기 위해서, reserve 서비스의 리소스를 줄여서 재배포한다.
    <img width="703" alt="스크린샷 2021-02-28 오후 2 51 19" src="https://user-images.githubusercontent.com/33116855/109409248-7d785d80-79d4-11eb-95ce-4af79b9a7e72.png">

- reserve 시스템에 replica를 자동으로 늘려줄 수 있도록 HPA를 설정한다. 설정은 CPU 사용량이 15%를 넘어서면 replica를 10개까지 늘려준다.
```
kubectl autoscale deploy reserve --min=1 --max=10 --cpu-percent=15
```

- hpa 설정 확인  
  <img width="631" alt="스크린샷 2021-02-28 오후 2 56 50" src="https://user-images.githubusercontent.com/33116855/109409360-6a19c200-79d5-11eb-90a4-fc5c5030e92b.png">

- hpa 상세 설정 확인  
  <img width="1327" alt="스크린샷 2021-02-28 오후 2 57 37" src="https://user-images.githubusercontent.com/33116855/109409362-6ede7600-79d5-11eb-85ec-85c59bdefcaf.png">
  <img width="691" alt="스크린샷 2021-02-28 오후 2 57 53" src="https://user-images.githubusercontent.com/33116855/109409364-700fa300-79d5-11eb-8077-70d5cddf7505.png">

  
- siege를 활용해서 워크로드를 2분간 걸어준다. (Cloud 내 siege pod에서 부하줄 것)
```
kubectl exec -it (siege POD 이름) -- /bin/bash
siege -c1000 -t120S -r100 -v --content-type "application/json" 'http://20.194.45.67:8080/reserves POST {"userId":1, "roomId":"3"}'
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다.
```
watch kubectl get all
```
- 스케일 아웃이 자동으로 되었음을 확인

  <img width="656" alt="스크린샷 2021-02-28 오후 3 01 47" src="https://user-images.githubusercontent.com/33116855/109409423-eb715480-79d5-11eb-8b2c-0a0417df9718.png">

- 오토스케일링에 따라 Siege 성공률이 높은 것을 확인 할 수 있다.  

  <img width="412" alt="스크린샷 2021-02-28 오후 3 03 18" src="https://user-images.githubusercontent.com/33116855/109409445-18be0280-79d6-11eb-9c6f-4632f8a88d1d.png">

## Self-healing (Liveness Probe)
- reserve 서비스의 yml 파일에 liveness probe 설정을 바꾸어서, liveness probe 가 동작함을 확인

- liveness probe 옵션을 추가하되, 서비스 포트가 아닌 8090으로 설정, readiness probe 미적용
```
        livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8090
            initialDelaySeconds: 5
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
```

- reserve 서비스에 liveness가 적용된 것을 확인  
  <img width="824" alt="스크린샷 2021-02-28 오후 3 31 53" src="https://user-images.githubusercontent.com/33116855/109409951-1bbaf200-79da-11eb-9a39-a585224c3ca0.png">

- reserve 서비스에 liveness가 발동되었고, 8090 포트에 응답이 없기에 Restart가 발생함   
  <img width="643" alt="스크린샷 2021-02-28 오후 3 34 35" src="https://user-images.githubusercontent.com/33116855/109409994-7c4a2f00-79da-11eb-8ab7-e542e50fd929.png">

## ConfigMap 적용
- reserve의 application.yaml에 ConfigMap 적용 대상 항목을 추가한다.

  <img width="558" alt="스크린샷 2021-02-28 오후 4 01 52" src="https://user-images.githubusercontent.com/33116855/109410475-4f981680-79de-11eb-8231-0679b6f5f55b.png">

- reserve의 service.yaml에 ConfigMap 적용 대상 항목을 추가한다.

  <img width="325" alt="스크린샷 2021-02-28 오후 4 05 07" src="https://user-images.githubusercontent.com/33116855/109410532-c03f3300-79de-11eb-8e61-71752818c41d.png">


- ConfigMap 생성하기
```
kubectl create configmap apiurl --from-literal=conferenceapiurl=http://conference:8080 --from-literal=roomapiurl=http://room:8080
```

- Configmap 생성 확인, url이 Configmap에 설정된 것처럼 잘 반영된 것을 확인할 수 있다.  
```
kubectl get configmap apiurl -o yaml
```
  <img width="447" alt="스크린샷 2021-02-28 오후 4 08 16" src="https://user-images.githubusercontent.com/33116855/109410590-33e14000-79df-11eb-93ed-bdfb04778cd8.png">
  <img width="625" alt="스크린샷 2021-02-28 오후 4 10 11" src="https://user-images.githubusercontent.com/33116855/109410630-8884bb00-79df-11eb-99d4-f6311cbe37bd.png">


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

- RestAPI 기반 Request/Response 요청이 과도할 경우 CB 를 통하여 장애격리 하도록 설정함.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 설정

- conference의 Application.yaml 설정
```
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- reserve에 Thread 지연 코드 삽입
  <img width="702" alt="스크린샷 2021-03-01 오후 2 40 46" src="https://user-images.githubusercontent.com/33116855/109456415-22aa3900-7a9c-11eb-9a30-4e63323312c2.png">

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://52.141.56.203:8080/conferences POST {"roomId": "1", "userId":"1", reserveId:"1"}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 reserve에서 처리되면서 다시 conference를 받기 시작 

  <img width="409" alt="스크린샷 2021-03-01 오후 2 32 14" src="https://user-images.githubusercontent.com/33116855/109455911-00fc8200-7a9b-11eb-8d95-f5df5ef249fd.png">

- CB 잘 적용됨을 확인


