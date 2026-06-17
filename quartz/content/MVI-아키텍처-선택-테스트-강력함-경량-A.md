# MVI 아키텍처 선택: 테스트 강력함을 기준으로 한 경량 A 방식

이 문서는 팀 iOS 프로젝트의 MVI(Model-View-Intent) 아키텍처 레이어 선택 과정을 다룹니다. 특히 **테스트 강력함**을 최우선 기준으로, 기존에 선택했던 E 방식(`inline Task`)에서 **경량 A 방식(순수 reduce + Effect)**으로 변경한 결정과 그 배경 및 구현 범위를 설명합니다.

#### 요약

| 항목 | 내용 |
|---|---|
| 기존 결정 | E (B + NavigationEffect, inline `Task`) |
| 변경 결정 | **경량 A** (순수 reduce + `Effect` + Response Action + NavigationEffect 공통) |
| 기준 전환 | 보일러플레이트 최소 → **테스트 강력함** (AI 시대 근거) |
| 구현 범위 | **L1 ~ L3 실용** (자체 TestStore ~190줄). L3 production·L4 제외 |
| TCA | **도입 안 함**. 자체 구현이 못 미치는 지점 = TCA 재평가 트리거 |

#### MVI 후보 A와 E 선정 배경

이 결정은 3축([[세-축-통합-Coordinator-MVI-DI|세 축 통합]])([[세-축-통합-Coordinator-MVI-DI|MVI]] + [[Coordinator-아키텍처-설계-및-컨벤션|Coordinator]] + [[의존성-주입-DI-번들-주입-선택-적용|DI]]) 조합 중 MVI 레이어(화면 내부 상태를 단방향으로 관리하는 방식)의 선택입니다. 원래 5개 후보(A~E)를 검토하여 E를 선택했으나, 본 문서에서는 A와 E 두 후보를 재비교합니다.

**프로젝트 운영 조건** (아키텍처 선택에 영향을 미치는 주요 전제):

| 조건 | 내용 |
|---|---|
| 성격 | MVP → 성장 가능 앱 (3개월 내 MVP 출시) |
| 팀 | 최대 3명 유지 (확장 계획 없음) |
| 도메인 | 소셜/콘텐츠 소비 중심 (결정성 요구 낮음) |
| 화면 규모 | MVP 15개 미만 → 성장 시 20~30개 (중간 규모) |
| 외부 라이브러리 | TCA 등 **강한 회피 선호** |
| 플랫폼 | iOS 17 / Swift 5.10 / SwiftUI / `@Observable` / async-await |
| 문서 목적 | 면접 대비 — **노선을 하나로 통일**(단일 노선 방침) |

**후보 E (이전 결정)** — "B + NavigationEffect"
ViewModel이 `State`를 보유하고 `send(Intent)`로 사용자 의도를 받습니다. 비동기는 핸들러 안에서 **inline `Task`**로 실행하고 결과를 `state`에 직접 반영합니다. **순수 reduce 함수가 없습니다.** 화면 전환 신호만 [[AsyncStream-Combine-비동기-기술-선택-가이드|AsyncStream]]`<NavigationEffect>` 채널로 분리하여 Coordinator가 `for await`로 구독하도록 하므로, ViewModel은 Coordinator를 모릅니다.

*   **선택 근거**: 보일러플레이트 최소화 + async/await 1:1 가독성. 소셜/콘텐츠 도메인의 낮은 결정성·시간여행 가치를 고려했습니다.

```swift
// E: reduce가 ViewModel에 inline. 비동기 결과를 받은 자리에서 바로 state에 씀.
func send(_ intent: HomeIntent) {
    switch intent {
    case .refresh:
        Task {
            state.isLoading = true
            state.items = await repo.fetch()   // 의도와 결과가 한 자리에
            state.isLoading = false
        }
    case .tapDetail(let id):
        nav.yield(.goToDetail(id))             // 화면 전환은 NavigationEffect로
    }
}
```

**후보 A (이번에 채택)** — "순수 reduce + Effect + Response Action"
reduce를 ViewModel 외부의 **순수 함수** `(inout State, Action) -> Effect<Action>`로 분리합니다. 비동기는 reduce가 직접 실행하지 않고 `Effect` 값으로 **반환**합니다. 외부 실행기가 이를 실행하고, 결과를 **Response Action**(`xxxResponse`)으로 다시 dispatch하여 state를 갱신합니다. 화면 전환은 E와 동일하게 NavigationEffect 채널을 사용합니다.

*   **주요 단점**: Action 개수 폭발(요청/응답 쌍) + `Effect` 인프라 직접 구현 + async/await 우회.

```swift
// A: reduce는 순수 함수. 비동기는 Effect로 반환하고 결과는 Response Action으로 복귀.
func homeReducer(_ state: inout HomeState, _ action: HomeAction) -> Effect<HomeAction> {
    switch action {
    case .refresh:
        state.isLoading = true
        return .run { send in await send(.refreshResponse(.success(await repo.fetch()))) }
    case .refreshResponse(.success(let items)):    // ← 멀리 떨어진 곳에서 마무리
        state.items = items; state.isLoading = false
        return .none
    }
}
```

**경량 A**: A를 풀세트(시간여행·Store scope·`@dynamicMemberLookup`)까지 도입하지 않고, **순수 reduce + 최소 `Effect`/`Store`(~40줄)**만 직접 구현하는 절제된 형태입니다. 이는 본 문서에서 채택하는 구체적인 방식이며, NavigationEffect는 E와 동일하게 공통으로 유지합니다.

**"작은 TCA" 함정**: 순수 reduce를 *실용적으로* 만들려면 `Effect`/`Store`/`TestStore`/취소 레지스트리/시간여행을 전부 직접 구현해야 합니다. 이 모든 것을 만들면 결국 **TCA를 직접 재구현하는 일**이 됩니다. 직접 구현 비용이 가파르게 오르는 특정 시점부터는 "그냥 TCA를 쓰는 게 합리적"이 되는데, 본 프로젝트는 TCA를 회피하므로 **그 선을 넘지 않기 위해 경량 형태로 절제합니다.** 그 절제선이 아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#경량 유지 규칙 (절제) 및 트리거]]에 명시된 TCA 재평가 트리거입니다.

#### 판단 기준 전환 - AI 시대와 테스트 강력함

E를 선택했던 핵심 근거는 **"보일러플레이트 최소화 + async/await 가독성"**이었습니다. 이 기준 자체를 재검토합니다.

**AI 사용 시 테스트 강력함이 1순위**

AI가 코드 작성을 대신하는 환경에서는 **"적게 쓰고 읽기 쉬움"의 가치가 하락**하고, **"AI가 짠 코드가 맞는지 검증하는 안전망(테스트)"의 가치가 상승**합니다. 따라서 아키텍처 선택 기준을 **테스트 강력함**으로 승격합니다.

**AI가 낮춘 비용과 그렇지 않은 비용**

| 비용 | AI가 낮췄나 | 비고 |
|---|---|---|
| **작성**: Action enum·Response case·reduce 분기 타이핑 | ✅ 거의 0 | A의 최대 단점(보일러플레이트)이 해소됩니다 |
| **인프라 구축**: `Effect`/`Store`/TestStore 골격 | ✅ 순식간 | |
| **읽기·추론**: 흩어진 시나리오 따라가기 | △ AI가 도와도 최종 판단은 사람 | |
| **동시성 검증**: 취소·race·누락이 맞는지 | ❌ AI가 미묘하게 틀려도 검증은 사람 | **테스트가 필요한 자리** |
| **잘못된 추상화의 고착** | ❌ 오히려 악화(빨리·널리 박힘) | 절제 규칙으로 방어 (아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#경량 유지 규칙 (절제) 및 트리거]] 참고) |

AI는 보일러플레이트 타이핑(accidental complexity)은 해결했지만 상태·동시성·검증(essential complexity)은 해결하지 못했습니다. 후자를 다루는 핵심 도구는 테스트입니다. AI를 많이 쓸수록 테스트가 강력해야 하는 이유입니다.

**부수 효과: A에 유리한 두 가지 변화**

1.  **E의 "단순함" 프리미엄 하락**: 보일러플레이트를 AI가 생성해 주므로, "A는 코드가 길어 별로"라는 논거가 약해집니다.
2.  **검증 가치 상승**: A의 **결정적 reduce 테스트**는 AI 생성 로직을 mock·대기 없이 검증하기 좋습니다. 따라서 AI 시대에는 A의 가치가 오히려 올라갑니다.

**TCA는 도입하지 않습니다**

테스트 강력함의 정점은 TCA의 `TestStore`입니다. 그럼에도 TCA는 **강한 회피를 유지**합니다.

| 이유 | 설명 |
|---|---|
| 프레임워크 종속 | 버전 추적·마이그레이션 비용, 외부 의존성 |
| 중규모 과함 | 화면 15~30개 소셜/콘텐츠 도메인에 풀세트는 부담스럽습니다 |
| 원리 통제 | 단방향/Effect/TestStore **원리를 직접 구현**해 도입 여부를 직접 결정합니다 |

테스트 강력함을 끝까지 추구하면 논리적 종착지는 TCA입니다([[MVI-아키텍처-선택-테스트-강력함-경량-A#배경: MVI 후보 A와 E]]의 "작은 TCA" 함정). 본 결정은 자체 구현으로 충분한 범위까지만 진행하고, 그 경계를 넘으면 TCA를 재평가합니다. 경계선은 아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L3 production 대안책 (타이밍·취소·중첩)]] 및 [[MVI-아키텍처-선택-테스트-강력함-경량-A#경량 유지 규칙 (절제) 및 트리거]]에 트리거로 명시되어 있습니다.

#### A와 E 재비교

판단 기준이 "테스트 강력함"으로 바뀐 기준으로 두 노선을 다시 비교합니다.

**경량 A의 장단점**
**경량 A 정의**: 순수 reduce(`(inout State, Action) -> Effect<Action>`) + 최소 `Effect`/`Store`(~40줄) + 비동기 결과를 Response Action으로 복귀시키는 방식. NavigationEffect(`AsyncStream`)는 E와 동일하게 공통 유지. 시간여행·Store scope·`@dynamicMemberLookup`은 미구현.

| | 내용 |
|---|---|
| **장점** | ① 순수 reduce → **결정적 테스트**(mock·대기 0) ② 사건 기반 로그(Action 시퀀스로 인과 관계를 직접 파악 가능) ③ reduce 한 곳에서 상태 변경이 구조적으로 강제됨 ④ 비동기 결과까지 시나리오 테스트(Response Action) ⑤ 로깅 등 미들웨어가 단일 채널(reduce 입구 한 곳) |
| **단점** | ① Action 개수 폭발(Response Action) ② async/await 1:1 가독성 손실 (단, AI 시대에는 비용이 감소) ③ `Effect` 인프라 직접 구현(경량이라 ~40줄) ④ 경량 유지에 절제 규율 필요(아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#경량 유지 규칙 (절제) 및 트리거]] 참고) ⑤ `merge` 콤비네이터 복잡 effect는 자체 테스트 한계(아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L3 production 대안책 (타이밍·취소·중첩)]] 참고) |

**E (inline Task)의 장단점**

| | 내용 |
|---|---|
| **장점** | ① 보일러플레이트 최소 ② async/await 1:1 직독 ③ `@Observable`/`@MainActor` 자연 통합 ④ 취소·병합이 `Task`/`async let` 같은 언어 기능으로 내장되어 있음(취소 = `task.cancel()` 한 줄, 병렬 = `async let`/`TaskGroup`. Effect 타입 없이 표준 기능만으로 해결됨) |
| **단점** | ① 테스트가 **ViewModel 전체를 mock하고 Task 완료를 대기해야 함** → 결정성이 약함 ② **비동기 시나리오 테스트가 까다로움**(L2 불가) ③ state 변경이 `Task` 곳곳 → 구조 강제 없음(규율에 의존해야 함) ④ 추적성이 state 스냅샷 수준에 머무름 |

**A를 선정한 이유**

새로운 1순위 가치인 **테스트 강력함**을 기준으로 두 노선을 다시 비교하면 결과가 달라집니다.

| 테스트 레벨 | E | 경량 A |
|---|---|---|
| L1 순수 reduce 결정적 테스트 | ❌ (ViewModel 전체를 mock하고 대기해야 함) | ✅ |
| L2 비동기 시나리오 결정적 테스트 | ❌ | ✅ (자체 TestStore) |
| L3 exhaustive | ❌ | ✅ (실용형) |

*   **E는 L1조차 불가능**합니다. reduce가 ViewModel에 인라인되어 있어 순수 함수가 아니므로, 결과를 검증하려면 의존성을 mock하고 `Task` 완료를 기다려야 합니다(시간 의존).
*   A는 reduce를 ViewModel 외부의 순수 함수로 분리하므로 L1~L3 테스트가 성립합니다.
*   A의 최대 약점(보일러플레이트·가독성)은 **AI가 무력화**(위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#판단 기준 전환: AI 시대와 테스트 강력함]] 참고)했습니다.

기준이 테스트 강력함으로 바뀌었으므로, 결론도 E에서 A로 뒤집힙니다.

#### 경량 A 구현 범위 (L1 ~ L3 실용)

"A로 간다"는 결정이 곧 "TCA를 직접 다 만든다"는 의미는 아닙니다. 테스트 레벨을 정의하고, 어디까지 자체 구현할지 정합니다.

**L1 ~ L4 테스트 레벨 정의**

*   **L1: 순수 reduce 상태 전이 테스트**
    `reduce(state, action) -> state`가 순수 함수이므로, 입력만 주어지면 출력이 결정됩니다.

    ```swift
    var s = HomeState()
    _ = homeReducer(&s, .refreshResponse(.success([.fixture, .fixture])))
    #expect(s.items.count == 2)
    #expect(s.isLoading == false)
    ```

    *   **검증 대상:** "이 action이 들어오면 state가 이렇게 변한다" (비즈니스 로직의 대부분)
    *   **자체 구현 비용:** 0 (인프라 불필요, 함수 호출)
    *   **한계:** reduce가 반환한 `Effect`는 실행하지 않으므로, "비동기가 올바른 action을 보내는지"는 검증하지 않습니다.

*   **L2: 비동기 Effect 시나리오 테스트**
    action 전송 → `Effect` 반환 → 실행 → 비동기 완료 → 결과 action 재dispatch → 최종 state. 이 **전체 흐름**을 결정적으로 검증합니다.

    ```swift
    let store = TestStore(HomeState(), reduce: homeReducer, env: .mock(items: [.fixture]))
    await store.send(.refresh) { $0.isLoading = true }            // 동기 변화 단언
    await store.receive(.refreshResponse(.success([.fixture]))) {  // effect가 보낸 action 단언
        $0.items = [.fixture]
        $0.isLoading = false
    }
    ```

    *   **검증 대상:** "refresh 누르면 → 로딩 → 데이터 도착 → 반영 → 로딩 해제"의 시나리오 전체
    *   **자체 구현 비용:** 중 (~150줄, 1회). 비동기 완료 제어와 받은 action 매칭이 핵심 요소입니다.
    *   **결정성 비결:** 의존성(`env`)을 mock으로 즉답시켜 시간 의존을 제거합니다.

*   **L3: exhaustive**
    L2 + **"명시하지 않은 state 변화가 있으면 실패"** + **"받기로 한 effect action을 모두 받지 않으면 실패"**.

    ```swift
    await store.send(.refresh) { $0.isLoading = true }
    // ↑ refresh가 isLoading 말고 error도 건드렸다면 → 단언에 없으니 자동 실패
    // ↑ effect가 보낸 action을 다 안 받고 테스트 끝내면 → "미처리 effect" 실패
    ```

    *   **검증 대상:** 의도하지 않은 부작용 및 누락된 effect를 **자동 검출**합니다.
    *   **자체 구현 비용:** 핵심(완전 일치 + 잔여 감지)은 L2 골격에 거의 추가 비용이 없습니다. 편의 기능(필드 diff)은 +~40줄이 소요됩니다.
    *   **전제:** `State: Equatable`(본 프로젝트는 화면 상태를 단일 `Equatable` struct로 둡니다 — 위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#배경: MVI 후보 A와 E]]의 "프로젝트 운영 조건" 참고).

*   **L4: 시간여행 / 재생**
    모든 (action, state) 이력 기록 → 임의 시점 되감기 → 다른 분기 재실행.

    *   **성격:** **테스트가 아니라 런타임 디버깅 도구**입니다 (아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L4를 포함하지 않는 이유]]에서 제외 결정).
    *   **자체 구현 비용:** 상 (이력 저장은 쉬우나 재실행·UI 연동 복잡).

**테스트 레벨 요약**

| 레벨 | 무엇 | 누가 주나 | 본 결정 |
|---|---|---|---|
| L1 | 순수 reduce 전이 | 경량 A 자체 | ✅ 채택 |
| L2 | 비동기 시나리오 | 자체 TestStore | ✅ 채택 |
| L3 | exhaustive | 자체 TestStore(실용형) | ✅ 채택 |
| L4 | 시간여행 | TCA-internals | ❌ 제외 (아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L4를 포함하지 않는 이유]] 참고) |

**L3 실용을 채택한 이유**

**결정:** **L1 ~ L3 실용**을 채택합니다. L3 실용은 L2(시나리오)에 완전 일치 강제, 잔여 effect 감지, 필드 diff, 그리고 exhaustive 켜고/끄기 기능을 더한 것입니다. **총 ~190줄 1회 투자로 전 화면에서 재사용 가능합니다.**

**근거: exhaustive의 본질은 비용이 적다**

| L3 요소 | 비용 | 자체 구현 |
|---|---|---|
| state 완전 일치 강제 | **0** | `state == expected` (Equatable) |
| 잔여 effect 감지 | **0** | `pending.isEmpty` |
| 필드별 diff 메시지 | 낮음(~40줄) | `Mirror` 기반 |
| exhaustive 끄고/켜기 | ~1줄 | 플래그 분기 (작성 시 피로 완화) |

exhaustive의 **본질**(완전 일치 + 잔여 감지)은 TestStore 골격에 거의 추가 비용 없이 포함됩니다. 비용이 큰 부분(타이밍·취소·중첩)은 L3의 본질이 아니라 **복잡 effect 인프라**이며, 그것은 아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L3 production 대안책 (타이밍·취소·중첩)]]에서 대체 방안을 제시합니다.

**자체 TestStore 골격 (실용형 L3)**

```swift
// 골격 — Swift Testing 기준 의사코드.
// Issue.record(_:)는 테스트 실패 보고. fieldDiff(_:_:)는 Mirror 기반 별도 헬퍼(아래 '필드 diff' 참조).
@MainActor
final class TestStore<State: Equatable, Action: Equatable> {
    private var state: State
    private let reduce: (inout State, Action) -> Effect<Action>
    private var pending: [Action] = []                   // effect가 보낸 미수신 action
    var exhaustive = true                                 // 작성 피로 완화용 끄고/켜기

    init(_ initial: State, reduce: @escaping (inout State, Action) -> Effect<Action>) {
        self.state = initial; self.reduce = reduce
    }

    func send(_ action: Action, expect: (inout State) -> Void) async {
        var expected = state; expect(&expected)
        let effect = reduce(&state, action)
        assertState(expected)                             // L3: 완전 일치 강제
        await effect.operation { [weak self] in self?.pending.append($0) }
    }

    func receive(_ action: Action, expect: (inout State) -> Void) async {
        guard let i = pending.firstIndex(of: action) else {
            Issue.record("기대한 effect action 없음: \(action)"); return   // 순서 무관 매칭
        }
        pending.remove(at: i)
        var expected = state; expect(&expected)
        _ = reduce(&state, action)
        assertState(expected)
    }

    // L3 exhaustive: 안 받은 effect action이 남으면 실패
    func finish() {
        if exhaustive, !pending.isEmpty { Issue.record("미처리 effect 남음: \(pending)") }
    }

    // 완전 일치 + 필드 diff
    private func assertState(_ expected: State) {
        guard exhaustive, state != expected else { return }
        Issue.record("state 불일치:\n\(fieldDiff(expected, state))")
    }
}
```

본 프로젝트는 화면 상태를 단일 `Equatable` struct로 두므로(위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#배경: MVI 후보 A와 E]]의 "프로젝트 운영 조건" 참고) exhaustive와 정합이 완벽합니다.

**자체 TestStore 사용 예시**

골격을 실제로 사용하면 테스트는 항상 **`send → receive → finish` 3박자**로 진행됩니다. `send`/`receive`의 trailing closure(= `expect` 파라미터)에 "기대하는 state"를 적고, `finish`로 마무리합니다.

```swift
// 대상: HomeState/HomeAction/homeReducer (위 L1·L2 예시와 동일)
@Test
func refresh하면_로딩후_데이터를_반영한다() async {
    let store = TestStore(HomeState(), reduce: homeReducer)   // env는 .mock(items: [.fixture, .fixture]) 주입 가정

    await store.send(.refresh) {            // ① 사용자 action → 동기 변화 단언
        $0.isLoading = true
        $0.error = nil
        // items는 안 적음 = "아직 안 바뀐다"는 단언. 바뀌면 L3 ①(완전 일치)로 자동 실패
    }
    await store.receive(.refreshResponse(.success([.fixture, .fixture]))) {  // ② effect가 보낸 action 단언
        $0.items = [.fixture, .fixture]
        $0.isLoading = false
    }
    store.finish()                          // ③ 안 받은 effect 남으면 L3 ②(잔여 감지)로 실패
}
```

exhaustive가 탐지하는 두 가지(L3 본질):

```swift
// ① 완전 일치 — 단언 안 한 필드가 바뀌면 실패
await store.send(.refresh) {
    $0.isLoading = true
    // reducer가 실수로 items=[]까지 비웠다면 → 여기 안 적었으니 자동 실패
}

// ② 잔여 감지 — effect가 보낸 action을 안 받고 finish하면 실패
await store.send(.refresh) { $0.isLoading = true }
store.finish()   // receive(.refreshResponse...) 누락 → "미처리 effect 남음" 실패
```

작성 피로가 큰 화면(점진 렌더링 등)은 `exhaustive`를 비활성화하여 최종 상태만 검증합니다:

```swift
store.exhaustive = false
await store.send(.refresh) { $0.isLoading = true }       // error 안 적어도 통과
await store.receive(.refreshResponse(.success([.fixture]))) { }  // 중간 state 단언 생략
#expect(store.state.items == [.fixture])                 // 최종만 직접 확인
```

**L3 production 대안책 (타이밍, 취소, 중첩)**

비용이 큰 부분은 **자체 구현하지 않고 '불필요화'하는 방향**으로 접근합니다. 즉, production TestStore가 *감당*해야 할 복잡함을 애초에 *만들지 않는* 전략입니다.

| 문제 | production TestStore 방식 | **본 결정의 대체** |
|---|---|---|
| **타이밍** (지연·디바운스·폴링) | 내부 스케줄러 | **Clock 의존성 주입** — 테스트 시 `ImmediateClock`으로 즉시 진행 |
| **취소** (latest-wins·이탈) | exhaustive 추적 | effect 시나리오에서 **분리해 별도 단위 테스트** (`Task.cancel`은 Swift 표준) |
| **동시/중첩** | `merge` 콤비네이터 결정적 테스트 | **결과를 1 action으로 합침** (`async let`으로 1개 종합 action 반환). `merge` 콤비네이터 금지 |

**`merge` 및 정밀 취소를 자체 구현하지 않는 이유**
비용이 큰 부분은 호출부(`.merge`/\`.cancellable\` 몇 줄)가 아니라 **그것을 결정적으로 테스트하는 밑단 인프라**(도착 타이밍 통제·effect 추적·취소 레지스트리)입니다. 사실상 **작은 TCA 재구현**(위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#배경: MVI 후보 A와 E]]의 "작은 TCA" 함정)입니다. AI가 골격은 순식간에 짜주지만 ① 동시성이 *맞는지*는 보장할 수 없으며(테스트 도구가 미묘하게 틀리면 그것으로 돌린 모든 테스트의 신뢰가 무너집니다) ② 쉽게 만들어주는 만큼 잘못된 추상화가 빨리·널리 **고착**됩니다(위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#판단 기준 전환: AI 시대와 테스트 강력함]] 참고). 따라서 "만들 수 있다"가 아니라 **"안 만든다"를 규칙으로 삼습니다**(아래 [[MVI-아키텍처-선택-테스트-강력함-경량-A#경량 유지 규칙 (절제) 및 트리거]] 참고). 정밀 취소를 포기한 대가는 `Task` 핸들 1개의 **전체 취소**로 충분합니다(structured concurrency — 부모 `Task` 취소 시 내부 `async let` 자식까지 함께 취소됩니다. latest-wins는 `loadTask?.cancel()` 한 줄로 구현 가능합니다).

[[AsyncStream-Combine-비동기-기술-선택-가이드|비동기]] 패턴별 표현법(모두 단순 TestStore로 테스트 가능):

```swift
// 순차 의존 — (a) Response 체인: 중간 상태가 UI/테스트에 중요할 때
case .login: return .run { send in await send(.loginResponse(...)) }
case .loginResponse(.success(let token)): return .run { send in await send(.profileResponse(...)) }

// 순차 의존 — (b) 한 effect 안 순차 await: 중간 노출 불필요할 때
return .run { send in
    let token = try await auth.login(cred)
    let profile = try await repo.profile(token)
    await send(.loaded(profile))
}

// 동시 호출 — async let → 1 action: 동시성 + 테스트 보장 둘 다
return .run { send in
    async let a = repo.items()
    async let b = repo.profile()
    await send(.loaded(try await a, try await b))
}
```

**"동시 2개 호출"은 자유롭게 할 수 있습니다**. 보장이 약해지는 경우는 동시 실행 때문이 아니라 **결과를 2개 action으로 흩어서 도착 순서가 비결정적**일 때뿐입니다(`merge`). 결과를 1 action으로 합치면 동시성과 결정적 테스트를 모두 얻습니다.

**예외**: 결과를 도착하는 대로 점진 렌더링해야 하는 화면은 결정적 테스트를 일부 포기하고 **통합 테스트**로 처리합니다. 이런 화면이 반복되면 그 시점이 **TCA 재평가 트리거**입니다.

**L4를 포함하지 않는 이유**

| 이유 | 설명 |
|---|---|
| **테스트가 아님** | L4는 런타임 디버깅 도구(되감기·재생)이지 검증 도구가 아닙니다. 본 결정의 1순위 가치(테스트 강력함)와 결이 다릅니다. |
| **비용 대비 가치 없음** | 재실행·UI 연동·effect 재현 구현 비용이 높지만, 테스트 커버리지는 늘지 않습니다. |
| **디버깅은 다른 수단으로 충분** | 사건 기반 로그(Action 시퀀스, reduce 입구 한 곳 로깅)로 "무엇이 왜 일어났나"를 추적 가능합니다. |

L4는 구현하지 않습니다. 디버깅은 Action 로그로, 검증은 L1~L3로 분담합니다.

#### 경량 유지 규칙 및 트리거

| 규칙 | 내용 | 어기고 싶어지는 순간 = 트리거 (TCA 재평가) |
|---|---|---|
| **effect 단순성** | 1 effect = 1 작업 = 1 결과 action. `merge`/`flatMap` 콤비네이터 사용 금지. 순차 처리는 (a)/(b), 병렬은 `async let`으로 1개 종합 action 반환 | 점진 렌더링(흩어진 결과)이 여러 화면에 반복 |
| **취소 경량** | `Task` 핸들 1개로 latest-wins 패턴 적용. `EffectID`/`.cancellable` 직접 구현하지 않음 | 화면별 정교한 취소 정책이 다수 필요 |
| **미들웨어 최소** | reduce 입구 로깅만 한 줄로 제한. 분석·디버그 타임라인 등 추가하지 않음 | 미들웨어 도입 욕구가 반복적으로 발생 |
| **Store 단순** | 화면 단위 독립 Store. `.scope`/자식 store 만들지 않음 | 화면 간 state 공유·스코핑 요구가 여러 번 발생 |
| **TestStore 경량** | L3 실용까지만. 타이밍·취소·중첩 exhaustive는 위 [[MVI-아키텍처-선택-테스트-강력함-경량-A#L3 production 대안책 (타이밍·취소·중첩)]]으로 대체합니다 | 복잡 effect의 결정적 테스트가 실제로 필요 |

이 규칙들이 깨지는 시점이 곧 **TCA 재평가 트리거**입니다. 도달 전까지는 경량 A가 정답이고, 도달하면 자체 강화 vs TCA 도입을 그 시점에서 판단합니다.

#### 최종 결정 요약

| # | 결정 | 핵심 근거 |
|---|---|---|
| 1 | 판단 기준 = **테스트 강력함** | AI가 작성 비용을 해결 → 검증력이 1순위 |
| 2 | 노선 = **경량 A** (E에서 변경) | E는 L1조차 불가, A는 L1~L3 가능 |
| 3 | TCA **미도입** | 프레임워크 종속·중규모 과함으로 회피 유지. 경계는 트리거로 |
| 4 | 구현 범위 = **L1~L3 실용** | exhaustive 본질은 ~190줄로 저렴 |
| 5 | L3 production = **불필요화** | Clock 주입 / 취소 분리 / 결과 1 action 합치기 |
| 6 | L4 **제외** | 테스트 아닌 디버깅 도구, 낮은 가치 |
