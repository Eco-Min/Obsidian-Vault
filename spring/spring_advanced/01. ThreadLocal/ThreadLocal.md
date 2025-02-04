- 쓰레드 로컬은 해당 쓰레드만 접근할 수 있는 특별한 저장소를 말한다. 
  쉽게 이야기해서 물건 보관 창구를 떠올리면 된다. 
- 여러 사람이 같은 물건 보관 창구를 사용하더라도 창구 직원은 사용자를 인식해서 
  사용자 별로 확실하게 물건을 구분해준다.
- 사용자A, 사용자B 모두 창구 직원을 통해서 물건을 보관하고, 꺼내지만 창구 지원이 
  사용자에 따라 보관한 물건을 구분해주는 것이다.

**일반적인 변수 필드**
여러 쓰레드가 같은 인스턴스의 필드에 접근하면 처음 쓰레드가 보관한 데이터가 사라질 수 있다.

- 쓰레드 로컬을 통해서 데이터를 조회할 때도 `thread-A` 가 조회하면 
  쓰레드 로컬은 `thread-A` 전용 보관소에서
- `userA` 데이터를 반환해준다. 물론 `thread-B` 가 조회하면 `thread-B` 
  전용 보관소에서 `userB` 데이터를 반환해준다.
- 자바는 언어차원에서 쓰레드 로컬을 지원하기 위한 `java.lang.ThreadLocal` 클래스를 제공한다.

```java
@Slf4j
public class ThreadLocalService {
	private ThreadLocal<String> nameStore = new ThreadLocal<>();
	public String logic(String name) {
		log.info("저장 name={} -> nameStore={}", name, nameStore.get());
		nameStore.set(name);
		sleep(1000);
		log.info("조회 nameStore={}",nameStore.get());
		return nameStore.get();
	}
	
	private void sleep(int millis) {
		try {
			Thread.sleep(millis);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```

## ThreadLocal 사용법
- 값 저장: `ThreadLocal.set(xxx)`
- 값 조회: `ThreadLocal.get()`
- 값 제거: `ThreadLocal.remove()`

>**주의**
해당 쓰레드가 쓰레드 로컬을 모두 사용하고 나면 `ThreadLocal.remove()` 를 호출해서 
쓰레드 로컬에 저장된 값을 제거해주어야 한다.  (아직 프세스가 실행 중인 상태일 때)

```java
@Slf4j
public class ThreadLocalServiceTest {
	private ThreadLocalService service = new ThreadLocalService();
	@Testvoid 
	threadLocal() {
		log.info("main start");
		Runnable userA = () -> {
			service.logic("userA");
		};
		
		Runnable userB = () -> {
			service.logic("userB");
		};
		
		Thread threadA = new Thread(userA);
		threadA.setName("thread-A");
		
		Thread threadB = new Thread(userB);
		threadB.setName("thread-B");
		
		threadA.start();
		sleep(100);
		threadB.start();
		sleep(2000);
		log.info("main exit");
	
	}
	private void sleep(int millis) {
		try {
			Thread.sleep(millis);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
- result
```shell
[Test worker] main start
[Thread-A] 저장 name=userA -> nameStore=null
[Thread-B] 저장 name=userB -> nameStore=null
[Thread-A] 조회 nameStore=userA
[Thread-B] 조회 nameStore=userB
[Test worker] main exit
```

## ThreadLocal 주의 사항
쓰레드 로컬의 값을 사용 후 제거하지 않고 그냥 두면 WAS(톰켓)처럼 `쓰레드 풀` 을 사용하는 경우
심각한 문제를 발생 할 수 있다.

1. Thread 생성은 많은 자원을 먹는다.
2. ThreadPool 을 이요하여 Thread 를 삭제 하지 않고 pool 에 반환 하여 재 사용한다
   (시간이 지나면 없어지긴 하지만 언제 없어지는지는 아무도 모른다.) 

위 두 관점을 생각해보고 시나리오를 작성 해보자
### 시나리오
- 사용자A 가 저장 Http 요청
	- Thread-A -> UserA 생성 후 ThreadPool  반환
	- ThreadPool 내부에 Thread-A 반환 후 UserA 데이터를 보관
- 사용자B 가 조회 Http 요청
	- ThreadPool 내부에서 Thread-A 를 재사용
	- ex) User class 가 null 이면 조회 null 이 아니면  반환 이라 가정
	- Thread-A 는  보관하고 있떤 UserA 데이터를 사용
	- 사용자B 는 UserA 의 데이터를 사용하게 된다.
	- 
결과적으로 사용자B는 사용자A의 데이터를 확인하게 되는 심각한 문제가 발생하게 된다.
이런 문제를 예방하려면 사용자A의 요청이 끝날 때 쓰레드 로컬의 값을 
`ThreadLocal.remove()` 를 통해서 꼭 제거해야 한다.