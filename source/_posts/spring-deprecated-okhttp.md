---
title: 'Spring 프레임워크의 OkHttp 의존성 제거'
date: 2024-12-04 14:44:40
category:
  - Java
  - Spring
---

# OkHttp란?

[OkHttp](https://github.com/square/okhttp)는 HTTP/2 지원, 내장 캐시, 쉬운 사용법 등으로 Android 계열에서 널리 사용되고 0에 가까운 의존성으로 Java 라이브러리 진영에서도 자주 사용되**었**던 오픈소스 HTTP 클라이언트입니다.

Square Inc.(현재는 BLOCK Inc.)의 오픈소스 팀에서 관리하는 오픈소스 프로젝트 중에 하나입니다.

[OkHttp의 특징](https://github.com/square/okhttp/issues/3472)

이런 특징으로 인해 Spring 진영을 비롯해서 단위테스트 프레임워크 계의 핫 아이템인 [Testcontainers](https://github.com/testcontainers/testcontainers-java)에서도 사용하는 등 범용적이고 신뢰할 수 있는 클라이언트입니다.


## Spring Framework에서 사용하는(했던) OkHttp

Spring Framework에서 REST API를 호출하기 위한 모듈인 `RestTemplate`이나 `RestClient`는 HTTP 통신을 위한 추상화 모듈만 제공할 뿐이고 실제 HTTP 통신은 HTTP Client 모듈에 의존합니다.

`org.springframework.http.client` 패키지를 보면 대략 어떤 HTTP Client 모듈이 있는지 알 수 있습니다.

{% asset_img image.png %}
*spring-web 6.0 버전의 org.springframework.http.client 패키지*

위 이미지를 보면 알 수 있듯이 Spring은 6.0 버전 대까지는 아래 HTTP 클라이언트들을 기본 제공했습니다.

- `HttpComponentsClientHttpRequestFactory` : Apache의 `HttpClient`
- `OkHttp3ClientHttpRequestFactory` : OkHttp3의 `OkHttpClient`
- `SimpleClientHttpRequestFactory` : JDK의 `HttpURLConnection`


# 사건의 발단

문제는 이렇게 잘 사용되던 OkHttp에서 2019년 하나의 변화를 시도하면서 시작되었습니다.

{% asset_img image1.png %}
*OkHttp의 Kotlin 전환 선언*

바로 `OkHttp 3`의 코드를 모두 Kotlin으로 전환하고, Kotlin standard library를 추가하여 `OkHttp 4`로의 변화를 시도한 것입니다.

> 이 무렵 Square에서 Kotlin에 굉장히 매료되었는지 Java로 된 오픈소스 프로젝트들을 대거 Kotlin으로의 전환을 시도합니다.
> Android 진영에 많은 기여를 하던 Square이어서 그런지 Kotlin으로의 전환은 빨랐습니다.

그러면서 자신들의 라이브러리가 두루 이용되고 있다는 것을 알고 있던 Square였기 때문에 이전 버전과 100% 호환성을 제공하겠다고 약속했습니다.


# 반응

하지만 해당 이슈에서 볼 수 있듯이 Zero-dependency에 가깝던 Java 라이브러리가 Kotlin 런타임의 큰 의존성이 생긴다는 점은 Java 오픈소스 진영에겐 반갑지 않았습니다.

[Testcontainers for Java](https://java.testcontainers.org/)의 제작자이면서 [Java Champion](https://javachampions.org/members.html)이기도 한 [Sergei Egorov](https://github.com/bsideup)는 아쉬움을 표현했습니다.

{% asset_img image2.png %}
*Sergei Egorov의 코멘트*

> 최고의 JVM HTTP 라이브러리 중 하나였던 OkHttp가 이제 더이상 좋은 선택이 아니게 되어서 참 유감입니다.
> 세상이 이전의 실수(Scala, Groovy로 만든 Java 라이브러리)에서 교훈을 얻지 못한 것이 슬픕니다.
>
> OkHttp가 정말 좋았던 점은 다른 의존성이 없었고 [shade](https://maven.apache.org/plugins/maven-shade-plugin/)하기 좋았던 점이었습니다. 이제 더이상 그렇지 않겠네요.
> 여기에 댓글을 남겨놓을테니, 다음에 'OkHttp가 Kotlin X.Y.Z 버전과 호환되지 않습니다'와 같은 이슈를 만나면 여기에 와서 👍를 눌러주세요.

또, 자신의 라이브러리를 **OkHttp보다는 덜 유명하긴 한데 많이 쓰이는 Testcontainers**라고 소개하면서 보일러플레이트를 제거하려고 다른 언어까지 쓰는 OkHttp를 더이상 사용할 수 없다며 약간의 조롱 섞인 코멘트도 덧붙였습니다.


## Spring의 반응

선언한대로 OkHttp 4는 2019년에 릴리즈되었습니다. 하지만 Spring은 OkHttp3이 아주 잘 만들어진 라이브러리여서 그런지 바로 걷어내지 않았습니다.

Spring Boot에서도 2.6.X 버전까지는 OkHttp 3의 버전을 관리해주다가 2.7.0부터는 OkHttp 4로 버전을 업그레이드하는 등 계속해서 사용하는 듯한 모습이었습니다.

하지만 더이상 지원되지 않는 버전의 라이브러리는 계속해서 사용할 수 없었고, 특히 Web과 관련된 라이브러리는 더욱더 그렇습니다. TLS의 최신 싸이퍼에 대한 지원이 되지 않는 점 등이 치명적인 보안 취약점이 될 수 있기 때문입니다.

그리고 Spring Boot가 공식 지원하는 OkHttp 4 버전을 계속 쓰게 놔두자니 자신들의 프로젝트에 거대한 Kotlin 라이브러리 의존성이 생기는 것이 탐탁치 않았을 것이라고 추측합니다.

결국 Spring Web은 `OkHttp3Client` 시리즈를 6.1 버전에서 `Deprecated` 처리하고 6.2 버전에서 삭제하기로 합니다.

{% asset_img image3.png %}
*Spring 이슈*

> 이름에 걸맞게 OkHttp3ClientHttpRequestFactory는 OkHttp 3을 기반으로 합니다. 이후 3과 호환이 가능한 OkHttp4가 출시되었지만, 여기에는 Kotlin 런타임이 필요합니다.
> OkHttp 5는 현재 개발 중이며 이전 버전과 호환되지 않는 것으로 보입니다. 어떤 종류의 백 포팅 정책이 있는지 불분명하고 오래된 종속성을 유지하면 보안 위험이 발생하므로 OkHttp3ClientHttpRequestFactory를 더 이상 사용하지 않아야 합니다.
> 또한 현재 및 향후 OkHttp 구현에는 런타임에 Kotlin이 필요하므로 Java 사용에는 적합하지 않습니다. 6.1에 새로운 `ClientHttpRequestFactory`가 도입되어 `OkHttp3ClientHttpRequestFactory`를 대체할 수 있는 다양한 선택의 폭이 생겼습니다.


### 새로운 HTTP 클라이언트 도입

Java HTTP 클라이언트의 주류였던 OkHttp를 제거함과 동시에 다른 좋은 HTTP 클라이언트를 도입했습니다.

- `JdkClientHttpRequestFactory` : JDK 11부터 추가된 `HttpClient`
- `JettyClientHttpRequestFactory` : Jetty의 `HttpClient`
- `ReactorClientHttpRequestFactory` : Netty의 `HttpClient` (6.2 버전)


## Spring Boot의 대응

Spring Framework에서 OkHttp3을 `Deprecated` 처리함에 따라 가장 밀접한 Spring Boot 역시 기민하게 움직였습니다.

{% asset_img image4.png %}
*Spring Boot 이슈*

Spring Boot v3.4 부터는 spring-boot-dependencies의 BOM에서 OkHttp의 의존성 버전을 제거했습니다. 따라서, Spring Boot 프로젝트에서 OkHttp 클라이언트를 사용하기 위해서는 이제 버전을 직접 명시해야 합니다.


# 소견

처음에는 Kotlin으로 만든 라이브러리가 무슨 문제가 있는지 의아했습니다. 하지만 Kotlin으로 컴파일된 jar 라이브러리를 Java 프로젝트에서 실행하려면 `kotlin-stdlib` 모듈이 필요하다는 것이 치명적인 단점이었습니다.

`kotlin-stdlib`이 Java 프로젝트에 포함되면 Kotlin을 실행할 수 있는 런타임이 들어오게 되어서 Java 파일 안에서 Kotlin에만 있는 함수나 coroutine 등을 실행할 수 있게 됩니다.
잘 사용하면 장점이 될 수도 있는 강력한 라이브러리이지만, Kotlin을 사용할 계획이 없는 프로젝트에 HTTP 호출만을 위해 1MB가 넘는 라이브러리가 포함되는 것은 분명히 단점도 있습니다. 많은 Java 오픈소스 프로젝트들은 이런 이유로 OkHttp를 걷어내는 작업들을 진행한 것으로 보입니다.

하지만 품질 좋은 HTTP API를 제공하는 [Feign](https://github.com/OpenFeign/feign)은 아직 OkHttp에 대한 지원을 중단하지 않았습니다.
요즘 [Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign)을 사용하는 프로젝트가 많아졌는데, Feign의 HTTP 클라이언트를 OkHttp로 선택한다면 자신의 Java 프로젝트에 불필요한 Kotlin 런타임이 포함된다는 사실을 인지해야 할 것입니다.