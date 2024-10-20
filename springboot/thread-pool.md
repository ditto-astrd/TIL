## 0. 서두
spring boot에서 API 콜이 한 번 호출되고 결과값이 반환될 때 까지 thread pool 1개가 필요된다.
비동기로 호출 될 경우, 비동기로 호출되는 메서드의 개수 만큼 thread pool이 더 필요된다.

그렇다면 이왕이면 더 큰값의 thread-pool을 설정하면 되지 않을까? 그런 호기심에서 thread pool에 대한 탐구가 시작됐다.

## 1. 그냥 큰 값으로 맘대로 설정하면 안돼?

우선 spring boot가 다중 요청을 처리할 때, 요청 하나당 생성되는 thread는 spring에서 생성 <-> 소멸을 관리해주는게 아니라 내장형 Tomcat, 즉, WAS에서 처리를 해준다.

참고로 **HTTP + 동기 요청**에 대해서 언급한다.
이는 @RestController나 @Controller를 통해 호출되는 일반적인 동기 API를 말한다.
(비동기에 쓰이는 thread-pool이나 batch에 쓰이는 thread-pool은 후술한다.)

기본적으로 아래의 루트를 거쳐 thread의 생성과 소멸이 이루어진다.
```
- Tomcat은 다중 요청을 처리하기 위해서, 부팅할 때 Thread의 컬렉션인 Thread Pool을 생성한다.
- 유저 요청(HttpServletRequest)가 들어오면 Thread Pool에서 하나씩 Thread를 할당한다. 해당 Thread에서 스프링부트에서 작성한 Dispathcer Servlet을 거쳐 유저 요청 을 처리한다.
- 작업을 모두 수행하고 나면 Thread는 Thread Pool로 반환한다.
```

보통 application.yml이나 application.properties에서 아래 처럼 thread-pool에 대한 지정이 가능한데,
``` yml
# application.yml (적어놓은 값은 default) 
server: 
	tomcat: 
		threads: 
			max: 200 # 생성할 수 있는 thread의 총 개수 
			min-spare: 10 # 항상 활성화 되어있는(idle) thread의 개수 
			max-connections: 8192 # 수립가능한 connection의 총 개수 
			accept-count: 100 # 작업큐의 사이즈 
			connection-timeout: 20000 # timeout 판단 기준 시간, 20초 
	port: 8080 # 서버를 띄울 포트번호

```
thread-pool은 다다익선일것같은데, 이왕이면 큰 값으로 지정해주면 되지 않을까? 

그렇다면 thread-pool이 어떤 자원을 토대로 생성이 되는지 이해 할 필요가 있다.
#### (1) CPU 자원
- 각 thread는 CPU 코어에서 실행되기 때문에 CPU 자원의 제한이 걸림
- CPU 코어 수 보타 많은 스레드가 동시에 활성화 되면 Context switching이 비용이 급격히 증가하여 성능이 저하됨
- 컨텍스트 스위칭이 많아지면 CPU가 스레드 간 작업을 전환하는 데 대부분의 시간을 소비하게 되어 처리량(throughput)이 급격히 떨어짐
#### (2) 메모리 사용량 증가
- 각 스레드는 고유의 스택 메모리와 힙 메모리를 사용
- 일반적으로 JVM의 각 스레드 스택은 1MB로 크기가 설정되며, 많은 스레드를 생성하면 그만큼 메모리가 사용됨
- 예시로 스레드가 1,000개라면 최소 1GB의 메모리가 스택메모리로만 사용되며, 이 수치가 커질수록 힙 메모리 부족 내지는 OOM(Out Of Memory Error)가 발생할 위험이 있음

#### (3) OS레벨의 리소스를 사용하기 때문에 운영체제에서 관리할 수 있는 스레드 수의 제한

#### (4) 스레드가 많아질 수록 공유 자원에 대한 경쟁 발생

#### (5) 많은 스레드로 인한 네트워크 I/O가 동시에 발생시, 그만큼 요청 지연 및 TimeoutException이 발생

이런 점들을 고려해야한다.
특히 1번과 2번의 사유로, Spring Boot가 올라간 인스턴스, 즉 서버의 CPU 코어와 JVM의 힙 메모리 크기를 잘 파악해야한다.

추가로 API를 호출하여 DB에 적재된 데이터를 가져오는 일반적인 thread pool의 경우, \아래와 같은 이슈도 함께 고려해야 한다.

#### (1) thread pool > DB Connection
- 동시에 많은 스레드가 DB 커넥션을 요구하게 되지만 사용 가능한 커넥션이 제한되어 있기 때문에 병목 현상이 발생할 수 있음
#### (2) thread pool < DB Connection
- 많은 DB 커넥션이 유휴 상태로 남게 되어 자원이 낭비될 수 있음

이러한 이슈들 때문에 외부 API를 호출하지 않고 내가 가진 DB만 connection하는 API로 구성된 프로젝트라 하더라도, 마음대로 thread-pool을 늘릴 수는 없는 노릇이다.

DB connection pool은 사용중인 DB의 성능과도 관련이 있으므로, 결론적으로 thread pool은 마음대로 늘리고 싶어도 쉽게는 할 수 없는 노릇이디.


## 2. Async의 thread pool은 달라?

1번에서 언급한 HTTP + 동기 (@Controller, 또는 @RestController를 통한 호출)은 SpringBoot의 내장형 WAS의 thread-pool을 사용하지만, Async의 thread-pool은 다르다.

@Async로 인해 비동기로 호출되는 API는 TaskExecutor 인터페이스에 의해 관리되며, 내장형 WAS의 thread 풀과는 독립적인 thread pool로 동작하기 때문이다.

#### 한줄 요약 : 비동기는 SpringBoot 자체 thread-pool, 동기는 내장형 Tomcat thread-pool


## 3. Batch의 thread pool은 달라?

2번과 동일하다.
배치를 돌릴때 필요되는 @Scheduled 스레드 풀 (= Spring TaskScheduler)은 다르다.

## 4. 그래서 결론은?

동기 thread-pool, 비동기 thread-pool, 그리고 batch thread-pool 모두 독립적인 thread-pool에서 동작한다.

개발자가 마음만 먹으면 각각의 thread-pool 설정을 이룰 수 있지만, 인스턴스의 환경 및 DB Connection pool의 상태와도 밀접하게 연관되어있기 때문에 신중하게 값을 조정하는 과정이 요구된다.

---
### 참고 자료
https://f-lab.kr/insight/thread-pool-and-db-connection-pool-optimization
https://russell-seo.tistory.com/9
