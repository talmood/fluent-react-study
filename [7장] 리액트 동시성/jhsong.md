# 동기식 렌더링의 문제

## 사용자 경험 저하

동기식 렌더링의 문제는 메인 스레드를 가로막아서 사용자 경험이 저하된다는 것, **문제가 발생하면 UI 반응이 느려지거나 응답을 하지 않아 사용자가 불편**을 겪게 됨.

- 이로 인해 **리액트에서 상태를 저장하는 setter 함수를 비동기로 구현**해둠
    - 여러 업데이트를 일괄 처리해서 메인 스레드에서 수행되는 작업을 최소화
    (Batch Update based on Diffing Algorithm ⇒ Reconciliation)

## 우선순위가 없다

동기식 렌더링은 우선순위가 없어 일괄 처리를 사용해봤자 더 복잡해질 뿐이다. 모든 업데이트를 동일하게 취급하고, 업데이트가 사용자에게 보이는지 여부를 따지지 않음.

- 예를 들어, 동기식 렌더링에서는 보이지 않는 탭, 모달창 뒤에 있는 콘텐츠, 로딩 상태 콘텐츠 등 **사용자가 볼 수 없는 항목도 동등한 수준으로 렌더링**함.
    - 이로 인해 Main Thread가 Block될 수 있음.
    - **사용자가 볼 수 있고 상호 작용할 수 있는 항목이 우선적으로 먼저 렌더링**되어야 함.

동시성 기능이 도입되기 전에는 중요하지 않은 업데이트가 중요한 업데이트를 가로막아 사용자 경험이 저하되는 현상이 종종 부각됨. (우선순위가 없기 때문!)

이를 해결하기 위해 **concurrent rendering(동시성 렌더링)을 사용하여 업데이트 작업의 중요도와 긴급도에 따라 우선순위**를 정하고, 중요한 업데이트가 덜 중요한 업데이트에 Blocking되지 않도록 함.

- 예를 들어, 사용자는 버튼을 마우스로 가리키거나 클릭할 때 해당 동작에 대한 즉각적인 피드백이 표시되기를 기대함. 그러나 리액트가 크고 긴 목록을 렌더링하는 도중이라면, 목록을 모두 렌더링할 때까지 버튼 색상 같은 피드백이 미뤄짐.
- 하지만, 동시성 렌더링을 사용하면 CPU 연산이 많은 작업(목록 렌더링)은 우선순위가 낮아지고, 사용자 상호 작용이나 애니메이션 같은 더 중요한 작업이 우선적으로 처리됨.

# 업데이트 예약과 지연

리액트에서 예약(schedule)하거나 지연(defer)하는 기능은 애플리케이션의 응답성을 유지하는 데 매우 중요함.

**파이버(fiber) 재조정자**는 스케줄러와 여러 효율적인 API에 의존해 구현함.

- 스케줄러는 `setTimeout` , `MessageChannel` 등의 브라우저 API를 사용해서 작업의 우선순위를 부여한 후 예약 및 관리하는 시스템
    - 예를 들어, 채팅 기능을 만든다고 한다면 **메시지 목록 업데이트와 사용자의 메세지 입력 상호 작용 작업을 효율적으로 관리**해야 함 → 이를 동시성 렌더링을 통해 해결
        - **사용자의 메세지 입력하거나 전송**할 때, 텍스트 입력 업데이트를 다른 업데이트보다 **우선적으로 처리**함 ⇒ 부드러운 사용자 경험 보장
        - 서버에서 새 메세지가 도착해 렌더링해야 할 때, 렌더 레인을 통해 렌더링을 진행
            - **렌더 레인은 동기식으로 DOM을 업데이트** 함 ⇒ 다른 업데이트를 가로막게 되어 사용자 입력에 지연이 발생
            ⇒ `useTransition` 의 `startTransition` 으로 **새 메세지 목록을 렌더링하는 작업의 우선순위를 낮출 수 있음**

# Render Lane

## 작동방식

컴포넌트가 업데이트되거나 새 컴포넌트가 렌더 트리에 추가되면, 해당 업데이트의  우선순위에 따라 레인을 할당함.

- 우선순위는 **업데이트 종류**(사용자 상호 작용, 데이터 패치, 백그라운드 작업 등)와 **컴포넌트 가시성**(visibility) 같은 요인에 따라 정해짐
- 업데이트가 발생하면 다음과 같은 순서로 업데이트를 할당함
    1. 업데이트 콘텍스트 확인
        - 업데이트 콘텍스트 분석
    2. 콘텍스트에 따른 우선순위 추정
        - 18.3.1 기준 Lane 종류 (해당 레인 기반 우선순위 추정)
          
            | Lane 이름 | 비트 표현 | 설명 |
            | --- | --- | --- |
            | NoLanes/NoLane | 0b0000000000000000000000000000000 | 레인 없음 (초기 상태) |
            | SyncHydrationLane | 0b0000000000000000000000000000001 | 동기식 하이드레이션 |
            | SyncLane | 0b0000000000000000000000000000010 | 동기식 업데이트 (최우선) |
            | InputContinuousHydrationLane | 0b0000000000000000000000000000100 | 입력 연속 하이드레이션 |
            | InputContinuousLane | 0b0000000000000000000000000001000 | 사용자 입력 처리 (클릭 등) |
            | DefaultHydrationLane | 0b0000000000000000000000000010000 | 기본 하이드레이션 |
            | DefaultLane | 0b0000000000000000000000000100000 | 기본 업데이트 (네트워크 요청 등) |
            | TransitionHydrationLane | 0b0000000000000000000000001000000 | 전환 하이드레이션 |
            | TransitionLanes (1-16) | 0b0000000011111111111111110000000 | UI 전환 작업 (집합) |
            | RetryLanes (1-4) | 0b0000111100000000000000000000000 | 재시도 작업 (집합) |
            | SelectiveHydrationLane | 0b0001000000000000000000000000000 | 선택적 하이드레이션 |
            | IdleHydrationLane | 0b0010000000000000000000000000000 | 유휴 하이드레이션 |
            | IdleLane | 0b0100000000000000000000000000000 | 유휴 작업 (낮은 우선순위) |
            | OffscreenLane | 0b1000000000000000000000000000000 | 화면 밖 콘텐츠 (최하위 우선순위) |
    
     3. 우선순위 재정의가 있는지 확인
    
    - `useTransition` , `useDeferredValue` , `flushSync` 등의 우선순위 변동 사항이 직접적으로 조작됐는지 확인
    1. 레인에 업데이트 할당
