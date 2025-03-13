### 동기식 렌더링의 문제

- 업데이트 시 메인 스레드를 가로막아서 사용자 경험이 저하
  - 해결 방법 1. 여러 업데이트를 일괄 처리해서, 메인 스레드에서 수행되는 작업을 최소화
  - 해결 방법 2. 작업의 중요도와 긴급도에 따라 우선순위를 정하고, 중요한 업데이트를 먼저 수행

### 스케줄러

- 리액트가 업데이트를 예약하고 지연하기 위해, 파이버 재조정자는 스케줄러 등의 기능에 의존
- 덕분에 유휴 시간 동안 작업을 수행하고, 가장 적절한 시기에 업데이트를 예약
- 본래 리액트 팀에서 최초에는 스케줄링 기능을 위하여 사용자가 현재 input event를 처리중 인지를 확인하기 위한 isInputPending 이라는 API를 제안하였음
  - https://wicg.github.io/is-input-pending/
- 이후 Web API로 활용할 수 있는 Scheduler API가 추가되었고 더 이상 isInputPending이 아니라 Scheduler API를 활용하는 것을 권장함.
  - https://developer.mozilla.org/en-US/docs/Web/API/Scheduling/isInputPending
  - https://developer.mozilla.org/en-US/docs/Web/API/Scheduler
- 그러나 아직 불완전한 상태이므로, 리액트에서도 해당 방식은 아직 사용하지 않는 것으로 보임. 리액트 내의 스케줄러는 런타임 환경을 구분하여 가장 최적의 API를 활용
  - https://github.com/facebook/react/blob/e62a8d754548a490c2a3bcff3b420e5eedaf11c0/packages/scheduler/src/forks/Scheduler.js#L551
  - MessageChannel API를 사용할 수 있다면 사용한다.
  - 만약 불가능하다면, setTimeout을 사용한다. (최소 4ms의 딜레이가 존재하기 때문에, MessageChannel API보다는 느림)

### 렌더 레인

- 렌더 레인은 렌더링 주기에서 가장 우선적으로 처리해야 하는 작업을 표현
- 리액트 18에서 도입되었으며, 이전에는 만료 시간이 있는 예약 매커니즘이 활용되었음
- 작동 방식
  - 업데이트 수집: 업데이트 작업을 예약 시, 우선순위에 따라 각 레인에 할당
  - 레인 처리: 우선 순위가 높은 레인에 할당된 업데이트부터 처리함. 같은 레인의 업데이트는 일괄 처리
  - 커밋 단계: 업데이트 작업 후 실제로 DOM에 반영해야 할 내용을 적용하고 Effect를 처리
  - 반복: 렌더링이 발생할 때마다 위 과정을 반복
- 우선 순위 결정 방식
  - 업데이트의 컨텍스트 확인: 사용자의 상호작용, prop or state 등의 변화, 서버 응답에 의한 변화 등 업데이트를 발생하게 한 컨텍스트를 평가
  - 우선순위 측정: 컨텍스트에 따라 업데이트의 우선순위를 추정
  - 우선순위 재정의 확인: `useTransition`, `useDeferredValue` 등으로 업데이트의 우선순위가 명시적으로 재정의된 경우 설정된 우선순위 사용
  - 올바른 레인에 업데이트 할당: 우선순위에 알맞는 레인에 업데이트를 할당. (비트마스크 연산 활용)
