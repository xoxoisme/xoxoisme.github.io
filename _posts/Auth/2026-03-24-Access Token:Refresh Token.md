---
title: Access Token/Refresh Token
date: 2026-03-24 21:00
categories:
- Auth
tags:
- Auth
- JWT
---

> "**JWT는 발금한 후, 삭제가 불가능하기에 유효기간을 따로 부여하여 사용자의 접근성을 편하게 했다.**"

---

## 목표

> "Acess Token과 Refresh Token의 각각 역할에 대해 알아봅니다."

---

## Access Token
- 해당 토큰만 사용하는 경우, 제 3자에게 토큰을 탈취당하면 보안에 취약하다.
- 발급이 되면 서버에 저장되지 않고 **토큰 자체로 검증하며 사용자 권한을 인증**한다.
- 이로 인해, 탈취되면 토큰을 획득한 누구나 권한 접근이 가능해지는 문제가 발생한다.

## Refresh Token
- 그래서, **Access Token의 유효기간은 짧게 Refresh Token의 유효기간은 보다 길게 설정**한다.
- **Acess Token은 접근**에 관여하는 토큰, **Refresh Token은 재발급**에 관여하는 토큰이다.

## 원리
- 사용자가 처음 로그인하게 되면, 서버는 클라이언트에게 Acess Toekn과 Refresh Token을 동시에 발급한다.
- 서버는 DB에 Refresh Token을 저장하고, 클라이언트는 Acess Token과 Refresh Token을 쿠키, 세션 혹은 웹스토리지에 저장하고 요청이 있을 때마다 둘을 헤더에 담아 보낸다.
- Acess Token이 만료되면 DB에 저장된 Refresh Token과 헤더에서 온 Refresh Token을 비교해 일치하면 Acess Token을 재발급한다.
- 사용자가 로그아웃 하면 DB에 저장된 Refresh Token을 삭제한다.

---

## 코드로 본다면

### 토큰 생성
```java
    public String createAccessToken(String subjectEmail, String role) {
        Instant now = Instant.now();
        Instant exp = now.plusSeconds(accessTokenTtlSeconds);
        return Jwts.builder()
                .issuer(issuer)                     // 누가 발급했는지
                .subject(subjectEmail)              // 토큰의 주인
                .issuedAt(Date.from(now))           // 발급 시각
                .expiration(Date.from(exp))         // 만료 시각
                .claims(Map.of(
                        "role", role,
                        "tokenType", "access"
                ))                                 // 추가 정보(권한, 타입)
                .signWith(key)                      // 비밀키
                .compact();                         // JWT 문자열 생성
    }

    public String createRefreshToken(String subjectEmail) {
        Instant now = Instant.now();
        Instant exp = now.plusSeconds(refreshTokenTtlSeconds);
        return Jwts.builder()
                .issuer(issuer)
                .subject(subjectEmail)
                .issuedAt(Date.from(now))
                .expiration(Date.from(exp))
                .claims(Map.of(
                        "tokenType", "refresh",
                        "jti", UUID.randomUUID().toString()
                ))
                .signWith(key)
                .compact();
    }
```
- 위처럼, `tokenType`을 두어 Access/Refresh를 구분한다.

## 재발급
```java
    @PostMapping("/refresh")
    public TokenResponse refresh(HttpServletRequest request, HttpServletResponse response) {
        String refreshToken = extractRefreshToken(request); // DB에 저장된 refreshToken을 가져온다.
        if (refreshToken == null || refreshToken.isBlank()) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "refresh token not found");
        }

        try {
            // 해당 refreshToken의 내용을 확인 후, acessToken이 만료된 회원을 찾는다.
            // DB에서 가져온 refreshToken과 일치한지 확인한다.
            // DB 기준 만료시간을 확인한다(이중 검증)

            String newAccessToken = jwtService.createAccessToken(member.getEmail(), member.getType().name());
            String newRefreshToken = jwtService.createRefreshToken(member.getEmail());  // rotation 방식이다.(탈취 피해 줄임)

            member.updateRefreshToken(newRefreshToken, LocalDateTime.now().plusSeconds(jwtService.getRefreshTokenTtlSeconds()));
            memberRepository.save(member);
            addRefreshCookie(response, newRefreshToken);

            return TokenResponse.bearer(newAccessToken);
        } catch (JwtException | IllegalArgumentException ex) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED, "invalid refresh token");
        }
   }
```

## 로그아웃
```java
    @PostMapping("/logout")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void logout(HttpServletRequest request, HttpServletResponse response) {
        String refreshToken = extractRefreshToken(request);
        if (refreshToken != null && !refreshToken.isBlank()) {
            memberRepository.findByRefreshToken(refreshToken)
                    .ifPresent(member -> {
                        member.clearRefreshToken(); // refreshToken 삭제
                        memberRepository.save(member);
                    });
        }
        expireRefreshCookie(response); // refreshToken을 maxAge = 0으로하여 브라우저의 쿠키도 만료시킨다.
    }
```

---

## Reference
- https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-Access-Token-Refresh-Token-%EC%9B%90%EB%A6%AC-feat-JWT