# 의존성 주입([[세-축-통합-Coordinator-MVI-DI|DI]]) 전략: 번들 주입 선택 및 적용

## 0. 요약

팀 iOS 프로젝트의 의존성 주입([[세-축-통합-Coordinator-MVI-DI|DI]]) 전략으로 **번들 주입** 방식을 채택합니다. `AppDependencies` 구조체를 Composition Root에서 한 번 조립하고, [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] 트리를 통해 의존성을 전달합니다. 로거와 애널리틱스 등 횡단 관심사에는 경량 전역 Facade를 사용하며, 전역 가변 상태를 두지 않는 것을 원칙으로 합니다.

| 항목 | 내용 |
|---|---|
| **결정** | **번들 주입** — `AppDependencies` struct를 Composition Root에서 한 번 조립해 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] 트리로 주입. 전역 가변 상태 없음. |
| 횡단 인프라 | **로거·애널리틱스만 작은 전역 facade** (상태 없는 write-only이므로 테스트에 영향 없음). |
| Scope | **소유 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]의 수명** (세션 = 로그인 Coordinator, 플로우 = 플로우 Coordinator). |
| 트리 밖 접근 | Share Extension은 별도 프로세스에 자체 DI, 딥링크는 루트 라우팅으로 처리. |
| 핵심 근거 | 테스트 격리·병렬성, 컴파일 안전성, 레이어 청결성, Scope의 자연스러운 처리. 번들 주입으로 전역의 유일한 강점(깊이 무관)을 1개 파라미터로 해결. |
| 대안 | 트리 밖 접근이 잦아지거나 Scope 처리가 복잡해지면 전역 컨테이너(자체 경량 Factory)를 재평가. |

---

## 1. 번들 주입 결정

의존성 주입([[세-축-통합-Coordinator-MVI-DI|DI]]) 방식으로 **번들 주입**을 채택합니다.
의존성을 `AppDependencies` 구조체로 묶어 **Composition Root(`@main`/`AppCoordinator`)에서 한 번 조립**하고, [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] `init`을 통해 하위 트리로 전달합니다. 전역 가변 `Container.shared`와 같은 방식은 사용하지 않습니다.
단, **로거 및 애널리틱스**와 같이 상태가 없는 횡단 인프라는 작은 전역 Facade 형태로 사용합니다. 이 결정은 이전에 논의되었던 "자체 경량 Factory(전역 Container)" 방식을 **대체**하며, 전역 컨테이너는 특정 상황(트리 밖 접근의 빈번함 등)의 **대안**으로 간주합니다.

---

## 2. 주입 방식의 장점 (vs 전역 컨테이너)

번들 주입 방식은 전역 컨테이너 방식과 비교했을 때 다음과 같은 장단점을 가집니다.

| 항목 | **주입 (번들/개별)** | 전역 컨테이너 |
|---|---|---|
| 전역 가변 상태 | ✅ 없음 | ❌ 있음 (Service Locator) |
| Pass-through | ⚠️ 있음 (번들이면 1개 파라미터) | ✅ 없음 (깊이 무관) |
| 의존성 가시성 | ✅ 명시적 (init 시그니처) | ❌ 숨겨짐 |
| 컴파일 안전성(완전성) | ✅ 빠뜨리면 컴파일 에러 | ⚠️ 미등록 시 런타임 크래시 발생 가능 |
| 테스트 격리 | ✅ 자동 (각 테스트에서 Mock 주입) | ⚠️ `shared` 인스턴스 누수로 인해 `.serialized` 등 수동 격리 필요 |
| 테스트 병렬성 | ✅ | ⚠️ 직렬화 필요 |
| 레이어 모듈화 | ✅ 자연스러움 (프로토콜만 전달) | ⚠️ Infra 레이어 누수 위험 |
| Scope 처리 | ✅ [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] 수명에 자연스럽게 일치 | ⚠️ 기본적으로 App-lifetime, Scope 추가 시 복잡도 증가 |
| 트리 밖 접근 | 루트 라우팅 또는 번들 | ✅ 어디서든 직접 접근 가능 |
| 배선 복잡도 | 수동 배선 (규모가 커지면 증가) | ✅ 적음 (등록 한 곳) |
| 되돌리기 비용 | — | 전역에서 주입으로 전환 시 **높음** / 주입에서 전역으로 전환 시 **낮음** |
| 테스트 강력함 1순위 | ✅ 강함 | ⚠️ 약함 (런타임 의존성으로 인한) |

전역 컨테이너가 주입 방식보다 유리한 점은 `pass-through`가 없고 트리 밖 접근이 용이하다는 두 가지뿐입니다. 그러나 우리 애플리케이션에서는 이 두 가지 모두 큰 문제가 되지 않습니다.
*   `pass-through`는 **번들 주입으로 하나의 파라미터**(`AppDependencies`)로 줄어듭니다. 따라서 호출 깊이와 무관하게 의존성 전달이 간결해집니다.
*   트리 밖 접근의 경우, **Share Extension은 별도 프로세스로 자체적인 DI**를 구성하며, **딥링크는 루트 라우팅**으로 처리됩니다. 남는 횡단 인프라는 로거와 애널리틱스뿐이며, 이들은 작은 전역 Facade로 관리 가능합니다. 비즈니스 로직 의존성에서는 트리 밖 접근이 거의 발생하지 않습니다.

반면 주입 방식은 프로젝트의 최우선 가치인 **테스트 강력함** (참고: [[MVI-아키텍처-선택-테스트-강력함-경량-A]])과 정확히 일치하는 테스트 격리, 병렬성, 컴파일 안전성, 레이어 청결성, Scope 처리, 의존성 가시성 등의 장점을 제공합니다. 이러한 이유로 주입 방식이 더 우위에 있습니다.

> **오해 교정**: "앱이 커지면 결국 전역을 사용하게 된다"는 관점은 오히려 역설적입니다. [[모듈화-전략-레이어-기능-하이브리드|모듈화]], 테스트, Scope 관리에 대한 요구사항이 커질수록 전역(God Singleton) 방식은 더욱 비효율적입니다. "Scope가 필요하면 전역"이라는 주장도 반대입니다. 전역은 기본적으로 App-lifetime Scope를 가지므로 Scope 관리에 불리하며, **[[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]의 수명**이 세션/플로우 Scope를 자연스럽게 처리하는 데 더 적합합니다. 또한, 수동 생성자 주입 자체는 이미 "경량 컴파일타임 DI" 입니다. Swift 타입 시스템이 컴파일 완전성 검사를 자동으로 제공하기 때문입니다.

---

## 3. 번들 주입 구현

### 3.1 AppDependencies와 Composition Root

의존성은 Domain 프로토콜로 선언된 `AppDependencies` 구조체로 묶습니다. 구현체(`~Impl`)의 조립은 앱 모듈의 Composition Root(`MyApp`)에서 한 번만 수행됩니다.

```swift
// 의존성 묶음 — Domain 프로토콜로 선언 (구현체 아님)
struct AppDependencies {
    let homeRepo: HomeRepository
    let userRepo: UserRepository
    let savedLinks: SavedLinksRepository
}

// Composition Root (App 모듈) — 한 번만 조립. 구체 impl(Infra)은 여기서만 보임
@main
struct MyApp: App {
    @State private var root: AppCoordinator
    init() {
        let api = APIClientImpl()
        let deps = AppDependencies(
            homeRepo:   HomeRepositoryImpl(api: api),
            userRepo:   UserRepositoryImpl(api: api),
            savedLinks: SavedLinksRepositoryImpl(appGroup: "group.com.ourapp")
        )
        _root = State(initialValue: AppCoordinator(deps: deps))
    }
    var body: some Scene { WindowGroup { RootView(coordinator: root) } }
}
```

**레이어 [[모듈화-전략-레이어-기능-하이브리드|모듈화]] 정합**: `AppDependencies`는 Domain 프로토콜만을 참조하며, 구체적인 구현체 조립은 App 모듈(Composition Root)에서만 이루어집니다. 이로써 Presentation/Feature 레이어는 프로토콜만 보고 Infra 레이어에 대한 직접적인 의존성을 가지지 않습니다.

### 3.2 Coordinator 주입 및 자식 전달

[[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]는 `init`을 통해 `AppDependencies` 번들을 주입받고, 이를 다시 자식 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]에게 전달합니다.

```swift
@Observable @MainActor
final class HomeCoordinator: Coordinator {
    private let deps: AppDependencies
    init(deps: AppDependencies) { self.deps = deps }      // ← 부모가 주입

    func makeRootStore() -> HomeStore {
        let store = HomeStore(HomeState(), reduce: homeReducer(repo: deps.homeRepo))
        observe(store)
        return store
    }
    func showDetail(_ id: Item.ID) {
        let child = DetailCoordinator(deps: deps)          // ← 같은 번들 그대로 전달 (1개 파라미터)
        // present/push child ...
    }
}
```

이 방식으로 `pass-through` 파라미터가 `deps` 하나로 줄어들며, 의존성 전달의 깊이와 무관하게 간결함을 유지합니다.

### 3.3 Store/Reduce 생성자 주입 (MVI 경량 A)

Reduce 및 Store는 기존 [[MVI-아키텍처-선택-테스트-강력함-경량-A]]의 "경량 A 방식"과 같이 생성자 주입 방식을 그대로 유지합니다. 의존성은 순수 Reduce 팩토리 함수의 인자로만 전달됩니다.

```swift
// reduce는 repo를 인자로만 받음 (순수 시그니처 유지)
func homeReducer(repo: HomeRepository) -> (inout HomeState, HomeAction) -> Effect<HomeAction, HomeNav> { ... }
let store = HomeStore(HomeState(), reduce: homeReducer(repo: deps.homeRepo))
```

특히 순수 Reduce (L1 테스트)는 의존성을 직접 호출하지 않으므로, DI 메커니즘을 전혀 거치지 않습니다.

### 3.4 테스트와 Preview

모든 테스트는 전역 의존성을 사용하지 않고 각자 Mock 의존성을 주입하므로, 자동으로 격리되고 병렬 테스트에 안전합니다. 별도의 격리 장치가 필요 없습니다.

```swift
// ① 순수 reduce (L1) — DI·mock 무관 (의존성 호출 안 함)
@MainActor @Test func home_refreshResponse_reduces() {
    var s = HomeState()
    _ = homeReducer(repo: DummyRepo())(&s, .refreshResponse([.fixture]))
    #expect(s.items == [.fixture])
}

// ② 비동기 Effect (L2/L3) — Mock Repository 직접 주입. 전역 없음
@MainActor @Test func home_refresh_flow() async {
    let store = TestStore(HomeState(), reduce: homeReducer(repo: MockHomeRepo(items: [.fixture])))
    await store.send(.refresh) { $0.isLoading = true }
    await store.receive(.refreshResponse([.fixture])) { $0.items = [.fixture]; $0.isLoading = false }
    store.finish()
}

// ③ Coordinator — Mock Dependencies 주입. 전역을 건드리지 않으므로 병렬 안전 (.serialized 불필요)
@MainActor @Test func coordinator_flow() async {
    let deps = AppDependencies(homeRepo: MockHomeRepo(items: [.fixture]), userRepo: /* ... */, savedLinks: /* ... */)
    let coord = HomeCoordinator(deps: deps)
    // ...
}

// ④ Preview — 헬퍼가 Mock Dependencies 조립
#Preview {
    AppCoordinator(deps: .preview).start()    // .preview = Mock 의존성 번들
}
```

---

## 4. Scope와 Coordinator 수명

세션(Session)이나 플로우(Flow)와 같은 특정 구간의 Scope는 **해당 구간을 소유한 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]의 수명**에 따라 자동 처리됩니다. Scope가 필요한 의존성을 해당 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]에 얹고 자식에게 주입하면, 의존성의 생성, 공유, 파괴가 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]의 라이프사이클을 따릅니다.

```swift
// 세션 Scope = 로그인 Coordinator의 수명
@Observable @MainActor
final class LoggedInCoordinator: Coordinator {
    private let session: AuthSession            // ← 세션 Scope 의존성, 이 coordinator가 소유
    private let deps: AppDependencies
    init(session: AuthSession, deps: AppDependencies) { self.session = session; self.deps = deps }
    func makeHomeStore() -> HomeStore {
        HomeStore(HomeState(), reduce: homeReducer(repo: deps.homeRepo))   // 자식들이 session 공유
    }
}

// Root = 로그인/로그아웃에서 생성/파괴
final class RootCoordinator: Coordinator {
    private let deps: AppDependencies
    var loggedIn: LoggedInCoordinator?
    func didLogin(_ token: Token) { loggedIn = LoggedInCoordinator(session: AuthSession(token), deps: deps) }
    func didLogout()              { loggedIn = nil }    // 세션 자동 해제 (다음 로그인 = 새 세션)
}
```

플로우 Scope 또한 동일한 방식으로 처리됩니다. 플로우 Coordinator가 `CheckoutContext` 등을 소유하고, `finish` 시 부모 Coordinator가 이를 해제하면 의존성도 함께 해제됩니다. Scope 경계가 곧 Coordinator의 생성/해제이므로, 별도의 Scope 관리 장치가 필요 없습니다.

> Scope가 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] 경계와 일치하지 않거나(예: 형제 플로우 간 공유), Scope 및 의존성이 복잡하게 얽혀 Lazy 로딩이나 캐싱 등이 필요해지는 경우, 그때 전역 컨테이너 또는 [[세-축-통합-Coordinator-MVI-DI|DI]] 도구의 도입을 재평가할 수 있습니다. 그러나 세션 및 몇 개의 플로우 수준에서는 위의 패턴만으로도 충분하며 더 명확합니다.

---

## 5. 로거 및 애널리틱스: 경량 전역 Facade

앱 어디서든 `init`을 거치지 않고 직접 호출해야 하는 횡단 인프라(Cross-cutting Concern)는 주입 방식이 번거로울 수 있습니다. **로거 및 애널리틱스는 작은 전역 Facade** 형태로 두되, 비즈니스 로직에 직접적인 영향을 미치는 Repository와는 절대 섞지 않습니다.

```swift
enum Log {
    nonisolated(unsafe) static var backend: LogBackend = OSLogBackend()
    static func debug(_ m: @autoclosure () -> String) { backend.log(.debug, m()) }
}
enum Analytics {
    nonisolated(unsafe) static var backend: AnalyticsBackend = NoopAnalytics()
    static func track(_ e: Event) { backend.track(e) }
}
```

**왜 로거/애널리틱스는 전역이어도 되는가?** [[세-축-통합-Coordinator-MVI-DI|DI]]에서 전역 방식이 비판받는 주요 이유(테스트 격리 문제, 숨겨진 의존성, Mock 어려움)는 로거와 애널리틱스에는 거의 적용되지 않습니다.
*   **상태 없음 (Write-only)**: 로그 및 이벤트는 단순히 정보를 기록하고 잊는(fire-and-forget) 방식이므로, 비즈니스 로직의 상태를 가지지 않습니다.
*   **결과에 영향 없음**: Reduce의 상태 전이나 Store의 결정성에 영향을 미치지 않습니다. L1, L2, L3 테스트에서 로거/애널리틱스의 동작을 신경 쓸 필요가 없습니다.
*   **테스트에서 무시 가능**: 기본적으로 `no-op` 백엔드를 사용하며, 필요 시 `Log.backend = SpyBackend()`와 같이 Mock 백엔드로 교체하여 동작을 확인할 수 있습니다.

즉, 로거와 애널리틱스는 "테스트 강력함"을 **해치지 않는** 종류의 전역입니다. 반면, **Repository와 같이 상태를 가지거나 비즈니스 로직 결과에 영향을 미치는 의존성은 절대 전역에 두지 않습니다.** 전역으로 허용되는 표면은 로거와 애널리틱스 두 가지로만 제한합니다.

---

## 6. 트리 밖 접근 처리

트리 밖에서 의존성에 접근해야 하는 진입점들은 다음과 같이 처리합니다.

| 진입점 | 처리 방식 | 전역 컨테이너 필요 여부 |
|---|---|---|
| **Share Extension** (인스타그램 공유 등) | **별도 타깃 및 별도 프로세스** → 메인 앱의 의존성 트리가 닿지 않음. 자체 Composition Root와 App Group 공유 저장소(`UserDefaults(suiteName:)`/파일/DB)를 통해 데이터 저장 및 접근. 메인 앱은 `savedLinks` Repository를 통해 읽음. | ❌ (프로세스 분리) |
| **딥링크 / 유니버설 링크** | `onOpenURL`이 **루트([[Coordinator-아키텍처-설계-및-컨벤션|AppCoordinator]], `AppDependencies` 보유)** 로 진입 → [[Coordinator-아키텍처-설계-및-컨벤션|AppCoordinator]].handle(url:)`에서 주입된 의존성을 활용하여 라우팅(`path` 구성). 진입점이 소수에 한정됨. | ❌ (루트 라우팅) |
| **푸시 알림 / 위젯 / App Intents** | 딥링크와 동일하게 처리 — 루트로 라우팅하거나 각 익스텐션의 자체 DI를 구성. | ❌ |

이처럼 트리 밖으로 보이는 진입점들도 실제로는 "별도 프로세스" 또는 "루트 진입"으로 처리되어 전역 컨테이너를 도입할 필요가 없습니다.

---

## 7. 대안: 전역 컨테이너 (경량 Factory)

트리 밖 접근이 **실제로 빈번**해지거나, 수동 의존성 배선이 관리하기 어려워지는 경우, 전역 컨테이너를 대안으로 고려할 수 있습니다. 핵심 특징은 다음과 같습니다.

*   **형태**: 의존성을 `Container`의 **프로퍼티 + KeyPath**로 선언합니다 (`resolve(\.homeRepo)`). 이는 딕셔너리 기반의 `register/resolve` 방식이 아닌 프로퍼티 방식이므로, **미등록 시 컴파일 에러**를 발생시켜 런타임 크래시를 방지할 수 있습니다. **전역 접근** 방식으로 깊이에 무관하게 의존성을 가져올 수 있습니다.
*   **레이어 정합 (방법 A)**: `Container` 선언은 Domain 레이어만 의존하도록 유지하고(기본값을 `fatalError`로 미등록 표식으로 사용), 구체적인 구현체 등록은 Composition Root인 `Container.live()`에서 수행합니다. 이를 통해 Presentation 레이어가 Infra 레이어에 전이 의존하지 않도록 합니다.
*   **트레이드오프**: 전역 가변 상태가 부활하여 테스트 시 `.serialized`와 같은 수동 격리 작업이 필요하며, `fatalError`로 등록 완전성 검사(부팅 시 런타임 확인) 및 Service Locator 성격의 단점이 다시 발생합니다. 이는 2절에서 주입 방식을 택한 주된 이유입니다.

**대안으로 전환하는 트리거**:
*   트리 밖 접근(딥링크 외 Ad hoc 전역 서비스 호출)이 다수의 화면에서 반복적으로 발생하는 경우.
*   Scope 및 의존성 그래프의 라이프사이클이 Coordinator 경계만으로는 해결하기 어려운 복잡한 경우가 다수 발생하는 경우.
*   수동 의존성 배선이 실제로 관리 불가능할 정도로 의존성 주입이 폭발적으로 증가하는 경우.

현재는 코드 구현 초기 단계이므로 전환 비용은 낮습니다 (주입 ↔ 전역 컨테이너는 Composition Root의 국소 변경으로 가능합니다). 하지만 **주입 방식에서 전역 컨테이너로의 전환은 쉽지만, 전역 컨테이너에서 주입 방식으로의 전환은 어렵습니다.** 따라서 불확실성이 있을 때는 주입 방식으로 시작하는 것이 추후 옵션을 더 많이 보존하는 방법입니다.

---

## 8. 결정 요약

| # | 결정 내용 | 핵심 근거 |
|---|---|---|
| 1 | [[세-축-통합-Coordinator-MVI-DI|DI]] = **번들 주입** 채택 | 전역 방식이 포기하는 테스트 격리/병렬성, 컴파일 안전성, 레이어 청결성, Scope 처리 능력을 회수. 프로젝트 1순위 가치(테스트 강력함)에 부합. |
| 2 | 전역 가변 `Container` **불채택**(대안으로 강등) | Service Locator 패턴 및 런타임 의존성으로 인해 테스트의 결정성 및 가시성 저해. |
| 3 | `AppDependencies` **번들** 주입 | `pass-through` 파라미터를 1개로 최소화하여 의존성 전달 깊이에 무관하게 간결함 유지. |
| 4 | Scope = **[[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] 수명**으로 처리 | 세션/플로우 경계가 곧 해당 [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]]의 생성/해제로 이어져 자연스러운 Scope 관리. |
| 5 | 로거·애널리틱스 = **작은 전역 Facade** | 상태 없는 Write-only 성격으로 테스트에 영향을 주지 않음. Repository 등 비즈니스 로직 의존성과 분리. |
| 6 | 트리 밖 접근 = **별도 프로세스 / 루트 라우팅** | Share Extension은 자체 [[세-축-통합-Coordinator-MVI-DI|DI]], 딥링크는 루트 [[Coordinator-아키텍처-설계-및-컨벤션|AppCoordinator]]를 통한 라우팅으로 처리하여 전역 의존성 불필요. |
