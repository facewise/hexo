---
title: Spring의 Transaction 관리
date: 2024-07-29 16:44:41
category:
  - Java
  - Spring
---

스프링 부트로 프로젝트를 하면서 `@Transactional`이 주는 편리함을 당연하다고만 받아들였다. 
그러다 보니 어떤 원리로 `@Transactional` 메소드가 트랜잭션으로서 작동하는지 모른 채로 사용해왔다.

나와 같은 고민을 한 사람들에게 스프링이 트랜잭션을 컨트롤하는 방법에 대해 공유하려고 한다. 

# 개요

스프링이 트랜잭션을 관리하는 방법을 알아보기 위해 스프링이 없는 상태에서 트랜잭션을 관리하기부터 시작해서
스프링의 설계 사상을 한 단계씩 적용해가면서 차근차근 이해해보려고 한다.

즉, JDBC 레벨에서 트랜잭션을 관리하는 방법부터 시작하면서 스프링이 트랜잭션 관리를 설계하기 위해 어떤 생각들을
적용하는지 하나하나 알아보는 것이 기초를 다지기 좋다고 생각해서이다.

# JDBC의 트랜잭션 관리

트랜잭션을 관리하는 방식이 JDBC이든, 스프링이든, Hibernate이든 관계없이 DB 트랜잭션은 아래와 같은 과정을 거친다.

```java
import javax.sql.DataSource;
import java.sql.Connection;

public void doTransaction(DataSource dataSource) {
    Connection connection;
    try {
        connection= dataSource.getConnection();
        connection.setAutoCommit(false);
        // SQL 실행
        connection.commit();
    } catch (Exception e) {
        connection.rollback();
    } finally {
        connection.close();
    }
}
```

Java에서 데이터베이스의 트랜잭션을 시작하는 방법은 이 방법 밖에는 없다.
즉, 스프링도 최종적으론 이 JDBC API를 호출함으로써 트랜잭션을 관리하고 있다.
사실 스프링이 `@Transactional` 메소드에 대해서 해주는 역할은 위 동작을 해주는 것 뿐이다.

## JDBC 트랜잭션 전파 옵션과 격리 수준

여기서 트랜잭션의 전파(propagation) 방식과 격리(isolation) 수준을 다르게 한다면 아래와 같다.

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.Savepoint;

public void doTransaction(DataSource dataSource) {
    Connection connection;
    Savepoint savepoint;
    try {
        connection = dataSource.getConnection();
        // 격리 수준 설정
        connection.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
        // 전파 옵션 설정
        savepoint = connection.setSavepoint();
        connection.setAutoCommit(false);
        // SQL 실행
        connection.commit();
    } catch (Exception e) {
        connection.rollback(savepoint);
    } finally {
        connection.close();
    }
}
```

# 스프링의 트랜잭션 관리

스프링에서는 위 작업을 단순화하기 위해서 `PlatformTransactionManager` 인터페이스를 만들었고 이 인터페이스는
JDBC가 트랜잭션을 시작하던 방법을 추상화했다.

## 프로그래밍적 트랜잭션 관리

`PlatformTransactionManager`를 통해 트랜잭션을 시작하고 관리하는 코드를 작성한다면 아래와 같이 작성할 수 있다.

```java
public Object doTransaction(PlatformTransactionManager platformTransactionManager) {
    // 트랜잭션 옵션 설정
    TransactionDefinition transactionDefinition;
    transactionDefinition.setPropagationBehavior(PROPAGATION_REQUIRES_NEW);
    transactionDefinition.setIsolationLevel(ISOLATION_READ_COMMITTED);
    
    // Connection 얻어오기
    TransactionStatus transactionStatus = platformTransactionManager.getTranscation(transactionDefinition);
    try {
        // SQL 실행
    } catch (Exception e) {
        platformTransactionManager.rollback();
        throw new RuntimeException(e);
    }
    platformTransactionManager.commit();
    return result;
}
```

스프링이 구체적으로 제시한 사용법은 `TransactionTemplate`에 잘 구현되어 있다.

*TransactionTemplate.java*
```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");

    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager cpptm) {
        return cpptm.execute(this, action);
    } else {
        TransactionStatus status = this.transactionManager.getTransaction(this);
        T result;
        try {
            result = action.doInTransaction(status);
        } catch (RuntimeException | Error ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);
            throw ex;
        } catch (Throwable ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
        }
        this.transactionManager.commit(status);
        return result;
    }
}
```

이런 추상화 덕분에 스프링 사용자는 다음과 같이 간편해졌다.
- 더이상 JDBC의 `Connection`을 `open`, `commit`, `close`등을 호출할 필요가 없어짐
- 예외 발생 시 `RuntimeException`으로 변경하기 때문에 `try-catch` 구문을 작성할 필요가 없어짐
- 트랜잭션 안에서의 동작만 기술하면 됨

하지만 이렇게 코드를 통한 트랜잭션 실행은 스프링이 근본적으로 추구하는 방향은 아니었다.

## 선언적 트랜잭션 관리

스프링에서 XML 방식으로 설정하는 것이 기본이었을 때에는 트랜잭션 또한 XML로 설정해야 했다.
XML을 통한 설정법은 이미 레거시(Legacy)가 되었으므로 자세히 알아보지는 않겠지만 이런 방식이 가능하다는 것을 알아두면 좋을 것 같다.

```xml
<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!-- get으로 시작하는 메소드는 readOnly를 true로 설정 -->
        <tx:method name="get*" read-only="true"/>
        <tx:method name="*"/>
    </tx:attributes>
</tx:advice>
```

스프링 AOP를 통해 Advice를 위와 같이 구성하고 아래와 같이 특정 빈에 적용할 수 있었다.

```xml
<aop:config>
    <aop:pointcut id="userServicePointcut" expression="execution(* x.y.service.UserService.*(..))"/>
    <aop:advisor advice-ref="transactionAdvice" pointcut-ref="userServicePointcut"/>
</aop:config>

<bean id="userService" class="x.y.service.UserService"/>
```

그리고 `UserService` 클래스는 아래와 같을 것이다.

```java
public class UserService {
    public Long save(User user) {
        // SQL 실행
        return id;
    }
}
```

AOP를 이용한 선언적 트랜잭션 관리를 사용함으로써 프로그래밍적 트랜잭션보다 코드는 훨씬 더 간단해졌다.
하지만 이를 위해 XML 파일을 작성하고 설정해야 했으며 XML 파일은 내용이 너무 장황했다.

## @Transactional

이제 최근 스프링의 트랜잭션 관리 방법인 `@Transactional` 사용을 보자.

```java
public class UserService {
    @Transactional
    public Long save(User user) {
        // SQL 실행
        return id;
    }
}
```

이제 더이상 XML 설정도 필요 없고 `try-catch`, `commit`, `rollback` 같은 코드도 필요하지 않다.

하지만 다음 두 가지 설정을 해야한다.

- Spring Configuration 중에 `@EnableTransactionManagement`가 적용된 Configuration이 있어야 한다.
  (Spring Boot에서는 자동으로 설정해준다.)
- `PlatformTransactionManager` 빈이 등록되어 있어야 한다.

이 두 설정만 마치면 스프링은 자동으로 `@Transactional`이 있는 _**public**_ 메소드에 대해 트랜잭션을 관리할 것이다.

즉, 스프링은 위의 코드를 아래와 같이 _번역해_ 줄 것이다.

```java
public class UserService {
    public Long save(User user) {
        Connection connection;
        try {
            connection = dataSource.getConnection();
            connection.setAutoCommit(false);
            // SQL 실행
            return id;
        } catch (Exception e) {
            connection.rollback();
        } finally {
            connection.close();
        }
    }
}
```

어떻게 이게 가능한지 이제 파헤쳐보자.

# CGLIB, JDK 프록시

내가 작성한 `UserService`의 `save()` 메소드는 SQL을 실행하고 `id`를 리턴할 뿐인데 스프링이 내가 짠 코드를 변경할 방법은 없다.

그 과정에서 스프링은 꼼수를 부린다. 내가 만든 `UserService`만 객체로 만드는 것이 아니라 **프록시 객체도 같이 생성한다**.

그리고 스프링의 IoC 컨테이너는 `UserService`빈을 필요로 하는 다른 곳에 의존성으로 주입해줄 때 **프록시를 대신 주입한다.**


CGLIB 라이브러리는 `UserService`를 상속한 프록시 객체를 만들어준다. (JDK 프록시는 방식이 약간 다르긴 여기서 다루진 않겠다.)

```java
public class UserController {
    
    @Autowired
    UserService userService;
    
    public Long post(User user) {
        return userService.save(user);
    }
}
```

즉, `UserController`에서 `UserService`의 `save()` 메소드를 호출하면, 진짜 `UserService`가 호출되는 것이 아니라
가짜로 만든 프록시의 `save()`가 호출되는 것이다.

# `PlatformTransactionManager` 역할

그렇다면 이 과정에서 `PlatformTransactionManager`는 왜 필요한 걸까?

생성된 프록시는 직접 트랜잭션을 컨트롤하지 않는다. 대신 `PlatformTransactionManager`에게 이런 임무를 맡겨버린다.

예를 들어 `DataSourceTransactionManager`의 `doBegin()` 메소드의 소스코드를 보면 우리가 초반부에 알아본 방법과
동일하다는 것을 알 수 있다.

_DataSourceTransactionManager.java_
```java
public class DataSourceTransactionManager implements PlatformTransactionManager {
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        Connection con;
        try {
            con = this.obtainDataSource().getConnection();
            // ..
            con.setAutoCommit(false);
            // JDBC 트랜잭션 시작 방법과 동일
        } catch (Exception e) {
            // 생략
        }
    }
}
```

스프링이 구현한 `DataSourceTransactionManager`는 우리가 처음에 알아본 JDBC로 트랜잭션을 시작하는 방법과 완전히 똑같다.

# 요약

트랜잭션을 편리하게 할 수 있게 스프링이 우리에게 해준 것은 반복된 코드를 추상화해준 것이다.

1. 스프링은 `@Transactional`이 붙은 메소드(또느 클래스)를 찾아서 프록시 클래스를 만든다.
2. 프록시는 `TransactionManager`를 자체적으로 가지고 있고 트랜잭션 관리를 여기에 맡겨버린다.
3. `TransactionManager`는 JDBC로 트랜잭션을 관리하는 방법과 동일하게 트랜잭션을 관리한다.
