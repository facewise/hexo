---
title: Spring AOP 포인트컷(Pointcut)
date: 2024-04-11 09:21:20
tag: AOP
category: 
  - Java
  - Spring
---

# 포인트컷(Pointcut)이란?

포인트컷은 Advice가 언제 실행될지 조인 포인트 중에서 골라내는 작업이라고 할 수 있다.

즉, 어드바이스가 어떤 메소드들에서 실행될지 골라내는 작업이다.

스프링 AOP에서는 스프링 빈의 메소드 실행에 대한 조인 포인트만 제공하기 때문에 스프링에서 포인트컷은 스프링 빈의 메소드와 연결된다고 볼 수 있다.

<br>

# 포인트컷 선언

포인트컷을 선언할 때는 다음 두 가지가 필요하다.

1. 시그니쳐 (Signature)
2. 포인트컷 표현식 (Pointcut Expression)

AspectJ에서 시그니쳐란 쉽게 말해 Java의 메소드이다. 이름과 파라미터로 식별 가능한 고유한 메소드이고 꼭 `void` 타입이어야 한다.

포인트컷 표현식이란 어떤 메소드가 실행될 때 이 애스펙트가 적용되어야 하는지 나타내는 표현식이다.

AspectJ에서 포인트컷은 다음과 같이 선언할 수 있다.

```java
@Pointcut("execution(* transfer(..))") // 포인트컷 표현식
private void anyOldTransfer() {} // Signature
```

위 포인트컷의 시그니쳐는 `anyOldTransfer`라는 이름과 0개의 파라미터로 구성돼있다.

그리고 포인트컷 표현식은 `@Pointcut("execution(* transfer(..))")`가 된다.

<br>

# 스프링 AOP의 포인트컷 표현식

스프링 AOP에서 지원하는 AspectJ의 포인트컷 지정자 (Pointcut designator, PCD) 는 다음과 같다:

## `execution`

어떤 메소드가 실행될 때인지를 지정한다.

`execution([접근자] 타입 [클래스].이름(파라미터))`

- 접근자: `public`, `private` 등의 접근 제어자. 생략 가능.
- 타입: 메소드의 리턴 타입.
- 클래스: 패키지명을 포함한 클래스명. 생략 가능.
- 이름: 메소드 이름.
- 파라미터: 메소드의 파라미터 타입.
  - `*`: 모든 타입.
  - `..`: 0개 이상의 모든 파라미터.

```java
@Pointcut("execution(public String com.baeldung.pointcutadvice.dao.FooDao.findById(Long))")
```

예를 들어 위와 같은 포인트컷 표현식이 있다면

접근자는 `public`, 리턴 타입은 `String`, 클래스는 `com.baeldung.pointcutadvice.dao.FooDao`, 메소드 이름은 `findById`, 파라미터는 `Long` 타입 1개만 갖고 있는 메소드를 지정하는 것이다.

```java
@Pointcut("execution(* com.baeldung.pointcutadvice.dao.FooDao.*(..))")
```

위 표현식에서는 접근자는 생략되었고 리턴 타입은 `*` 로 모든 타입이고, 클래스는 `com.baeldung.pointcutadvice.dao.FooDao`, 메소드 이름은 `*`로 모든 메소드이며 파라미터는 `..`로 갯수와 타입에 상관없이 모든 메소드를 지정하게 된다.

## `within`

어떤 타입 안에 있는 메소드인지 지정한다.

```java
@Pointcut("within(com.baeldung.pointcutadvice.dao.FooDao)")
```

는 `FooDao` 내의 모든 메소드를 의미하고

```java
@Pointcut("within(com.baeldung..*)")
```

는 `com.baeldung` 패키지와 그 하위의 모든 패키지의 모든 타입 안의 모든 메소드를 의미한다.

## `this` 와 `target`

`this`와 `target`은 AOP가 프록시를 생성할 때 CGLIB 방식인지 JDK dynamic 방식인지에 따라 선택해야 한다.

만약 `IFooService`라는 인터페이스가 있고 이를 구현한 `FooServiceImpl` 클래스가 있다고 가정해보자.

```java
public class FooServiceImpl implements IFooService {
    ...
}
```

이 경우에는 스프링 AOP는 JDK dynamic 프록시를 사용하기 때문에 `target` 표현식을 사용해야 한다.

```java
@Pointcut(target(com.xyz.IFooService))
```

`A instanceof IFooService` 가 true인 모든 A의 메소드를 의미한다.

<br>

만약 `FooServiceImpl`가 다음처럼 아무 인터페이스도 구현하지 않고 있다면,

```java
public class FooServiceImpl {
    ...
}
```

이 경우에는 생성되는 프록시가 `FooServiceImpl`의 자식 클래스로 생성되기 때문에

```java
@Pointcut(target(com.xyz.FooServiceImpl))
```

처럼 사용해야 한다.

## `args`

메소드의 파라미터로 전달된 런타임 객체의 타입이 지정된 타입일 경우에 적용된다.

예를 들어 어떤 메소드의 파라미터로 `String` 타입이 전달될 때 적용하고 싶다면

```java
@Pointcut("args(java.lang.String)")
```

### `execution`과 차이점

`execution(* *(java.lang.String))`은 `String` 타입을 파라미터로 받는다고 선언된 메소드를 의미하고

`args(java.lang.String)`은 런타임에 전달된 파라미터의 타입이 `String` 타입인 메소드를 의미한다.

다음과 같이 두 메소드가 있는데 하나는 `List` 타입으로 선언돼있고 하나는 `ArrayList` 타입으로 선언돼있다고 가정하자.

```java
public void method(List<String> list) {}

public void method2(ArrayList<String> arrayList) {}
```

그리고 만약 `method(new ArrayList<>());`를 호출한다고 하면

이 경우에 `execution(* *(java.util.ArrayList))`는 `method2`에만 적용되는 것이고

`args(java.util.ArrayList)`는 `method`와 `method2` 둘 다에 적용되는 것이다.

## `@target`

위에서 설명한 `target`과 다르다는 것을 주의해야 한다.

지정한 애너테이션을 갖고 있는 클래스의 모든 메소드를 의미한다.

```java
@Pointcut("@target(org.springframework.stereotype.Repository)")
```

## `@args`

위에서 설명한 `args`와 다르다는 것을 주의해야 한다.

메소드의 파라미터로 전달된 런타임 객체의 실제 타입이 지정된 애너테이션을 갖고 있는 클래스일 경우에 적용된다.

예를 들어 어떤 메소드의 파라미터로 `@Entity`가 붙은 클래스의 객체가 전달될 때 적용하고 싶다고 가정하면

```java
@Pointcut("@args(com.baeldung.pointcutadvice.annotations.Entity)")
public void methodsAcceptingEntities() {}
```

로 지정하게 되면 `@Entity`가 붙은 객체가 전달되는 모든 메소드를 의미한다.

이렇게 지정한 포인트컷의 파라미터에 접근하려면 advice에서 `JoinPoint` 를 파라미터로 받아서 접근 가능하다.

```java
@Before("methodsAcceptingEntities()")
public void logMethodAcceptionEntityAnnotatedBean(JoinPoint jp) {
    Object entity = jp.getArgs()[0]; // @Entity가 선언된 객체
    logger.info("Accepting beans with @Entity annotation: " + entity);
}
```

## `@within`

`within`과 다르다는 점을 주의한다.

`within`과 거의 비슷하지만 `@within`은 지정한 애너테이션을 갖고 있는 타입의 모든 메소드를 의미한다.

```java
@Pointcut("@within(org.springframework.stereotype.Repository)")
```

는 아래와 동일한 표현식이다.

```java
@Pointcut("within(@org.springframework.stereotype.Repository *)")
```

## `@annotation`

지정된 애너테이션을 갖고 있는 메소드에 적용된다.

`@within`은 지정된 애너테이션을 갖고 있는 타입의 모든 메소드에 적용되는 것이고 `@annotation`은 지정된 애너테이션을 갖고 있는 메소드에 적용되는 차이점이 있다.

예를 들어 `@Loggable`이라는 애너테이션을 만들어서 메소드에 추가한다면

`@Loggable`이 붙은 모든 메소드에 대해 아래와 같이 적용할 수 있다.

```java
@Pointcut("@annotation(com.baeldung.pointcutadvice.annotations.Loggable)")
public void loggableMethods() {}

@Before("loggableMethods()")
public void logMethod(JoinPoint jp) {
    String methodName = jp.getSignature().getName();
    logger.info("Executing method: " + methodName);
}
```

## `bean`

이 지정자는 **AspectJ에는 없는 지정자**로 스프링 AOP에서 자체 지원한다.

지정한 빈의 모든 메소드를 의미한다. `*`를 통한 와일드카드 표현식을 지원한다.

## 이외의 PCD

스프링 AOP에서는 위에서 소개한 포인트컷 지정자 이외에 다른 AspectJ 지정자를 사용하면 `IllegalArgumentException`이 발생한다.

<br>

# 포인트컷 조합

포인트컷은 적용될 대상 메소드를 집합처럼 나타내는 표현식이므로 여러 포인트컷을 조합해서 집합처럼 사용할 수 있다.

사용 가능한 연산자는 다음과 같다.

- `&&` : 두 포인트컷의 교집합
- `||` : 두 포인트컷의 합집합
- `!` : 여집합

<br>

# 포인트컷 공유

규모가 큰 애플리케이션을 개발할 때는 자주 사용되는 공통 포인트컷을 묶어서 하나의 클래스로 만들 것을 추천한다.

예를 들면, 아래와 같이 `CommonPointcuts`라는 클래스를 만들고

```java
package com.xyz;

import org.aspectj.lang.annotation.Pointcut;

public class CommonPointcuts {

	/**
	 * dao 패키지 밑의 모든 클래스의 모든 메소드에 적용
	 */
	@Pointcut("within(com.xyz.dao..*)")
	public void inDataAccessLayer() {}

	/**
	 * service 패키지 하위에 있는 모든 타입의 모든 메소드 지정.
	 *
	 * 만약 Service 빈 이름이 모두 ~Service로 끝난다면
     * bean(*Service)로 지정해도 좋다.
	 */
	@Pointcut("execution(* com.xyz..service.*.*(..))")
	public void businessService() {}

}
```

Advice를 만들 때 공통 포인트컷을 사용할 수 있다.

```java
@Before("com.xyz.CommonPointcuts.businessService()")
public void beforeBusinessServiceMethods() {
    ...
}
```

# 포인트컷 잘 작성하기

어떤 메소드가 해당 포인트컷에 적합한지 찾아내는 것은 비용이 많이 드는 작업이다.

특히 동적으로 생성된 빈의 메소드는 더욱더 비용이 많이 든다.

그래서 AspectJ는 컴파일할 때 포인트컷을 모두 찾아내서 검사하기 쉬운 메소드 순서로 정렬한다.

좋은 포인트컷을 작성하기 위해서는 이 포인트컷의 목표를 생각하고 최대한 메소드 집합의 범위를 좁혀야 한다.

포인트컷 지정자(PCD)는 다음 세 그룹으로 분류할 수 있다.

- Kinded
  - 특정 조인포인트(메소드)만 선택 : `execution`
- Scoping
  - 조인포인트의 그룹을 선택 : `within`
- Contextual
  - 문맥에 따라 다르게 선택 : `this`, `target`, `@annotation`

포인트컷을 잘 작성하기 위해서는 최소한 `kinded`와 `scoping`을 포함해야 한다.

`Kinded` 지정자만 선언하거나 `Contextual` 지정자만 선언한 포인트컷은 작동은 하지만 시간, 메모리 효율성에서 성능에 영향을 줄 수 있다.

`Scoping` 지정자는 특정 그룹만 대상으로 하면 되기 때문에 조인포인트를 찾아내는 게 매우 빠르다.

**즉, 잘 작성된 포인트컷은 `Scoping` 지정자를 포함한 포인트컷이다.**
