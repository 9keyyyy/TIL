# JWT access/refresh token 구현하기 (with redis)

### **⚙️** 개발 환경

- aws linux2 ec2 + docker (github action 자동배포)
- springboot
- gradle
- java 17
- redis

<br>

현재 진행하고 있는 프로젝트에서 github action + docker를 통해 cicd 구축을 해놓은 상태이기 때문에, 배포 서버에서 workflow를 수정하고 docker-compose를 통해 redis 환경을 구축하는 것까지 진행했다.

전체 코드는 [여기](https://github.com/SejongChatbot/server)서 확인할 수 있다. 

기존에 구현했던 회원가입, 로그인 코드에 jwt access/refresh token 인증 방식을 적용했다.

<br>

## 📍 인증 순서

1. 클라이언트의 로그인 요청
2. id/pw 검증 후 access/refresh token 발급
3. refresh token은 redis에 저장하고, 클라이언트에 access/refresh 토큰 응답
4. 클라이언트가 API 호출 시, access token 만료되었으면 유저의 refresh token을 검증하여 access 토큰 재발급(reissue)

<br><br>


## 📍 의존성 추가 및 jwt/redis 관련 환경변수 설정

### build.gradle

```jsx
implementation 'io.jsonwebtoken:jjwt:0.9.1'
implementation 'org.springframework.boot:spring-boot-starter-data-redis'

// com.sun.xml.bind : jwt
implementation 'com.sun.xml.bind:jaxb-impl:4.0.1'
implementation 'com.sun.xml.bind:jaxb-core:4.0.1'
// javax.xml.bind : jwt
implementation 'javax.xml.bind:jaxb-api:2.4.0-b180830.0359'
```

위와 같이 spring security, jwt, redis를 사용할 수 있도록 의존성을 추가했다. 

### application.yml

```jsx
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://${SEJONGMATE_RDS_URL}:3306/sejongmatedb?serverTimezone=Asia/Seoul
    username: ${SEJONGMATE_RDS_USER}
    password: ${SEJONGMATE_RDS_PW}

  jpa:
    hibernate:
      ddl-auto: create 
    properties:
      hibernate:
        format_sql: true

  driver:
    path: chromedriver

  jwt:
    secret: ${SEJONGMATE_JWT_KEY}
    token:
      access-expiration-time: 43200000    # 12시간
      refresh-expiration-time: 604800000   # 7일

  data:
    redis:
      host: redis # 로컬에서 테스트 할 때는 localhost로 사용  
      port: 6379
```

jwt secret key 및 토큰 만료 시간을 설정하고, redis DB 관련 설정을 한다.

<br><br>


## 📍 프로젝트에 Redis 적용 및 배포 서버 도커에 설치

### RedisConfig

```java
package com.sejongmate.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Value("${spring.data.redis.port}")
    private int port;

    @Value("${spring.data.redis.host}")
    private String host;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        // redisTemplate를 받아와서 set, get, delete를 사용
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
        // setKeySerializer, setValueSerializer 설정
        // redis-cli을 통해 직접 데이터를 조회 시 알아볼 수 없는 형태로 출력되는 것을 방지
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());

        return redisTemplate;
    }

}
```

- Springboot 프로젝트에서 redis를 사용하기 위한 설정
- Lettuce Redis Client 사용 → `RedisTemplate` 의 메서드로 Redis 서버에 명령을 수행할 수 있음
- `application.yml`의 환경변수를 `@Value` 어노테이션을 사용해 변수에 주입

<br>

### EC2 docker-compose.yaml

```jsx
version: "3"
services:
  redis:
    image: redis
    container_name: sejongmate_redis
    restart: always
    hostname: redis
    ports:
      - 6379:6379

  app:
    image: [도커hub 유저아이디]/sejongmate:latest
    restart: always
    container_name: sejongmate
    ports:
      - 8080:8080
    depends_on:
      - redis
```

- hostname으로 설정한 값을 스프링부트 환경 변수 redis host로 설정해주면 됨
- github action workflow에서 도커 이미지 하나를 run 하는 대신, docker-compose 실행 ([상세코드](https://github.com/SejongChatbot/server/blob/dev/.github/workflows/gradle.yml))

<br><br>

## 📍 Security 및 JWT 설정

Authentication Token을 Authentication Manager가 넘겨받아 Authentication 객체를 생성하고, 
이를 Provider에게 전달하여 Token을 생성하게 된다.

### JWT

- `JwtTokenProvider` : 유저 정보로 jwt access/refresh 토큰 생성 및 재발급 + 토큰으로부터 유저 정보 받아옴
- `JwtFilter` : request 앞단에 붙이는 필터. http request에서 토큰을 받아와 정상 토큰일 경우 security context에 저장

### Spring Security

- `JwtSecurityConfig` : JwtFilter를 Spring Security Filter Chain에 추가하기 위한 설정
- `SecurityConfig` : 기본적으로 스프링 시큐리티에 필요한 설정 → jwt 적용 및 authentication 필요한 API 주소 설정
- `JwtAccessDeniedHandler`: 접근 권한 없을 때 403 에러
- `JwtAuthenticationEntryPoint`: 인증 정보 없을 때 401 에러

<br><br>

### JwtTokenProvider

```java
@Component
@RequiredArgsConstructor
@Log4j2
public class JwtTokenProvider {

    private final RedisTemplate<String, String> redisTemplate;

    @Value("${spring.jwt.secret}")
    private String secretKey;

    @Value("${spring.jwt.token.access-expiration-time}")
    private long accessExpirationTime;

    @Value("${spring.jwt.token.refresh-expiration-time}")
    private long refreshExpirationTime;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    /**
     * Access 토큰 생성
     */
    public String createAccessToken(Authentication authentication){
        Claims claims = Jwts.claims().setSubject(authentication.getName());
        Date now = new Date();
        Date expireDate = new Date(now.getTime() + accessExpirationTime);

        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(expireDate)
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .compact();
    }

    /**
     * Refresh 토큰 생성
     */
    public String createRefreshToken(Authentication authentication){
        Claims claims = Jwts.claims().setSubject(authentication.getName());
        Date now = new Date();
        Date expireDate = new Date(now.getTime() + refreshExpirationTime);

        String refreshToken = Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(expireDate)
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .compact();

        // redis에 저장
        redisTemplate.opsForValue().set(
                authentication.getName(),
                refreshToken,
                refreshExpirationTime,
                TimeUnit.MILLISECONDS
        );

        return refreshToken;
    }

    /**
     * 토큰으로부터 클레임을 만들고, 이를 통해 User 객체 생성해 Authentication 객체 반환
     */
    public Authentication getAuthentication(String token) {
        String userPrincipal = Jwts.parser().
                setSigningKey(secretKey)
                .parseClaimsJws(token)
                .getBody().getSubject();
        UserDetails userDetails = userDetailsService.loadUserByUsername(userPrincipal);

        return new UsernamePasswordAuthenticationToken(userDetails, "", userDetails.getAuthorities());
    }

    /**
     * http 헤더로부터 bearer 토큰을 가져옴.
     */
    public String resolveToken(HttpServletRequest req) {
        String bearerToken = req.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    /**
     * Access 토큰을 검증
     */
    public boolean validateToken(String token){
        try{
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
            return true;
        } catch(ExpiredJwtException e) {
            log.error(EXPIRED_JWT.getMessage());
            throw new BaseException(EXPIRED_JWT);
        } catch(JwtException e) {
            log.error(INVALID_JWT.getMessage());
            throw new BaseException(INVALID_JWT);
        }
    }
}
```

- `createAccessToken` , `createRefreshToken`
    - 유저 정보를 넘겨받아 토큰 생성
    - 넘겨받은 authentication의 `getName()` 메소드를 통해 `username` 가져옴 (`username` : User의 num 필드로 설정함)
    - 각각 expiration time 설정
- `getAuthentication`
    - 토큰을 복호화해 토큰에 들어있는 유저 정보 꺼냄
    - 이후 authentication 객체 반환
- `resolveToken`
    - http 헤더로부터 bearer 토큰 가져옴
- `validateToken`
    - 토큰 정보 검증
    - Jwts 모듈이 각각 상황에 맞는 exception 던져줌


<br>

### JwtFilter

```java
/**
 * 헤더(Authorization)에 있는 토큰을 꺼내 이상이 없는 경우 SecurityContext에 저장
 * Request 이전에 작동
 */

public class JwtFilter extends OncePerRequestFilter {
    private final JwtTokenProvider jwtTokenProvider;

    public JwtFilter(JwtTokenProvider jwtTokenProvider) {
        this.jwtTokenProvider = jwtTokenProvider;
    }

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        String token = jwtTokenProvider.resolveToken(request);
        try {
            if (token != null && jwtTokenProvider.validateToken(token)) {
                Authentication auth = jwtTokenProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(auth); // 정상 토큰이면 SecurityContext에 저장
            }
        } catch (RedisConnectionFailureException e) {
            SecurityContextHolder.clearContext();
            throw new BaseException(REDIS_ERROR);
        } catch (Exception e) {
            throw new BaseException(INVALID_JWT);
        }

        filterChain.doFilter(request, response);
    }
}
```

- `OncePerRequestFilter` 인터페이스 구현
- `doFilterInternal` 함수 오버라이드
    - 필터링 로직 수행
    - request header에서 token을 꺼내고 유효성 검사 후 유저 정보를 꺼내 `Security Context` 에 저장
    - `SecurityConfig` 에 인증을 설정한 API에 대한 request 요청은 모두 이 필터를 거치기 때문에 토큰 정보가 없거나 유효하지 않은 경우 정상적으로 수행되지 않음


<br>

### JwtSecurityConfig

```java
/**
 * JwtTokenProvider과 JwtFilter를 SecurityConfig에 적용
 */
@RequiredArgsConstructor
public class JwtSecurityConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public void configure(HttpSecurity http) throws Exception {
        JwtFilter customFilter = new JwtFilter(jwtTokenProvider);
        http.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
    }

}
```

- jwtTokenProvider 주입받음
- JwtFilter를 Spring Security Filter Chain에 추가

<br>

### SecurityConfig

```java
@Configuration
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {

    private final JwtTokenProvider jwtTokenProvider;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtAccessDeniedHandler jwtAccessDeniedHandler;

    @Bean
    public PasswordEncoder getPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authenticationConfiguration
    ) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .httpBasic().disable()  // 비인증시 login form redirect X (rest api)
                .csrf().disable()       // crsf 보안 X (rest api)
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) // jwt token으로 인증 > 세션 필요없음

                .and()
                .authorizeRequests()    // 다음 리퀘스트에 대한 사용권한 체크
//                .requestMatchers("/**").permitAll() // 모든 주소 허용
                .requestMatchers("/api/users/login", "/api/users/signup").permitAll() // 허용된 주소
                .anyRequest().authenticated() // Authentication 필요한 주소

                .and()                  // exception handling for jwt
                .exceptionHandling()
                .accessDeniedHandler(jwtAccessDeniedHandler)
                .authenticationEntryPoint(jwtAuthenticationEntryPoint);

        // jwt 적용
        http.apply(new JwtSecurityConfig(jwtTokenProvider));
        return http.build();
    }
}
```

<br>

### JwtAccessDeniedHandler

```java
package com.sejongmate.config.security;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * 유저 정보는 있으나 자원에 접근할 수 있는 권한이 없는 경우 : SC_FORBIDDEN (403) 응답
 */
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        // 필요한 권한이 없이 접근하려 할때 403
        response.sendError(HttpServletResponse.SC_FORBIDDEN);
    }
}
```

유저 정보는 있으나 자원에 접근할 수 있는 권한이 없는 경우 : SC_FORBIDDEN (403) 응답

<br>

### JwtAuthenticationEntryPoint

```java
package com.sejongmate.config.security;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * 유저 정보 없이 접근한 경우 : SC_UNAUTHORIZED (401) 응답
 */
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        // 유효한 자격증명을 제공하지 않고 접근하려 할때 401
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
    }
}
```

유저 정보 없이 접근한 경우 : SC_UNAUTHORIZED (401) 응답

<br><br>

## 📍 인증 객체 생성

### UserDetailServiceImpl

```java
@Service
@Log4j2
public class UserDetailsServiceImpl implements UserDetailsService {
    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String num) throws BaseException {
        User user = userRepository.findByNum(num)
                .orElseThrow(() -> {
                    log.error(INVALID_USER_NUM.getMessage());
                    return new BaseException(INVALID_USER_NUM);
                });

        Set<GrantedAuthority> grantedAuthorities = new HashSet<>();
        return new org
                .springframework
                .security
                .core
                .userdetails
                .User(user.getNum(), user.getPassword(), grantedAuthorities);
    }
}
```

- `UserDetailsService` 인터페이스를 구현한 클래스
- `loadUserByUsername` 메소드를 오버라이드 : 넘겨받은 `UserDetails` 와 `Authentication` 의 패스워드를 비교하고 검증하는 로직을 처리
- 유저에 대한 검증이 완료되면 Authentication 객체 리턴

<br><br>

## 📍 로그인 관련 Service 구현

```java
@Transactional
public TokenDto login(UserLoginReqDto userLoginReqDto) throws BaseException {
    try {
        Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        userLoginReqDto.getNum(),
                        userLoginReqDto.getPassword()
                )
        );

        TokenDto tokenDto = new TokenDto(
                jwtTokenProvider.createAccessToken(authentication),
                jwtTokenProvider.createRefreshToken(authentication)
        );

        return tokenDto;

    }catch(BadCredentialsException e){
        log.error(INVALID_USER_PW.getMessage());
        throw new BaseException(INVALID_USER_PW);
    }
}
```

- 로그인 성공 시 Token 반환
- authenticationManager 통해 id, password 검증
    - `authenticate()` 메소드 실행 시 `UserDetailsServiceImpl` 에서 만든 `loadUserByUsername` 메소드 실행됨
    - 검증 실패 시 exception 발생
    - 검증 성공 시 토큰 생성 (access/refresh token)



<br><br>

## 📍 로그인 API 작동 확인
<img width="942" alt="스크린샷 2023-05-06 오전 2 06 01" src="https://user-images.githubusercontent.com/62213813/236522053-b3b7754e-8335-46da-b9e9-933614cb4af0.png">




<br><br><br><br><br><br><br>


참고 자료

[https://hou27.tistory.com/entry/Spring-Security-JWT](https://hou27.tistory.com/entry/Spring-Security-JWT)

[https://bcp0109.tistory.com/301](https://bcp0109.tistory.com/301)

[https://do5do.tistory.com/14](https://do5do.tistory.com/14)