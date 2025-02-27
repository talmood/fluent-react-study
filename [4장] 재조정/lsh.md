## 재조정

- 리액트의 가상 DOM은 우리가 화면에 보여지길 원하는 UI 상태에 대한 일종의 청사진
- 상태 업데이트를 동시에 여러 번 수행한다고 해도, DOM을 매 번 업데이트 하는 것이 아니라 최종 변경 사항을 한 번만 DOM에 Commit 하는 식으로 배칭 처리
- JSX 구문으로 작성한 컴포넌트의 형태는 빌드 시 React.createElement() 함수를 호출하는 형태로 변환되는데, 이 Element의 정보를 가지고 Fiber 노드를 생성함
- ReactElement는 일시적이고 상태가 없지만, Fiber 노드는 상태를 저장하고 수명이 길음. (한 번 mount된 파이버 노드의 재사용)

### Stack 기반의 재조정자

- 스택 기반 방식으로 동작하던 기존의 리액트 재조정 알고리즘
- 더 긴급한 UI 변경 요청이 들어와도, 업데이트를 중간에 중단할 수 없음. (우선순위 조정 불가)

### Fiber 기반의 재조정자

- 특정 시점에 존재하는 리액트 컴포넌트 트리의 정보를 표현하여 관리하는 리액트 내부 데이터 구조
- 해당 컴포넌트가 특정 시점에 갖는 상태, 프롭, 하위 컴포넌트 등의 정보를 보관하고 있음
- 각 파이버 노드는 포인터 프로퍼티를 통해 다른 파이버 노드를 가리키고 있음
  - return: 부모 노드
  - sibling: 형제 노드
  - child: 자식 노드
- 각 파이버 노드는 재조정 시 더블 버퍼링 방식으로 새로운 UI를 계산함
  - current: 현재 파이버 노드
  - workInProgress: 현재 파이버 노드를 복사한 뒤, 새로운 반영사항을 작업 중인 파이버 노드

## 렌더 레인

- 우선 순위가 높은 작업과 낮은 작업을 구분하기 위해 비트 마스킹 방식으로 구현된 개념
  - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberLane.js#L41
- 재조정 과정 중에 변경 사항이 발생하면, 당시 상황에 따라 적절한 레인에 변경 사항이 할당됨
- 우선순위가 더 높은 레인에 할당된 작업부터 업데이트를 우선적으로 수행

## 렌더링 프로세스

![Image](https://github.com/user-attachments/assets/cbee24d5-9c1d-43df-af3a-f6332d48d4a5)

- didReceiveUpdate: 업데이트 플래그 변수
- beginWork: 이전 파이버 노드와, 새로운 파이버 노드가 달라지면 변경되었음을 나타내는 플래그 변수를 true로 업데이트)
  - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberBeginWork.js#L3860
- updateFunctionComponent: 함수 컴포넌트로부터 생성된 파이버 노드의 업데이트를 반영하는 함수로, 업데이트 플래그 변수가 false이면 bailout하고 이후 렌더링 과정을 생략
  - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberBeginWork.js#L1176
- reconcileChildren: 해당 파이버 노드의 자식 노드를 생성하거나, 재조정
  - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactFiberBeginWork.js#L333
- reconcileChildFiber: reconcileChildren 함수에 의해서 사용되는 파이버 트리 순회 로직을 모아놓은 구현체.
  - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactChildFiber.js#L1928
  - reconcileSingleElement: 노드가 단일 리액트 컴포넌트일 때 사용하는 함수
    - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactChildFiber.js#L1615
  - reconcileChildrenArray: 노드가 배열 형태일 때 사용하는 함수로, sibling 탐색
    - https://github.com/facebook/react/blob/v19.0.0/packages/react-reconciler/src/ReactChildFiber.js#L1142
