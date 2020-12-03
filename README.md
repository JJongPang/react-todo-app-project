# react-todo-app-project

## 일정 관리 웹 애플리케이션 만들기
### 일정관리 애플리케이션 개발 흐름
- 프로젝트 준비하기
- UI 구성하기
- 기능 구현하기

<br />

### UI 구성하기
- TodoTemplate: 화면을 가운데에 정렬시켜 주며, 앱 타이틀(일정 관리)을 보여 줍니다.  children으로 내부 JSX를 props로 받아 와석 렌더링해 줍니다.
- TodoInsert: 새로운 항목을 입력하고 추가할수 있는 컴포넌트 입니다. state를 통해 인풋의 상태를 관리 합니다.
- TodoListItem: 각 할 일 항목에 대한 정보를 보여 주는 컴포넌트입니다. todo 객체를 props로 받아 와서 상태에 따라 다른 스타일의 UI를 보여 줍니다.
- TodoKList: todos 배열을 props로 받아 온 후, 이를 배열 내장 함수 map을 사용해서 여러 개의 TodoListItem 컴포넌트로 변환하여 보여 줍니다.

### 기능 구현하기
- 항목 추가 기능
- 항목 지우기 기능 구현하기

<br />

## 컴포넌트 성능 최적화
### 컴포넌트 성능 최적화
- 많은 데이터 렌더링하기
- 크롬 개발자 도구를 통한 성능 모니터링
- React memo를 통한 콤포넌트 리렌더링 성능 최적화
- onToggle과 onRemove가 새로워지는 현상 방지하기
- react-virtualized를 사용한 렌더링 최적화

<br />

## 많은 데이터 렌더링하기
~~~js
import React, { useState, useCallback, useRef } from 'react';
import TodoTemplate from './components/TodoTemplate';
import TodoInsert from './components/TodoInsert';
import TodoList from './components/TodoList';

function createBulkTodos() {
  const array = [];
  for (let i = 1; i <= 2500; i++) {
    array.push({
      id: i,
      text: `할 일 ${i}`,
      checked: false,
    });
  }
  return array;
}

const App = () => {
  const [todos, setTodos] = useState(createBulkTodos);

  //고윳값으로 사용될 id
  //ref를 사용하여 변수 담기
  const nextId = useRef(2501);

  const onInsert = useCallback(
    (text) => {
      const todo = {
        id: nextId.current,
        text,
        checked: false,
      };
      setTodos(todos.concat(todo));
      nextId.current += 1; //nextId 1씩 더하기
    },
    [todos],
  );

  const onRemove = useCallback(
    (id) => {
      setTodos(todos.filter((todo) => todo.id !== id));
    },
    [todos],
  );

  const onToggle = useCallback(
    (id) => {
      setTodos(
        todos.map((todo) =>
          todo.id === id ? { ...todo, checked: !todo.checked } : todo,
        ),
      );
    },
    [todos],
  );

  return (
    <TodoTemplate>
      <TodoInsert onInsert={onInsert} />
      <TodoList todos={todos} onRemove={onRemove} onToggle={onToggle} />
    </TodoTemplate>
  );
};

export default App;
~~~
여기서 useState(createBulkTodos())라고 작성하면 리렌더링될 때마다 createBulkTodos 함수가 호출되지만, useState(createBulkTodos)처럼 파라미터를 함수 형태로 넣어 주면 컴포넌트가 처음 렌더링될 때만 createBulkTodos 함수가 실행될 것입니다.

<br />

## 느려지는 원인 분석
- 자신이 전달받은 props가 변경될 때
- 자신의 state가 바뀔 때
- 부모 컴포넌트가 리렌더링될 때
- forceUpdate 함수가 실행 될 때

<br />

## React.memo를 사용하여 컴포넌트 성능 최적화
~~~js
import React from 'react';
import {
  MdCheckBoxOutlineBlank,
  MdCheckBox,
  MdRemoveCircleOutline,
} from 'react-icons/md';
import cn from 'classnames';
import './TodoListItem.scss';

const TodoListItem = ({ todo, onRemove, onToggle }) => {
  const { id, text, checked } = todo;
  return (
    <div className="TodoListItem">
      <div className={cn('checkbox', { checked })} onClick={() => onToggle(id)}>
        {checked ? <MdCheckBox /> : <MdCheckBoxOutlineBlank />}
        <div className="text">{text}</div>
      </div>
      <div className="remove" onClick={() => onRemove(id)}>
        <MdRemoveCircleOutline />
      </div>
    </div>
  );
};

export default React.memo(TodoListItem);
~~~
TodoListItem 컴포넌트는 todo, onRemove, onToggle이 바뀌지 않으면 리렌더링을 하지 않습니다.

<br />

## onToggle, onRemove 함수가 바뀌지 않게 하기
React.memo를 사용하는 것만으로 컴포넌트 최적화가 끝나지 않습니다. 현재 프로젝트에서는 todos 배열이 업데이트되면 onRemove와 onToggle 함수도 새롭게 바뀌기 때문입니다. onRemove와 onToggle 함수는 배열 상태를 업데이트하는 과정에서 최신 상태의 Todos를 참조하기 때문에 todos 배열이 바뀔 때마다 함수가 새로 만들어집니다.

<br />

#### useState 함수형 업데이트
기존에 setTodos 함수를 사용할 때는 새로운 상태를 파라미터로 넣어 주었습니다. setTodos를 사용할 때 새로운 상태를 파라미터로 넣는 대신, 상태 업데이트를 어떻게 할지 정의해 주는 업데이트 함수를 넣을 수도 있습니다.
~~~js
import React, { useState, useCallback, useRef } from 'react';
import TodoTemplate from './components/TodoTemplate';
import TodoInsert from './components/TodoInsert';
import TodoList from './components/TodoList';

function createBulkTodos() {
  const array = [];
  for (let i = 1; i <= 2500; i++) {
    array.push({
      id: i,
      text: `할 일 ${i}`,
      checked: false,
    });
  }
  return array;
}

const App = () => {
  const [todos, setTodos] = useState(createBulkTodos);

  //고윳값으로 사용될 id
  //ref를 사용하여 변수 담기
  const nextId = useRef(2501);

  const onInsert = useCallback((text) => {
    const todo = {
      id: nextId.current,
      text,
      checked: false,
    };
    setTodos((todos) => todos.concat(todo));
    nextId.current += 1; //nextId 1씩 더하기
  }, []);

  const onRemove = useCallback((id) => {
    setTodos((todos) => todos.filter((todo) => todo.id !== id));
  }, []);

  const onToggle = useCallback(
    (id) => {
      setTodos((todos) =>
        todos.map((todo) =>
          todo.id === id ? { ...todo, checked: !todo.checked } : todo,
        ),
      );
    },
    [todos],
  );

  return (
    <TodoTemplate>
      <TodoInsert onInsert={onInsert} />
      <TodoList todos={todos} onRemove={onRemove} onToggle={onToggle} />
    </TodoTemplate>
  );
};

export default App;
~~~

<br />

#### useReducer 사용하기
useReducer를 사용할 때는 원래 두 번째 파라미터에 초기 상태를 넣어 주어야 합니다. 지금은 그 대신 두 번째 파라미터에 undefined를 넣고, 세 번째 파라미터에 초기 상태를 만들어주는 함수인 createBulkTodos를 넣어 주었는데요. 이렇게 하면 컴포넌트가 맨 처음 렌더링될 때만 createBulkTodos 함수가 호출됩니다.
~~~js
import React, { useState, useCallback, useRef, useReducer } from 'react';
import TodoTemplate from './components/TodoTemplate';
import TodoInsert from './components/TodoInsert';
import TodoList from './components/TodoList';

function createBulkTodos() {
  const array = [];
  for (let i = 1; i <= 2500; i++) {
    array.push({
      id: i,
      text: `할 일 ${i}`,
      checked: false,
    });
  }
  return array;
}

function todoReducer(todos, action) {
  switch (action.type) {
    case 'INSERT': //새로추가
      return todos.concat(action.todo);
    case 'REMOVE':
      return todos.filter((todo) => todo.id !== action.id);
    case 'TOGGLE':
      return todos.map((todo) =>
        todo.id === action.id ? { ...todo, checked: !todo.checked } : todo,
      );
    default:
      return todos;
  }
}

const App = () => {
  const [todos, dispatch] = useReducer(todoReducer, undefined, createBulkTodos);

  //고윳값으로 사용될 id
  //ref를 사용하여 변수 담기
  const nextId = useRef(2501);

  const onInsert = useCallback((text) => {
    const todo = {
      id: nextId.current,
      text,
      checked: false,
    };
    dispatch({ type: 'INSERT', todo });
    nextId.current += 1;
  }, []);

  const onRemove = useCallback((id) => {
    dispatch({ type: 'REMOVE', id });
  }, []);

  const onToggle = useCallback((id) => {
    dispatch({ type: 'TOGGLE', id });
  }, []);

  return (
    <TodoTemplate>
      <TodoInsert onInsert={onInsert} />
      <TodoList todos={todos} onRemove={onRemove} onToggle={onToggle} />
    </TodoTemplate>
  );
};

export default App;
~~~

<br />

## 불변성의 중요성
리액트 컴포넌트에서 상태를 업데이트할 때 불변성을 지키는 것은 매우 중요합니다.
기존의 값을 직접 수정하지 않으면서 새로운 값을 만들어 내는 것을 '불변성을 지킨다'고 합니다.
~~~js
// 예제

const todos = [
  { id: 1, checked: true },
  { id: 2, checked: true },
];

const nextTodos = [...todos];

//nextTodos[0].checked = false; //아직가지는 똑같은 객체를 가리키고 있기 때문에 true

console.log(todos);
console.log(nextTodos);
console.log(todos[0] === nextTodos[0]);

nextTodos[0] = {
  ...nextTodos[0],
  checked: false,
};

console.log(todos[0] === nextTodos[0]); // 새로운 객체를 할당해 주었기에 false
~~~

<br />

## react-virtualized를 사용한 렌더링 최적화
react-virtualized를 사용하면 리스트 컴포넌트에서 스크롤되기 전에 보이지 않는 컴포넌트는 렌더링하지 않고 크기만 차지하게끔 할 수 있습니다. 그리고 만약 스크롤되면 해당 스크롤 위치에서 보여 주어야 할 컴포넌트를 자연스럽게 렌더링시키죠. 이 라이브러리를 사용하면 낭비되는 자원을 아주 쉽게 아낄 수 있습니다.

#### 최적화 준비
~~~
$ yarn add react-virtualized
~~~

~~~js
// TodoList.js

import React, { useCallback } from 'react';
import { List } from 'react-virtualized';
import TodoListItem from './TodoListItem';
import './TodoList.scss';

const TodoList = ({ todos, onRemove, onToggle }) => {
  const rowRenderer = useCallback(
    ({ index, key, style }) => {
      const todo = todos[index];
      return (
        <TodoListItem
          todo={todo}
          key={key}
          onRemove={onRemove}
          onToggle={onToggle}
          style={style}
        />
      );
    },
    [onRemove, onToggle, todos],
  );
  return (
    <List
      className="TodoList"
      width={512}
      height={513}
      rowCount={todos.length}
      rowHeight={57}
      rowRenderer={rowRenderer}
      list={todos}
      style={{ outline: 'none' }}
    />
  );
};

export default React.memo(TodoList);
~~~
List 컴포넌트를 사용할 때는 해당 리스트의 전체 크기와 각 항목의 높이, 각 항목을 렌더링할 때 사용해야 하는 함수, 그리고 배열을 props로 넣어 주어야 합니다.

~~~js
// TodoListItem.js

import React from 'react';
import {
  MdCheckBoxOutlineBlank,
  MdCheckBox,
  MdRemoveCircleOutline,
} from 'react-icons/md';
import cn from 'classnames';
import './TodoListItem.scss';

const TodoListItem = ({ todo, onRemove, onToggle, style }) => {
  const { id, text, checked } = todo;
  return (
    <div className="TodoListItme-virtualized" style={style}>
      <div className="TodoListItem">
        <div
          className={cn('checkbox', { checked })}
          onClick={() => onToggle(id)}
        >
          {checked ? <MdCheckBox /> : <MdCheckBoxOutlineBlank />}
          <div className="text">{text}</div>
        </div>
        <div className="remove" onClick={() => onRemove(id)}>
          <MdRemoveCircleOutline />
        </div>
      </div>
    </div>
  );
};

export default React.memo(
  TodoListItem,
  (prevProps, nextProps) => prevProps.todo === nextProps.todo,
);
~~~

~~~css
// TodoListItem.scss

.TodoListItme-virtualized {
  //엘리먼트 사이사이에 테두리를 넣어 줌
  & + & {
    border-top: 1px solid #dee2e6;
  }
  &:nth-child(even) {
    background: #f8f9fa;
  }
}

.TodoListItem {
  padding: 1rem;
  display: flex;
  align-items: center;
  .checkbox {
    cursor: pointer;
    flex: 1;
    display: flex;
    align-items: center; //세로 중앙 정렬
    svg {
      font-size: 1.5rem;
    }
    .text {
      margin-left: 0.5rem;
      flex: 1; //차지할 수 있는 영역 모두 차지
    }
    //체크되었을 때 보여 줄 스타일
    &.checked {
      svg {
        color: #22b8cf;
      }
      .text {
        color: #adb5bd;
        text-decoration: line-through;
      }
    }
  }
  .remove {
    display: flex;
    align-items: center;
    font-size: 1.5rem;
    color: #ff6b6b;
    cursor: pointer;
    &:hover {
      color: #ff8787;
    }
  }
}
~~~

<br />

## Project 실행
```md
./react-todo-app-project/todo-app

$ yarn start
```