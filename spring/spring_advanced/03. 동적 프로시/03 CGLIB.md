### CGLIB: Code Generator Library
- CGLIB는 바이트코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공하는 라이브러리이다.
- CGLIB를 사용하면 인터페이스가 없어도 구체 클래스만 가지고 동적 프록시를 만들어낼 수 있다.
- CGLIB는 원래는 외부 라이브러리인데, 스프링 프레임워크가 스프링 내부 소스 코드에 포함했다. 따라서 스프링을 사용한다면 별도의 외부 라이브러리를 추가하지 않아도 사용할 수 있다

- 준비코드
```java
public interface ServiceInterface {
	void save();
	void find();
}
```
```java
@Slf4j
public class ServiceImpl implements ServiceInterface {

	@Override
	public void save() {
		log.info("save 호출");
	}
	@Override
	public void find() {
		log.info("find 호출");
	}
}
```
```java
@Slf4j
public class ConcreteService {
	public void call() {
		log.info("ConcreteService 호출");
	}
}
```

### 예제 코드
- MethodInterceptor - CGLIB
```java
package org.springframework.cglib.proxy;

public interface MethodInterceptor extends Callback {
	Object intercept(Object obj, Method method, Object[] args, 
		MethodProxy	proxy) throws Throwable;
}
```
`obj` : CGLIB가 적용된 객체
`method` : 호출된 메서드
`args` : 메서드를 호출하면서 전달된 인수
`proxy` : 메서드 호출에 사용

- 예제 코드 - TimeMethodInterceptor
```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

@Slf4j
public class TimeMethodInterceptor implements MethodInterceptor {
	private final Object target;
	
	public TimeMethodInterceptor(Object target) {
		this.target = target;
	}
	
	@Override
	public Object intercept(Object obj, Method method, Object[] args,
		MethodProxy proxy) throws Throwable {
		
		log.info("TimeProxy 실행");
		long startTime = System.currentTimeMillis();
		
		Object result = proxy.invoke(target, args);
		
		long endTime = System.currentTimeMillis();
		long resultTime = endTime - startTime;
		
		log.info("TimeProxy 종료 resultTime={}", resultTime);
		return result;
	}
}
```

### 적용
```java
import hello.proxy.cglib.code.TimeMethodInterceptor;
import hello.proxy.common.service.ConcreteService;
import hello.proxy.common.service.ServiceImpl;
import hello.proxy.common.service.ServiceInterface;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.cglib.proxy.Enhancer;

@Slf4j
public class CglibTest {

	@Test
	void cglib() {
		ConcreteService target = new ConcreteService();
		Enhancer enhancer = new Enhancer();
		
		enhancer.setSuperclass(ConcreteService.class);
		enhancer.setCallback(new TimeMethodInterceptor(target));
		
		ConcreteService proxy = (ConcreteService)enhancer.create();
		
		log.info("targetClass={}", target.getClass());
		log.info("proxyClass={}", proxy.getClass());
		proxy.call();
	}
}
```
```shell
# 출력
CglibTest - targetClass=class hello.proxy.common.service.ConcreteService
CglibTest - proxyClass=class hello.proxy.common.service.ConcreteService$
$EnhancerByCGLIB$$25d6b0e3
TimeMethodInterceptor - TimeProxy 실행
ConcreteService - ConcreteService 호출
TimeMethodInterceptor - TimeProxy 종료 resultTime=9
```
`CGLIB` 가 생성한 클래스의 형태: `대상클래스$$EnhancerByCGLIB$${임의코드}`

- `ConcreteService` 는 인터페이스가 없는 구체 클래스이다
- `Enhancer`: CGLIB는 `Enhancer` 를 사용해서 프록시를 생성한다.
- `enhancer.setSuperclass(ConcreteService.class)` : CGLIB는 구체 클래스를 상속 받아서 프록시를 생성할 수 있다. 어떤 구체 클래스를 상속 받을지 지정한다.
- `enhancer.setCallback(new TimeMethodInterceptor(target))`: 프록시에 적용할 실행 로직을 할당
- `enhancer.create()` : 프록시를 생성한다. 앞서 설정한 `enhancer.setSuperclass(Class class)` 에서 지정한 클래스를 상속 받아서 프록시가 만들어 진다.

JDK 동적 프록시는 인터페이스를 구현(implement)해서 프록시를 만든다. CGLIB는 구체 클래스를 상속(extends)해서 프록시를 만든다.

이걸 그림으로 표현하면
![[CGLIB 그림.png]]

### CGLIB 제약
- 클래스 기반 프록시는 상속을 사용하기 때문에 몇가지 제약이 있다.
	- 부모 클래스의 생성자를 체크해야 한다. CGLIB는 자식 클래스를 동적으로 생성하기 때문에 기본 생성자가 필요하다.
	- 클래스에 `final` 키워드가 붙으면 상속이 불가능하다. CGLIB에서는 예외가 발생한다.
	- 메서드에 `final` 키워드가 붙으면 해당 메서드를 오버라이딩 할 수 없다. CGLIB에서는 프록시 로직이동작하지 않는다.