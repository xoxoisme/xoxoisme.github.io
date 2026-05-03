---
title: HTTP client 비교
date: 2026-05-03-15:00
tags:
- Java
categories:
- Java
- Http
---

> "**HTTP client는 기술의 발전, 요구사항의 다양화, 환경마다 철학이 달라졌기에 그에 맞게 진화했다.**"

---

## 목표

> HTTP client의 종류와 개념에 대해 알아봅니다.

---

## HTTP client

- 다른 서버로 HTTP 요청을 보내고, 응답을 받아오는 라이브러리이다.
- 보통 REST(GET/POST/PUT) 요청을 구성한다.
- 응답 코드 및 body와 같은 응답을 수신한다.

---

## 종류

### RestTemplate

- **동기** 방식을 사용한다.
- **블로킹** 방식이라 트래픽이 몰리면 서버가 멈출 수 있다.


```java
@RestController
public class ApiController {

    private final RestTemplate restTemplate = new RestTemplate();

    @GetMapping("/call-api")
    public String callApi() {
        String url = "https://api.example.com/data";
        ResponseEntity<String> response = restTemplate.getForEntity(url, String.class); // 위 url로 GET 요청
        return response.getBody();
    }
}
```

### WebClient

- **비동기** 방식이며 **논블로킹을** 지원한다.(동기 방식도 사용가능하다.)
- 내부적으로 Nett의 이벤트 루프를 사용한다.
- 리액티브 프로그래밍(데이터가 오면 그 때 처리하는 방식) 패러다임을 알아야 해서 코드 구현과 디버깅이 어렵다.
- 성능 면에서는 최강이다.
- 보통 `.block()`으로 응답을 기다린 후 처리하는 데 이는 Spring 공식 문서가 RestTemplate이 유지보수 모드가 되어서 권고하는 추세라 그렇다.
  - 그리고 이는 동기 + 블로킹이지만 마지막 `.block()` 이전에는 비동기 + 논블로킹으로 진행되기에 병렬로 처리된다는 장점이 있다.

```java
webClient.get()              // Netty 이벤트 루프에 등록 (논블로킹)
  .retrieve()                // 요청 전송 (논블로킹)
  .bodyToMono(String.class)  // 응답 대기 (Netty가 비동기 처리)
  .block()                   // ← 여기서 내 스레드가 멈춤 (블로킹!)
```

### OpenFeign
- 구현 코드를 짜지 않고, 인터페이스에 어노테이션만 붙이면 알아서 구현체를 만들어준다.
- **블로킹** 방식이고 Spring Clound에 대한 **의존성이 무겁다**.(MSA 인프라 패키지까지 가져오기 때문이다.)

```java
// 1. 인터페이스 정의
@FeignClient(name = "order-client", url = "https://api.orders.com")
public interface OrderClient {

    @GetMapping("/orders/{orderId}")
    OrderDto getOrder(@PathVariable("orderId") String orderId);

    @PostMapping("/orders")
    void createOrder(@RequestBody OrderRequest request);
}

// 2. 서비스에서 사용
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderClient orderClient; // 자동으로 주입

    public void processOrder(String orderId) {
        OrderDto order = orderClient.getOrder(orderId); // 일반 메소드 호출하듯이 사용
        System.out.println("주문 정보: " + order);
    }
}
```

### RestClient

- **동기 + 블로킹** 방식이다.
- 다만, Java 21 가상 스레드와 함께 쓰면, 복잡한 비동기 코드 없이도 엄청난 성능이 나온다.
  - **JVM**이 자동으로 I/O 대기가 발생하면 가상 스레드가 하던 걸 멈추고 다른 작업을 할 수 있게 한다.
- WebClient와 동일한 함수형 API를 제공하나, 리액티브 타입이 아닌 일반 자바 객체를 반환한다.

---

## References

- https://twofootdog.tistory.com/349