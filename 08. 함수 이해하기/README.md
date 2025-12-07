# 함수 이해하기

## 함수, 함수 어디에나 함수

| 상황/목표 | 객체지향 (OOP)                                                  | 함수형 (FP) |
| :--- |:------------------------------------------------------------| :--- |
| **큰 프로그램을 작은 구성 요소로 조립할 때** | 구성 요소가 **클래스와 객체**일 것입니다.                                   | 구성 요소가 **함수**입니다. |
| **프로그램의 부분을 전달받거나<br>결합도를 줄일 때** | **인터페이스**와 **의존 주입(dependency injection)** 을 사용할 것입니다.      | **함수 매개변수**를 받습니다. |
| **코드를 재사용하고 싶을 때<br>('중복을 피하라')** | **상속**이나 **데커레이터 패턴(decorator pattern)** 같은 기술을 사용할 수 있습니다. | 모든 재사용 가능한 코드를 함수로 만들고, **함수 합성**을 통해 연결합니다. |

## 함수가 주인공

함수형 프로그래밍에서는 함수 자체를 독립적인 주체로 간주합니다.  
그리고 함수가 주인공이므로, 다른 함수에 입력값으로 전달할 수도 있습니다.

### 주인공인 함수

다음은 기존 함수 정의입니다.  
세 번째 정의는 익명 함수(람다 표현식)에 square라는 이름을 붙입니다.  
네 번째 정의는 plus3 함수에 addThree라는 이름을 붙입니다.

```typescript
function plus3(x: number): number {
  return x + 3;
}
function times2(x: number): number {
  return x * 2;
}
const square = (x: number) => x * x;
const addThree = plus3;
```


```kotlin
fun plus3(x: Int) = x + 3
fun times2(x: Int) = x * 2
val square: (Int) -> Int = { it * it }
val addThree = ::plus3
```

<br>

함수를 리스트에 넣을 수도 있습니다.

```typescript
// listOfFunctions : List<(i: number) => number>
const listOfFunctions = [addThree, times2, square];
```

```kotlin
// listOfFunctions : List<(Int) -> Int>
val listOfFunctions = listOf(addThree, ::times2, square)
```

### 입력으로서 함수

함수를 주인공으로 다루는것은 입력과 출력으로 사용할 수 있다는 뜻입니다.

```typescript
const evalWith5ThenAdd2 = (fn: (i: number) => number) => fn(5) + 2;
```

```kotlin
fun evalWith5ThenAdd2(fn: (Int) -> Int) = fn(5) + 2
```

<br>

```typescript
const add1 = (x: number) => x + 1;
evalWith5ThenAdd2(add1);
```

```kotlin
fun add1(x: Int) = x + 1
evalWith5ThenAdd2(add1)
```

### 출력으로서 함수

함수를 출력으로 반환하는 큰 이유는 특정 매개변수를 함수에 고정(bake in)할 수 있기 떄문입니다.

```typescript
const add1 = (x: number) => x + 1;
const add2 = (x: number) => x + 2;
const add3 = (x: number) => x + 3;
```

```kotlin
fun add1(x: Int) = x + 1
fun add2(x: Int) = x + 2
fun add3(x: Int) = x + 3
```

이 중복을 제거하려면 어떻게 해야 할까요 ??  
가산기 생성기(adder generator)를 만들어 특정값을 미리 설정한 더하기 함수를 반환하면 됩니다.

```typescript
const adderGenerater = (numberToAdd: number) => (x: number) => numberToAdd + x;
```

```kotlin
fun adderGenerator(numberToAdd: Int): (Int) -> Int = { it + numberToAdd }
```

### 커링

함수를 반환하는 트릭을 사용하면 여러 매개변수를 받는 함수를, 단일 매개변수를 연이어 받는 함수로 변환할 수 있습니다.  
예를 들어 두 개의 매개변수를 받는 add 함수가 있다고 가정해봅시다.

```typescript
const add = (n: number) => (x: number) => x + n;
const add3 = add(3);
```

```kotlin
fun add(n: Int, x: Int): Int = x + n
fun addCurried(x: Int): (Int) -> Int = { add(it, x) }
val add3 = addCurried(3)
```

### 부분 적용

모든 함수가 커리 함수라면, 다중 매개변수 함수에 인수 하나만 전달해도 나머지 매개변수들을 입력받는 새로운 함수를 얻을 수 있다는 뜻입니다.  
예를 들어 sayGreeting 함수는 두 개의 매개변수를 가집니다.

```typescript
const sayGreeting = (greeting: string) => (name: string) =>
console.log(`${greeting} ${name}`)
```

```kotlin
fun sayGreeting(greeting: String, name: String) =
    println("$greeting $name")
```

<br>

하지만 하나의 인수만 전달하여 새로운 함수를 만들 수 있습니다.

```typescript
const sayHello: (name: string) => void = sayGreeting("Hello");
const sayGoodbye: (name: string) => void = sayGreeting("Goodbye");
```

```kotlin
fun sayHello(name: String): Unit { sayGreeting("Hello", name) }
fun sayGoodbye(name: String): Unit { sayGreeting("Goodbye", name) }
```

이렇게 매개변수를 고정하는 방식을 부분 적용(partial application)이라고 합니다.

## 완전 함수

함수형 프로그래밍에서의 함수는 모든 가능한 입력을 출력으로 연결합니다. 즉, 모든 입력값에 해당하는 출력값이 존재하도록 합니다.  
이런 함수를 **완전 함수**라고 합니다. 이는 가능한 명확하게 표현하고 모든 효과를 타입 시그니처에 명시적으로 기록하기 위함입니다.

```typescript
const twelveDividedBy = (n: number) =>
  match(n)
    .with(6, _ => Some(2))
    // ...
    .with(0, _ => None)
    .otherwise(/* ... */);

// const twelveDividedBy: (n: number) => Option<number>
```

```kotlin
fun twelveDividedBy(n: Int): Int? = when(n) {
    6 -> 2
    // ...
    0 -> null
    else -> /* ... */
}

// val twelveDividedBy: (Int) -> Int?
```

이 시그니처는 정수를 입력받아 유효한 입력값일 경우 정수를 반환할 수 있지만, 때에 따라 반환값이 없을 수 있다는 뜻입니다.

## 함수 합성

함수 합성은 앞선 함수의 결과를 다음 함수의 입력으로 연결하여 함수를 결합하는 방식입니다.  
이러한 합성의 중요한 측면 중 하나는 **정보 은닉**입니다.  
최종 합성된 함수가 더 작은 함수들로 이루어져 있다는것을 알 수 없으며, 더 작은 함수들이 무엇을 처리했는지 알 수 없습니다.

### TypeScript와 Kotlin의 함수 합성

두 함수의 첫 번째 함수 출력 타입이 두 번째 함수의 입력 타입과 동일하기만 하면 두 함수를 결합할 수 있습니다.  
이를 일반적으로 **파이핑(piping)**이라고 부릅니다.

```typescript
const add1 = (x: number) => x + 1;
const square = (x: number) => x * x;
const add1ThenSquare = (x: number) => pipe(x, add1, square);
add1ThenSquare(5);
```

add1ThenSquare 함수 정의를 보면 단지 add1에 전달하기 위해 화살표 함수로 별도의 매개변수를 받아서 pipe()의 시작값으로 넣어주는 형태로 자주 발생하는 불필요한 보일러 플레이트 코드입니다.  
따라서 많은 경우 flow로 합성함수를 만듭니다.

```typescript
const add1ThenSquare = flow(add1, square);
```

```typescript
const isEven = (x: number) => x % 2 === 0;
const printBool = (x: boolean) => `value is ${x}`;
const isEvenThenPrint = flow(isEven, printBool);
isEvenThenPrint(2);
```

<br>

Kotlin은 확장 함수를 활용하기 때문에 메서드 체이닝으로 파이프라인을 구현합니다.

```kotlin
fun Int.add1() = this + 1
fun Int.square() = this * this
val add1ThenSquare: (Int) -> Int = { it.add1().square() }
add1ThenSquare(5)

fun Int.isEven() = (this % 2) == 0
fun Boolean.printBool() = "value is $this"
val isEvenThenPrint: (Int) -> String = { it.isEven().printBool() }
isEvenThenPrint(2)
```

### 녹록지 않은 함수 합성

출력과 입력이 일치하는 두 함수 합성은 매우 간단합니다. 하지만 입출력이 맞지 않을때는 어떻게 해야 할까요 ?  
기본적으로 입출력 타입은 동일하지만 `모양`이 다른 경우가 일반적입니다.

<img width="641" height="175" alt="Image" src="https://github.com/user-attachments/assets/c5b71e8a-161d-45ec-9146-50dd59eb769d" />

<br>

함수 합성에서 발생하는 많은 문제는 함수의 입력과 출력을 조정하여 맞추는 과정에서 일어납니다.  
인기있는 방식 중 하나는 양쪽을 동일한 타입으로 변환하는 것입니다. 즉, 양쪽에서 공통적으로 사용할 수 있는 `최소공배수`를 찾아내는 것입니다.  
예를 들어 출력이 `int`이고 입력이 `Option<int>`인 경우, 두 타입을 포함하는 가장 작은 타입은 `Option`입니다.

<img width="760" height="161" alt="Image" src="https://github.com/user-attachments/assets/d78d3415-b874-40ed-8cc3-5b5e6288d64f" />

```typescript
const add1 = (x: number) => x + 1;
const printOption = (x: Option<number>) => match(x)
  .with(P.instanceOf(Some), i => console.log("The int is ", i))
  .with(P.instanceOf(None), _ => console.log("No value"))
  .exhaustive();
pipe(5, add1, some, printOption);
```

```kotlin
fun add1(x: Int) = x + 1
fun Int?.printOptional() = println(this?.let { "The int is $it" } ?: "No value")
add1(10).printOptional() // "The int is 11" 출력
```















