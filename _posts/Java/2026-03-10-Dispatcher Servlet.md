---
title: Dispatcher Servlet
date: 2026-03-10 02:00
tags:
- Java
categories:
- Java
---

> "**서블릿은 Java를 사용하여 웹 페이지를 동적으로 생성하는 서버측 프로그램이다. 디스패처 서블릿은 이후 나온 중앙 관리자 역할이다.**"

---

## 목표

> 서블릿의 한계로 인해 디스패처 서블릿이 나타난 이유가 무엇인지 알아봅니다.

---

## Servlet

- **동적 웹 페이지를 만들 때** 사용되는 자바 기반의 웹 애플리케이션 프로그래밍 기술이다.
- 개발자가 요청과 응답을 일일이 처리하는 게 아니라 비즈니스 로직에 집중할 수 있게 해준다.
- 웹 요청과 응답을 **간단한 메서드 호출로 다룰 수 있게 해주는 기술**이다.

### 주요 특징

- HTML을 사용하여 응답한다.
- Java의 **스레드를 이용**하여 동작한다.
- MVC 패턴에서 **컨트롤러로 이용**된다.
- `HttpServlet`클래스를 상속받는다.
- HTML 변경 시 Servlet을 **재컴파일** 해야한다는 단점이 있다.

### 동작 흐름

```
Browser
  ↓
HTTP Request
  ↓
Web Server / Servlet Container (Tomcat)
  ↓
Servlet
  ↓
Business Logic
  ↓
HTTP Response
  ↓
Browser
```

#### 여기서 중요한 건 Servlet은 **싱글톤**이라 보통 1개 객체만 생성한다.

- 요청마다 객체를 생성하면 성능과 메모리 비용이 크게 증가하기 때문에 하나의 객체만 생성 후 **여러 스레드가 공유하도록 설계**되었다.
- 또한, 동시성은 Controller가 **stateless**(상태를 가지지 않음) 특징을 가지고 각 스래드 스택에 **지역 변수들이 저장되기에** 충돌하지 않는다.

### 생명 주기

|   메서드    |                    역할                    |
| :---------: | :----------------------------------------: |
|  `init()`   | 최초 1번 실행, 변경 시 `destroy()` 후 실행 |
| `service()` |               요청마다 실행                |
| `destroy()` |             서버 종료 시 실행              |

### Servlet의 한계

- URL마다 Servlet을 직접 매핑해야 한다.
- 공통 로직(로그인 검사, 인증 등)을 매번 구현해야 한다.
- Controller 역할을 하는 Servlet이 많아지면 관리가 어려워진다.

---

## Dispatcher Servlet

> 이러한 문제를 해결하기 위해 등장한 것이 **Front Controller 패턴**이며, Spring MVC에서는 이를 **Dispatcher Servlet**이 담당한다.

### 주요 특징

- 요청에 맞는 Controller를 찾아 실행한다.
- `ViewResolver`를 통해 적절한 View를 반환한다.

### 동작 흐름

```
Browser
↓
HTTP Request
↓
Tomcat (Servlet Container)
↓
DispatcherServlet
↓
HandlerMapping
↓
Controller
↓
Service
↓
Repository
↓
DB
↓
Response
```

- 처리 결과를 `View` 또는 `JSON` 형태로 반환한다.

---

## Reference

- https://iamjooon2.tistory.com/39