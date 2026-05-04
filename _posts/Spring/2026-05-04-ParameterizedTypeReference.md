---
title: ParameterizedTypeReference
date: 2026-05-04-13:00
categories:
- Spring
tags:
- Spring
---

> "**ParameterizedTypeReference는 제네릭 타입을 갖는 객체의 타입 정보를 보존하기 위한 목적으로 사용된다.**"

---

## 목표

> 제네릭 타입의 특징에 대해서 알아봅니다.

---

## 타입 소거

- 먼저 제네릭은 **컴파일 타임에만 존재하고, 런타임에는 타입 정보가 사라진다**.
- 이를 타입 소거라고 한다.

### 컴파일

```java
List<String> list = new ArrayList<>();
list.add(123); // 컴파일 에러
```

- 직접 값을 넣으면 걸리지만, **런타임 시 외부에서 오는 데이터는 확인하지 못한다**.
- 즉, Jackson이 역직렬화 할 때 타입 정보를 전달할 수 없다는 뜻이다.

### 런타임

```java
// 작성한 코드
List<String> list = new ArrayList<>();

// 컴파일 후 바이트코드
List list = new ArrayList();
```

- 이 때는 `List<String>` 과 `List<Integer>`는 JVM 입장에서 동일한 `List`로 인식된다.

---

## ParameterizedTypeReference

- 일반 제네릭 클래스를 익명 클래스로 상속하면, 서브 클래스의 바이트코드에 부모 타입 정보가 남는다.

### 사용 안할 경우

```java
// Map으로 받으면 매번 직접 캐스팅
Map<String, Object> userMap = (Map<String, Object>) list.get(0);
String name = (String) userMap.get("name");

// ParameterizedTypeReference 사용
List<User> users = response.getBody();
String name = users.get(0).getName(); // 훨씬 깔끔
```

- Jackson이 타입 정보 없이 JSON을 파싱하면 자동으로 `LinkedHashMap`으로 만든다.
- 그래서, 익명 클래스로 현재 자기 자신의 자식을 즉석으로 만들어 부모 타입의 정보를 런타임시에도 남게 만드는 것이다.

### 사용 예시

```java
    public TmdbPageResponse<TmdbTvResult> searchTvShows(String query, int page) {
        return restClient.get()
                .uri("/search/tv?query={query}&page={page}&language=ko-KR", query, page)
                .retrieve()
                .body(new ParameterizedTypeReference<>() {});
    }
```

- `ParameterizedTypeReference<>()`를 사용함으로써 `TmdbPageResponse<TmdbTvResult>`로 **역직렬화가 가능**하게 된다.
- 사용 안하고 캐스팅을 하게 되면 매번 **직접 캐스팅을 해주어야 하는 불편함**이 생긴다.
- 유지보수와 타입 안정성이 최악이 된다는 말이다.