# 코드 생성과 타입이 관계없음을 이해하기

큰 그림에서 보면, 타입스크립트 컴파일러는 두 가지 역할을 수행한다.

- 최신 TS/JS를 브라우저에서 동작할 수 있도록 구버전의 JS로 트랜스파일한다.
- 코드의 타입 오류를 체크한다.

## 타입 오류가 있는 코드도 컴파일이 가능하다

컴파일을 타입 체크와 독립적으로 동작하기 때문에, 타입 오류가 있는 코드도 컴파일이 가능하다. -> `.js`파일이 생성됨.

타입스크립트 오류는 C나 자바 같은 언어들의 warning와 비슷하다. 문제가 될 만한 부분을 알려주지만, 빌드를 멈추지는 않는다.

> `컴파일과 타입 체크`  
> 코드에 오류가 있을 때 "컴파일에 문제가 있다"고 말하는 경우를 보았을 것이다. 그러나 이는 기술적으로 틀린 말이다. 엄밀히 말하면 오직 코드 생성만이 "컴파일"이라고 할 수 있기 때문이다. 작성한 TS가 유효한 JS라면 TS 컴파일러는 컴파일을 해 낸다. 그러므로 코드에 오류가 있을 때 "타입 체크에 문제가 있다"고 말하는 것이 더 정확한 표현이다.

만약 오류가 있을 때 컴파일하지 않으려면, tsconfig.json에 `noEmitOnError`를 설정하거나 빌드 도구에 동일하게 적용하면 된다.

## 런타임에는 타입 체크가 불가능하다

```js
interface Square {
  width: number;
}
interface Rectangle extends Square {
  height: number;
}
type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
  if (shape instanceof Rectangle) {
    // Rectangle은 형식만 참조하지만, 여기서는 값으로 사용되고 있다.
    return shape.width * shape.height; // Shape 형식에 height 속성이 없다.
  } else {
    return shape.width * shape.width;
  }
}
```

`instanceof` 체크는 런타임에 일어나지만, `Rectangle은 타입이기 때문에 런타임 시점에 아무런 역할을 할 수 없다.` TS의 타입은 "제거 가능(erasable)"하다. 실제로 JS로 컴파일되는 과정에서 모든 인터페이스, 타입, 타입 구문은 그냥 제거되어 버린다.
앞의 코드에서 다루고 있는 shape타입을 명확하게 하려면, 런타임에 타입 정보를 유지하는 방법이 필요하다. 하나의 방법은 height 속성이 존재하는지 체크해 보는 것이다.

```js
function calculateArea(shape: Shape) {
  if ("height" in shape) {
    shape; // 타입이 Rectangle
    return shape.width * shape.height;
  } else {
    shape; // 타입이 Square
    return shape.width * shape.width;
  }
}
```

속성 체크는 런타임에 접근 가능한 값에만 관련되지만, 타입 체커 역시도 shape의 타입을 Rectangle로 보정해 주기 때문에 오류가 사라진다.
타입 정보를 유지하는 또 다른 방법으로는 런타임에 접근 가능한 타입 정보를 명시적으로 저장하는 "태그"기법이 있다.

```js
interface Square {
  kind: "square";
  width: number;
}

interface Rectangle extends Square {
  kind: "rectangle";
  height: number;
  width: number;
}

type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
  if (shape.kind === "rectangle") {
    shape; // 타입이 Rectangle
    return shape.width * shape.height;
  } else {
    shape; // 타입이 Square
    return shape.width * shape.width;
  }
}
```

여기서 Shape의 타입은 "태그된 유니온"의 한 예다. 이 기법은 런타임에 타입 정보를 손쉽게 유지할 수 있기 때문에, TS에서 흔하게 볼 수 있다.  
타입(런타임 접근 불가)과 값(런타임 접근 가능)을 둘 다 사용하는 기법도 있다. 타입을 클래스로 만들면 된다. Square와 Rectangle을 클래스로 만들면 오류를 해결할 수 있다.

```js
class Square {
  constructor(public width: number) {}
}
class Rectangle extends Square {
  constructor(public width: number, public height: number) {
    super(width);
  }
}
type Shape = Square | Rectangle;

function calculateArea(shape: Shape) {
  if(shape instanceof Rectangle) {
    shape; // 타입이 Rectangle
    return shape.width * shape.height;
  } else {
    shape; // 타입이 Square
    return shape.width * shape.width; // 정상
  }
}
```

인터페이스는 타입으로만 사용 가능하지만, Rectangle을 클래스로 선언하면 타입과 값으로 모두 사용할 수 있으므로 오류가 없다.  
`type Shape = Square | Rectangle;`에서 Rectangle은 타입으로 참조되지만, `shape instanceof Rectangle`에서는 값으로 참조된다.

## 타입 연산은 런타임에 영향을 주지 않는다.

string 또는 number 타입인 값을 항상 Number로 정제하는 경우를 가정한다. 다음 코드는 타입 체커를 통과하지만 잘못된 방법을 썼다.

```js
function asNumber(val: number | string): number {
  return val as number;
}
```
