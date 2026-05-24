---
title: JSON Web Token(JWT)
date: 2026-05-11 22:00
categories:
- Spring-Security
tags:
- Java
- Spring
- Jwt
- CSR
---

> "서버가 상태를 저장하지 말고, 클라이언트가 인증 정보를 들고 다닌다."

---

## 목표

> Jwt와 그와 관련된 내용에 대해서 알아봅니다.

---

## 간략한 사전 지식

#### 쿠키

- HTTP는 기본적으로 무상태(**stateless**) 프로토콜이다.
- 이를 해결하기 위해 서버가 브라우저에 저장시키는 작은 텍스트 데이터이다.
- 브라우저는 자동으로 쿠키를 붙여서 서버에 보낸다.
- 쿠키 안에는 만료 시간, **XSS**(HttpOnly), **CSRF**(SameSite) 등이 있다.

#### 세션

- 쿠키는 데이터를 브라우저에 저장하지만, 세션은 서버에 저장한다.
- 세션을 생성 후, db에 데이터를 저장하고 쿠키에 sessionId를 저장한다.(`Set-Cookie: sessionId=abc123`)
- 서버에 저장되기에 분산 환경에서 구조가 복잡해지고 서버에 부담이 크기에 Jwt가 등장했다.

##### 보통, 쿠키와 세션을 같이 사용하고 쿠키에는 `sessionId`만 세션에는 유저 정보를 저장한다.

---

#### JWT 구조

```text
{Header}.{Payload}.{Signature}
```
- 헤더는 암호화 알고리즘을 명시한다.
- 페이로드는 유저 정보를 명시한다.(유저이름, 권한, 만료시간 정도)
- 헤더와 페이로드는 누구나 디코딩 가능해 **민감한 정보는 절대 담지 않는다**.
- 시그니처에는 헤더와 페이로드와 비밀키를 이용해 **HMAC** 결과값이 들어간다.
- 결국, **비밀키는 서버에만 있으며** 비밀키를 알지 못하면 해킹할 수 없다.

---

#### CSRF

- **브라우저가 쿠키를 자동으로 전송하는 특성**을 이용한 해킹 공격이다.
- Jwt를 쓰면 쿠키가 아닌 `Authorization` 헤더로 전송하기에 비활성화한다.
- 다만, Jwt를 쿠키에 저장하는 방식을 쓴다면 비활성화 하면 안된다.
- 보안은 항상 **트레이드 오프**이기에 각 상황에 맞게 사용하면 된다.

#### CORS

- **다른 출처(Origin)의 요청을 브라우저가 차단**하는 보안 정책이다.
- Jwt를 헤더로 보낼 때 브라우저는 먼저 `OPTIONS` 메서드로 사전 요청(Preflight)를 보낸다.
- 이 때, CORS 설정이 없으면 사전 요청이 막혀 요청 자체가 되지 않는다.

---

## 구현

#### 순서

1. JwtTokenProvider
2. JwtAuthentificationFilter
3. SecurityConfig
4. UserController

##### JwtTokenProvider

```java
    // 토큰 생성 (로그인 성공 시 호출)
    public String generateToken(Long userId) {
        Date now = new Date();
        return Jwts.builder()
                .subject(userId.toString())
                .issuedAt(now)
                .expiration(new Date(now.getTime() + expiration))
                .signWith(key)
                .compact();
    }

    // 토큰에서 userId 추출
    public Long getUserId(String token) {
        String subject = Jwts.parser()
                .verifyWith(key).build()
                .parseSignedClaims(token)
                .getPayload().getSubject();
        return Long.parseLong(subject);
    }

    // 토큰 유효성 검사
    public boolean validate(String token) {
        try {
            Jwts.parser().verifyWith(key).build().parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
```

- 로그인 시 클라이언트에게 `accessToken`을 전송하기 위해 **토큰을 생성하고 검증하는 역할**을 한다.
- 해당 클래스에서 `secretKey`와 `expiration`을 설정한다.
- `UserDetailService`를 통해 db 조회를 함으로써 권한을 관리할 수 있고, OAuth2를 적용할 수 있다.

##### JwtAuthentificationFilter

```java
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String token = resolveToken(request);

        if (token != null && jwtTokenProvider.validate(token)) {
            Long userId = jwtTokenProvider.getUserId(token);
            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(userId, null, List.of());
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);  // 토큰 없어도 통과 (막는 건 SecurityConfig)
    }

    private String resolveToken(HttpServletRequest request) {
        String bearer = request.getHeader("Authorization");
        if (StringUtils.hasText(bearer) && bearer.startsWith("Bearer ")) {
            return bearer.substring(7);
        }
        return null;
    }
```

- `OncePerRequestFilter`를 상속해 **요청당 정확히 한 번만** 실행되게 한다.
- 필터는 인증 정보 세팅만 담당하고, 차단 결정은 `SecurityConfig`에 맡긴다.
- `SecurityContext`에 인증된 사용자의 정보를 저장한다.

##### SecurityConfig

```java
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/users/login", "/api/users/signup").permitAll()
                .anyRequest().authenticated()
            )
            // JWT 필터를 Spring Security 기본 인증 필터 앞에 등록
            .addFilterBefore(
                new JwtAuthenticationFilter(jwtTokenProvider),
                UsernamePasswordAuthenticationFilter.class
            );

        return http.build();
    }
```

- **세션을 사용하지 않게** 설정한다.
- Jwt 필터를 Spring Security 기본 필터 앞에 둔다.
- `authenticated()`는 `SecurityContext`에 비어있으면 403을 반환한다.
  
##### 로그인에서 토큰 발급

- 이렇게 하면 `UsernamePasswordAuthenticationFilter`를 커스텀하는 방식보다 단순하다.
- 다만, 현재 사이드 프로젝트의 핵심 기능을 빨리 구현하기 위해 커스텀하지 않았으나 이후 고도화는 고민 중에 있다.