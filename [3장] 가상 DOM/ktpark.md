
### 가상 DOM 소개

실제 DOM은 노드 객체, 가상 DOM은 설명 역할을 하는 평범한 JS 객체로 구성
setState또는 다른 메커니즘을 통해 UI 변경 -> 가상 DOM 업데이트 -> 그에 맞게 실제 DOM 업데이트

가상 DOM이 나온 이유는 실제 DOM의 Reflow의 증가에 따른 성능 영향에 비효율성을 줄이기 위해 등장

자바스크립트 엔진을 최대한 활용하는 다양한 알고리즘(Diffing Algorithm)에 접근해 빠르고 효율적으로 조작

### 실제 DOM

주로 querySelector, getElementById를 사용 각각 단점들이 존재 
(querySelector는 문서의 크기가 크면 느려지고, getElementById는 브라우저는 ID의 고유성을 강제하지 않아 충돌 발생 위험)

getBoundingClientRect()를 이용해서 레이아웃 스레싱을 최소화할 수 있지만 대기 중인 레이아웃 변경 사항이 있을 때는 리플로를 일으킬 수 있습니다. 
-> React는 알아서 한 번의 작업으로 필요한 정보를 모두 가져온다. 

다양한 항목이 존재할 때 새로운 항목을 추가 하였을 때 실제 DOM은 레이아웃을 다시 그린다. -> 리액트는 변화한 곳만 렌더링

브라우저 간 호환성 문제도 존재한다. -> React에서는 **SyntheticEvent**를 이용해 이벤트를 커스텀 한다. 

문서 조각이 있어 DOM 노드를 저장하는 가벼운 컨테이너 역할을 한다. -> React가 더 효울적 (일괄 업데이트, 효율적인 비교 알고리듬, 단일 렌더링)

### 가상 DOM 작동 방식

가상 DOM에서는 실제 DOM에서 사용하는 API를 사용할 수 있다. 

리액트 element는 각각 props와 state를 가지고 있다. 


```ts
{
	$$typeof: Symbol(react.element),  //element 종류
	type: "div",                     //DOM 엘리먼트 '호스트 컴포넌트'
	key: null,                       
	ref: null,
	props: {
		className: "my-class"
		children: "Hello, world!"
	}, 
	_owner: null,                    //element 추적용(코드 사용 금지)
	_store: null                     //추가 데이터 저장(코드 사용 금지)
}
```


가상 DOM은 실제 DOM에 반영하기 위해서 재조정 프로세스를 거친다. 

diffing 알고리즘을 통해 가상 DOM과 실제 DOM의 차이를 비교하여 업데이트 한다. 
루트 노드가 변경 되지 않는 이상 다시 생성하지 않고 변경된 노드만 업데이트 하는 방식으로 진행한다. 

불필요한 리렌더링이 발생하는 경우도 있다. (부모 컴포넌트가 렌더링 하는 경우 자식 컴포넌트도 렌더링 한다. ) -> memo와 useMemo를 통해 최적화 할 수 있다. 
