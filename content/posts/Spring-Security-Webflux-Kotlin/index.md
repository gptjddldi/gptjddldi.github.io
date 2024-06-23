---
title: "코틀린 스프링 웹플럭스 세션 인증"
date: 2024-01-13T18:12:02+09:00
---

Spring Security 개발을 처음 시작하면서 많은 블로그의 소스 코드를 참고했다. 스프링에 처음 입문하고, 블로그 마다 구현 방식이 제각각이고, 최신 Spring Securty 6 버전을 사용한 코드를 찾기 어려워서 적용에 어려움이 많았다.

spring security 의 내부 구현을 살펴보면서 프로젝트에 세션 인증을 적용해 보자.

소스코드는 https://github.com/gptjddldi/spring-tutorial/tree/master/webflux-security-tutorial 여기에서 확인할 수 있다.

---

## 내부 구현 확인

{{< figure src="images/diagram.png" caption="WebAuthentication Filter 순서도">}}

```java
// AuthenticationWebFilter
@Override
public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
    return this.requiresAuthenticationMatcher.matches(exchange)
        .filter((matchResult) -> matchResult.isMatch())
        .flatMap((matchResult) -> this.authenticationConverter.convert(exchange))
        .switchIfEmpty(chain.filter(exchange).then(Mono.empty()))
        .flatMap((token) -> authenticate(exchange, chain, token))
        .onErrorResume(AuthenticationException.class, (ex) -> this.authenticationFailureHandler
            .onAuthenticationFailure(new WebFilterExchange(exchange, chain), ex));
}

private Mono<Void> authenticate(ServerWebExchange exchange, WebFilterChain chain, Authentication token) {
    return this.authenticationManagerResolver.resolve(exchange)
        .flatMap((authenticationManager) -> authenticationManager.authenticate(token))
        .switchIfEmpty(Mono
            .defer(() -> Mono.error(new IllegalStateException("No provider found for " + token.getClass()))))
        .flatMap(
                (authentication) -> onAuthenticationSuccess(authentication, new WebFilterExchange(exchange, chain)))
        .doOnError(AuthenticationException.class,
                (ex) -> logger.debug(LogMessage.format("Authentication failed: %s", ex.getMessage())));
}

// ...
protected Mono<Void> onAuthenticationSuccess(Authentication authentication, WebFilterExchange webFilterExchange) {
    ServerWebExchange exchange = webFilterExchange.getExchange();
    SecurityContextImpl securityContext = new SecurityContextImpl();
    securityContext.setAuthentication(authentication);
    return this.securityContextRepository.save(exchange, securityContext)
        .then(this.authenticationSuccessHandler.onAuthenticationSuccess(webFilterExchange, authentication))
        .contextWrite(ReactiveSecurityContextHolder.withSecurityContext(Mono.just(securityContext)));
}
```

먼저 requiresAuthenticationMatcher.matches(exchange) 에서 request url 이 로그인 요청을 처리할 url 인지 판별한다.  
이후에 authenticationConverter.convert(exchange) 는 request 에서 인증 정보(id, password)를 가져와 Authentication 객체로 변환한다.  
이렇게 변환된 Authenticate 객체는 authenticate 메소드를 통해 유효한 인증인지 확인된다.  
유효한 인증인 경우 securityContextREpository.save() 를 통해 repository 에 인증 정보가 저장되고 onAuthenticationSuccess 메소드가 실행된다. 유효하지 않은 객체인 경우 AuthenticationException 을 발생시키고 authenticationFailureHandler 메소드가 실행된다.



<!-- // AbstractUserDetailsReactiveAuthenticationManager
@Override
public Mono<Authentication> authenticate(Authentication authentication) {
    String username = authentication.getName();
    String presentedPassword = (String) authentication.getCredentials();
    // @formatter:off
    return retrieveUser(username)
            .doOnNext(this.preAuthenticationChecks::check)
            .publishOn(this.scheduler)
            .filter((userDetails) -> this.passwordEncoder.matches(presentedPassword, userDetails.getPassword()))
            .switchIfEmpty(Mono.defer(() -> Mono.error(new BadCredentialsException("Invalid Credentials"))))
            .flatMap((userDetails) -> upgradeEncodingIfNecessary(userDetails, presentedPassword))
            .doOnNext(this.postAuthenticationChecks::check)
            .map(this::createUsernamePasswordAuthenticationToken);
    // @formatter:on
} -->

<!-- 예시로 UserDetailsRepositoryReactiveAuthenticationManager 클래스의 인증 과정을 살펴보자.

Authentication Manager 는 Authentication 객체에 있는 id, password 값과 실제 저장소(db) 의 id, password 값이 일치하는지 확인한다. 
일치하는 경우 createUsernamePasswordAuthenticationToken 를 통해 유효한 Authenticate 객체를 리턴한다. -->

---
### 인증 프로세스 정리

```
1. 사용자는 identifier 와 password 를 포함하여 인증 서버로 로그인 요청을 보낸다.

2. ServerAuthenticationConverter 는 이 요청을 Authentication 객체로 변환하여 Authentication Manager 에 전달한다.

3. Authentication Manager 는 전달받은 Authentication 객체의 정보와 실제 사용자의 정보 비교해 인증 여부를 결정한다.

4. 인증에 성공한 경우 인증 정보를 Repository 에 저장하고 onAuthenticationSuccess 가 실행되며, 실패한 경우 onAuthenticationFailure 가 실행된다.

5. 인증에 성공한 경우 세션이 생성되며 사용자에게 세션 정보가 반환된다. 
(ex: Set-Cookie:SESSION={session code}; Path=/; HttpOnly; SameSite=Lax)

6. 사용자는 이후 요청에 세션을 포함하여 서버에 인증된 요청을 보낼 수 있다.
```
---

## 구현

우리가 직접 구현할 빈은 다음과 같다.

```
- SecurityWebFilterChain
- AuthenticationWebFilter
- ReactiveAuthenticationManager
  - ReactiveUserDetailsService
    - UserDetails
- ServerAuthenticationConverter
- AuthenticationSuccessHandler
```
---

먼저 spring security 설정을 모아둘 config 클래스를 생성한다.

```kotlin
@Configuration
@EnableWebFluxSecurity
class SecurityConfig {
    @Bean
    fun securityWebFilterChain(http: ServerHttpSecurity, sessionLoginFilter: AuthenticationWebFilter): SecurityWebFilterChain {

        return http
            .csrf { it.disable() }
            .formLogin { it.disable() }
            .logout { it.disable() }
            .httpBasic { it.disable() }
            .authorizeExchange{
                it
                    .pathMatchers("/login")
                    .permitAll()
                    .pathMatchers("/register")
                    .permitAll()
                    .anyExchange()
                    .authenticated()
            }
            .addFilterAt(sessionLoginFilter, SecurityWebFiltersOrder.AUTHENTICATION)
            .build()
    }

    @Bean
    fun serverSecurityContextRepository(): ServerSecurityContextRepository? {
        return WebSessionServerSecurityContextRepository()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder {
        return BCryptPasswordEncoder()
    }

    @Bean
    fun sessionAuthenticationManager(
        customUserDetailsService: CustomUserDetailsService,
        passwordEncoder: PasswordEncoder
    ): ReactiveAuthenticationManager {
        val manager = UserDetailsRepositoryReactiveAuthenticationManager(customUserDetailsService)
        manager.setPasswordEncoder(passwordEncoder)
        return manager
    }

}
```
securityWebFilterChain 에서 
```.pathMatchers("/login").permitAll()``` 와 ```.addFilterAt(sessionLoginFilter, SecurityWebFiltersOrder.AUTHENTICATION)``` 이 두 부분에 주목하자.
```/login``` path 로 오는 요청은 인증을 거치지 않는다는 의미이다. 회원가입 페이지, 로그인 페이지, 인증 없이 접근할 수 있는 페이지의 path 를 등록하면 되겠다. 이번 실습에서는 로그인 테스트만 하기 때문에 ```/login``` path 만 허용하고 나머지 요청은 인증이 필요하도록 한다.  
그 다음은 AUTHENTICATION 단계에 sessionLoginFilter 필터를 추가한다는 의미이다. Spring Security 는 여러 단계의 Filter 로 구성되어 있는데, 특정 단계에 우리가 만든 Custom Filter 를 삽입함으로써 다양한 케이스의 환경을 다룰 수 있다. 

---
``` kotlin
// SecurityConfig

@Bean
fun sessionLoginFilter(
    sessionAuthenticationManager: ReactiveAuthenticationManager,
    serverSecurityContextRepository: ServerSecurityContextRepository,
    customConverter: ServerAuthenticationConverter
) : AuthenticationWebFilter {
    val filter = AuthenticationWebFilter(sessionAuthenticationManager)

    filter.setSecurityContextRepository(serverSecurityContextRepository) // 1
    filter.setRequiresAuthenticationMatcher { ServerWebExchangeMatchers.pathMatchers(HttpMethod.POST, "/login").matches(it) } // 2
    filter.setServerAuthenticationConverter(customConverter) // 3
    filter.setAuthenticationSuccessHandler { webFilterExchange, authentication -> // 4
        webFilterExchange.exchange.response.statusCode = HttpStatus.OK
        val a = webFilterExchange.exchange.response.bufferFactory().wrap("hey, ${authentication.name}".toByteArray())
        webFilterExchange.exchange.response.writeWith(Mono.just(a))
    }

    return filter
}
```
sessionLoginFilter 에서 인증과 관련된 로직들을 설정한다. 
1. Filter 를 만들 때 Authentication Managr 로 sessionAuthenticationManager 를 전달한다. 인증 프로세스 정리의 3번에서 ```Authentication Manager 는 전달받은 Authentication 객체의 정보와 실제 사용자의 정보 비교해 인증 여부를 결정한다.``` 라고 언급했다. 실제 사용자의 정보를 가져올 때 customUserDetailsService 를 통해 가져온다.
2. "/login" POST 요청이 오는 경우에 인증을 수행하도록 한다.
3. Authenticaion 요청에서 인증과 관련된 정보를 추출하여 Authention 객체로 변환하는데, 우리는 name 과 password 를 추출해 객체로 변환시키는 CustomConverter 를 만들어 사용할 것이다.
4. 인증에 성공했을 때 처리다. 200 OK 응답과 요청된 name 파라미터를 전달할 것이다.
5. 인증에 실패했을 경우도 위와 같이 ```filter.setAuthenticationFailureHandler``` 를 설정하면 된다. 실습에서는 401 Unauthorized 로 충분하기 충분하기 때문에 커스텀하지 않고 기본 핸들러를 사용한다.

---
``` kotlin
@Service
class CustomUserDetailsService(
    private val userRepository: UserRepository,
) : ReactiveUserDetailsService {
    override fun findByUsername(username: String): Mono<UserDetails?> {
        return userRepository
            .findByName(username)
            .map {u ->
                User.withUserDetails(CustomUserDetails(u!!)).build()
            }
            .switchIfEmpty(Mono.empty())
    }
}
```
``` java
// MapReactiveUserDetailsService
@Override
public Mono<UserDetails> findByUsername(String username) {
    String key = getKey(username);
    UserDetails result = this.users.get(key);
    return (result != null) ? Mono.just(User.withUserDetails(result).build()) : Mono.empty();
}
```
CustomUserDetailsService 는 findByUsername 메소드를 구현해야 한다. 이 메소드는 UserDetails 타입을 리턴해야 한다.
```MapReactiveUserDetailsService``` 를 참고하여 구현했다.

---
``` kotlin
@Component
class CustomConverter: ServerAuthenticationConverter {
    private val decoder = Jackson2JsonDecoder()

    override fun convert(exchange: ServerWebExchange): Mono<Authentication> {
        val elementType: ResolvableType = ResolvableType.forClass(LoginData::class.java)

        return exchange.request.body
            .next()
            .flatMap { buffer ->
                Mono.justOrEmpty(decoder.decode(buffer, elementType, null, null)).cast(LoginData::class.java) // 1
            }
            .switchIfEmpty(Mono.error(UsernameNotFoundException("username not found")))
            .map {
                UsernamePasswordAuthenticationToken(it.name, it.password) // 2
            }
    }
}
private data class LoginData (val name: String, val password: String)
```
CustomConverter 는 convert 메소드를 구현한다.  
요청(Exchange)으로부터 인증 정보 name, password 를 추출해서 (1)  
Authentication 객체를 리턴한다. (2)

## 결과
{{< figure src="images/result.png" caption="정상적인 인증 요청 결과">}}

"/login" 라우트로 POST 요청을 보냈을 때 response header 에 200ok 와 세션 정보가 담긴 Set-cookie 를 확인할 수 있고 response body 에서 usernname 을 확인할 수 있다.

## 나아가기
지금까지 세션 인증을 살펴봤다. 하지만 세션 인증은 로그인 정보를 서버에 저장하기 때문에 Stateless 환경에서 사용하기 어렵다. Stateless 환경에서는 JWT 를 사용하는 걸 고려해 볼 수 있다.  
그렇다면 세션 인증에서 JWT 인증으로 변경하려면 어느 부분을 수정해야 할까?  

