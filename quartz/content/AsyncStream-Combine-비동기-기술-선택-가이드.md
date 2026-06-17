# AsyncStream과 Combine: 비동기 처리 기술 선택 가이드

팀 iOS 프로젝트에서 비동기 이벤트를 효과적으로 다루기 위해 `Combine`과 `AsyncStream` 중 어떤 기술을 선택하고 활용할지 결정하는 가이드입니다.

### 0. 프로젝트 기술 스택 및 특징
민잘팀 iOS 프로젝트는 다음과 같은 기술 스택과 특징을 갖습니다:
*   SwiftUI (필요에 따라 부분 UIKit)
*   MVI (Model-View-Intent)
*   SPM 모듈화
*   지도 서비스: 사용자 현재 위치 지속 관찰, 지도 위에 장소 저장

이러한 특성을 고려할 때, 프로젝트에서는 **끊임없는 단일 소스 스트림** 처리와 **여러 구독자에게 퍼지는 상태** 관리가 중요합니다.

### 1. iOS 비동기 처리의 진화
Swift에서 비동기 이벤트를 처리하는 방식은 다음과 같이 발전해왔습니다.

*   **~iOS 12**: Delegate, Completion Handler, Closure Callback
*   **iOS 13 (2019)**: Combine 등장 (선언형 리액티브 프레임워크)
*   **Swift 5.5 (2021)**: `async`/`await`, `Task`, `Actor` 도입; `AsyncStream` (콜백을 AsyncSequence로 변환)
*   **iOS 17 (2023)**: `CLLocationUpdate.liveUpdates()`와 같이 Apple이 직접 제공하는 AsyncSequence API 증가

#### 1.1. 콜백 방식의 한계
`Combine`이나 `AsyncStream`이 없던 시절, `CoreLocation`은 델리게이트, 네트워크는 클로저 콜백으로 비동기를 표현했습니다. 이 방식은 다음과 같은 문제점을 갖습니다.

*   **흐름 분산**: 비동기 작업의 시작, 값 수신, 변형, 정지 로직이 여러 메서드나 프로퍼티에 흩어져 코드 가독성을 저해합니다.
*   **합성 어려움**: "위치를 1초마다 `throttle`하고, 중복을 제거하며, 권한이 통과한 뒤에만 흘려보낼 것"과 같은 복잡한 요구사항을 콜백만으로 구현하기 어렵습니다.
*   **수동적인 생명주기 관리**: 시작과 정지를 개발자가 직접 관리해야 하며, 누락 시 리소스 누수(예: 배터리 소모)로 이어질 수 있습니다.
*   **콜백 중첩 (Callback Hell)**: 순차적인 비동기 작업(예: 권한 요청 → 위치 한 번 수신 → 서버 조회) 시 콜백이 깊게 중첩되어 코드 복잡도가 증가합니다.

### 2. Combine 개요
`Combine`은 콜백 방식의 한계를 극복하기 위해 등장한 선언형 리액티브 프레임워크입니다. 값이 시간에 따라 흐르는 파이프라인을 선언형으로 조립하여, 흩어진 콜백을 하나로 묶고 다양한 연산자를 통해 값을 변형·조합할 수 있게 합니다.

*   **Publisher**: `Output` 타입의 값을 시간에 따라 방출합니다.
*   **Subscriber**: `Publisher`가 방출하는 값을 받아 처리합니다.
*   **Operator**: `Publisher`를 받아 변형된 `Publisher`를 반환하여 파이프라인을 구성합니다.

`Publisher<Output, Failure>`와 같이 값 타입과 에러 타입이 컴파일 타임에 고정됩니다. 에러가 발생하지 않는 스트림은 `Failure == Never`로 표현합니다.

### 3. AsyncStream 개요
`AsyncStream`은 별도의 프레임워크가 아닌 Swift 동시성(Concurrency)의 일부로, 언어 차원의 기능입니다. `AsyncSequence` 프로토콜을 준수하며, `for await ... in` 구문을 통해 일반적인 for문처럼 값을 순회할 수 있습니다.

`AsyncStream`은 기존 콜백/델리게이트 기반 비동기 소스를 `AsyncSequence`로 감싸주는 **브릿지** 역할을 수행합니다.

#### 3.1. Continuation
`AsyncStream` 구현에는 `continuation`이 핵심적인 역할을 합니다. 이는 콜백 기반 API를 `async`/`await` 세계로 연결해주는 도구입니다.

*   **`withCheckedContinuation`**: 단 한 번의 결과를 반환하는 비동기 작업 (예: 권한 요청)에 사용됩니다. `resume(returning:)`은 정확히 1회만 호출되어야 합니다.
*   **`AsyncStream.Continuation`**: 여러 번의 값을 방출하는 비동기 작업에 사용됩니다. `yield(_:)`를 여러 번 호출하여 값을 흘려보내고, `finish()`로 스트림을 종료합니다.

#### 3.2. 버퍼링 정책 (`BufferingPolicy`)
생산자(값 방출)가 소비자(값 처리)보다 빠르게 값을 `yield`할 때, 값을 어떻게 쌓아둘지 결정하는 정책입니다.

*   `.unbounded` (기본): 무제한으로 값을 쌓아둡니다. 끊임없이 들어오는 스트림(예: 위치 업데이트)에서는 메모리 증가에 주의해야 합니다.
*   `.bufferingNewest(n)`: 최신 `n`개의 값만 유지하고, 오래된 값은 버립니다. (예: 최신 위치 정보만 중요한 경우)
*   `.bufferingOldest(n)`: 가장 오래된 `n`개의 값만 유지하고, 새 값은 버립니다.

`AsyncStream`은 `Combine`의 Demand 기반 backpressure 메커니즘이 없습니다. 버퍼가 가득 차면 정책에 따라 이전 값 또는 새 값을 버립니다.

#### 3.3. `onTermination`과 구조적 동시성 취소
`AsyncStream` 생성 시 `continuation.onTermination` 클로저를 등록할 수 있습니다. 이 클로저는 소비자가 스트림(`for await` 루프)을 빠져나가거나 `Task`가 취소될 때 호출됩니다. `onTermination`에서 리소스 정리(예: 위치 업데이트 중지) 로직을 구현하여, `Combine`에서 `AnyCancellable`을 수동으로 관리해야 했던 문제가 언어 차원에서 자동으로 관리됩니다.

#### 3.4. 현재 위치 스트림 예제
기존 `CLLocationManagerDelegate` 기반의 위치 업데이트를 `AsyncStream`으로 변환하는 예제입니다.

```swift
import CoreLocation

final class LocationService: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    private var continuation: AsyncThrowingStream<CLLocation, Error>.Continuation?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
    }

    func currentLocationStream() -> AsyncThrowingStream<CLLocation, Error> {
        AsyncThrowingStream(CLLocation.self, bufferingPolicy: .bufferingNewest(1)) { continuation in
            self.continuation = continuation
            self.manager.startUpdatingLocation()

            continuation.onTermination = { [weak self] _ in
                self?.manager.stopUpdatingLocation() // 스트림 종료 시 자동 정리
                self?.continuation = nil // 순환 참조 방지 및 자원 해제
            }
        }
    }

    // CLLocationManagerDelegate
    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        guard let latest = locations.last else { return }
        continuation?.yield(latest) // 스레드 안전하게 값 방출
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        // 에러 발생 시 스트림 종료
        continuation?.finish(throwing: error)
        continuation = nil // 에러로 종료 후 continuation 무효화
    }
}
```

소비는 `for await` 구문으로 간결하게 이루어집니다.

```swift
Task { @MainActor in
    let service = LocationService() // 예시를 위한 인스턴스 생성
    do {
        for try await location in service.currentLocationStream() {
            // 메인 액터에서 안전하게 UI 갱신
            print("현재 위치: \(location.coordinate)")
        }
    } catch {
        print("위치 스트림 에러: \(error)")
    }
}
```

> **참고**: iOS 17+ 환경에서는 Apple이 제공하는 `CLLocationUpdate.liveUpdates()`와 같은 네이티브 AsyncSequence API를 직접 활용하여 더욱 간결하게 구현할 수 있습니다.

### 4. AsyncStream의 장점

*   **콜백/델리게이트 통합**: 기존 비동기 소스를 `for await` 기반의 단일 흐름으로 자연스럽게 통합합니다.
*   **취소 및 생명주기 자동화**: [[Swift-AsyncSequence-이해-및-활용]]에서 설명하는 구조적 동시성을 통해 `Task` 취소 신호가 전파되어 `onTermination`에서 리소스가 자동으로 정리됩니다. SwiftUI의 `.task` 수정자와 함께 사용하면 보일러플레이트 코드를 줄일 수 있습니다.
*   **낮은 진입 장벽**: `Publisher`, `Subscriber`, `Operator`, `Scheduler`, `AnyCancellable`과 같은 `Combine` 고유의 개념 없이 `async`/`await`만 이해하면 쉽게 사용할 수 있습니다.
*   **순차 + 연속 비동기의 통합**: `await` (단발성)와 `for await` (연속성)가 같은 Swift Concurrency 세계관 내에서 자연스럽게 연결됩니다.

### 5. AsyncStream의 한계

*   **멀티캐스트 부재**: `AsyncStream`은 기본적으로 단일 소비자 모델입니다. 여러 구독자에게 동시에 값을 전달하려면 직접 팬아웃(fan-out) 로직을 구현해야 합니다.
*   **연산자 부족**: 표준 라이브러리에는 `map`, `filter` 등의 기본적인 연산자만 제공됩니다. `debounce`, `throttle`, `combineLatest`, `merge` 등 복잡한 연산자가 필요한 경우 `swift-async-algorithms`와 같은 외부 패키지를 추가해야 합니다.
*   **Demand 기반 Backpressure 부재**: 생산자가 소비자보다 빠르게 값을 `yield`할 때, 버퍼 정책에 따라 값이 버려지거나(버퍼링 정책) 무제한으로 쌓입니다. `Combine`처럼 소비자의 수요(Demand)에 따라 생산 속도를 조절하는 메커니즘이 내장되어 있지 않습니다.

#### 5.1. 멀티캐스트 구현 시 수작업 비용
저장된 장소 목록을 여러 화면이 동시에 구독하는 상황을 `AsyncStream`으로 구현하려면 `actor`를 활용하여 다음과 같이 직접 멀티캐스트 로직을 구현해야 합니다. 이는 `Combine`의 `CurrentValueSubject`를 사용하는 것보다 훨씬 많은 수작업 비용을 발생합니다.

```swift
actor SavedPlacesBroadcaster {
    private var current: [SavedPlace] = []
    private var continuations: [UUID: AsyncStream<[SavedPlace]>.Continuation] = [:]

    func stream() -> AsyncStream<[SavedPlace]> {
        AsyncStream {
            // 신규 구독자에게 현재값 즉시 전달 (CurrentValueSubject와 유사)
            $0.yield(current)

            let id = UUID()
            continuations[id] = $0

            $0.onTermination = { [weak self] _ in
                Task { await self?.remove(id) }
            }
        }
    }

    private func remove(_ id: UUID) { continuations[id] = nil }

    func save(_ place: SavedPlace) {
        current.append(place)
        for c in continuations.values { c.yield(current) } // 모든 구독자에게 방송
    }
}
```

### 6. Combine과 AsyncStream 간 상호 운용성
`Combine`과 `AsyncStream`은 상호 배타적인 기술이 아니며, 서로 변환하여 함께 사용할 수 있습니다.

*   **`Combine` Publisher → `AsyncSequence`**: `Combine` Publisher의 `.values` 프로퍼티를 통해 `AsyncSequence`로 변환하여 `for await`로 소비할 수 있습니다.

    ```swift
    for await place in store.places.values {
        // ... AsyncSequence로 소비
    }
    ```

*   **`async` 작업 → `Combine` 파이프라인**: `Future`로 `async` 작업을 감싸 `Combine` 파이프라인에 통합할 수 있습니다.

    ```swift
    let publisher = Future<CLAuthorizationStatus, Never> { promise in
        Task { promise(.success(await service.requestAuthorization())) }
    }
    ```

이러한 변환 기능을 통해 각 기술이 강점을 갖는 경계에서 최적의 선택을 하고 필요한 경우 연결할 수 있습니다.

### 7. 프로젝트 적용 방안
민잘팀 프로젝트의 MVI 아키텍처와 특징을 고려할 때, `AsyncStream`과 `Combine`의 역할은 자연스럽게 나뉩니다.

#### 7.1. MVI 아키텍처 관점
프로젝트 MVI는 **`Store`를 두되, 상태 전이는 순수 `reduce` 함수로, 비동기 작업은 `Effect`로 분리**하는 구조를 갖습니다. 의존성은 `AppDependencies` 구조체를 통한 [[의존성-주입-DI-번들-주입-선택-적용|번들 주입]] 방식으로 받습니다.

이러한 구조에서:

*   **현재 위치 스트림**: [[Swift-AsyncSequence-이해-및-활용]]의 `AsyncStream`을 기반으로 `LocationService`를 구현하여 `Effect` 내부에서 소비하고, 그 값을 다시 `Intent`로 되먹여 단방향 흐름(Intent → reduce → State)을 유지합니다. `for await`는 이 "스트림 소비 → Intent 변환" 고리를 직관적으로 표현합니다.
*   **저장된 장소 목록 (`SavedPlacesStore`)**: 여러 화면이 공유하는 상태이므로 `Combine`의 `CurrentValueSubject` 기반으로 구현하는 것이 멀티캐스트 및 "최신 상태 즉시 전달" 측면에서 더 자연스럽습니다.

`AppDependencies`는 이 두 종류의 의존성을 함께 묶어 [[세-축-통합-Coordinator-MVI-DI]] 아키텍처에 통합합니다.

```swift
struct AppDependencies {
    let locationService: LocationService
    let savedPlacesStore: SavedPlacesStore // Combine CurrentValueSubject 기반
}

@MainActor
final class MapStore: ObservableObject {
    @Published private(set) var state = MapState()

    private let dependencies: AppDependencies
    private var trackingTask: Task<Void, Never>?

    init(dependencies: AppDependencies) {
        self.dependencies = dependencies
    }

    func send(_ intent: MapIntent) {
        let effect = reduce(&state, intent) // 순수 상태 전이
        run(effect)                          // 비동기 Effect 실행
    }

    private func run(_ effect: MapEffect) {
        switch effect {
        case .startTracking:
            trackingTask = Task {
                for await location in dependencies.locationService.currentLocationStream() {
                    self.send(.locationUpdated(location.coordinate))
                }
            }
        case .stopTracking:
            trackingTask?.cancel()
        case .save(let place):
            dependencies.savedPlacesStore.save(place)
        case .none:
            break
        }
    }
}
// MapState, MapIntent, MapEffect, reduce 함수는 본문에서 생략
```

#### 7.2. SwiftUI 관점
SwiftUI의 `.task` 수정자는 뷰의 생명주기에 `Task`를 바인딩하여 뷰가 사라질 때 `Task`를 자동으로 취소해줍니다. `AsyncStream`과 함께 사용하면 `Combine`의 `Set<AnyCancellable>`과 같은 명시적인 `cancellable` 관리 없이 위치 추적 스트림을 뷰와 함께 시작·종료할 수 있어 보일러플레이트가 줄어듭니다.

```swift
.task {
    // 뷰가 사라지면 자동으로 스트림 종료 + onTermination 호출
    for await location in service.currentLocationStream() {
        // ...
    }
}
```

#### 7.3. SPM 모듈화 관점
실무적으로 중요한 결정 지점입니다.

*   **Combine**: iOS 내장 프레임워크이므로 어느 모듈에서 사용해도 **외부 의존성이 없습니다**.
*   **AsyncStream**: `AsyncStream` 자체는 Swift 내장 기능이지만, `throttle`, `debounce`, `combineLatest` 등 복잡한 연산자가 필요한 경우 `swift-async-algorithms`라는 **외부 의존성**이 필요합니다. 이 외부 의존성이 모듈 그래프에 미치는 영향을 고려해야 합니다. 위치 스트림과 같이 기본적인 연산자만으로 충분한 경우 의존성이 발생하지 않을 수 있습니다.

#### 7.4. 지속 위치 추적 관점
끊임없이 들어오는 위치 스트림에서는 backpressure와 메모리 관리가 중요합니다.

*   대부분의 경우 위치는 **최신값만 의미 있는** 특성을 갖습니다. 이 경우 `AsyncStream`의 `.bufferingNewest(1)` 정책이 가장 적합하며, 메모리 부담을 최소화할 수 있습니다.
*   만약 "모든 위치 포인트를 경로로 누적"하는 기능이 필요하다면, `AsyncStream`의 `.unbounded` 버퍼 사용 시 메모리 증가나 `Combine`의 Demand 기반 제어 방식을 고려해야 합니다.

#### 7.5. 팀 관점

*   **최소 지원 버전**: `async`/`await` 및 `AsyncStream`은 Xcode 13.2+에서 iOS 13까지 백디플로이(back-deploy)됩니다. `Combine` 또한 iOS 13+에서 지원되므로, **버전으로 인한 제약은 없습니다.** (단, `CLLocationUpdate.liveUpdates()`와 같은 네이티브 `AsyncSequence` API는 iOS 17+에서 사용 가능합니다.)

### 8. 역할 기반 혼용 전략
전면적인 단일 기술 통일보다는 각 기술의 강점을 살리는 **"역할 기반 혼용" 전략**을 채택합니다.

*   **신규 코드의 기본값은 언어 표준에 수렴하는 `AsyncStream`으로 설정**합니다. 특히 단일 소스·단일 소비자 스트림(예: 현재 위치 추적)의 경우, SwiftUI의 `.task`와의 시너지와 낮은 진입 장벽을 활용합니다. `swift-async-algorithms`와 같은 외부 의존성이 필요 없는 단순한 스트림 처리에 적합합니다.
*   **`Combine`이 명백히 우월한 멀티캐스트 및 복잡한 합성(Operator) 지점에서만 의도적으로 `Combine`을 사용**합니다. (예: 여러 화면이 구독하는 공유 상태 관리에 `CurrentValueSubject` 활용)
*   두 기술 간의 경계는 `.values` (Publisher → AsyncSequence) 또는 `Future` (async → Combine)와 같은 **브릿지**를 통해 유연하게 연결합니다.

### 9. 참고 자료
*   [Swift Concurrency: AsyncStream / AsyncThrowingStream (Apple Docs)](https://developer.apple.com/documentation/swift/asyncstream)
*   [Combine (Apple Docs)](https://developer.apple.com/documentation/combine)
*   [swift-async-algorithms (Apple, GitHub)](https://github.com/apple/swift-async-algorithms)
*   [CLLocationUpdate.liveUpdates() (Apple Docs)](https://developer.apple.com/documentation/corelocation/cllocationupdate/3956976-liveupdates/)
*   WWDC21 "Meet async/await in Swift", "Meet AsyncSequence"
