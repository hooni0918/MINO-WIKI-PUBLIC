# 세 축 통합: [[Coordinator-아키텍처-설계-및-컨벤션]] + [[MVI-아키텍처-선택-테스트-강력함-경량-A]]([[MVI-아키텍처-선택-테스트-강력함-경량-A]]) + [[의존성-주입-DI-번들-주입-선택-적용]]([[의존성-주입-DI-번들-주입-선택-적용]])

이 문서는 프로젝트의 [[Coordinator-아키텍처-설계-및-컨벤션]], [[MVI-아키텍처-선택-테스트-강력함-경량-A]], [[의존성-주입-DI-번들-주입-선택-적용]]에 대한 최종 [[Coordinator-아키텍처-설계-및-컨벤션]] 결정과 주요 변경사항의 근거를 요약하고, Coordinator, MVI ([[MVI-아키텍처-선택-테스트-강력함-경량-A]]), DI ([[의존성-주입-DI-번들-주입-선택-적용]]) 세 가지 아키텍처 축의 통합 방안을 상세히 설명합니다.

## 1. 아키텍처 결정 요약

이 섹션에서는 각 [[Coordinator-아키텍처-설계-및-컨벤션]] 축의 핵심 결정 사항을 간략히 요약합니다.

### 1.1 Coordinator
단일 `Coordinator` 프로토콜을 사용하며 (`path`, `sheet`, `cover`, `finish` 통합), 화면 전환 메서드는 `extension`으로 기본 구현됩니다. 사용하지 않는 Sheet/Cover는 `Never`, 닫히지 않는 `finish`는 빈 `FlowFinish<Void>()`로 표현합니다. `@Observable` 클래스로, App에서 `@State`로 보유합니다 (iOS 17). `init(deps:)`로 의존성을 주입하고 자식에게 전달하며, `makeXxxStore()`에서 `Store`를 생성하고 `NavigationEffect`를 구독합니다. [[Coordinator-아키텍처-설계-및-컨벤션]]에서 더 자세한 내용을 확인할 수 있습니다.

### 1.2 MVI (경량 A)
순수 `reduce ((inout State, Action) -> Effect<Action>)` 함수와 최소한의 `Effect`/`Store` (약 40줄)를 사용합니다. 비동기 결과는 `Response Action`으로 복귀시키며, `NavigationEffect`는 [[AsyncStream-Pull-방식-Unfolding-클로저]]을 활용합니다. `State`는 단일 `Equatable` `struct`로 정의합니다. 테스트는 자체 `TestStore`를 활용하여 L1~L3 수준의 실용적인 테스트 (약 190줄)를 지원하며, L4, 시간 여행, Production `TestStore`는 도입하지 않습니다. TCA는 도입하지 않습니다. [[MVI-아키텍처-선택-테스트-강력함-경량-A]]에서 더 자세한 내용을 확인할 수 있습니다.

### 1.3 DI ([[의존성-주입-DI-번들-주입-선택-적용]])
`AppDependencies` `struct`를 Composition Root에서 한 번 조립한 뒤, Coordinator의 `init`으로 주입하고 Coordinator 트리를 따라 자식에게 전달합니다 (전역 의존성 없음). `reduce` 및 `Store`는 생성자를 통해 의존성을 주입합니다. 의존성 스코프는 소유 Coordinator의 수명 (세션/플로우)과 동일합니다. 로거, 애널리틱스와 같은 횡단 관심사에는 경량 전역 Facade를 사용합니다. 전역 컨테이너 (자체 경량 Factory)는 대안으로 고려되며, 필요 시 재평가합니다. [[의존성-주입-DI-번들-주입-선택-적용]]에서 더 자세한 내용을 확인할 수 있습니다.

### 1.4 세 축 통합
세 가지 아키텍처 축은 `Store`를 중심으로 수렴됩니다. Coordinator가 `deps`에서 의존성을 꺼내 `Store`를 생성하면, `Store`는 순수 `reduce`와 `Effect`를 처리하고, `Effect.navigate`는 Coordinator가 구독하여 화면 전환을 수행합니다. 출력 채널은 `NavigationEffect` (화면 내)와 `FlowFinish` (화면 간 1회) 두 가지입니다.

## 2. 주요 변경 사항

기존 아키텍처 통합 초안(MVI=E, DI=미결)과 비교해, 확정된 결정에 따라 세 접합부가 다음과 같이 변경됩니다.

| 접합부 | 기존 (E / DI 미결) | **확정 ([[MVI-아키텍처-선택-테스트-강력함-경량-A]] / [[의존성-주입-DI-번들-주입-선택-적용]])** |
|---|---|---|
| ① [[의존성-주입-DI-번들-주입-선택-적용]] 주입 | Coordinator에 개별 의존성 주입 | **`AppDependencies` [[의존성-주입-DI-번들-주입-선택-적용]]을 `init`으로 주입 → 자식에 그대로 전달** (전역 0개, pass-through 1개) |
| ② ViewModel | `@Observable` ViewModel + `send(intent)` + inline Task | **`Store<State, Action, Nav>`** — 순수 reduce + `Effect` |
| ③ NavigationEffect | ViewModel이 `nav.yield` 직접 | **reduce가 `Effect.navigate` 반환** → Store가 yield (순수 유지) |

이 세 가지 축은 `Store` 하나로 수렴됩니다. `Store`는 (1) 주입받은 의존성을 reduce에 묶고, (2) [[MVI-아키텍처-선택-테스트-강력함-경량-A]]의 reduce/Effect를 실행하며, (3) NavigationEffect를 Coordinator로 흘려보내는 역할을 한다.

## 3. 세 축 통합 아키텍처 다이어그램

```
[Composition Root (@main)]
   │ AppDependencies 조립 (구현체 → 프로토콜) + 로거/애너리틱스 전역 facade
   ▼
[AppCoordinator(deps)] ── 자식 strong 보유 ──▶ [HomeCoordinator(deps)]
                                            │ Coordinator: path/sheet/cover/finish
                                            │ DI: deps 주입받아 자식에 전달  ← init(deps:)
                                            │ effectTasks 보관, deinit 시 cancel
                                            │
                                            │ makeRootStore() ──┐ 1. deps.homeRepo 꺼냄
                                            │                   │ 2. Store 생성(생성자 주입)
                                            │                   │ 3. navigationEffects 구독
                                            ▼                   ▼
                                       [HomeStore = Store<State, Action, Nav>]
                                            │ 순수 reduce + Effect (경량 A)
                                            │
                          send(action) ─▶ reduce ─▶ Effect ─┬─ .run   → Task → Response Action 재진입
                                            │                ├─ .navigate(nav) → navigationEffects.yield
                                            │                └─ .none
                                            ▲                         │
[HomeRootView]                              │                         ▼
   @Environment(HomeCoordinator)            │              [Coordinator가 for await 구독]
   @State store = coordinator.makeRootStore()│                        │
   store.send(.refresh) ────────────────────┘                        ▼
                                                              push/present/dismiss

[자식 Flow Coordinator] ── finish(.completed) ──▶ [부모 Coordinator]   (FlowFinish, 화면 간 1회)
```

## 4. 접합부 상세 구현

이 섹션에서는 각 아키텍처 축의 구체적인 구현과 접합부의 동작 방식을 설명합니다.

### 4.1 접합부 ① — DI([[의존성-주입-DI-번들-주입-선택-적용]])와 Coordinator

#### 변화: Coordinator의 `deps` 수신 및 자식 전달

비즈니스 의존성은 `AppDependencies` 번들로 묶여 **Composition Root에서 한 번 조립**되고, Coordinator의 `init(deps:)`로 주입되어 트리 아래로 흐릅니다. Coordinator는 자기 factory에서 필요한 의존성을 `deps`에서 꺼내 `Store`에 주입하고, 자식 Coordinator를 만들 때 **동일한 `deps`를 그대로 전달**합니다. 이 덕분에 pass-through 파라미터는 `deps` 한 개로 제한되며, 깊이에 상관없이 유지됩니다. 전역 가변 상태는 0개입니다.

> **레이어 [[모듈화-전략-레이어-기능-하이브리드]] 정합**: `AppDependencies`는 Domain 프로토콜만 참조하고, 구현체(`~Impl`) 조립은 App 모듈에서만 이루어집니다. Presentation(Coordinator) 레이어는 프로토콜만 바라보며 Infra 레이어를 알지 못합니다.

```swift
@Observable @MainActor
final class HomeCoordinator: Coordinator {
    // MARK: - Capabilities
    var path: [HomeRoute] = []
    var sheet: HomeSheet? = nil
    var cover: HomeCover? = nil
    let finish = FlowFinish<Void>() // 탭 루트라 미사용

    // MARK: - Dependencies / Lifecycle
    private let deps: AppDependencies
    init(deps: AppDependencies) { self.deps = deps } // ← 부모가 주입
    private var effectTasks: [Task<Void, Never>] = []
    deinit { effectTasks.forEach { $0.cancel() } }

    // MARK: - Store Factories
    func makeRootStore() -> HomeStore {
        let store = HomeStore(
            HomeState(),
            reduce: homeReducer(repo: deps.homeRepo) // ← DI 접합 (주입받은 번들에서)
        )
        observe(store)
        return store
    }

    // MARK: - Effect Routing
    private func observe(_ store: HomeStore) {
        let task = Task { @MainActor [weak store, weak self] in
            guard let store else { return }
            for await nav in store.navigationEffects { // ← MVI Pull 구독
                self?.handle(nav)
            }
        }
        effectTasks.append(task)
    }

    private func handle(_ nav: HomeNav) {
        switch nav {
        case .goToDetail(let id):
            let child = DetailCoordinator(deps: deps) // ← 자식에 같은 번들 전달
            // present/push child ...
        }
    }
}
```

> **주의(메모리 캡처)**: 구독 `Task`가 `store`를 strong 캡처하면 `store`의 수명이 `Task`에 묶일 수 있습니다. `[weak store]`로 `store`를 약하게 캡처하여 이를 방지해야 합니다 ([[AsyncStream-Pull-방식-Unfolding-클로저|AsyncStream Pull 구독]]의 공통 주의점).

> **테스트 격리**: Coordinator는 mock `deps`를 주입하여 테스트합니다(`HomeCoordinator(deps: .mock)`). 전역 상태를 통하지 않으므로 **병렬 안전합니다**(`.serialized` 불필요). reduce/Store는 생성자 주입 방식이므로 `Store` 단위(특히 순수 reduce) 테스트는 DI와 무관합니다.

#### scope = Coordinator 수명

세션/플로우와 같은 구간 스코프는 **해당 구간을 소유한 Coordinator의 수명**으로 관리됩니다. `AuthSession`이나 `CheckoutContext`와 같은 스코프 의존성을 해당 Coordinator의 프로퍼티로 보유하여 자식에게 주입하면, 생성·공유·파괴가 Coordinator 라이프사이클에 맞춰집니다 (예: 로그인 시 생성, 로그아웃/`finish` 시 해제). 별도의 스코프 관리 메커니즘은 불필요합니다.

### 4.2 접합부 ② — MVI ([[MVI-아키텍처-선택-테스트-강력함-경량-A]])와 Coordinator: Store

#### 4.2.1 통합 인프라: `Store`와 `Effect`

[[MVI-아키텍처-선택-테스트-강력함-경량-A|경량 A]]의 reduce/Effect에 **navigation 채널**을 더한 형태입니다. reduce는 순수하게 `Effect` 값만 반환하며, `Store`가 그 `Effect`를 실행합니다. `.navigate`는 [[Swift-AsyncSequence-이해-및-활용|AsyncStream]]을 통해 yield하고, `.run`은 `Task`로 실행합니다.

```swift
// Effect — 비동기(run) + navigation(navigate) 두 결과를 값으로 표현
enum Effect<Action, Nav> {
    case none
    case run((@escaping (Action) -> Void) async -> Void) // 비동기 → Response Action
    case navigate(Nav)                                   // → Coordinator로 송출
}

@Observable @MainActor
final class Store<State, Action, Nav> {
    private(set) var state: State
    let navigationEffects: AsyncStream<Nav>
    private let navContinuation: AsyncStream<Nav>.Continuation
    private let reduce: (inout State, Action) -> Effect<Action, Nav>
    private var tasks: [Task<Void, Never>] = []

    init(_ initial: State, reduce: @escaping (inout State, Action) -> Effect<Action, Nav>) {
        self.state = initial
        self.reduce = reduce
        (navigationEffects, navContinuation) = AsyncStream.makeStream()
    }
    deinit { tasks.forEach { $0.cancel() }; navContinuation.finish() }

    func send(_ action: Action) {
        // log.debug("Action: \(action)") // 미들웨어 1곳 (사건 로그)
        execute(reduce(&state, action))
    }

    private func execute(_ effect: Effect<Action, Nav>) {
        switch effect {
        case .none: break
        case .navigate(let nav): navContinuation.yield(nav) // ← 접합부 ③
        case .run(let op):
            let task = Task { [weak self] in await op { self?.send($0) } }
            tasks.append(task)
        }
    }
}
```

> 취소·병합은 `Effect.run` 내부의 `Task`로 처리합니다 (취소는 `task.cancel()` 한 줄, 병렬은 `async let`/`TaskGroup`). `Effect` 타입을 별도로 만들 필요가 없는 이유는 [[MVI-아키텍처-선택-테스트-강력함-경량-A|MVI 경량 A]] 문서에서 설명합니다.

#### 4.2.2 화면 구성: State, Action, Nav, Reducer

```swift
struct HomeState: Equatable {
    var items: [Item] = []
    var isLoading = false
    var error: String? = nil
}

enum HomeAction: Equatable { // Equatable → TestStore 매칭용
    case refresh
    case refreshResponse([Item]) // Response Action (성공)
    case refreshFailed(String)   // Response Action (실패) — Error 대신 String으로 Equatable 유지
    case tapDetail(Item.ID)
}

enum HomeNav: Equatable {
    case goToDetail(Item.ID)
}

typealias HomeStore = Store<HomeState, HomeAction, HomeNav>

// 순수 reduce — 의존성은 Effect.run 안에서만 사용 (reduce 시그니처는 순수 유지)
func homeReducer(repo: HomeRepository) -> (inout HomeState, HomeAction) -> Effect<HomeAction, HomeNav> {
    { state, action in
        switch action {
        case .refresh:
            state.isLoading = true
            return .run { send in
                do    { await send(.refreshResponse(try await repo.fetch())) }
                catch { await send(.refreshFailed(error.localizedDescription)) }
            }
        case .refreshResponse(let items):
            state.items = items; state.isLoading = false; return .none
        case .refreshFailed(let message):
            state.error = message; state.isLoading = false; return .none
        case .tapDetail(let id):
            return .navigate(.goToDetail(id)) // ← navigation도 Effect로 (reduce 순수)
        }
    }
}
```

#### 4.2.3 View 구현

```swift
struct HomeRootView: View {
    @Environment(HomeCoordinator.self) private var coordinator
    @State private var store: HomeStore?

    var body: some View {
        Group {
            if let store { HomeContentView(store: store) }
            else { ProgressView() }
        }
        .task { if store == nil { store = coordinator.makeRootStore() } } // ← 접합부 ②
    }
}

struct HomeContentView: View {
    let store: HomeStore
    var body: some View {
        List(store.state.items) { item in
            Button(item.title) { store.send(.tapDetail(item.id)) }
        }
        .overlay { if store.state.isLoading { ProgressView() } }
        .task { store.send(.refresh) }
    }
}
```

### 4.3 접합부 ③ — 두 출력 채널: NavigationEffect와 FlowFinish

[[MVI-아키텍처-선택-테스트-강력함-경량-A]]에서 화면이 외부로 내보내는 신호는 두 채널로, 본질이 달라 공존합니다.

| 채널 | 방향 | 메커니즘 | 빈도 | [[MVI-아키텍처-선택-테스트-강력함-경량-A]]에서 |
|---|---|---|---|---|
| **NavigationEffect** | `Store` → `Coordinator` | reduce가 `Effect.navigate` 반환 → [[Swift-AsyncSequence-이해-및-활용|AsyncStream]] yield → `for await` | 다수 | `case .tapDetail: return .navigate(...)` |
| **FlowFinish** | 자식 `Coordinator` → 부모 | `FlowFinish<Output>` + `bind` + `callAsFunction` | 1회 | 자식 flow 종료 시 `finish(.completed(data))` |

NavigationEffect는 reduce (순수 함수)에서 값으로 반환되므로 **테스트 가능하다** (reduce가 `.navigate`를 반환하는지 단언). FlowFinish는 화면 간 일회성 결과를 전달하는 목적이므로 별도로 유지됩니다.

```swift
// 자식 flow Coordinator (시트 안)
@Observable @MainActor
final class EditFlowCoordinator: Coordinator {
    var path: [EditRoute] = []
    var sheet: EditSheet? = nil
    var cover: EditCover? = nil
    let finish = FlowFinish<EditResult>() // ← 화면 간 1회성

    func complete(_ result: EditResult) { finish(result) } // 부모가 bind로 수신
}
```

## 5. [[Coordinator-아키텍처-설계-및-컨벤션]] 결정 변경 상세: MVI (E → A), DI (전역 Container vs init 주입)

이 섹션에서는 [[MVI-아ki텍처-선택-테스트-강력함-경량-A]] [[Coordinator-아키텍처-설계-및-컨벤션]]를 E 방식에서 [[MVI-아키텍처-선택-테스트-강력함-경량-A]] 방식으로 변경한 이유와, [[의존성-주입-DI-번들-주입-선택-적용]] 방식 선택의 배경을 상세히 설명합니다.

### 5.1 MVI: E → A 변경 사유

#### 5.1.1 두 방식의 구조

**E 방식** — ViewModel 내부에서 `Task`로 비동기를 실행하고 결과를 `state`에 직접 반영합니다. 순수 `reduce` 함수가 없습니다.

```swift
case .refresh:
    Task {
        state.isLoading = true
        state.items = await repo.fetch()
        state.isLoading = false
    }
```

**A 방식** — `reduce`를 ViewModel 외부의 순수 함수로 분리합니다. 비동기 작업은 직접 실행하지 않고 `Effect`로 반환하며, 비동기 결과는 `Response Action`으로 재디스패치하여 `state`를 갱신합니다.

```swift
case .refresh:
    state.isLoading = true
    return .run { send in await send(.refreshResponse(.success(await repo.fetch()))) }
case .refreshResponse(.success(let items)):
    state.items = items; state.isLoading = false
    return .none
```

#### 5.1.2 변경의 근거: 판단 기준의 전환

| 구분 | 이전 기준 | 변경 기준 |
|--- |--- |--- |
| 1순위 | 보일러플레이트 최소·가독성 | 테스트 강력함 |
| 전환 이유 | — | AI가 작성 비용을 흡수, 검증 비용은 미흡수 |

AI가 코드 작성을 대신하는 환경에서 "적게 쓰고 읽기 쉬움"의 가치는 하락했습니다. A 방식의 최대 단점인 보일러플레이트는 AI가 거의 0 비용으로 처리합니다. 반면, AI가 대신하지 못하는 영역은 "작성된 코드가 맞는지 검증"하는 테스트입니다. 따라서 1순위 기준을 테스트 강력함으로 승격했습니다.

#### 5.1.3 새 기준 적용 결과

테스트 레벨은 "어디까지 결정적으로 검증할 수 있나"를 단계로 나눈 것입니다.

*   **L1** — Action을 넣었을 때 `state`가 의도대로 바뀌는지 (순수 `reduce` 상태 전이)를 검증합니다.
*   **L2** — "버튼 → 로딩 → 데이터 도착 → 반영"처럼 비동기 결과까지 이어지는 흐름 전체를 검증합니다.
*   **L3** — L2에 더해, 단언하지 않은 `state` 변화나 누락된 `effect`가 있으면 자동으로 실패시킵니다 (부작용·누락 자동 검출).

| 테스트 레벨 | E 방식 | A 방식 |
|--- |--- |--- |
| L1 순수 reduce 상태전이 | ❌ | ✅ |
| L2 비동기 시나리오 | ❌ | ✅ |
| L3 exhaustive (부작용·누락 자동 검출) | ❌ | ✅ |

E 방식은 `reduce`가 ViewModel에 인라인되어 순수 함수가 아니므로 L1 테스트조차 불가능합니다. 검증 시 의존성 mock과 `Task` 완료 대기가 필요하며, 이는 시간 의존적이고 비결정적입니다. A 방식은 `reduce`가 순수 함수이므로 입력과 출력이 결정적입니다. mock이나 대기 없이 상태 전이를 검증하고, 비동기 결과까지 시나리오 테스트가 가능합니다. A 방식의 단점 (코드량, 가독성)은 AI가 무력화합니다.

#### 5.1.4 결론

기준이 "짧은 코드"에서 "테스트 강력함"으로 바뀌면서 A 방식의 단점은 상쇄되고 장점만 유효해져 **E 방식에서 A 방식으로 변경**합니다. 단, 풀세트가 아닌 **[[MVI-아키텍처-선택-테스트-강력함-경량-A]]** (순수 `reduce` + 최소 `Effect`/`Store` + 자체 `TestStore` 약 190줄, L1~L3까지)을 채택합니다. 이 경계를 넘는 복잡도가 발생하면 TCA 도입을 재평가합니다.

#### 5.1.5 TestStore 제작 및 테스트 진행

L2/L3 테스트는 자체 `TestStore`로 진행합니다.

##### 핵심 골격

`TestStore`는 `state`와 `reduce`를 관리하며, `effect`가 되돌린 `action`을 큐 (`pending`)에 모아두는 작은 클래스입니다. 동작은 `send` (동기 변화 + `effect` 적재), `receive` (되돌아온 `action` 처리), `finish` (잔여 `effect` 검증) 세 가지뿐입니다. (Swift Testing 기준, `Issue.record`는 실패를 보고합니다.)

```swift
@MainActor
final class TestStore<State: Equatable, Action: Equatable> {
    private var state: State
    private let reduce: (inout State, Action) -> Effect<Action>
    private var pending: [Action] = []     // effect가 보낸, 아직 안 받은 action
    var exhaustive = true                   // L3 엄격성 on/off

    init(_ initial: State, reduce: @escaping (inout State, Action) -> Effect<Action>) {
        self.state = initial; self.reduce = reduce
    }

    // 내가 보내는 action: 동기 변화 단언 + effect는 pending에 적재
    func send(_ action: Action, expect: (inout State) -> Void) async {
        var expected = state; expect(&expected)
        let effect = reduce(&state, action)
        assertState(expected)                                   // L3 ① 완전 일치
        await effect.operation { [weak self] in self?.pending.append($0) }
    }

    // effect가 되돌린 action: pending에서 찾아 처리 + 단언
    func receive(_ action: Action, expect: (inout State) -> Void) async {
        guard let i = pending.firstIndex(of: action) else {
            Issue.record("기대한 effect action 없음: \(action)"); return
        }
        pending.remove(at: i)
        var expected = state; expect(&expected)
        _ = reduce(&state, action)
        assertState(expected)
    }

    // 마무리: 안 받은 effect가 남으면 실패
    func finish() {
        if exhaustive, !pending.isEmpty { Issue.record("미처리 effect 남음: \(pending)") }  // L3 ②
    }

    private func assertState(_ expected: State) {
        guard exhaustive, state != expected else { return }
        Issue.record("state 불일치:\n\(fieldDiff(expected, state))")   // 필드 diff는 Mirror 기반 헬퍼
    }
}
```

##### 진행: send, receive, finish 3단계

의존성은 mock으로 즉시 응답하도록 하여 대기 시간을 없앱니다. 따라서 매번 같은 결과 (결정적)를 보장합니다.

```swift
@Test
func refresh하면_로딩후_데이터를_반영한다() async {
    // ① 생성: mock repo를 reduce에 주입 (전역 없음)
    let store = TestStore(
        HomeState(),
        reduce: homeReducer(repo: MockHomeRepo(items: [.fixture, .fixture]))
    )

    // ② send: 사용자 action → 즉시 일어나는 동기 변화를 단언
    await store.send(.refresh) {
        $0.isLoading = true
        // items는 안 적음 = "아직 안 바뀐다"는 단언. 바뀌면 L3가 자동 실패
    }

    // ③ receive: effect가 되돌린 action → 그 시점 state 단언
    await store.receive(.refreshResponse(.success([.fixture, .fixture]))) {
        $0.items = [.fixture, .fixture]
        $0.isLoading = false
    }

    // ④ finish: 안 받은 effect가 남아 있으면 실패
    store.finish()
}
```

*   `send` = "내가 누른 버튼". 직후 동기 변화를 단언하고 `fetch effect`는 `pending`에 쌓입니다.
*   `receive` = "비동기가 끝나 되돌아온 결과". `pending`에서 찾아 처리하고 그 시점 `state`를 단언합니다.
*   `finish` = "받기로 한 것을 모두 받았나" 확인합니다.
*   동시 호출은 `async let`으로 묶어 **결과를 1개 `action`으로 합쳐** 도착 순서의 비결정성을 피하면, 같은 3단계로 검증합니다.

### 5.2 DI: 전역 Container vs init 주입

#### 5.2.1 두 방식의 구조

**전역 Container** — 의존성을 `Container.shared`에 등록하고 필요한 곳에서 `resolve`로 꺼냅니다. 깊이와 무관하게 어디서든 접근할 수 있습니다.

**init 주입** — 의존성을 `AppDependencies` 번들로 묶어 Composition Root (`@main`)에서 1회 조립하고, Coordinator의 `init`을 통해 주입하여 트리 하향으로 전달합니다. 전역 가변 상태가 없습니다.

```swift
// 1회 조립
let deps = AppDependencies(homeRepo: HomeRepositoryImpl(api: api), ...)
_root = State(initialValue: AppCoordinator(deps: deps))

// 자식에 동일 번들 전달 (파라미터 1개)
let child = DetailCoordinator(deps: deps)
```

#### 5.2.2 항목별 비교

| 항목 | init 주입 ([[의존성-주입-DI-번들-주입-선택-적용]]) | 전역 Container |
|---|---|---|
| 전역 가변 상태 | 없음 | 있음 (Service Locator) |
| 의존성 가시성 | 명시적 (init 시그니처) | 숨음 |
| 누락 시 | 컴파일 에러 | 부팅 시 런타임 크래시 |
| 테스트 격리 | 자동 (각자 mock) | `shared` 누수 → `.serialized` 수동 |
| 테스트 병렬 | 안전 | 직렬화 필요 |
| 레이어 [[모듈화-전략-레이어-기능-하이브리드]] | 자연 (프로토콜만 하향) | Infra 누수 위험 |
| scope (세션/플로우) | Coordinator 수명에 자연 처리 | 기본 `app-lifetime`, 별도 관리 기계 필요 |
| pass-through | 있음 (번들이면 1개) | 없음 (깊이 무관) |
| 트리 밖 접근 | 루트 라우팅 / 번들 | 어디서든 바로 |

#### 5.2.3 선택 근거

전역 방식이 주입 방식보다 우위에 있는 지점은 두 가지뿐이며, 두 가지 모두 본 프로젝트에서는 중요도가 낮습니다.

| 전역의 강점 | 본 프로젝트에서의 처리 |
|---|---|
| `pass-through` 없음 | 번들 1개로 축소 → 트리 깊이와 무관 |
| 트리 밖 접근 | Share Extension = 별도 프로세스 자체 DI / 딥링크 = 루트 라우팅 / 남는 횡단 인프라는 로거·애널리틱스뿐 (작은 전역 facade로 분리) |

반면, 주입 방식이 제공하는 가치 (테스트 격리·병렬, 컴파일 안전, 레이어 청결, 스코프 자연 처리)는 프로젝트의 1순위 가치인 "테스트 강력함"과 일치합니다.

#### 5.2.4 결론

전역 방식의 두 가지 강점은 [[의존성-주입-DI-번들-주입-선택-적용]]과 루트 라우팅으로 상쇄되며, 주입 방식의 강점은 1순위 가치와 정렬되므로 **init [[의존성-주입-DI-번들-주입-선택-적용]] 방식을 채택**합니다. 전환 비용 비대칭성 측면에서, 주입 방식에서 전역 방식으로의 전환은 쉽지만, 전역 방식에서 주입 방식으로의 전환은 어렵습니다. 불확실성이 있을 경우 주입 방식으로 시작하는 것이 향후 선택지를 보존하는 데 유리합니다. 트리 밖 접근이 실제로 빈번해지면 전역 컨테이너 (자체 경량 Factory)를 대안으로 재평가합니다.

## 6. 테스트 통합 (자체 L3 TestStore + deps 주입)

세 축이 테스트에서 만나는 방식입니다. **세 레이어를 다른 깊이로** 테스트합니다.

```swift
import Testing

// ① Store 시나리오 — 경량 A의 자체 L3 TestStore (가장 많은 테스트)
//    DI 무관: reducer에 mock repo를 직접 주입 (deps조차 안 거침)
@MainActor @Test func home_refresh_flow() async {
    let store = TestStore(HomeState(), reduce: homeReducer(repo: MockRepo(items: [.fixture])))
    await store.send(.refresh) { $0.isLoading = true } // 동기 변화
    await store.receive(.refreshResponse([.fixture])) { // effect 결과
        $0.items = [.fixture]; $0.isLoading = false
    }
    store.finish() // L3: 미처리 effect 없음
}

// ② navigation 검증 — reduce가 순수라 직접 호출로 단언
//    Effect는 .run(closure) 때문에 전체 Equatable 불가 → 패턴 매칭으로 검증
@MainActor @Test func tapDetail_navigates() {
    var state = HomeState()
    let effect = homeReducer(repo: MockRepo())(&state, .tapDetail("42"))
    guard case .navigate(.goToDetail(let id)) = effect else {
        Issue.record("navigate effect 기대"); return
    }
    #expect(id == "42")
}

// ③ Coordinator flow — mock deps 주입. 전역 없음 → 병렬 안전(.serialized 불필요)
@MainActor @Test func coordinator_pushes_detail() async {
    let deps = AppDependencies(homeRepo: MockRepo(items: [.fixture])) // 간략화된 mock deps
    let coord = HomeCoordinator(deps: deps)
    let store = coord.makeRootStore()
    store.send(.tapDetail("42"))
    // navigationEffects 구독으로 coord.path == [.detail("42")] 검증 (비동기 대기)
}
```

**대부분의 로직은 ① (Store L3 테스트)로 DI와 무관하게 결정적으로 검증합니다.** ①·②는 mock repository를 직접 주입하므로 DI와 무관하며, ③ 역시 mock `deps`를 주입할 뿐 **전역 상태를 통하지 않아 전부 병렬 테스트에 안전합니다** (`.serialized` 불필요). 이는 "테스트 강력함 1순위"가 DI까지 일관되게 적용된 모습입니다.

## 7. 라이프사이클

```
앱 시작
   → AppDependencies 조립 (구현체 → 프로토콜) + 로거/애너리틱스 전역 facade
   ↓
AppCoordinator(deps) 생성
   → 자식 HomeCoordinator(deps) 생성 (deps 주입·전달)
   ↓
View 표시
   → .task { store = coordinator.makeRootStore() }
   ↓
makeRootStore 실행
   → deps.homeRepo를 사용하여 Store 생성(생성자 주입)
   → navigationEffects 구독 Task를 effectTasks에 추가
   ↓
화면 동작: View → store.send(action) → reduce → Effect 실행
                                          ├ .run → Task 실행 → Response Action 재진입
                                          └ .navigate → AsyncStream yield → Coordinator handle 메서드 실행 → 화면 전환 (push/present)
   ↓
화면 이탈
   → View 해제
   → @State store 해제 (Store deinit: 내부 tasks cancel + continuation finish)
   ↓
Coordinator deinit
   → effectTasks 전부 cancel (구독 Task 종료)
```

`Store`는 `Coordinator` factory가 만들고, 그 시점에 NavigationEffect 구독을 시작합니다. `Coordinator`가 해제되면 `effectTasks`를 모두 cancel하여 `Task` 누수를 방지합니다.

## 8. 최종 결정 요약

이 표는 세 축 통합 아키텍처의 최종 결정 사항을 간략히 요약합니다.

| # | 접합부 | 확정 |
|---|---|---|
| ① | [[의존성-주입-DI-번들-주입-선택-적용]] × Coordinator | `AppDependencies` [[의존성-주입-DI-번들-주입-선택-적용]]을 `init(deps:)` 주입 → 자식 전달. reduce/Store는 생성자 주입. 전역 0 |
| ② | [[MVI-아키텍처-선택-테스트-강력함-경량-A]] × Coordinator | Coordinator factory가 `Store` 생성 + `navigationEffects` 구독 |
| ③ | 출력 채널 | `Effect.navigate`(NavigationEffect, 화면 내·다수) + `FlowFinish`(화면 간·1회) 공존 |
| 인프라 | Store/Effect | `Store<State, Action, Nav>` + `Effect`(none/run/navigate) — 순수 reduce |
| 테스트 | 통합 | Store L3 테스트(DI 무관) + Coordinator는 mock `deps` 주입(전부 병렬) |
