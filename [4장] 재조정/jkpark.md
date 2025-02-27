### 파이버 노드?

- 생성 - createFiberFromTypeAndProps 함수, 이 함수의 타입과 프롭은 리액트 엘리먼트라고 할 수도 있음.
- 작업 루프(work loop)를 사용해 UI 업데이트, 업데이트에 필요한 경우 파이버 노드를 ‘dirty’로 표시
- 루프의 끝에 도달하면 브라우저의 DOM와 분리된 새 DOM 트리를 메모리에 생성하고 화면에 반영(flushed).
- **beginWork**는 위에서 아래로 이동하며 ‘업데이트가 필요함’을 표시
- **completeWork**는 다시 위로 이동하며 브라우저에서 분리된 실제 DOM 엘리먼트 트리를 메모리에 구성

### 더블 버퍼링?

두 개의 버퍼(or 메모리 공간)를 생성하여 일정 간격으로 두 버퍼를 전환해 깜박임을 줄이고 체감 성능을 개선

### 더블 버퍼링이 파이버 재조정과 비슷한 이유?

- 업데이트 발생 시 파이버 트리가 포크 → 업데이트 = 렌더링
- 현재 트리를 대체할 새 트리가 준비되어 더블 버퍼링처럼 현재 파이버 트리와 교체 = 커밋

### 파이버 재조정

렌더링 - 커밋

**렌더링 단계**

- beginWork 함수에서 발생
- 상태 변경 이벤트 발생 시 시작
- 각 파이버를 재귀적, 단계적으로 순회, 업데이트 보류 중이라는 신호 플래그를 성정하여 대체 트리에 오프스크린 변경 작업 수행

**beginWork**

- 작업용 트리의 맨 아래까지 돌며 파이버 노드의 업데이트 필요 여부를 나타내는 플래그 설정
- 작업 완료 시 completeWork를 호출하고 다시 올라가며 순회

```jsx
function beginWork(
	current: Fiber | null, // 현재 파이버 노드에 대한 참조, 비교용
	workInProgress: Fiber, // 업데이트 중인 파이버 노드, 'dirty'로 표시되어 반환
	renderLanes: Lanes,
): Fiber | null;
```

- renderLanes
    - 기존의 renderExpirationTime을 대체하는 개념
    - 업데이트 우선순위를 잘 정하고, 업데이트 프로세스를 효율적으로 하는 데 도움이 됨.
    - 변경의 우선순위가 높을수록 더 높은 레인이 할당
    - 타임 슬라이싱이라는 기술을 사용해 실행 시간이 긴 업데이트를 더 작고 관리하기 쉽게 분할

**completeWork**

- 작업용 파이버 노드에 업데이트 적용, 업데이트된 실제 DOM 트리를 새롭게 생성

```jsx
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null;
```

- beginWork와 동일한 시그니처
- completeWork가 트리의 맨 위에 도달해서 DOM 트리를 구성하면 렌더링 단계가 완료 → 커밋 단계로 넘어감

커밋 단계

- 렌더링 단계에서 가상 DOM에 적용된 변경 사항을 실제 DOM에 반영
    1. 변형 단계
        - commitMutationEffects 함수를 호출해 파이버 노드에 적용된 업데이트를 실제 DOM에 반영
    2. 레이아웃 단계
        - commitLayoutEffects라는 함수를 이용해 DOM에서 업데이트된 노드의 새 레이아웃 계산
- 커밋 단계에서 발생하는 효과
    - 배치 효과
    - 업데이트 효과
    - 삭제 효과
    - 레이아웃 효과
