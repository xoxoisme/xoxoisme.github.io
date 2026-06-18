---
title: 토스 승인/내부 DB 저장 실패
date: 2026-06-18 23:00
category: Trouble_Shooting
tags:
- Toss
- Transaction
- PG
- Buzzer Bidder
---

> "외부 API 연동 시, 내부 DB 데이터 정합성에 신경써야 한다."

---

## 목표

> TossPayments의 결제 흐름에 대해 알아봅니다.

---

## 문제 상황

#### TossPayments는 결제 인증과 승인이 분리되어 있다.

<img src="/images/2026-06-18-토스 승인 내부 DB 저장 실패/스크린샷 2026-01-13 오후 9.33.43.png">

- 토스페이먼츠는 데이터 정합성과 연동 편의를 위해서 요청과 승인을 따로 하는 방식으로 만들었다.
- 결제 인증과 승인이 비동기적으로 일어나기에 한 번에 결제의 결과를 받으려면 반드시 **웹훅을 연동**해야 한다.
    - `고객 브라우저(고객의 폰/PC)`: 카드 정보 넣고 본인인증 하는 곳
    - `상점 서버(백엔드)`: 주문 정보를 갖고 있고, 돈을 실제로 빠지게 할지 결정하는 곳
    - `토스 서버`: 실제로 카드사와 통신해서 결제를 처리하는 곳
    - 인증과 승인을 한 번에 묶으면 상점 서버는 인증 과정에 끼지 않기에 결과를 기다릴 수 밖에 없다.
    - 그래서, 웹훅을 이용해서 결과를 받아야하지만 이 정합성을 보장하기에 여러 작업이 필요하다.
- 그래서, **결제 요청과 승인을 따로 하는 방식**으로 데이터 정합성을 보장하고, 상점에서 직접 해야 할 일을 줄이고자 했다.

```text
1) [FE] POST /payments          → 백엔드가 orderId 발급 (Payment PENDING)
2) [FE] 토스 결제창 오픈 (orderId, amount 사용)
3) [사용자] 결제창에서 카드 인증/결제
4) [토스] successUrl로 리다이렉트 → paymentKey, orderId, amount 전달
5) [FE] 위 3개 값으로 POST /payments/confirm  ← 여기가 "승인 요청"
6) [백엔드] 토스 /v1/payments/confirm 호출 → status=DONE → 지갑 충전
```

- 즉, 승인(confirm) 시점에 2가지 **외부/내부 작업이 연달아서** 발생한다.
    1. 토스 서버에 승인 요청한다.(성공하면 실제로 사용자 돈이 빠져나감)
    2. 내부 DB에서 지갑에 결제한 금액만큼 충전된다.

#### 초기 코드의 문제

```java
@Service
@RequiredArgsConstructor
@Transactional                       // ← 클래스 전체가 하나의 트랜잭션
public class PaymentService {

    private final WalletRepository walletRepository;
    private final WalletService walletService;

    public PaymentConfirmResponseDto confirm(Long userId, PaymentConfirmRequestDto request) {
        Payment payment = paymentRepository.findByOrderId(orderId)...;

        if (!payment.isPending()) throw ...;          // 중복 승인 방지
        if (!payment.getAmount().equals(amount)) ...; // 금액 검증

        TossConfirmResponseDto response =
                tossPaymentsClient.confirmPayment(request);   // ★ 외부 호출이 DB 트랜잭션 안에 있음
        if (!response.status().equals("DONE")) throw ...;

        payment.confirm(...);
        paymentRepository.save(payment);

        User user = userRepository.findById(userId)...;
        walletService.chargeBizz(user, amount);       // ★ 여기서 터지면?
        return ...;
    }
}
```

- 위 구조는 2가지 위험이 존재한다.
    1. 외부 호출이 DB 트랜잭션 안에 있다.
        1. 토스 승인 호출(네트워크 지연 수백 ms~수 초)이 **DB 트랜잭션 안**에서 일어나므로, 커넥션과 row 락을 그 시간 내내 점유한다.
        2. 동시 결제가 몰리면 **커넥션 pool 고갈**로 이어질 수 있다.
    2. 보상 로직이 없다.
        1. 토스 승인은 성공했는데 DB에 충전하는 로직이 실패(DB 장애, 락 타임아웃, 지갑 로직 예외)하면 트랜잭션은 롤백된다.
        2. **토스를 통해 빠져나간 돈이 충전이 안되는 문제**가 발생한다.

→ 즉, 사용자는 결제를 했음에도 불구하고 충전이 안되는 문제가 발생할 수 있다. 자동 복구 수단 또한 없다.

---

## 해결 과정

#### 트랜잭션 분리 및 보상 트랜잭션 도입

```java
@Transactional
public void markLocked(String orderId, Long amount) {
    Payment payment = paymentRepository.findByOrderIdForLock(orderId)...; // 비관적 락
    if (!payment.isPending()) throw new BusinessException(NOT_PENDING_PAYMENT); // 중복 방지
    if (!payment.getAmount().equals(amount)) throw ...(INVALID_AMOUNT);          // 금액 위변조 방지
    payment.markLocked();   // PENDING → LOCKED
}

@Transactional
public Payment confirmAndChargeWallet(...) {
    Payment payment = paymentRepository.findByOrderIdForLock(orderId)...;
    if (!payment.isLocked()) throw ...;
    payment.confirm(paymentKey, PaymentMethod.fromToss(response.method()), response.approvedAt());
    walletService.chargeBizz(user, amount);
    return payment;
}
```

- 위처럼, DB 작업 단위를 **별도 서비스의 짧은 단위로 분리**했다.

```java
public PaymentConfirmResponseDto confirm(Long userId, PaymentConfirmRequestDto request) {
    paymentTransactionService.markLocked(orderId, amount);          // ① 짧은 TX: 락 + 검증

    TossConfirmResponseDto response =
            tossPaymentsClient.confirmPayment(request);            // ② 외부 호출 (TX 밖!)
    if (!response.status().equals("DONE")) throw ...;

    try {
        Payment payment = paymentTransactionService
                .confirmAndChargeWallet(...);                      // ③ 짧은 TX: 충전
        return PaymentConfirmResponseDto.from(payment);
    } catch (Exception e) {                                        // ④ 충전 실패 = 보상 시작
        PaymentCancelRequestDto cancelRequest =
                new PaymentCancelRequestDto("결제 승인 후 내부 처리 실패로 인한 취소");
        try {
            tossPaymentsClient.cancelPayment(paymentKey, cancelRequest);   // 토스에 환불 요청
            paymentTransactionService.markCanceled(orderId, "INTERNAL_FAIL", ...);
        } catch (Exception ex) {
            paymentTransactionService.markCancelFailed(orderId, "CANCEL_FAIL", "결제 취소 실패");
        }
        throw new BusinessException(ErrorCode.INTERNAL_PAYMENT_ERROR);
    }
}
```

- `confirm()` 을 외부 API 호출이 섞여있기에 **오케스트레이터**로 활용하고, 보상 트랜잭션을 실행한다.
- 다만, 여기서도 한계점은 있다.
- 바로 **토스 취소하는 요청 자체가 실패**했을 경우(토스 서버 장애)이다.

#### 결제 취소 재시도 스케줄러

1. `CANCEL_PENDING` → `CANCEL_FAILED` 로 변경했다.

```java
catch (Exception ex) {
    // 취소 실패 시, CANCEL_PENDING 상태 유지(스케줄러로 재시도)
    paymentTransactionService.markCancelPending(
            paymentKey, orderId, "CANCEL_FAIL", "결제 취소 실패", OffsetDateTime.now());
}
```

1. 10초 주기마다 최대 3회 실행하도록 했다.

```java
@Scheduled(fixedDelay = 10000)   // 10초마다
@Transactional
public void retryCancel() {
    List<Payment> payments =
            paymentRepository.findDueCancelPending(OffsetDateTime.now(), PageRequest.of(0, 50));
    for (Payment payment : payments) retry(payment.getOrderId());
}

private void retry(String orderId) {
    paymentRepository.findByOrderIdForLock(orderId).ifPresent(payment -> {
        if (payment.getStatus() != CANCEL_PENDING) return;
        if (payment.getNextCancelRetryAt().isAfter(now())) return;   // 백오프 대기
        if (payment.getCancelRetryCount() >= 4) {
            payment.failAfterThreeRetry(...);   // 한도 초과 → CANCEL_FAILED (수동개입 격리)
            return;
        }
        try {
            tossPaymentsClient.cancelPayment(payment.getPaymentKey(), request);
            payment.canceled(...);              // 성공 → CANCELED
        } catch (Exception ex) {
            // 실패 → 카운트 +1, 다음 재시도 시각 미루고 CANCEL_PENDING 유지
            payment.increaseCancelRetryCount(nextRetryAt);
            payment.cancelPending(...);
        }
    });
}

private long delaySeconds(int n) {  // 지수 백오프
    return switch (n) { case 1 -> 30; case 2 -> 120; case 3 -> 300; default -> 120; };
}
```

- 백오프 재시도로도 결제 취소가 이루어지지 않을 경우에는 `CANCEL_FAILED` 로 상태가 남게 된다.
- 이 경우에는 사람이 직접 수동으로 개입해야 한다.

---

## 결과

#### 최종 플로우차트

<img src="/images/2026-06-18-토스 승인 내부 DB 저장 실패/스크린샷 2026-06-18 오후 10.52.18.png">

- 토스에서 결제된 돈은 **반드시 충전되거나 환불되도록** 최종 일관성을 확보했다.
- 외부 호출이 DB 트랜잭션 밖으로 빠져 커넥션/락 점유 시간을 단축했다.
- 일시적 토스 장애는 스케줄러가 **백오프 재시도**로 방지했다.