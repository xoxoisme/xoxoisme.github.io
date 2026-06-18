---
title: STOMP 분기에서 JWT 인증 실패
date: 2026-06-18 17:16
category: My_Fish
tags:
- WebSocket
- STOMP
- JWT
- Trouble Shooting
---

> "WebSocket 핸드셰이크 요청에 Authorization 헤더를 실을 수 없다."

---

## 목표

> WebSocket 활용 시 인증 분기에 대해서 알아봅니다.

---

## 문제 상황

- 실시간 채팅에 JWT 인증을 붙이는 과정에서, **WebSocket 핸드셰이크 요청에 Authorization 헤더를 실을 수 없는 문제**에 부딪혔다.
- 브라우저 표준 WebSocket API는 생성자에 url과 subprotocol만 받고, 커스텀 헤더를 지정하는 입구가 없다.
- 그래서 일반 REST처럼 `Authorization: Bearer <accessToken>` 을 핸드셰이크에 붙이는 방식이 불가능했다.

```java
  if (StompCommand.CONNECT.equals(accessor.getCommand())) {
      String token = accessor.getFirstNativeHeader("Authorization"); // Bearer 토큰
      ...payloadOrNull(token)...      // 검증
      accessor.getSessionAttributes().put("memberId", memberId);
```

- 이를 우회하기 위해 위처럼 웹소켓은 그냥 연결하고, **STOMP CONNECT 프레임 헤더에 임시 토큰**을 실어 보내 CONNECT 시점에 인증하도록 구현했는데, 이 방식은 다른 문제를 안고 있었다.
    - CONNECT 헤더에 토큰을 넣으려면 JS가 토큰 값을 직접 들고 있어야 한다.→XSS 노출 위험
        - 노출을 줄이기 위해 WebSocket 전용 10분짜리 단기 토큰을 별도 발급하는 임시 방편을 도입했지만, 이는 토큰 발급 API, 토큰 생성 로직, CONNECT 인증 분기 등 불필요한 소켓 자원이 소모되었다.

---

## 해결 과정

#### 원인 분석

- 왜 10분짜리 단기 토큰을 발급해야 했었는지 **흐름을 역추적**했다.
    - STOMP CONNECT에서 인증 → 토큰을 헤더러 보냄 → JS가 토큰을 소유해야 함 → 노출 위험 으로 이어졌다.
    - 결국 인증 지점이 STOMP CONNECT 라서 생긴 문제임을 파악했다.

#### 대안 검토

- 인증을 다른 지점에서 할 수 없는지 찾아보았다.
- 브라우저는 WebSocket **핸드셰이크 요청(HTTP GET)에 쿠키를 자동으로 전송한다는 점**을 알 수 있었다.
- 또한, 현재 서비스는 브라우저와 동일 도메인인 환경이라 쿠키가 정상 전송되는 것을 확인했다.

#### 구조 변경

- 인증 시점을 `STOMP Handler.presend`에서 `WebSocketHandShakeInterceptor` 로 이동했다.
- `beforeHandshake()` 에서 쿠키의 accesToken을 검증하고 실제 회원 존재 확인까지 되면 검증된 memberId를 세션 attribute에 저장했다.
- 검증에 실패하면 HTTP 업그레이드 자체를 거부하도록 예외 처리했다.

#### 불필요 코드 제거

- 단기 토큰 발급 로직, 토큰 발급 API, STOMP CONNECT 인증 분기를 제거했고, 이후 CONNECT는 인증된 이후에만 가능하기에 별도로 인증하지 않았고, SUBSCRIBE/SEND는 세션에 저장된 memberId를 재사용하도록 했다.

#### 인증 분리

- 인증은 핸드셰이크(1회)로 끝냈다.
- SUBSCRIBE는 브로커가 직접 처리할 뿐, 컨트롤러 메서드를 호출하지 않기에 메세지가 브로커에 도달하기 전, 인바운드 채널을 가로채는 StompHandler.preSend에서 memberId로 권한을 검증하도록 했다.
- SEND는 컨트롤러 메서드로 라우팅(DB 저장) 되기에 컨트롤러/서비스에서 권한을 검증하도록 했다.

---

## 결과

- 단기 토큰이라는 우회책 자체가 없어졌기에 불필요했던 로직들이 제거 되었다.
- 인증 정보를 JS가 들고 있지 않게 되어 토큰 노출을 하지 않게 되었다.
- 인증 실패가 핸드셰이크에서 즉시 차단되어, 인증 안 된 소켓이 열렸다가 CONNECT에서 막히는 낭비를 제거했다.
- 일반 REST와 동일한 인증 매커니즘으로 **일관성을 확보**했다.
- 인증(핸드셰이크)/인가(SUBSCRIBE,SEND)의 **책임이 명확히 분리**되었다.

---

## 이전 플로우 차트 비교

<img src="/images/2026-06-18-STOMP 분기에서 JWT 인증 실패/스크린샷 2026-06-18 오후 3.35.37.png">

- 이는 초기의 실시간 채팅 플로우 차트이다.
- 웹소켓 연결 시에 JWT 검증을 하지 않고 STOMP CONNECT 이전에 STOMP Handler가 임시 토큰의 JWT를 검증한다. 이후, 세션에 memberId를 저장한다.
- 이 당시, HttpOnly 로 인해 JS가 쿠키를 알 수 없어 토큰 여부를 확인하지 못했었는데 임시방편으로 웹소켓 연결 시에 **10분 단기용 임시 토큰을 발행**했었다.
- 하지만, 해당 프로젝트는 쿠키를 신뢰할 수 없거나 못 쓰는 환경(크로스 사이트,네이티브/비 브라우저 클라이언트)가 아니었다.
    - 그래서 임시 토큰 발급 API, 별도 검증 로직이 필요없었고, 인증 실패 시 CONNECT까지 가지 않고 핸드셰이크에서 즉시 연결 거부함으로써 불필요한 소켓 자원을 줄일 수 도 있다.

<img src="/images/2026-06-18-STOMP 분기에서 JWT 인증 실패/스크린샷 2026-06-18 오후 4.15.32.png">

- 그래서 위처럼, 핸드셰이크 시에 브라우저가 accessToken을 쿠키에 자동으로 넣어 보냄으로써 JWT 검증을 할 수 있게 되어 빠르게 사전 차단할 수 있게 되었다.
- 이후, 불필요한 소켓 자원을 낭비하는 것을 방지했다.