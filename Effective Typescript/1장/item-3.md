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

string 또는 number 타입인 값을 항상 Number로 정제하는 경우(`as number`)를 가정한다. 다음 코드는 타입 체커를 통과하지만 잘못된 방법을 썼다.

```ts
function asNumber(val: number | string): number {
  return val as number;
}
```

변환된 js 코드를 보면 코드에 아무런 정제 과정이 없다. `as number`(타입 단언문)는 타입 연산이고, 런타임에 아무런 영향을 미치지 않는다.

```js
function asNumber(val) {
  return val;
}
```

값을 정제하기 위해서는 아래와 같이 명시적으로 런타임에 타입 체크를 해야한다.

```ts
function asNumber(val: number | string): number {
  return typeof val === "string" ? Number(val) : val;
}
```

## 런타임 타입은 선언된 타입과 다를 수 있다.

다음 함수를 보고 마지막의 console.log까지 실행될 수 있을지 생각해보자.

```ts
function setLightSwitch(value: boolean) {
  switch (value) {
    case true:
      turnLightOn();
      break;
    case false:
      turnLightOff();
      break;
    default:
      console.log("실행되지 않을까 봐 걱정됩니다.");
  }
}
```

ts는 일반적으로 실행되지 못하는 죽은(dead) 코드를 찾아내지만, 여기서는 strict를 설정하더라도 찾아내지 못한다. 그러면 마지막 부분을 실행할 수 있는 경우는 무엇일까?

`: boolean`이 타입 선언문이라는 것에 주목해보자. ts의 타입이기 때문에 런타임에 제거된다. js였다면 실수로 함수를 "ON"으로 호출할 수도 있다.  
순수 ts에서도 마지막 코드를 실행하는 방법이 존재한다. 예를 들어, 네트워크 호출로부터 받아온 값으로 함수를 실행하는 경우가 있다.

```ts
interface LightApiResponse {
  lightSwitchValue: boolean;
}
async function setLight() {
  const response = await fetch("/light");
  const result: LightApiResponse = await response.json();
  setLightSwitch(result.lightSwitchValue);
}
```

/light를 요청하면 그 결과로 `LightApiResponse`를 반환하라고 선언했지만, 실제로 그렇게 되리라는 보장은 없다. API를 잘못 파악해서 `LightApiResponse`가 실제로 문자열이었다면, 런타임에는 `setLightSwitch`함수까지 전달될 것이다.

## 타입스크립트 타입으로는 함수를 오버로드할 수 없다.

C++같은 언어는 동일한 이름에 매개변수만 다른 여러 버전의 함수를 허용한다. 이를 "함수 오버로딩"이라고 한다. 그러나 ts에서는 `타입과 런타임의 동작이 무관하기 때문에, 함수 오버로딩은 불가능하다.`  
ts가 함수 오버로딩 기능을 지원하기는 하지만, 온전히 타입 수준에서만 동작한다. 하나의 함수에 대해 여러 개의 선언문을 작성할 수 있지만, 구현체(implementation)는 오직 하나다.

```ts
function add(a: number, b: number): number;
function add(a: string, b: string): string;
function add(a, b) {
  return a + b;
}

const three = add(1, 2); // number
const twelve = add("1", "2"); // string
```

add에 대한 처음 두 개의 선언문은 타입 정보를 제공할 뿐이다. 이 두 선언문은 TS가 JS로 변환되면서 제거되며, 구현체만 남게 된다.

## 타입스크립트 타입은 런타임 성능에 영향을 주지 않는다.

타입과 타입 연산자는 JS 변환 시점에 제거되기 때문에, 런타임의 성능에 아무런 영향을 주지 않는다. TS의 정적 타입은 실제로 비용이 전혀 들지 않는다.

- 런타임 오버헤드가 없는 대신, 빌드타임 오버헤드가 있다. 파일러 성능을 매우 중요하게 생각하여 컴파일은 일반적으로 상당히 빠른 편이다. 오버헤드가 커지면 빌드 도구에서 트랜스타일만(transpile only)을 설정하여 타입 체크를 건너뛸 수 있다.
- TS가 컴파일하는 코드는 오래된 런타임 환경을 지원하기 위해 호환성을 높이고 성능 오버헤드를 감안할지, 호환성을 포기하고 성능 중심의 네이티브 구현체를 선택할지의 문제에 맞닥뜨릴 수도 있다.  
  예를 들어 제너레이터 함수가 ES5 타깃으로 컴파일되려면, TS 컴파일러는 호환성을 위한 특정 헬퍼 코드를 추가할 것이다. 이런 경우가 제너레이터의 호환성을 위한 오버헤드 또는 성능을 위한 네이티브 구현체 선택의 문제이다. 어떤 경우든지 호환성과 성능 사이의 선택은 컴파일 타깃과 언어 레벨의 문제이며 여전히 타입과는 무관하다.
