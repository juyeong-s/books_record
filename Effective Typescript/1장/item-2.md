# TS 설정 이해하기

TS 설정은 커맨트 라인 `tsc --noImplicitAny program.ts`
이나 `tsconfig.json` 파일로 설정할 수 있다.

## JS 프로젝트를 TS로 전환하는 게 아니라면 noImplicitAny를 설정하는 것이 좋다.

`noImplicitAny`는 변수들이 미리 정의된 타입을 가져야 하는지 여부를 제어한다. TS는 타입 정보를 가질때 가장 효과적이기 때문에 가능하면 `noImplicitAny` 를 설정해야 한다. 다음 코드와 같이 any타입이라도 명시적으로 선언해주어야 한다.

```js
function add(a: number, b: number) {
  return a + b;
}
```

## "undefinded는 객체가 아닙니다."같은 런타임 오류를 방지하기 위해 strictNullChecks를 설정하는 것이 좋다.

`strictNullChecks`는 null과 undefinded가 모든 타입에서 허용되는지 확인하는 설정이다.

```js
const x: number = null; // 'null'형식은 'number'형식에 할당할 수 없다.
```

`strictNullChecks`를 설정하면 위 코드는 오류로 판단된다. null 대신 undefinded를 써도 마찬가지다.  
null을 사용하고 싶다면 명시적으로 `: number | null`로 설정해주면 된다.

반면, `strictNullChecks`는 오류를 잡아내기에는 좋지만 코드 작성을 어렵게 한다.
