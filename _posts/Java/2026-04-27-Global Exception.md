---
title: Global Exception
date: 2026-04-27 16:00
tags:
- Java
categories:
- Java
---

> "**전역 예외처리는 애플리케이션에서 발생하는 예외를 한 곳에서 모아 일관된 형식으로 처리해준다.**"

---

## 목표

> 전역 예외처리의 구조와 흐름을 알아봅니다.

---

## 관련 어노테이션

### `@ControllerAdvice`

- 모든 컨트롤러의 예외를 가로채 처리하는 클래스임을 명시한다.
- 전역 예외 처리 / 바인딩 / 모델 속성 추가 기능을 한다.
- 기본적으로 **뷰(View) 기반의 MVC 처리**에 적합하다.


### `@RestControllerAdvice`

- 예외 처리 결과를 JSON 또는 XML로 반환할 때 사용한다.
- 기능은 위와 동일하게 제공한다.
- 다만, 응답이 `@ResponseBody`로 처리되어 REST API에 적합하다.
- 프론트와 JSON 기반 통신할 때 용이하다.

### `@ExceptionHandler`

- 특정 예외가 발생했을 때 실행할 메서드를 지정한다.
- ResponseEntity, DTO, 문자열 등 다양한 형식으로 반환 가능하다.

---

## 활용 예시

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 사용자 예외 처리
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        ErrorCode errorCode = e.getErrorCode();
        return ResponseEntity
                .status(errorCode.getStatus())
                .body(ErrorResponse.of(errorCode));
    }

    // 수학 관련 검증 예외 처리
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException e) {
        return ResponseEntity
                .status(ErrorCode.INVALID_INPUT_VALUE.getStatus())
                .body(ErrorResponse.of(ErrorCode.INVALID_INPUT_VALUE));
    }

    // 이외의 모든 예외 처리
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        return ResponseEntity
                .status(ErrorCode.INTERNAL_SERVER_ERROR.getStatus())
                .body(ErrorResponse.of(ErrorCode.INTERNAL_SERVER_ERROR));
    }
}
```

### `BusinessException` 상속 구조

```java
Object
 └─ Throwable
     └─ Exception
         └─ RuntimeException
             └─ BusinessException
```

- 그렇기에 `super()`를 사용하여 에러 메세지를 저장해둔다.

```java
@Getter
@RequiredArgsConstructor
public enum ErrorCode {

    // ========== Common ==========
    INVALID_INPUT_VALUE("C001", HttpStatus.BAD_REQUEST, "잘못된 입력값입니다."),
    RESOURCE_NOT_FOUND("C002", HttpStatus.NOT_FOUND, "요청한 리소스를 찾을 수 없습니다."),
    INTERNAL_SERVER_ERROR("C003", HttpStatus.INTERNAL_SERVER_ERROR, "서버 내부 오류가 발생했습니다.");

    private final String code;
    private final HttpStatus status;
    private final String message;
}
```

- 이후 `final`을 활용해 불변 객체로 설계하여 이후 수정을 방지한다.

---

## References

- https://adjh54.tistory.com/79