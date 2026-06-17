# Swift AsyncSequence 이해 및 활용

## AsyncSequence가 필요한 이유

`async` 함수는 단 한 번만 값을 반환하고 종료됩니다.

```swift
func fetchUser() async throws -> User       // 1번 반환하고 끝
func currentLocation() async -> CLLocation  // 1번 반환하고 끝
```

그러나 현실에는 한 번의 결과로 끝나지 않고 지속적으로 발생하는 데이터가 많습니다.

*   **위치 업데이트**: 1초마다 새로운 좌표
*   **웹소켓**: 서버가 언제 메시지를 보낼지 모르는 스트림
*   **다운로드 진행률**: 0%, 12%, 34%, ... 100%
*   **사용자 키 입력**: 타이핑할 때마다 발생하는 이벤트
*   **노티피케이션**: 앱이 활성화될 때마다 발생하는 알림

이러한 상황에서는 `async` 함수처럼 단 한 번 `await`하여 값을 받는 방식으로는 연속적인 데이터 흐름을 표현할 수 없습니다.

### 시간축이라는 새로운 차원

비동기 작업에는 두 가지 종류가 있습니다.

| | 동기 | 비동기 |
|---|---|---|
| **단일 값** | `T` (일반 변수) | `async -> T` |
| **여러 값** | `[T]` / [[Swift-Sequence-프로토콜-이해하기|Sequence]] | **`AsyncSequence`** |

`async`가 동기 `T`를 비동기적으로 확장하여 단일 값을 처리한다면, `AsyncSequence`는 동기 `[[Swift-Sequence-프로토콜-이해하기|Sequence]]`를 비동기적으로 확장하여 시간에 걸쳐 발생하는 여러 값을 처리합니다. 이는 데이터 흐름에 **시간축이라는 차원이 하나 더 추가**된 것으로 이해할 수 있습니다.

```swift
// 동기 시퀀스: 값이 공간에 펼쳐져 있음 (이미 다 존재)
[1, 2, 3, 4, 5]

// 비동기 시퀀스: 값이 시간에 걸쳐 비동기적으로 생성됨
1 ---(0.3s)---> 2 ---(0.8s)---> 3 ---(2s)---> 4 ...
```

### 콜백/델리게이트 패턴의 한계

기존에는 콜백 클로저나 델리게이트 패턴을 사용하여 이러한 연속적인 비동기 데이터를 처리했습니다.

```swift
// 콜백 중첩 예시
manager.fetchFirst { first in
    manager.fetchSecond(after: first) { second in
        manager.fetchThird(after: second) { third in
            // ... 콜백 중첩으로 인한 가독성 저하
        }
    }
}

// 델리게이트 패턴 예시 — 여러 메서드에 로직 분산
extension MyClass: SomeDelegate {
    func didReceiveData(_ data: Data) { /* ... */ }
    func didFail(with error: Error) { /* ... */ }
    func didComplete() { /* ... */ }
}
```

이러한 패턴은 다음과 같은 문제점이 있습니다.

1.  **제어 흐름 역전**: 코드가 옆으로 들여쓰기되어 읽기 어렵고, 위에서 아래로 흐르는 직관적인 제어 흐름이 깨집니다.
2.  **함수형 조합 불가**: `filter`, `map`, `prefix(5)`와 같은 함수형 연산자를 직접 적용하기 어렵습니다.
3.  **취소의 어려움**: 비동기 작업의 중단 로직을 별도로 구현해야 하여 복잡합니다.
4.  **에러 처리 분산**: 델리게이트 패턴의 경우 성공, 실패, 완료 등의 상태가 여러 메서드에 흩어져 에러 처리가 일관적이지 못합니다.

### AsyncSequence의 장점

`AsyncSequence`를 사용하면 위의 문제를 해결하고, 비동기 스트림을 동기 코드처럼 다룰 수 있게 됩니다.

```swift
Task {
    do {
        for try await location in locationUpdates
            .filter { $0.horizontalAccuracy < 10 }   // 함수형 조합 가능
            .prefix(100) {                            // 100개만
            let result = try await api.send(location)
            handle(result)
        }
    } catch {
        print("에러:", error)   // 모든 에러 한 곳에서 처리
    }
}
// task.cancel() 한 줄로 루프 자체 종료 (취소도 깔끔)
```

이처럼 `AsyncSequence`는 제어 흐름을 동기처럼 위에서 아래로 유지하고, 함수형 메서드를 적용할 수 있으며, `Task.cancel()`으로 쉽게 중단하고, `do-catch` 블록으로 에러를 한 곳에서 처리할 수 있도록 돕습니다.

> **한 줄 요약**: `AsyncSequence`는 "여러 번 `await`할 수 있는 값"을 표현하는 타입으로, 현실의 대부분의 연속적인 비동기 데이터를 다루는 데 필수적입니다.

## Sequence vs AsyncSequence

`AsyncSequence`는 `Sequence`와 유사하지만, 값을 가져오는 데 시간이 걸린다는 근본적인 차이가 있습니다. 따라서 각 값을 꺼낼 때마다 `await` 키워드를 사용하여 "기다리겠다"고 명시해야 합니다.

| 특성 | [[Swift-Sequence-프로토콜-이해하기|Sequence]] (동기) | AsyncSequence (비동기) |
|---|---|---|
| **값의 존재 방식** | 모든 값이 이미 메모리에 존재하거나 즉시 생성 가능 | 값이 시간에 걸쳐 비동기적으로 생성됨 |
| **값 접근** | 즉시 접근 가능 (`for-in`) | 각 값을 기다려야 접근 가능 (`for-await-in`) |

```swift
// 동기 시퀀스
for n in [1, 2, 3] {        // 값이 이미 다 존재
    print(n)
}

// 비동기 시퀀스
for await n in someAsyncSeq {   // 각 값을 비동기적으로 기다림
    print(n)
}
```

## `for-await-in` 사용 규칙

`for-await-in` 루프는 비동기 컨텍스트(예: `Task`, `async` 함수) 안에서만 사용할 수 있습니다.

```swift
// 비동기 컨텍스트 안에서 사용
Task {
    for await item in asyncSeq {
        print(item)
    }
}
```

시퀀스가 에러를 던질 수 있는 경우, `try await`를 사용하여 에러를 처리해야 합니다.

```swift
Task {
    do {
        for try await item in asyncSeq {
            print(item)
        }
    } catch {
        print("에러:", error)
    }
}
```

`continue`, `break`와 같은 제어 흐름 문법은 `for-await-in`에서도 동일하게 사용할 수 있습니다. 또한, 해당 `Task`를 `cancel()`하면 `for-await-in` 루프가 즉시 종료됩니다.

## `for-await-in`의 동작 원리

`for-in`이 `while` 루프로 컴파일되었던 것처럼, `for-await-in`도 내부적으로는 `while` 루프와 유사하게 동작합니다. 단, `next()` 메서드 호출에 `await`이 붙어 비동기적으로 다음 값을 기다린다는 차이가 있습니다.

```swift
// 우리가 사용하는 코드
for await n in counter {
    print(n)
}

// 개념적인 컴파일 결과
var iter = counter.makeAsyncIterator()
while let n = await iter.next() {   // ← await가 붙은 next() 호출
    print(n)
}
```

`Task`가 `next()`를 호출하면, 값이 준비되기 전까지 **자신을 suspend(일시 중단)** 시킵니다. 이 동안 스레드는 다른 작업을 수행할 수 있으며, 값이 준비되면 `Task`는 **resume(재개)**되어 다음 코드를 실행합니다. 이러한 방식으로 `AsyncSequence`는 비동기 작업을 효율적으로 처리합니다.

> 🔗 **공식 문서**: [AsyncIteratorProtocol](https://developer.apple.com/documentation/swift/asynciteratorprotocol)

## 커스텀 AsyncSequence 직접 만들어보기

`AsyncSequence`를 직접 구현하는 것은 `Sequence` 구현과 유사하지만, 반복자의 `next()` 메서드가 `async`를 사용한다는 차이가 있습니다.

### 분리형 구현 예제

```swift
// 1. 비동기 이터레이터 구현
struct CounterIterator: AsyncIteratorProtocol {
    typealias Element = Int
    let howHigh: Int
    var current = 1

    mutating func next() async -> Int? {
        guard current <= howHigh else { return nil }
        // 비동기 작업: 1초 대기 (Task를 suspend하고 CPU는 다른 일 수행)
        try? await Task.sleep(for: .seconds(1))
        defer { current += 1 }
        return current
    }
}

// 2. 비동기 시퀀스 구현
struct Counter: AsyncSequence {
    typealias Element = Int
    typealias AsyncIterator = CounterIterator
    let howHigh: Int

    func makeAsyncIterator() -> CounterIterator {
        return CounterIterator(howHigh: howHigh)
    }
}

// 사용 예시 — 1초마다 하나씩 출력됨
Task {
    for await number in Counter(howHigh: 10) {
        print("숫자:", number)
    }
}
```

`Task.sleep`이 동작하는 동안 `Task`는 `suspend` 상태가 되고, 해당 스레드는 다른 작업을 처리할 수 있습니다. 이는 스레드가 잠들어버려 다른 작업이 막히는 동기적인 `sleep`과는 다릅니다.

### 공식 문서가 권하는 패턴 — 중첩 타입

Apple 공식 문서의 예제들은 반복자(`AsyncIterator`)를 시퀀스 타입의 **내부 중첩 타입**으로 두는 스타일을 사용합니다. 이는 외부 네임스페이스를 깔끔하게 유지하는 장점이 있습니다.

```swift
struct MyCounter: AsyncSequence {
    typealias Element = Int

    let howHigh: Int

    // AsyncIterator를 중첩 타입으로 정의
    struct AsyncIterator: AsyncIteratorProtocol {
        let howHigh: Int
        var current = 1

        mutating func next() async -> Int? {
            guard current <= howHigh else { return nil }
            try? await Task.sleep(for: .milliseconds(500))
            defer { current += 1 }
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(howHigh: howHigh)
    }
}
```

[[AsyncStream-Pull-방식-Unfolding-클로저|AsyncStream]], `AsyncMapSequence` 등 Swift 표준 라이브러리의 많은 `AsyncSequence` 관련 타입들이 이 패턴을 따르고 있습니다.

## Swift 6의 변화: Typed Error와 격리 (SE-0421)

Swift 6에서는 `AsyncIteratorProtocol`에 다음과 같은 주요 개선 사항이 도입되었습니다.

1.  **`Failure` 연관 타입 추가**: 시퀀스가 던질 수 있는 에러 타입을 명시적으로 선언할 수 있게 됩니다.
2.  **`next(isolation:)`**: 격리 컨텍스트를 명시할 수 있는 새로운 메서드가 추가되었습니다.

```swift
enum CounterError: Error {
    case unluckyNumber
}

struct SafeCounterIterator: AsyncIteratorProtocol {
    typealias Element = Int
    typealias Failure = CounterError    // <- Swift 6에서 추가된 연관 타입

    let howHigh: Int
    var current = 1

    // 격리 컨텍스트를 받고, 타입드 에러를 던지는 새로운 메서드
    mutating func next(
        isolation actor: isolated (any Actor)?
    ) async throws(CounterError) -> Int? {
        guard current <= howHigh else { return nil }
        let value = current
        current += 1
        if value == 13 {                       // 13은 불운한 숫자라고 가정
            throw .unluckyNumber
        }
        return value
    }
}
```

`throws(CounterError)`는 이 함수가 `CounterError` 타입의 에러만 던질 수 있음을 컴파일러에 약속하는 것입니다. 이를 통해 호출자는 불필요한 에러 타입을 `catch`할 필요가 없습니다.

> 🔗 **참고**: [Swift Evolution SE-0421 — Generalize AsyncSequence](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0421-generalize-async-sequence.md)

## 함수형 연산자

`[[Swift-Sequence-프로토콜-이해하기|Sequence]]`와 마찬가지로 `AsyncSequence`도 `map`, `filter`, `reduce`, `dropFirst`, `prefix`, `zip` 등 다양한 함수형 연산자를 기본으로 제공합니다. 이 연산자들은 기존 `AsyncSequence`를 변환하여 새로운 `AsyncSequence`를 반환합니다.

```swift
Task {
    let evens = MyCounter(howHigh: 20)
        .filter { $0 % 2 == 0 }     // 짝수만 필터링
        .map { $0 * 10 }            // 10배로 변환
        .prefix(5)                   // 처음 5개만 취득

    for await v in evens {
        print(v)   // 20, 40, 60, 80, 100 출력
    }
}
```

`filter`, `map`, `prefix`와 같은 연산자들은 각각 새로운 `AsyncSequence` 인스턴스를 반환하여, 마치 Combine 프레임워크의 오퍼레이터 체인처럼 비동기 시퀀스를 함수형으로 조합할 수 있게 합니다.

더 다양한 비동기 전용 연산자(예: `debounce`, `throttle`, `merge`, `combineLatest` 등)가 필요하다면, Apple이 제공하는 별도 라이브러리인 `Swift Async Algorithms`를 활용할 수 있습니다.

> 🔗 **참고**: [Swift Async Algorithms (GitHub)](https://github.com/apple/swift-async-algorithms)
> 🔗 **참고**: [Introducing Swift Async Algorithms — Apple Developer Blog](https://www.swift.org/blog/swift-async-algorithms/)

## 표준 라이브러리가 제공하는 AsyncSequence들

개발자가 직접 `AsyncSequence`를 구현하지 않아도, Swift와 Foundation, CoreLocation 등 Apple 프레임워크는 이미 많은 `AsyncSequence` 형태의 API를 제공하고 있습니다.

### `URL.lines` — HTTP 응답을 한 줄씩 비동기로

`URLSession.data(from:)`가 전체 응답을 한 번에 가져오는 반면, `URL.lines`는 HTTP 응답이 도착하는 대로 **한 줄씩 비동기적으로** 처리할 수 있습니다. 이는 크기가 큰 텍스트 데이터나 스트리밍 데이터를 처리할 때 메모리 효율적입니다.

```swift
let url = URL(string: "https://www.example.com")!

Task {
    do {
        for try await line in url.lines {
            print(line) // 다운로드되는 즉시 처리됨
        }
    } catch {
        print("에러:", error)
    }
}

// 함수형 조합도 가능
let titleLines = url.lines
    .filter { $0.contains("<title>") }
    .map { $0.trimmingCharacters(in: .whitespaces) }
```

> 🔗 **공식 문서**: [URL.lines](https://developer.apple.com/documentation/foundation/url/3767315-lines)

### `FileHandle.bytes` — 파일/표준 입력을 바이트 단위로

파일 또는 표준 입력을 바이트 스트림으로 읽어들일 수 있습니다.

```swift
// 표준 입력을 한 줄씩 비동기로 처리
for try await line in FileHandle.standardInput.bytes.lines {
    print("입력:", line)
}

// 파일 전체를 바이트로 스트리밍
let handle = try FileHandle(forReadingFrom: someURL)
for try await byte in handle.bytes {
    // 한 바이트씩 처리
}
```

> 🔗 **공식 문서**: [FileHandle.bytes](https://developer.apple.com/documentation/foundation/filehandle/3766681-bytes)

### `URLSession.bytes` — 네트워크 응답을 바이트 단위로

네트워크 응답 본문을 바이트 단위로 실시간으로 처리할 수 있게 합니다.

```swift
let (bytes, response) = try await URLSession.shared.bytes(from: url)

guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
    throw URLError(.badServerResponse)
}

for try await byte in bytes {
    // 응답 바이트를 즉시 받아 처리
}
```

> 🔗 **공식 문서**: [URLSession.bytes(from:)](https://developer.apple.com/documentation/foundation/urlsession/3767351-bytes)

### `NotificationCenter.notifications` — 알림을 비동기 시퀀스로

`NotificationCenter`의 알림을 `addObserver` 방식 대신 비동기 시퀀스로 구독할 수 있습니다.

```swift
let center = NotificationCenter.default
let notis = center.notifications(named: UIApplication.didBecomeActiveNotification)

Task {
    for await noti in notis {
        print("앱이 활성화됨:", noti)
    }
}

// 첫 번째 알림만 기다리는 패턴도 가능
let first = await notis.first { _ in true }
```

> 🔗 **공식 문서**: [NotificationCenter.notifications(named:)](https://developer.apple.com/documentation/foundation/notificationcenter/3813137-notifications)

### `CLLocationUpdate.liveUpdates` — CoreLocation의 신형 API (iOS 17+)

iOS 17부터 CoreLocation은 델리게이트 없이 비동기 시퀀스만으로 위치 업데이트를 받을 수 있는 API를 제공합니다.

```swift
Task {
    for try await update in CLLocationUpdate.liveUpdates() {
        print("내 위치:", update.location ?? "없음")
        print("정지 상태:", update.isStationary)
    }
}
```

> 🔗 **공식 문서**: [CLLocationUpdate](https://developer.apple.com/documentation/corelocation/cllocationupdate)
> 🔗 **WWDC 2023**: [Discover streamlined location updates](https://developer.apple.com/videos/play/wwdc2023/10180/)

## 정리

### 핵심 요약

> `AsyncSequence`는 "여러 번 `await`할 수 있는 값"을 표현하는 타입입니다.

### 상세 내용

1.  **타입 관점** — 비동기 데이터는 두 가지 종류로 나눌 수 있습니다.

    | | 동기 | 비동기 |
    |---|---|---|
    | **단일 값** | `T` | `async -> T` |
    | **여러 값** | `[T]` / `Sequence` | **`AsyncSequence`** |

    `async`가 동기 `T`를 비동기로 확장하여 단일 값을 다룬다면, `AsyncSequence`는 동기 `Sequence`를 비동기로 확장하여 시간축 위에 놓인 여러 값을 다룹니다.

2.  **동작 관점** — `for-await-in`은 `while let n = await iter.next()`와 동일하게 동작합니다.

    `await`은 해당 `Task`를 일시 중단(suspend)시킬 뿐, 스레드 자체는 다른 일을 계속합니다. 값이 준비되면 `Task`는 재개(resume)되어 다음 코드를 실행합니다.

3.  **설계 관점** — 왜 `AsyncCollection`은 없을까요?

    `Collection` 프로토콜은 `count` 속성, 인덱스 기반 접근, 여러 번 순회 가능 등의 특성이 있습니다. 그러나 비동기 스트림의 특성상 `AsyncCollection`은 원천적으로 불가능합니다.
    *   `count`를 미리 알 수 없음: 스트림이 언제 끝날지 알 수 없습니다.
    *   인덱스 접근 불가: 예를 들어 3번째 값을 얻기 위해선 1, 2번째 값을 모두 `await`하여 순서대로 받아야 합니다.
    *   두 번 순회 불가: 시간이 흐르는 데이터이므로, 이미 지나간 값을 다시 가져올 수 없습니다. (예: 어제 발생한 위치 이벤트는 오늘 다시 발생하지 않습니다.)

    따라서 `AsyncSequence`는 `Sequence`의 개념을 본떠 설계되었으며, `Collection`의 특성을 계승하지 않습니다.

## 다음 단계: AsyncStream

지금까지 살펴본 `AsyncSequence`는 강력하지만, 직접 `AsyncIteratorProtocol`을 구현하는 과정은 번거로울 수 있습니다. 상태 관리, 비동기 `next()` 구현 등이 복잡하게 느껴질 수 있습니다.

이러한 보일러플레이트를 줄이고 `AsyncSequence`를 더 쉽게 생성할 수 있도록 Apple은 `AsyncStream`이라는 도구를 제공합니다.

> 🔗 **관련 문서**: [AsyncSequence — Apple Developer](https://developer.apple.com/documentation/swift/asyncsequence)
> 🔗 **관련 문서**: [AsyncIteratorProtocol — Apple Developer](https://developer.apple.com/documentation/swift/asynciteratorprotocol)
> 🔗 **관련 문서**: [SE-0421: Generalize AsyncSequence](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0421-generalize-async-sequence.md)
> 🔗 **관련 문서**: [Swift Async Algorithms](https://github.com/apple/swift-async-algorithms)
