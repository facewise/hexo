---
title: 가상 스레드 (Virtual Thread)
date: 2024-01-29 17:23:07
category: Java
---

# 가상 스레드 도입 배경

이전부터 [Project Loom](https://wiki.openjdk.org/display/loom/Main)에 대해서 관심이 많았다.

지금까지 자바 서버는 요청 트래픽이 몰리는 상황에서 값비싼 컨텍스트 스위치 비용을 지불해왔다.
<br>
**이를 해결하기 위해 다양한 시도를 해왔다.**

- Reactive Streams와 같은 비동기 API
- 코루틴을 자바에 추가하기

Spring WebFlux는 Reactive API를 훌륭하게 구현해서 스레드가 부족한 환경에서 성능을 개선했다.

또, JVM 진영의 코틀린에서 coroutine을 통해 메소드를 비동기적으로 호출할 수 있게 지원해줬다.
<br>
**하지만 위의 시도들은 다음과 같은 이유로 자바의 표준이 되지 못했다**

- 웹플럭스는 요청을 처리하는 순서가 보장되지 않아 디버깅을 단계별로 진행할 수 없고, 처리되는 스레드가 다를 수 있어 stack trace도 제공할 수 없었다.
<br>
- 자바 플랫폼에 코루틴 API를 적용하려면 엄청난 대규모 작업이 될 수 밖에 없다. 기존의 스레드 API를 사용해 개발했던 내용은 모두 걷어내야 한다.
<br>

코루틴을 사용하려면 코틀린을 사용하면 되지만, 여전히 자바를 사용하고 싶은 사람은 코루틴도 해결책이 아니었다.

이 때문에 **Project Loom**은 자바 방식으로 해결하기 위해 3가지 기능을 도입하기로 하는데 그 중 첫 번째가 [가상 스레드(Virtual Thread)](https://openjdk.org/jeps/425)이다.

<br>

# 가상 스레드란?

가상 스레드는 한마디로 JVM에 의해 관리되는 경량 스레드이다.

기존의 JVM 스레드는 플랫폼 스레드(Platform Thread)라고 부르며 이는 OS 스레드를 래핑(Wrapping)한 스레드여서 OS 스레드와 일대일 관계였다.
<br>

## 플랫폼 스레드와 차이

**플랫폼 스레드는 다음과 같은 단점이 있다.**
- OS 스레드의 래퍼로 구현하기 때문에 사용가능한 스레드 수가 제한됨
- OS에 의해 생성되고 스케줄링되기 때문에 비용이 비싸고 컨텍스트 스위칭 비용도 비싸다
<br>

**이에 비해 가상 스레드는 JDK에서 구현하고 제공하는 user-mode 스레드이다.**

가상 스레드는 특정 OS 스레드에 연결되어 있지 않고 M:N 스케줄링을 사용한다. 가상 스레드의 수(M)가 더 적은 수(N)의 OS 스레드에서 실행되게 예약한다.

{% asset_img vt.png %}

가상 스레드는 CPU에서 계산을 수행할 때만 OS 스레드를 사용한다.

가상 스레드에서 blocking I/O 작업을 시작할 때 자바는 non-blocking OS를 호출하고 가상 스레드를 임시로 중단한다.
<br>

# 기존 스레드 모델과 성능 비교

간단한 애플리케이션을 만들어서 성능 비교를 해보았다.

**스프링 부트 버전**
- Spring Boot 3.2.1

**성능 테스트 도구**
- Apache JMeter

**테스트 환경 (VM)**
- Centos7 x86_64
- CPU 1 core 1 Thread
- 1GB Memory
- JDK 21
<br>

요청을 받으면 300ms동안 sleep하고 리턴하는 간단한 API를 만들었다.

그리고 현재 스레드가 가상 스레드인지 확인하기 위해 `Thread.currentThread().isVirtual`을 사용하여 로그도 남겼다.

```kotlin
package com.example.study.controller

import org.slf4j.LoggerFactory
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

class UserController {

    private val log = LoggerFactory.getLogger(this.javaClass)

    @GetMapping
    fun get(): String {
        log.info("VT: {}", Thread.currentThread().isVirtual)
        Thread.sleep(300)
        return "ok"
    }
}
```
<br>

그리고 톰캣이 요청을 처리할 때 가상 스레드를 사용할 수 있게 Bean을 등록해주었다.

```kotlin
package com.example.study.config

import org.apache.coyote.ProtocolHandler
import org.springframework.boot.web.embedded.tomcat.TomcatProtocolHandlerCustomizer
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.concurrent.Executors

@Configuration
class ThreadConfig {
    @Bean
    fun virtualThreadExecutorCustomizer(): TomcatProtocolHandlerCustomizer<*> {
        return TomcatProtocolHandlerCustomizer { protocolHandler: ProtocolHandler ->
            protocolHandler.executor = Executors.newVirtualThreadPerTaskExecutor()
        }
    }
}
```
<br>

이 Bean 설정은 Spring Boot 3.2 버전 이상에서는 `application.properties`에서
`spring.threads.virtual.enabled=true`를 설정하면 자동으로 생성된다.
<br>

## 테스트 결과

처음에 가상유저를 100으로 했을 때는 플랫폼 스레드와 가상 스레드의 차이가 없었다.

이는 기본 스레드 풀 200개가 활성화되어서 별 차이가 없는 것 같았다.
<br>

## 500 가상 유저

가상 스레드는 최대 스레드 풀을 상회하는 요청이 들어왔을 때 진가를 발휘했다.

**기존 스레드**

TPS
{% asset_img non_vt_500threads.png %}

응답시간
{% asset_img non_vt_500threads_rtt.png %}

기존 스레드 모델에서는 650 ~ 700 TPS를 맴돌았고 평균 응답시간은 750ms였다.

**가상 스레드**

TPS
{% asset_img vt_500threads.png %}

응답시간
{% asset_img vt_500threads_rtt.png %}

가상 스레드로 실행한 서버는 1400 ~ 1700 TPS를 기록했고 평균 응답시간은 초반을 제외하면 300 ~ 400ms 정도였다.
<br>

## 1000 가상 유저

이번엔 훨씬 더 많은 유저를 가정하고 테스트해보았다.

**기존 스레드**

TPS
{% asset_img non_vt_1000threads.png %}

응답시간
{% asset_img non_vt_1000threads_rtt.png %}

기존 스레드의 TPS는 가상 유저가 500일 때와 별다르지 않았고 응답시간은 거의 2배가 되었다.
<br>

**가상 스레드**

TPS
{% asset_img vt_1000threads.png %}

응답시간
{% asset_img vt_1000threads_rtt.png %}

가상 스레드로는 2400 ~ 3000 TPS를 기록했고 평균 응답시간은 300 ~ 400ms였다.
<br>

확실히 기존 플랫폼 스레드만을 사용했을 때보다 유의미하게 성능이 개선되는 것을 확인할 수 있었다.
<br>

# 가상 스레드 주의사항

이렇게 좋아보이기만 하는 가상 스레드이지만 주의해서 사용하지 않으면 오히려 사용하지 않는 것보다도 못한 경우가 될 수 있다.
<br>

- **풀링하지 말 것**
  - 가상 스레드는 생성하는 비용은 적지만 하나의 작업을 실행하고 GC에 의해 제거되게 설계되었으므로 풀링해놓고 계속 사용하는 것이 더 낭비가 된다.
  - 톰캣의 스레드 풀 설정을 해제하고 `Executors.newVirtualThreadPerTaskExecutor()`를 설정해준다.
  - 스레드 풀을 사용했던 코드는 세마포어를 사용하게 변경해야 한다.
- **스레드로컬(ThreadLocal) 사용에 주의할 것**
  - 스레드 간에 리소스를 공유했던 스레드로컬(ThreadLocal)을 사용하는 대신 글로벌 캐싱 전략을 사용하도록 변경한다.
    - 예를 들면 스레드로컬에 DB 커넥션을 저장해두고 사용하는 방식이 있었다면 가상 스레드 환경에서는 성능이 저하될 수 있다.
    - 기존의 스레드 로컬을 보완하기 위한 [Scoped Values](https://openjdk.org/jeps/446) 역시 Project Loom이 준비 중인 핵심 기능이다.
- **CPU 연산이 필요한 코드는 실행하지 말 것**
  - 가상 스레드는 non-blocking I/O 작업 처리량을 늘리기 위해 설계된 스레드로써 CPU 연산 성능은 기존 플랫폼 스레드보다 떨어진다.
  - 데이터베이스 I/O, 네트워킹 같은 작업을 가상 스레드에게 맡기는 것이 효율적일 것이다.
- **synchronized 또는 native method을 호출하지 말 것**
  - 가상 스레드가 해당 함수를 만나면 캐리어 스레드(연결된 플랫폼 스레드)에 고정(**pinned**)되어 block하고 마운트를 해제할 수 없게 된다.
  - 플랫폼 스레드로 실행해도 마찬가지이지만 가상 스레드가 플랫폼 스레드에 pinned 되면 해당 OS 스레드도 block 상태가 된다.
  - 자주 사용되지 않거나 인메모리 작업을 보호하는 synchronized 까지 제거할 필요는 없다. 하지만 자주 호출되고 오랫동안 고정시키는 코드 블럭은 [ReentrantLock](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)을 사용하는 것을 고려한다.

