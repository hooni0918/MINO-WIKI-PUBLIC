# Coordinator 아키텍처 설계 및 컨벤션

## 0. 배경

### 0.1 Coordinator의 필요성: 화면 간 직접 의존성 문제 해결

화면(View/ViewController)이 다음 화면을 직접 호출하는 방식(예: `navigationController?.pushViewController(NextVC(), ...)`)은 다음과 같은 문제를 야기합니다.

*   **재사용성 저하**: 다른 Flow에 화면을 재사용하려면 화면 코드를 직접 수정해야 합니다.
*   **분기 로직 복잡성**: "신규 사용자는 온보딩, 기존 사용자는 홈, 결제 중이면 결제 화면으로 복귀"와 같은 복잡한 분기 로직이 화면 내부에 집중됩니다.
*   **테스트의 어려움**: 화면 전환을 테스트하기 위해 내비게이션 스택을 Mocking해야 하는 등 테스트가 복잡해집니다.
*   **Flow 파악의 어려움**: 하나의 Flow를 이해하려면 여러 화면의 코드를 일일이 확인해야 합니다.

**Coordinator**는 이러한 문제를 해결하기 위한 패턴입니다. Flow에 속한 화면들을 관리하고 화면 전환을 책임지는 상위 객체로서, 화면은 자신의 고유한 작업에만 집중하고 "작업 완료" 또는 "이동 의도"만을 외부에 알립니다.

### 0.2 Coordinator의 핵심 원칙 3가지

1.  **의존성은 위에서 아래로 흐릅니다.** (자식 Coordinator는 부모 Coordinator를 알지 못합니다.)
2.  **결과는 아래에서 위로 표준 채널을 통해 흐릅니다.** (본 프로젝트에서는 `FlowFinish`를 사용합니다.)
3.  **동일한 원칙이 재귀적으로 적용됩니다.** 부모-자식 Coordinator는 동일한 인터페이스(Route enum + Output + finish 채널)를 가지며, 깊이에 관계없이 동일한 패턴으로 조립되어 전체적으로 트리 구조를 이룹니다.

트리의 Root를 조립하는 위치를 **Composition Root**라고 합니다. 일반적으로 `AppCoordinator`가 앱 시작 시 한 번 의존성 그래프를 조립하며(API 클라이언트, Repository 등을 생성하여 자식에게 전달), 본 프로젝트에서는 여기서 `AppDependencies` 번들을 조립하여 Coordinator 트리로 [[의존성-주입-DI-번들-주입-선택-적용|주입]]합니다. 구체적인 구현체 조립은 App 레이어에만 담당하고, 하위 레이어는 프로토콜에만 의존합니다. 로거와 애널리틱스 같은 횡단 관심사는 경량 전역 Facade로 둡니다.

### 0.3 SwiftUI 시대의 Coordinator

SwiftUI는 `@State path: [Route]`와 `NavigationStack(path:)` 등을 통해 내비게이션을 **선언형**으로 만들었습니다. 핵심은 "상태가 곧 화면"이라는 점입니다.

*   사용자가 스와이프 백(swipe-back)하면 `path.popLast()`가 자동으로 발생합니다. (UIKit의 수동 양방향 동기화 및 `UINavigationControllerDelegate` 함정이 사라집니다.)
*   딥링크는 `path = [.posts, .post(42), .comments(7)]`와 같이 **한 줄**로 내비게이션 스택을 통째로 교체할 수 있게 합니다. (비동기 체인, 중간 사용자 개입, 콜드/웜 스타트 문제가 사라집니다.)

그럼에도 불구하고 Coordinator는 여전히 다음과 같은 이유로 필요합니다.

| 이유 | 설명 |
| :--- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| (a) 상태 생명 주기 관리 | `@State`는 View가 사라지면 함께 사라집니다. 모달을 닫았다 다시 띄울 때 상태를 유지하려면 View 외부에 상태가 존재해야 하며, 이는 `@Observable` 객체로 분리하여 관리합니다. |
| (b) 책임 분리 | Flow가 커지면 View 내부에 `path`/`sheet` 상태 및 분기 로직이 과도하게 증가합니다. Flow 결정을 별도의 객체(Coordinator)로 분리하여 View는 "보여주기"에만 집중하도록 합니다. |
| (c) Flow 재사용성 | "EditFlow"를 두 군데서 띄우려면 View 내부에 박힌 로직은 재사용이 어렵습니다. Coordinator로 Flow 로직을 추상화하여 재사용성을 높입니다. |
| (d) 딥링크 진입점 통합 | 앱의 어느 곳에서든 특정 Flow 진입을 한 곳으로 요청할 수 있는 단일 진입점을 제공합니다. |

> **플랫폼 전제**: iOS 17 + `@Observable` 매크로를 사용합니다. Coordinator는 `@Observable` 클래스이며, App 진입점에서 `@State`로 보유합니다. (기존의 `ObservableObject` + `@StateObject` 패턴은 사용하지 않습니다. 이하의 코드는 모두 `@Observable` + `@State`를 기준으로 작성됩니다.)

### 0.4 운영 조건 (결정 전제)

*   **성격**: MVP에서 성장 가능성 있는 앱 (3개월 내 MVP)
*   **팀**: 최대 3명
*   **화면 규모**: 15개 미만 → 성장 시 20~30개 (중간 규모)
*   **플랫폼**: iOS 17 / Swift 5.10 / SwiftUI / `@Observable`

본 문서는 이 전제를 바탕으로 **Coordinator 프로토콜의 형태**(1절)와 **채택 컨벤션**(2절)을 결정합니다. 화면 내부 상태 및 비동기 처리는 [[MVI-아키텍처-선택-테스트-강력함-경량-A|경량 A(MVI)]]가, 의존성 조립은 [[의존성-주입-DI-번들-주입-선택-적용|번들 주입(DI)]]이 담당하며, 이 세 가지 축의 결합은 "[[세-축-통합-Coordinator-MVI-DI|세 축 통합 문서]]"에서 다룹니다.

MVI 및 DI가 "여러 후보 중 하나를 선택한 결정"인 것과 달리, Coordinator는 `FlowCoordination` 패키지의 Coordinator 프로토콜을 채택하는 성격이 강하므로, "후보 비교"보다는 "설계 및 채택 컨벤션"에 가깝습니다. 따라서 본 문서는 프로토콜 형태와 컨벤션 두 가지만 다룹니다.

## 1. Coordinator 프로토콜 결정 (핵심)

**결정**: Coordinator를 능력별 프로토콜로 분리하지 않고, **단일 `Coordinator` 프로토콜**로 통합합니다. `path`/`sheet`/`cover`/`finish`를 하나의 프로토콜에서 모두 노출하고, 화면 전환 메서드(`push`/`pop`/`present`/`dismiss`)는 프로토콜 확장(protocol extension)으로 기본 구현을 제공합니다.

### 1.1 후보 비교

| 후보 | 형태 | 평가 |
| :--- | :-------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **(a) 단일 프로토콜** ✅ | `path`/`sheet`/`cover`/`finish`를 하나의 `Coordinator` 프로토콜에 통합 + extension 기본 구현 | 채택이 한 줄로 간결하며 모든 Coordinator가 동일한 형태를 가집니다. **단점**: 사용하지 않는 Sheet/Cover/finish도 타입을 채워야 합니다(`Never`/빈 enum). |
| (b) 능력 4분할 | 능력별 프로토콜(Navigating/Finishable/Sheet/Cover) + 조합 `typealias` | ISP(Interface Segregation Principle)를 준수하며, 존재하지 않는 능력의 호출을 컴파일 타임에 차단합니다. **단점**: 능력 조합에 대한 인지 부하, `typealias` 관리의 복잡성, 여러 `associatedtype`으로 인한 제네릭 및 `existential` 타입 사용의 어려움이 있습니다. |
| (c) 베이스 클래스 상속 | `class Coordinator` 상속 | `@Observable`의 상속 추적 기능이 깨지며, 단일 상속을 Coordinator에 사용하게 됩니다. |

### 1.2 선택 근거 — 왜 능력 분리를 채택하지 않았는가

초기에는 (b) 능력 4분할 방식을 고려했으나, 최종적으로 **(a) 단일 프로토콜 방식으로 변경**했습니다. 그 근거는 다음과 같습니다.

*   **우리 팀 규모에서는 능력 분리의 이득보다 비용이 더 큽니다.** 유일한 실질적인 이득은 "존재하지 않는 능력 호출을 컴파일 타임에 차단"하는 것이지만, 그 대가로 (1) 화면마다 능력 조합을 선택하는 인지 부하, (2) 4개에 달하는 `typealias` 관리, (3) 여러 능력에 흩어진 `associatedtype`으로 인해 `any TabCoordinator`와 같은 `existential` 및 제네릭 처리가 까다로워지는 비용이 **항상** 발생합니다. 중간 규모(20~30개 화면)의 3인 팀에서 컴파일 타임 차단 이득이 이러한 비용을 압도할 만큼 크지 않습니다.
*   **실전에서의 성공 사례 두 가지를 확인했습니다.** 단일 프로토콜 방식(Kokkok/HGDGDS-iOS의 `Coordinatorable`)과 Route 배열 통합 방식(TCACoordinators) 모두 **능력 분리 없이** "화면은 Coordinator를 알지 못한다"는 목표를 달성하며 실제 프로덕션에서 안정적으로 작동하고 있습니다. 능력 분리는 "타입 엄격성" 측면의 선택지일 뿐, Coordinator 패턴 구현에 필수적인 요소는 아닙니다.
*   **단순함과 일관성이 본 프로젝트 조건에 더 적합합니다.** 모든 Coordinator가 `Coordinator` 프로토콜 채택 한 줄로 동일한 형태를 가지므로 인수인계 및 온보딩이 쉽고, `typealias` 및 조합 결정 과정이 사라집니다.
*   **(c) 베이스 클래스 회피**: 단일 상속 자원 소모 및 `@Observable` 상속 추적 기능의 제약 때문에 베이스 클래스 상속 방식은 고려하지 않았습니다.

> **포기한 것**: 단일 프로토콜 방식은 "push만 사용하는 화면도 `sheet`/`cover`/`finish` API를 표면에 가진다"(ISP 위반)는 점과 "존재하지 않는 능력 호출을 컴파일 타임에 막을 수 없다"는 점을 받아들입니다. 다만 미사용 메서드는 프로토콜 확장으로 기본 구현되어 있으므로 런타임 비용은 0이며, 실질적인 불편함은 작다고 판단했습니다.

### 1.3 Coordinator 프로토콜 정의 (SoT: Single Source of Truth)

이 정의가 Coordinator 프로토콜의 단일 출처(Single Source of Truth)입니다. 다른 문서들은 여기를 참조합니다.

```swift
@MainActor
public protocol Coordinator: AnyObject {
    associatedtype Route: Hashable
    associatedtype Sheet: Identifiable = Never
    associatedtype Cover: Identifiable = Never
    associatedtype Output = Void

    var path: [Route] { get set }
    var sheet: Sheet? { get set }
    var cover: Cover? { get set }
    var finish: FlowFinish<Output> { get }
}

public extension Coordinator {
    func push(_ route: Route) { path.append(route) }
    func pop() { _ = path.popLast() }
    func popToRoot() { path.removeAll() }
    func present(_ sheet: Sheet) { self.sheet = sheet }
    func dismissSheet() { sheet = nil }
    func present(cover: Cover) { self.cover = cover }
    func dismissCover() { cover = nil }
}

// 사용하지 않는 Sheet/Cover를 Never로 채우기 위한 보조 채택
extension Never: Identifiable { public var id: Never { self } }
```

> **`FlowFinish<Output>`**: 자식 Coordinator가 부모 Coordinator에게 결과를 **1회** 보고하는 표준 채널입니다(클로저 콜백 패턴의 정형화). 부모는 `bind` 메서드로 연결하고, 자식은 `finish(.completed(output))` 또는 `finish(.cancelled)`로 신호를 발생시킵니다. 이중 발사 및 `bind` 누락은 라이브러리가 방지합니다. 이는 화면 전환 의도(NavigationEffect)와는 **별개의 "Flow 종료" 신호**입니다. 결과 전달 패턴은 역사적으로 클로저, 델리게이트, [[AsyncStream-Combine-비동기-기술-선택-가이드|AsyncStream]]의 세 가지가 있었으며, `FlowFinish`는 일회성 결과 전달에 가장 자연스러운 **클로저 변형**을 택한 것입니다.

### 1.4 사용하지 않는 능력의 처리

단일 프로토콜 방식이므로 모든 Coordinator는 `Route`/`Sheet`/`Cover`/`Output`/`finish`를 형식상 채워야 합니다.

| 사용하지 않는 것 | 채우는 방법 |
| :--------------- | :---------------------------------------------------------------------------------------------------------------- |
| Sheet / Cover | `associatedtype Sheet = Never` (기본값)으로 지정하고, `var sheet: Never? = nil`로 구현합니다. 컴파일러가 기본 추론하므로 선언을 생략할 수 있습니다. |
| finish (탭 Root 등 닫히지 않는 Coordinator) | `let finish = FlowFinish<Void>()`를 보유하되, 실제로는 발사하지 않습니다(사용되지 않는 프로퍼티). |
| Output | `associatedtype Output = Void` (기본값)으로 지정합니다. |

이는 능력 분리의 "사용하지 않으면 채택하지 않음" 원칙이 사라진 대가입니다. 대신 모든 Coordinator의 채택 선언과 형태가 일관성을 가집니다.

## 2. Coordinator 채택 컨벤션

`Coordinator` 프로토콜은 내비게이션 상태만을 요구하는 전제이며, 그 위에 본 프로젝트가 추가하는 컨벤션은 세 가지입니다. 의존성 주입, Store 생성, Effect 구독의 구체적인 코드는 "[[세-축-통합-Coordinator-MVI-DI|세 축 통합 문서]]"에서 다룹니다.

| 컨벤션 | 내용 |
| :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **deps 주입** | Coordinator는 `init(deps: AppDependencies)`로 의존성 번들을 받고, **자식 Coordinator를 생성할 때 동일한 `deps`를 그대로 전달합니다.** 전역 가변 상태는 두지 않으며, 번들 하나를 통한 pass-through 방식을 사용합니다([[의존성-주입-DI-번들-주입-선택-적용|DI 주입]] 문서 참조). 세션/플로우 스코프 의존성은 해당 Coordinator가 추가로 보유합니다. |
| **Store factory** | Coordinator는 `makeXxxStore()` 팩토리 메서드를 통해 화면의 `Store`를 생성하고, **그 안에서 `NavigationEffect` 구독 Task를 시작합니다.** ([[MVI-아키텍처-선택-테스트-강력함-경량-A|경량 A]]의 `Store`는 [[MVI-아키텍처-선택-테스트-강력함-경량-A|MVI 경량 A]] 문서에, 통합 형태는 "세 축 통합 문서"에서 다룹니다.) |
| **MARK 구획** | Coordinator 코드는 `Capabilities / Dependencies·Lifecycle / Store Factories / Effect Routing / Flow Control(Deep Link 등)`으로 구획합니다. |

```swift
@Observable @MainActor
final class HomeCoordinator: Coordinator {
    // MARK: - Capabilities (Coordinator 프로토콜 요구)
    var path: [HomeRoute] = []
    var sheet: HomeSheet? = nil
    var cover: HomeCover? = nil
    let finish = FlowFinish<Void>()          // 탭 루트라 미사용(죽은 프로퍼티)

    // MARK: - Dependencies / Lifecycle  (deps 주입)
    private let deps: AppDependencies
    init(deps: AppDependencies) { self.deps = deps }   // ← 부모가 주입
    private var effectTasks: [Task<Void, Never>] = []
    deinit { effectTasks.forEach { $0.cancel() } }

    // MARK: - Store Factories
    func makeRootStore() -> HomeStore {
        let store = HomeStore(
            HomeState(),
            reduce: homeReducer(repo: deps.homeRepo)   // ← 주입받은 번들에서 DI 접합
        )
        observe(store)                       // NavigationEffect 구독 시작
        return store
    }

    // MARK: - Effect Routing (NavigationEffect → Coordinator)
    private func observe(_ store: HomeStore) {
        let task = Task { @MainActor [weak store, weak self] in
            guard let store else { return }
            for await nav in store.navigationEffects { self?.handle(nav) }   // Pull 구독
        }
        effectTasks.append(task)
    }
    private func handle(_ nav: HomeNav) {
        switch nav { case .goToDetail(let id): push(.detail(id)) }
    }

    // MARK: - Flow Control (Deep Link 등)
    // path = [.posts, .post(42), .comments(7)]  처럼 스택 통째 교체
}
```

전체 통합 골격(Store/Effect 정의, View, 테스트)은 "세 축 통합 문서"에 있습니다.

### 2.1 컴파일러가 강제할 수 없는 부분

`Coordinator` 프로토콜은 내비게이션 상태만을 요구할 뿐, **deps 주입, Store factory, MARK 구획은 강제하지 못합니다.** 이러한 누락 위험은 **코드 리뷰 체크리스트 및 화면 템플릿**으로 보완합니다(3인 팀 및 인수인계를 대비한 가벼운 일관성 유지 장치). 더 강력한 강제 장치(SwiftLint 룰 등)는 화면 수가 25개를 넘거나 팀에 인원이 추가될 때 재검토합니다.

> **주입 방식 확정 경로**: 한때 "주입 `init`이 DI 방식(직접 주입 vs 컨테이너)에 종속되어 미결"이었으며 전역 컨테이너 방식으로 잠시 기울기도 했으나, 최종적으로 **번들 주입(`init(deps:)`) 방식으로 확정**되었습니다([[의존성-주입-DI-번들-주입-선택-적용|DI 주입]] 문서 참조). Coordinator는 `deps`를 받아 자식에게 전달하고, 전역 가변 상태를 두지 않습니다(로거 및 애널리틱스 제외).

## 3. 범위 — 무엇을 어디서 다루는가

본 문서는 Coordinator의 **형태와 채택 컨벤션**만을 결정합니다. 나머지 관련 주제는 다음 문서들에서 다룹니다.

| 주제 | 관련 문서 |
| :--- | :---------------------------------------------------------------- |
| 화면 내부 상태/비동기 (경량 A `reduce`·Effect·자체 L3 `TestStore`) | [[MVI-아키텍처-선택-테스트-강력함-경량-A|MVI 경량 A]] |
| 의존성 조립 (`AppDependencies` 번들 주입, 스코프, 로거/애널리틱스 전역) | [[의존성-주입-DI-번들-주입-선택-적용|의존성 주입(DI) 전략: 번들 주입 선택 및 적용]] |
| 세 축(Coordinator, MVI, DI)이 한 화면에서 맞물리는 전체 골격·View·테스트 | "세 축 통합 문서" (추후 작성 예정) |

`Route`/`Sheet`/`Cover` enum 설계, 자식 Coordinator 보유 및 해제, 딥링크와 같은 세부 사항은 0.2~0.3절에서 정리한 원칙(의존성 하향·결과 상향·재귀 트리 / 상태 = 화면·`path` 교체를 통한 딥링크)을 따르며, 구체적인 코드는 "세 축 통합 문서"의 화면 예시로 충분합니다.
