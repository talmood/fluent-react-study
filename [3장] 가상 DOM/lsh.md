## 가상 DOM

- **등장 이유**
  - React처럼 선언적인 인터페이스를 지향하는 라이브러리에서는, 렌더링을 트리거하는 setState 등의 이벤트가 발생할 때 해당 컴포넌트가 기존 상태로부터 새로운 상태를 어떻게 표현해야 할 지를 일일이 명령형으로 작성하지 않고, 단지 우리가 화면에 나타내고자 하는 바를 선언적으로 표현하는 것을 도와 개발자로 하여금 프로그램의 복잡도를 완화할 수 있도록 합니다.
  - 하지만, DOM 요소의 레이아웃을 계산하고 화면에 그리는 작업(layout, reflow은 비교적 비용이 높은 연산인데, 선언적인 인터페이스를 제공하기 위해서 만약 개발자가 이전 화면 및 상태를 고려하지 않고 화면으로 보여줘야 할 화면만 새롭게 그려낸다면 연산 비용이 높아질 것입니다.
  - 그래서 개발자는 매 snapshot마다 자신이 그리고 싶은 화면만 새롭게 표현하고, 실제 DOM 대신 가상의 자바스크립트 객체(가상 DOM)로 먼저 연산(Reconciliate)하자는 아이디어로 시작한 것이 React입니다.
  - https://www.youtube.com/watch?v=XxVg_s8xAms&ab_channel=MetaDevelopers

## SyntheticEvent

- 브라우저 네이티브 이벤트를 래핑해서 개발자가 통일된 인터페이스로 다룰 수 있도록 해주는 객체입니다. (https://github.com/facebook/react/blob/v19.0.0/packages/react-dom-bindings/src/events/SyntheticEvent.js#L46)
- **장점**
  - 브라우저와의 호환성: 예를 들어 이벤트가 발생한 엘리먼트 객체를 꺼내는 방법이 브라우저마다 다양한데, e.target 으로 꺼낼 수 있도록 인터페이스 통일화하였습니다. (e.target, e.srcElement, window 등)
    - https://github.com/facebook/react/blob/v19.0.0/packages/react-dom-bindings/src/events/getEventTarget.js#L17
  - Input 값이 변경될 때 동작하는 이벤트의 이름도 브라우저마다 `change`, `input` 등 다양한데, 마찬가지로 `onChange` 로 통일화하였습니다.
- **이벤트 풀링**
  - React 17 미만 버전에서는 이벤트 풀링이라는 개념이 있어서, 이벤트가 발생할 때 마다 SyntheticEvent를 새롭게 만드는 게 아니라, 일종의 SyntheticEvent 풀을 만들어서 이벤트가 발생할 때마다 가져와서 사용하는 형태로 구현되어 있었습니다. 언뜻 보아서는 이벤트 객체를 매번 생성하지 않아도 효율적일 것 같지만, 사실상 최신 브라우저에서는 이러한 풀링으로 얻을 수 있는 성능적인 이점은 없고, 개발자에게 혼란만 준다고 판단되어 17 버전에서는 제거되었습니다. (모던 리액트 딥 다이브 10장 참고)
  - 리액트 16 이전에서의 동작 방식
    - 이벤트 핸들러 내부에 비동기로 동작하는 로직이 있는 경우, 이벤트 객체는 이미 event pool로 들어가서 null로 초기화된 상황이기에 참조할 수 없는 문제가 발생했었습니다.
    ```jsx
    function handleChange(e) {
      setData((data) => ({
        ...data,
        // This crashes in React 16 and earlier:
        text: e.target.value,
      }));
    }
    ```
    - 그래서 이를 해결하려면 `e.persist()` 를 사용해야 했었는데, 직관적이지 않고 혼란스러운 동작이었습니다.
    ```jsx
    function handleChange(e) {
      e.persist();
      setData((data) => ({
        ...data,
        // This crashes in React 16 and earlier:
        text: e.target.value,
      }));
    }
    ```

## 가상 DOM 작동 방식

- 리액트 컴포넌트 내부에서 반환한 JSX 문법은, 빌드시 jsx 함수로 변환됩니다. 이 함수를 실행하면 리액트 컴포넌트의 메타 정보를 표현하는 ReactElement 객체가 생성됩니다.
  - https://github.com/facebook/react/blob/v19.0.0/packages/react/src/jsx/ReactJSXElement.js#L293
- 이 ReactElement는 몇 가지의 속성을 갖고 있습니다.
  - `$$typeof`: 해당 객체가 리액트 엘리먼트가 맞는지 확인할 수 있는 심볼 값
  - `type`: 해당 엘리먼트가 어떤 컴포넌트의 종류인지를 나타내는 값
    - ex) 리액트 컴포넌트의 경우 해당 컴포넌트의 이름, 호스트 컴포넌트의 경우 “div”, “span”, “button” 등
  - `ref`: 컴포넌트가 DOM 노드를 참조할 수 있도록 관리하는 속성으로, React 19에서는 제거되었습니다.
  - `props`: 컴포넌트에 전달된 모든 prop을 가리킵니다.
  - `_owner`: 개발 모드에서만 활용되는 디버깅 용도로, 이 엘리먼트를 생성한 부모 컴포넌트를 가리킵니다.

## 디핑 알고리즘

- 리액트 컴포넌트의 state나 prop이 변경되어 리액트 렌더링 프로세스가 트리거되면, 이전 트리와 새 트리를 비교하여 변경된 부분을 탐색하게 됩니다. 이 과정이 디핑 알고리즘입니다.
- 동작 과정
  1. 두 트리의 루트에 있는 노드가 변경되면, 모든 하위 트리를 새 트리로 대체
  2. 노드가 동일하다면, 노드의 속성을 비교하여 새 트리로 대체
  3. 자식 노드가 다르다면, 자식 노드를 다시 생성
  4. 자식 노드가 동일하지만 순서가 변경되었다면, 노드는 생성하지 않고 순서 재정렬

## References

- [https://medium.com/crossplatformkorea/react-리액트를-처음부터-배워보자-06-합성-이벤트와-event-pooling-6b4a0801c9b9](https://medium.com/crossplatformkorea/react-%EB%A6%AC%EC%95%A1%ED%8A%B8%EB%A5%BC-%EC%B2%98%EC%9D%8C%EB%B6%80%ED%84%B0-%EB%B0%B0%EC%9B%8C%EB%B3%B4%EC%9E%90-06-%ED%95%A9%EC%84%B1-%EC%9D%B4%EB%B2%A4%ED%8A%B8%EC%99%80-event-pooling-6b4a0801c9b9)
- [https://medium.com/hcleedev/web-react의-event-시스템-내부-구현-자세히-알아보기-react-v18-2-0-39d59ab45bec](https://medium.com/hcleedev/web-react%EC%9D%98-event-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EB%82%B4%EB%B6%80-%EA%B5%AC%ED%98%84-%EC%9E%90%EC%84%B8%ED%9E%88-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-react-v18-2-0-39d59ab45bec)
