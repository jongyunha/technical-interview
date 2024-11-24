# 동기(Synchronous)와 비동기(Asynchronous) 처리

## 주요 질문
"동기와 비동기 처리의 차이점에 대해 설명해주세요. 각각의 장단점과 적합한 사용 사례에 대해서도 설명해주세요."

## 1. 기본 개념

### 1.1 동기(Synchronous) 처리
- 요청과 결과가 동시에 일어남
- 요청을 보낸 후 응답을 받아야 다음 동작이 실행됨
- 실행 순서가 보장됨

```java
// 동기 처리 예시
public String readFile() {
    return Files.readString(Path.of("file.txt")); // 파일을 다 읽을 때까지 대기
}

String result = readFile();
System.out.println(result); // 파일 내용 출력
```

### 1.2 비동기(Asynchronous) 처리
- 요청과 결과가 동시에 일어나지 않음
- 요청을 보낸 후 응답과 관계없이 다음 동작이 실행됨
- 작업 완료 여부는 콜백함수, Promise 등으로 처리

```java
// CompletableFuture를 사용한 비동기 처리 예시
public CompletableFuture<String> readFileAsync() {
    return CompletableFuture.supplyAsync(() -> {
        try {
            return Files.readString(Path.of("file.txt"));
        } catch (IOException e) {
            throw new CompletionException(e);
        }
    });
}

readFileAsync()
    .thenAccept(result -> System.out.println(result)) // 파일 읽기가 완료되면 실행
    .exceptionally(error -> {
        System.err.println("Error: " + error);
        return null;
    });
```

## 2. 동기/비동기 처리의 장단점

### 2.1 동기 처리
장점:
- 설계가 직관적이고 간단함
- 쉬운 디버깅
- 순차적 실행 보장

단점:
- 요청이 완료될 때까지 블로킹
- 자원 활용도가 낮을 수 있음
- 응답 시간이 긴 경우 성능 저하

### 2.2 비동기 처리
장점:
- 자원의 효율적 활용
- 빠른 응답성
- 높은 성능과 처리량

단점:
- 복잡한 설계와 구현
- 디버깅이 어려움
- 콜백 지옥 가능성

## 3. 실제 구현 예시

### 3.1 자바의 비동기 처리
```java
// CompletableFuture 사용 예시
public class OrderService {
    public CompletableFuture<Order> processOrderAsync(Long orderId) {
        return CompletableFuture.supplyAsync(() -> findOrder(orderId))
            .thenApply(order -> validateOrder(order))
            .thenApply(order -> calculateTotal(order))
            .thenApply(order -> saveOrder(order));
    }
}

// 사용 예시
orderService.processOrderAsync(123L)
    .thenAccept(order -> sendNotification(order))
    .exceptionally(error -> {
        handleError(error);
        return null;
    });
```

### 3.2 Spring WebFlux를 사용한 비동기 웹 요청
```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return userRepository.findById(id)
            .flatMap(this::enrichUserData)
            .defaultIfEmpty(new User());
    }
}
```

### 3.3 JavaScript의 비동기 처리
```javascript
// Promise 사용
function fetchUserData(userId) {
    return fetch(`/api/users/${userId}`)
        .then(response => response.json())
        .then(user => enrichUserData(user))
        .catch(error => console.error(error));
}

// Async/Await 사용
async function fetchUserData(userId) {
    try {
        const response = await fetch(`/api/users/${userId}`);
        const user = await response.json();
        return await enrichUserData(user);
    } catch (error) {
        console.error(error);
    }
}
```

## 4. 적합한 사용 사례

### 4.1 동기 처리 적합 사례
- 트랜잭션 처리가 필요한 경우
- 순차적 실행이 중요한 경우
- 즉시 결과가 필요한 경우
- 에러 처리가 중요한 경우

### 4.2 비동기 처리 적합 사례
- 대용량 데이터 처리
- 외부 API 호출
- 파일 입출력
- 독립적으로 실행 가능한 작업들
- 실시간 데이터 처리

## 근거 자료

### 1. 공식 문서
- [Java CompletableFuture API](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html)
- [JavaScript Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

### 2. 기술 서적
- "Java Concurrency in Practice" by Brian Goetz
- "Reactive Programming with RxJava" by Tomasz Nurkiewicz
- "JavaScript: The Definitive Guide" by David Flanagan

### 3. 학술 자료
- ["Asynchronous Programming Patterns"](https://www.sciencedirect.com/science/article/pii/S0167642309000343)
- ["A Study of Asynchronous Programming Patterns"](https://ieeexplore.ieee.org/document/8453111)

### 4. 커뮤니티 자료
- [Node.js Event Loop Documentation](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [Reactive Streams Specification](https://www.reactive-streams.org/)

## 실제 면접 준비를 위한 추가 질문

1. "비동기 프로그래밍에서 발생할 수 있는 race condition을 어떻게 해결할 수 있나요?"

2. "동시에 여러 비동기 작업을 처리해야 할 때 어떤 방식으로 구현하시나요?"

3. "비동기 처리에서 에러 처리는 어떻게 하시나요?"

4. "Event Loop에 대해 설명해주세요."

실제 면접에서는 이러한 개념적인 설명과 함께, 본인이 실제 프로젝트에서 비동기 처리를 구현한 경험과 그 과정에서 겪은 문제점, 해결 방법 등을 구체적으로 설명하는 것이 좋습니다.