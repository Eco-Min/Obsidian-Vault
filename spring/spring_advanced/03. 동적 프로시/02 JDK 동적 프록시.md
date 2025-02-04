동적 프록시 기술은 개발자가 직접 프록시 클래스를 만들지 않고, 이름 그대로 객체를 동적으로 런타임에 개발자 대신 만들어준다. 그리고 동적 프록시에 원하는 실행 로직을 지정 할 수 있다.
> 주의
> JDK 동적 프로시는 인터페이스를 기반으로 프록시를 동적으로 만들어준다. 따라서 인터페이스가 필수.

- 준비 코드
```java
public interface AInterface {
	String call();
}
```
```java
@Slf4j
public class AImpl implements AInterface {
	@Override
	public String call() {
		log.info("A 호출");
		return "a";
	}
}
```
```java
public interface BInterface {
	String call();
}
```
```java
@Slf4j
public class BImpl implements AInterface {
	@Override
	public String call() {
		log.info("B 호출");
		return "b";
	}
}
```

### 예제 코드
JDK  동적 프록시가 제공하는 InvocationHandler
```java
package java.lang.reflect;

public interface InvocationHandler {
	public Object invoke(Object proxy, Method method, Object[] args)
		throws Throwable;
}
```
`Object proxy` : 프록시 자신
`Method method` : 호출한 메서드
`Object[] args` : 메서드를 호출할 때 전달한 인수

- 예제코드 - TimeInvocationHandler
```java
public class TimeInvocationHandler implements InvocationHandler {
	private final Object target;
	
	public TimeInvocationHandler(Object target) {
		this.target = target;
	}
	
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		log.info("TimeProxy 실행");
		long startTime = System.currentTimeMillis();
		
		Object result = method.invoke(target, args);
		
		long endTime = System.currentTimeMillis();
		long resultTime = endTime - startTime;
	
		log.info("TimeProxy 종료 resultTime={}", resultTime);
		return result;
	}
}
```

### 적용
```java
package hello.proxy.jdkdynamic;
import hello.proxy.jdkdynamic.code.*;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;import java.lang.reflect.Proxy;

@Slf4j
public class JdkDynamicProxyTest {

	@Test
	void dynamicA() {
		AInterface target = new AImpl();
		TimeInvocationHandler handler = new TimeInvocationHandler(target);
		AInterface proxy = (AInterface)
		Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[]
			{AInterface.class}, handler);
		proxy.call();
		log.info("targetClass={}", target.getClass());
		log.info("proxyClass={}", proxy.getClass());
}

	@Test
	void dynamicB() {
		BInterface target = new BImpl();
		TimeInvocationHandler handler = new TimeInvocationHandler(target);
		BInterface proxy = (BInterface)
		Proxy.newProxyInstance(BInterface.class.getClassLoader(), new Class[]
			{BInterface.class}, handler);
		proxy.call();
		log.info("targetClass={}", target.getClass());
		log.info("proxyClass={}", proxy.getClass());
	}
}
```
```shell
# 출력
TimeInvocationHandler - TimeProxy 실행AImpl - A 호출
TimeInvocationHandler - TimeProxy 종료 resultTime=0
JdkDynamicProxyTest - targetClass=class hello.proxy.jdkdynamic.code.AImpl
JdkDynamicProxyTest - proxyClass=class com.sun.proxy.$Proxy1
```
- 실행 순서
1. 클라이언트는 JDK 동적 프록시의 `call()` 을 실행한다.
2. JDK 동적 프록시는 `InvocationHandler.invoke()` 를 호출한다. `TimeInvocationHandler` 가
구현체로 있으로 `TimeInvocationHandler.invoke()` 가 호출된다.
3. `TimeInvocationHandler` 가 내부 로직을 수행하고, `method.invoke(target, args)` 를 호출해
서 `target` 인 실제 객체( `AImpl` )를 호출한다.
4. `AImpl` 인스턴스의 `call()` 이 실행된다.
5. `AImpl` 인스턴스의 `call()` 의 실행이 끝나면 `TimeInvocationHandler` 로 응답이 돌아온다. 시간
로그를 출력하고 결과를 반환한다.

- 동적 프록시 도입 전
![[동적프록시 도입전.png]]
- 동적 프록시 도입 후
![[동적프록시 도입 후.png]]

### 정리
예제를 보면 `AImpl`, `BImpl` 각각 프록시를 만들지 않았고 JDK 동적프록시로 `TimeInvocationHandler` 라는 공통을 사용하였다.
만약 적용 대상이 100개여도 동적 프록시를 통해서 생성하고, 각 필요한 `InvocationHandler`만 만들어 넣어주면 된다.
-> 결과적으로 프록시 클래스를 여러개 만들어야하는 문제, 부가기능 로직도 하나의 클래스에 모아서 단일 책임 원칙(SRP) 도 지킬 수 있다.

- JDK 동적 프록시의 한계
JDK 동적 프록시는 인터페이스가 필수이다.
그렇다면 V2 애플리케이션 처럼 인터페이스 없이 클래스만 있는 경우에는 어떻게 동적 프록시를 적용할 수 있을까?
이것은 일반적인 방법으로는 어렵고 `CGLIB` 라는 바이트코드를 조작하는 특별한 라이브러리를 사용해야 한다.