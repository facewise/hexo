---
title: '[Spring Security] 메소드 인가 (Method Security)'
date: 2024-04-12 10:59:11
tag: Security
category:
  - Java
  - Spring
---

> Spring Security 6.2.3 버전을 기준으로 작성했습니다.

# Method Security란?

스프링 시큐리티를 처음 접할 때 기본적으로 다루는 인가는 요청 URI에 대한 인가인 것 같다.

```java
@Bean
SecurityFilterChain web(HttpSecurity http) throws Exception {
	http
		.authorizeHttpRequests((authorize) -> authorize
			.requestMatchers("/user/**").hasAuthority("USER")
            .requestMatchers("/admin/**").hasAuthority("ADMIN")
			.anyRequest().authenticated()
		)

	return http.build();
}
```

위 코드처럼 빈을 만들고 `requestMatchers()`를 사용해서 API 엔드포인트에 인가를 적용하는 방식을 처음 접하게 된다.

규모가 작은 사이드 프로젝트에서는 해당 방식으로 인가를 처리해도 충분히 커버 가능하다.

그리고 도메인 권한이 중요하지 않으면 크게 문제가 없다.

하지만 도메인에 대한 권한을 엄격하게 관리하는 서비스이거나 대규모 프로젝트에서는

API 엔드포인트의 수도 늘어나기 때문에 엔드포인트만 검사하는 것만으로는 촘촘하게 권한을 검사하기 힘들 수 있다.

그래서 다음과 같은 경우에 **Method Security**를 사용하면 좋다:

- 권한 체크 로직이 복잡해서 세분화되어야 할 경우
- 서비스 레이어에서도 권한을 체크해야 할 경우
- 애너테이션 방식의 코드 스타일과 AOP를 지향할 경우

|                 | API 레벨       | 메소드 레벨   |
| --------------- | -------------- | ------------- |
| **권한 수준**   | 촘촘하지 않음  | 촘촘함        |
| **설정 방법**   | Config 빈      | 메소드에 선언 |
| **설정 스타일** | DSL            | 애너테이션    |
| **인가 표현식** | Ant, 정규식 등 | SpEL          |

**메소드 인가**는 **Spring AOP** 기반으로 작동한다.

즉, 메소드 호출의 **before**와 **after**에 권한을 체크하고 싶은 경우 사용하면 된다.

<br>

# 메소드 시큐리티 활성화

메소드 레벨 인가를 활성화하기 위해서는 `@Configuration`이 설정된 빈에 `@EnableMethodSecurity` 를 추가해주어야 한다.

```java
@EnableMethodSecurity
@Configuration
public class SecurityConfig {}
```

<br>

# 메소드 인가의 흐름

```java
@Service
public class MyService {
    @PreAuthorize("hasAuthority('permission:read')")
    @PostAuthorize("returnObject.owner == authentication.name")
    public Customer readCustomer(String id) { ... }
}
```

위와 같은 서비스가 있다고 가정하자.

그러면 `readCustomer()`라는 메소드에 대한 인가는 다음과 같은 흐름으로 적용된다.

1. **Spring AOP**는 `readCustomer()`의 프록시 메소드를 만들어서 감싼다.
2. `readCustomer()`의 프록시 메소드에서는 `AuthorizationManagerBeforeMethodInterceptor`를 호출한다.
3. **인터셉터**는 `PreAuthroizeAuthorizationManager#check()`를 호출한다.
4. **AuthorizationManager**는 `@PreAuthorize`안의 SpEL 표현식과 `Authentication`을 담은 컨텍스트를 **인터셉터**에 제공한다.
5. **인터셉터**는 컨텍스트에 담긴 `Authentication`과 표현식을 보고 권한 부여가 가능한지 체크한다.
6. 권한 체크에 통과하면 AOP는 `readCustomer()`를 호출한다.
7. 체크에 통과하지 못한다면 `AccessDeniedException`를 던진다.
8. `readCustomer()`의 호출이 끝나고 리턴되면 프록시가 `AuthorizationManagerAfterMethodInterceptor`를 호출한다.
9. **인터셉터**는 `PostAuthroizeAuthorizationManager#check()`를 호출한다.
10. `@PostAuthorize`안의 표현식이 통과하면 정상적인 흐름으로 종료된다.
11. 마찬가지로 통과하지 못하면 `AccessDeniedException`을 던진다.

# 애너테이션

메소드 인가를 설정하는 방법은 여러가지가 있지만 애너테이션 방식이 제일 권장된다.

애너테이션에서 권한을 검증하는 로직을 작성할 때는 여러가지 방법이 있는데

이 내용은 별도의 문서로 정리하겠다.

## `@PreAuthorize`

`@PreAuthorize`는 메소드 호출 전, 권한을 검증하고 싶을 때 사용한다.

```java
@Component
public class BankService {
	@PreAuthorize("hasRole('ADMIN')")
	public Account readAccount(Long id) {
	}
}
```

위 메소드는 호출하기 전에 `Authentication` 객체가 `ROLE_ADMIN` Role이 있는지 체크하고

Role이 없다면 `AccessDeniedException`이 던져진다.

`MockUser`를 사용해서 테스트를 작성할 수 있다.

```java
@Autowired
BankService bankService;

@WithMockUser(roles="ADMIN")
@Test
void readAccountWithAdminRoleThenInvokes() {
    Account account = bankService.readAccount("1");
}

@WithMockUser(roles="USER")
@Test
void readAccountWithUserRoleThenAccessDenied() {
    assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(
        () -> this.bankService.readAccount("1"));
}
```

## `@PostAuthorize`

`@PostAuthorize`는 메소드가 호출되고 리턴될 때 권한을 검증하고 싶을 때 사용한다.

```java
@Component
public class BankService {
	@PostAuthorize("returnObject.owner == authentication.name")
	public Account readAccount(Long id) {
        ...
        return account;
	}
}
```

위 메소드는 리턴되는 `Account` 객체의 `owner` 프로퍼티가

`Authentication` 객체의 `name` 프로퍼티와 같을 때만 정상적으로 리턴되고

같지 않다면 마찬가지로 `AccessDeniedException`이 발생한다.

아래와 같이 테스트해볼 수 있다.

```java
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void readAccountWhenOwnedThenReturns() {
    Account account = bankService.readAccount("1");
    // ... assertions
}

@WithMockUser(username="wrong")
@Test
void readAccountWhenNotOwnedThenAccessDenied() {
    assertThatExceptionOfType(AccessDeniedException.class).isThrownBy(
        () -> this.bankService.readAccount("1"));
}
```

## `@PreFilter`

`@PreFilter`는 메소드 호출 전 파라미터로 전달되는 객체들을 인가를 통해 필터링하고 싶을 때 사용한다.

```java
@Component
public class BankService {
	@PreFilter("filterObject.owner == authentication.name")
	public List<Account> updateAccounts(Account... accounts) {
        return accounts;
	}
}
```

위 메소드에서 파라미터로 전달되는 `accounts` 안에는

`owner` 프로퍼티가 `Authentication`의 `name` 프로퍼티와 같은 객체만 필터링돼서 전달된다.

```java
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void updateAccountsWhenOwnedThenReturns() {
    Account ownedBy = new Account("owner");
    Account notOwnedBy = new Account("not owner");
    List<Account> updated = bankService.updateAccounts(ownedBy, notOwnedBy);
    assertThat(updated).containsOnly(ownedBy);
}
```

위 테스트 코드에서 `updateAccounts()`의 파라미터로 2개의 `Account` 객체를 전달했지만,

`@PreFilter`로 인해 실제 메소드 내부에 전달되는 파라미터는 `ownedBy` 객체만 전달된다.

`@PreFilter`로 필터링할 수 있는 파라미터 타입은 배열, `Collection`, `Map`, `Stream`이다.
(`Stream`은 닫히지 않은 상태여야 한다.)

```java
@PreFilter
public void updateAccounts(Account[] accounts)

@PreFilter
public void updateAccounts(Collection<Account> accounts)

@PreFilter
public void updateAccounts(Map<String, Account> accounts)

@PreFilter
public void updateAccounts(Stream<Account> accounts)
```

## `@PostFilter`

`@PostFilter`는 메소드가 리턴하는 객체들을 인가를 통해 필터링하고 싶을 때 사용한다.

```java
@Component
public class BankService {
	@PostFilter("filterObject.owner == authentication.name")
	public Collection<Account> readAccounts(String... ids) {
        return accounts;
	}
}
```

위 코드는 리턴되는 `accounts` 컬렉션에서 객체들의 `owner` 프로퍼티가

`Authentication`의 `name` 프로퍼티와 같은 객체만 필터링되어 리턴된다는 의미이다.

아래와 같은 테스트코드로 확인해볼 수 있다.

```java
@Autowired
BankService bankService;

@WithMockUser(username="owner")
@Test
void readAccountsWhenOwnedThenReturns() {
    Collection<Account> accounts = bankService.updateAccounts("owner", "not-owner");
    assertThat(accounts).hasSize(1);
    assertThat(accounts.get(0).getOwner()).isEqualTo("owner");
}
```

`@PostFilter`로 필터링할 수 있는 리턴 타입은 배열, `Collection`, `Map`, `Stream`이다.

```java
@PostFilter
public Account[] readAccounts()

@PostFilter
public Collection<Account> readAccounts()

@PostFilter
public Map<String, Account> readAccounts()

@PostFilter
public Stream<Account> readAccounts()
```

**주의사항**
> `@PreFilter`나 `@PostFilter`를 사용해서 사이즈가 큰 컬렉션을 필터링하는 작업은
> 메모리를 매우 많이 소모한다. 자칫 잘못하면 OOM이 발생할 수도 있다.
> 그렇기 때문에 사이즈가 큰 컬렉션을 가져와서 필터링하기보다는
> 데이터를 가져올 때 먼저 필터링해서 가져오는 방식을 택해야 한다.
> 즉, SQL이나 NoSQL로 데이터를 가져올 때부터 적절한 조건식으로 데이터를 먼저 필터링해야 한다.

## 기타 애너테이션

### `@Secured`

`@Secured`는 `@PreAuthorize`의 레거시 버전이며 사용하려면

`@EnableMethodSecurity(securedEnabled = true)` 로 변경해야 한다.

### JSR-250 애너테이션

JSR-250 애너테이션에는 `@RolseAllowed`, `@PermitAll`, `@DenyAll` 등이 있으며 사용하려면

`@EnableMethodSecurity(jsr250Enabled = true)` 로 변경해야 한다.

## 클래스 레벨 애너테이션

```java
@Controller
@PreAuthorize("hasAuthority('ROLE_USER')")
public class UserApiController {
    @GetMapping("/endpoint")
    public String endpoint() { ... }
}
```

위와 같이 클래스 레벨로 애너테이션을 추가할 수 있고 해당 클래스의 모든 메소드에 적용된다.

```java
@Controller
@PreAuthorize("hasAuthority('ROLE_USER')")
public class UserApiController {
    @GetMapping("/admin")
    @PreAuthorize("hasAuthority('ROLE_ADMIN')")
    public String admin() { ... }
}
```

위처럼 예외로 다른 표현식을 사용해야할 경우에는 해당 메소드에 애너테이션을 오버라이드할 수 있다.

## 메타 애너테이션 (커스텀 애너테이션)

위에서 소개한 `@PreAuthorize`, `@PostAuthorize` 등의 애너테이션에 똑같은 표현식을

계속 적는 것은 분명 가독성과 유지보수 관점에서 좋지 않다.

이를 위해 메타 애너테이션이라 불리는 커스텀 애너테이션을 만들어서 대신 사용할 수 있다.

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasRole('ADMIN')")
public @interface IsAdmin {}
```

위처럼 애너테이션 인터페이스를 만들고

```java
@Component
public class BankService {
	@IsAdmin
	public Account readAccount(Long id) {
	}
}
```

와 같이 커스텀 애너테이션으로 `@PreAuthorize("hasRole('ADMIN')")`와 같은 효과를 낼 수 있다.
