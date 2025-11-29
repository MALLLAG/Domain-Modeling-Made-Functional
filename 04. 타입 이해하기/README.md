# 타입 이해하기

## 함수 이해하기

고등학교 수학시간에 함수는 입력과 출력이 있는 블랙박스 같은 것이라고 배웠습니다.  
예를 들어, 사과를 바나나로 바꾸는 어떤 함수는 입력과 출력을 화살표로 연결하여 다음과 같이 나타낼 수 있습니다

<img width="556" height="180" alt="Image" src="https://github.com/user-attachments/assets/5177ad24-5974-4a77-af77-e49384147ea9" />

`apple -> banana`같은 표현을 타입 시그니처 또는 함수 시그니처라 부릅니다.  
함수 시그니처를 이해하고 활용하는 것은 함수형 코딩에서 중요한 부분이므로 어떻게 동작하는지 잘 이해해야 합니다.

```typescript
function add1(x: number): number {
  return x + 1;
}

function add(x: number, y: number): number {
  return x + y;
}
```

```kotlin
fun add1(x: Int) = x + 1
fun add(x: Int, y: Int) = x + y
```

TypeScript와 Kotlin 모두 함수의 입출력 타입을 중요하게 생각합니다.  
두 언어 모두 출력 타입은 컴파일러가 어느정도 추론할 수 있지만, 코드의 안정성을 높이기 위해 함수 시그니처를 명시적으로 드러내기를 권장합니다.

Kotlin은 함수를 정의하기 위해 함수 타입을 갖는 `value`로 선언하는 것이 어색하게 느껴집니다.  
람다 표현식이 더 적합한 경우가 아니라면 일반적인 방식으로도 간결하게 함수를 정의할 수 있습니다.  
**합수가 입출력으로 여러 타입을 다룬다면, 아래처럼 여러 타입을 추상화하는 제네릭 타입을 활용합니다.**

```typescript
const areEqual = <T>(x: T) => (y: T) => x === y;
```

```kotlin
fun <T> areEqual(x: T, y: T) = x == y
```

## 타입과 함수

**함수형 프로그래밍에서 타입은 중요한 역할을 합니다. 타입은 함수의 입력 또는 출력이 될 수 있는 값들의 집합을 지칭합니다.**  
예를 들어 -32768에서 +32767까지의 범위 정수 집합을 `int16`이라고 지칭할 수 있습니다.

<img width="764" height="215" alt="Image" src="https://github.com/user-attachments/assets/304072d5-e1cf-414d-854d-b691bd49cabb" />

타입은 함수의 시그니처를 결정하므로, 이 함수의 시그니처는 다음과 같습니다.

`int16 -> 공역`

<br>

만약 모든 가능한 문자열들의 집합, string을 출력하는 함수라면 시그니처는 다음과 같습니다.

`정의역 -> string`

<br>

타입을 이루는 원소들이 꼭 원시 타입일 필요는 없습니다. 예를 들면 Person이라는 객체 집합을 다루는 함수도 있을 수 있습니다.  
개념적으로 보자면 타입을 이루는 원소는 실제 혹은 가상의 무엇이든 될 수 있습니다.  
함수를 원소로 갖는 함수 집할 또한 타입이 될 수 있습니다.  
TypeScript는 함수를 출력하는 함수를 화살표 함수로 쉽게 나타낼 수 있고, Kotlin은 입력을 함수로 받을때 람다 표현식을 씁니다.

```typescript
const add = (x: number) => (y: number) => x + y;
```

```kotlin
// fun ... either(block: Raise<Error>.() -> A): ...
either { ... }  // 람다 표현식을 입력으로 받는 either 함수
```

예시의 either 함수는 마지막 매개변수가 함수인 것을 알 수 있습니다.  
이런 경우 Kotlin은 주로 매개변수 block과 타입이 같은 함수를 전달하는 것이 아니라, 람다 표현식을 직접 작성하는 것을 선호합니다.  
이렇게 마지막 매개변수로 전달하는 람다 표현식을 **후행 람다**라고 부르며, 함수 호출시 매개변수를 묶는 소괄호 바깥에 람다 표현식을 두는 것을 허용하여 가독성을 증가시킵니다.

> ### 용어 주의: 값 vs. 객체 vs. 변수
> 함수형 프로그래밍 언어에서는 대부분의 것을 ‘값’이라 부릅니다. 객체지향 언어에서는 대부분의 것을 ‘객체’라 부릅니다. 그렇다면 ‘값’과 ‘객체’의 차이점은 무엇일까요?
> 값은 단순히 타입이라는 집합의 원소이며 입력이나 출력으로 사용됩니다. 예를 들어 `1`은 `int` 타입의 값이고, `"abc"`는 `string` 타입의 값입니다.  
> 함수도 값이 될 수 있습니다. `const add1 = (x: number) => x + 1;`과 같은 간단한 함수를 정의하면 `add1`은 그 타입의 형태가 `int->int`인 함수값입니다. 값은 불변이기에 변수라고 할 수 없습니다. 또한 값은 관련 동작이 없는 단순 데이터입니다.
> 반면 객체는 데이터 구조와 관련 동작(메서드)을 캡슐화한 것입니다. 일반적으로 객체는 상태(즉, 가변성)를 가지며, 내부 상태를 변경하는 모든 작업은 객체가 점 표기법으로 직접 제공해야 합니다.
> 따라서 객체가 없는 함수형 프로그래밍 세계에서는 ‘변수’ 또는 ‘객체’ 대신 ‘값’이라는 용어를 사용해야 합니다.

## 타입 합성

함수형 프로그래밍에서는 합성이라는 단어를 자주 만나는데, 이는 함수형 디자인이 합성에 기반하기 때문입니다.  
합성을 써서 작은 함수에서 새로운 함수를 만들고, 작은 타입에서 새로운 타입을 만듭니다.  
합수형 프로그래밍에서는 다음 두 가지 방법으로 새로운 타입을 합성합니다.

- AND로 연결
- OR로 연결

### AND 타입

예를 들어 과일 샐러드에는 사과, 바나나, 체리가 필요합니다.  
함수형 언어에서는 이를 곱 타입(product type)이라고 부릅니다. *이 책에서는 곱 타입을 레코드라 합니다.*  
FruitSalad 레코드 정의는 다음과 같습니다.

```typescript
class FruitSalad {
  constructor (
    readonly apple: AppleVariety,
    readonly banana: BananaVariety,
    readonly cherries: CherryVariety,
  ) { }
}
```

```kotlin
data class FruitSalad (
  val apple: AppleVariety,
  val banana: BananaVariety,
  val cherries: CherryVariety,
)
```

### OR 타입

새로운 타입을 만드는 다른 방법은 OR를 쓰는 것입니다. 예를 들어 과일 간식으로 사과, 바나나, 체리가 필요할 수 있습니다.  
함수형 프로그래밍에서는 이런 타입을 합 타입(sum type)이라고 부릅니다.
FruitSnack의 정의는 다음과 같습니다.

```typescript
type FruitSnack = AppleVariety | BananaVariety | CherryVariety;
type AppleVariety = "GoldenDelicious" | "GrannySmith" | "Fuji";
type BananaVariety = "Cavendish" | "GrosMichel" | "Manzano";
type CherryVariety = "Montmorency" | "Bing";
```

```kotlin
sealed interface FruitSnack
enum class AppleVariety: FruitSnack { GoldenDelicious, GrannySmith, Fuji }
enum class BananaVariety: FruitSnack { Cavendish, GrosMichel, Manzano }
enum class CherryVariety: FruitSnack { Montmorency, Bing }
```

Kotlin은 합 타입을 지원하지 않으므로 sealed interface로 선택 타입을 정의합니다.  
이는 외부 모듈과 패키지에서 이를 구현하지 못하게 막아서 FruitSnack의 구현체를 AppleVariety, BananaVariety, CherryVariety만으로 한정하려는 의도입니다.

### 단순 타입

우리는 종종 단일 필드를 포함하는 래퍼 타입(wrapper type)을 정의합니다.

```typescript
type ProductCode = {readonly value: string} & {"ProductCode": never};
```

```kotlin
@JvmInline
value class ProductCode(val value: String)
```

### 대수적 타입 시스템

대수적 타입 시스템(algebraic type system)은 더 작은 타입들을 곱 타입과 합 타입으로 연결하여, 모든 복합 타입을 구성하는 시스템을 말합니다.  
TypeScript는 대부분의 함수형 언어처럼 대수적 타입 시스템을 지원합니다.

새로운 데이터 타입을 구축하기 위해 AND, OR을 사용하는 것에 익숙해야 합니다.  
대수적 타입 시스템이 실제로 도메인을 모델링하는데 탁월한 도구임을 느낄 수 있습니다.

## TypeScript 및 Kotlin 타입 다루기

```typescript
// Person 레코드
type Person = {
    first: string;
    last: string;
}
```

```typescript
// 레코드 값을 만드는 방법
const alex: Person = {
    first: "Alex",
    last: "Smith",
}
```

레코드 정의에서 하위 타입이 있던 곳에 값을 넣으면 그대로 레코드의 값이 됩니다.  
클래스나 오브젝트같은 개념을 노출하지 않고 타입 정의에 따라 하위 타입의 값들을 합성하여 새로운 값을 만들었습니다.  
**클래스라는 개념이 없는 함수형 언어에서 레코드는 레코드 양식에 맞게 하위 타입 값들을 묶어두는 것만으로 충분합니다.**

<br>

이번엔 반대로 레코드의 값에서 하위 타입의 값을 분해하는 방법입니다.  

```typescript
const {first, last}: Person = alex;
```

```kotlin
val first = alex.first
val last = alex.last
```

<br>

이번엔 여러 하위 타입을 선택 타입으로 합성하는 방법입니다.

```typescript
type OrderQuantity = UnitQuantity | KilogramQuantity;
```

```typescript
// 선택 타입의 값 생성 방식 (하위 타입 중 하나로 객체를 만들어 할당)
const anOrderQtyInUnits: OrderQuantity = new UnitQuantity(10);
const anOrderQtyInKg: OrderQuantity = new KilogramQuantity(2.5);
```

```typescript
// 선택 타입의 값에서 하위 타입으로 타입 좁히기
import { match, P } from 'ts-pattern';

match(orderQuantity)
  .with(P.instanceOf(UnitQuantity), i => console.log(i.value))
  .with(P.instanceOf(KilogramQuantity), i => console.log(i.value))
  .exhaustive();
```


```kotlin
sealed interface OrderQuantity {
    @JvmInline
    value class Unit(val value: Int): OrderQuantity
    
    @JvmInline
    value class Kilogram(val value: Double): OrderQuantity
}
```

```kotlin
// 선택 타입의 값 생성 방식
val anOrderQtyInUnits = OrderQuantity.Unit(10)
val anOrderQtyInKg = OrderQuantity.Kilogram(2.5)
```

```kotlin
// 타입 좁히기
when (orderQuantity) {
    is OrderQuantity.Unit -> println(orderQuantity.value)
    is OrderQuantity.Kilogram -> println(orderQuantity.value)
}
```

## 타입으로 도메인 만들기

타입 합성은 도메인 주도 설계를 하는데 큰 도움이 됩니다. 타입들을 여러 방식으로 조합하여 복잡한 모델을 손쉽게 만들 수 있습니다.  
다음은 전자상거래 사이트의 결제 도메인 예시입니다.

```typescript
declare const checkNumber: unique symbol;
class CheckNumber {
  [checkNumber]!: never;
  constructor(readonly value: number) {}
}

declare const cardNumber: unique symbol;
class CardNumber {
  [cardNumber]!: never;
  constructor(readonly value: string) {}
}
```

`[checkNumber]!: never;` 같은 부분은 TypeScript의 구조적 서브타입때문에 생긴 미봉책입니다.  
단순 타입은 내부에 단일 원시 타입을 감싸므로 원시 타입이 같은 모든 단순 타입들은 동일한 구조를 가집니다.  
예를 들어 number를 감싸는 래퍼는 그 타입이 Kilogram이건 Pound건 모두 `{ value: number }`라는 동일한 구조를 가집니다.  
우리가 number를 쓰지 않고 Kilogram, Pound 이렇게 단순 타입을 정의한다는 것은, 서로 호환하지 못하게 타입으로 막으려는 의도입니다.  
하지만 두 구조가 같으므로 TypeScript는 Kilogram 타입에 Pound 오브젝트를 넣는 것이 가능합니다.
**따라서 원시 타입마다 unique symbol을 프로퍼티로 가지면 상호 호환을 막을 수 있습니다 !**  
이렇게 동일한 구조를 가진 타입간의 정적 호환을 막기 위한 타입 식별 장치를 추가하는 것을 **브랜딩**이라 합니다.

반면 Kotlin은 이름에 의한 서브타입(명목적 서브타입)과 래퍼용 `inline value class`를 지원하므로 간략한 형태로 구현할 수 있습니다.

```kotlin
@JvmInline
value class CheckNumber(val value: Double)

@JvmInline
value class CardNumber(val value: String)
```

<br>

이어서 저수준 타입들을 구축합니다. CardType은 선택 타입으로 Visa 또는 MasterCard 중 하나의 값을 가집니다.  
CreditCardInfo는 레코드로 CardType 및 CardNumber를 포합합니다.

```typescript
type CardType = "Visa" | "Mastercard";
class CreditCardInfo {
  constructor(
    readonly cardType: CardType,
    readonly cardNumber: CardNumber,
  ) { }
}
```

```kotlin
enum class CardType { Visa, Mastercard }
data class CreditCardInfo(
  val cardType: CardType,
  val cardNumber: CardNumber,
)
```

<br>

PaymentMethod라는 또 다른 선택 타입은 현금, 수표, 카드 중 하나를 선택합니다.

```typescript
type PaymentMethod = Cash | CheckNumber | CreditCardInfo;
```

```kotlin
sealed interface PaymentMethod {
  object Cash: PaymentMethod

  @JvmInline
  value class CheckNumber(val value: Double): PaymentMethod

  data class CreditCardInfo(val cardType: CardType, val cardNumber: CardNumber): PaymentMethod
}
```

<br>

PaymentAmount 및 Currency 정의는 다음과 같습니다.

```typescript
declare const paymentAmount: unique symbol;
class PaymentAmount {
  [paymentAmount]!: never;
  constructor(readonly value: number) {}
}

type Currency = "EUR" | "USD";
```

```kotlin
@JvmInline
value class PaymentAmount(val value: Double)

enum class Currency { EUR, USD }
```

<br>

마지막으로 Payment는 PaymentAmount, Currency, PaymentMethod를 포함하는 레코드입니다.

```typescript
class Payment {
  constructor(
    readonly amount: PaymentAmount,
    readonly currency: Currency,
    readonly method: PaymentMethod,
  ) {}
}
```

```kotlin
class Payment (
  val amount: PaymentAmount,
  val currency: Currency,
  val method: PaymentMethod,
)
```

<br>

객체지향 모델과 달리 함수형 모델은 타입에 직접 붙어있는 동작은 없습니다.  
다음으로 타입으로 수행 가능한 작업을 문서화하기 위해 함수 타입을 정의합니다.

```typescript
type PayInvoice = (i: UnpaidInvoice, j: Payment) => PaidInvoice;
```

```kotlin
typealias PayInvoice = (UnpaidInvoice, Payment) -> PaidInvoice
```

이는 UnpaidInvoice와 Payment를 입력하면 PaidInvoice가 만들어진다는 시그니처입니다.  

## 없어도 되는 값, 오류 및 컬렉션 모델링

도메인 모델링에서 종종 만나는 이슈들을 TypeScript, Kotlin 타입 시스템으로 어떻게 나타낼지 논의합니다.

- 없어도 되거나 누락된 값
- 오류
- 값을 반환하지 않는 함수
- 컬렉션

### 없어도 되는 값 모델링

이를 모델링하기 위해선 누락된 데이터의 의미를 살펴봐야 합니다. 이는 어떤 값이 존재하거나 전혀 존재하지 않음을 의미합니다.  
이를 `Option`이라는 선택 타입으로 모델링할 수 있습니다.

```typescript
type Option<A> = None | Some<A>;
```

Some은 A타입의 데이터가 들어있음을 의미하고, None은 데이터가 없다는 뜻입니다.  
예를 들어 PersonName 타입이 있고, 이름과 성은 필수지만 중간 이름은 없어도 되면 다음과 같이 모델링할 수 있습니다.

```typescript
import { Option } from 'fp-ts/Option';
class PersonName {
    readonly firstName: string,
    readonly middleInitial: Option<string>,
    readonly lastName: string,
}
```

```kotlin
data class PersonName(
  val firstName: String,
  val middleInitial: String?,
  val lastName: String,
)
```

### 오류 모델링

결제에 성공하거나 카드가 만료되어 결제가 실패하는 프로세스라면 어떻게 모델링해야 할까요 ?  
예외를 던질수도 있지만 실패가 발생할 수 있다는 사실을 명시적으로 함수 시그니처에 문서화할 수 있습니다.   
이를 위해 성공과 실패의 선택 타입이 필요하므로 `Either`라는 타입을 정의합니다.

```typescript
type Either<E, A> = Left<E> | Right<A>;
```

```kotlin
public sealed class Either<out A, out B> {
    public data class Left<out A> constructor(val value: A) : Either<A, Nothing>() { ... }
    public data class Right<out B> constructor(val value: B) : Either<Nothing, B>() { ... }
}
```

<br>

함수 호출에 성공하여 반환값이 있을때는 Right를 쓰고, 실패하여 오류가 나면 Left를 사용합니다.  
함수가 실패할 수 있음을 나타내려면 출력을 Either 타입으로 감쌉니다.

```typescript
type PayInvoice = (i: UnpaidInvoice, j: payment) => Either<PaymentError, PaidInvoice>;
```

```kotlin
typealias PayInvoice = (UnpaidInvoice, Payment) -> Either<PaymentError, PaidInvoice>
```

<br>

그 다음 PaymentError를 가능한 오류들의 선택 타입으로 정의할 수 있습니다.

```typescript
type PaymentError = CardTypeNotRecognized | PaymentRejected | PaymentProviderOffline;
```

```kotlin
sealed interface PaymentError

class CardTypeNotRecognized: PaymentError { ... }
class PaymentRejected: PaymentError { ... }
class PaymentProviderOffline: PaymentError { ... } }
```

### 값 자체가 없음 모델링

함수형 언어에서는 모든 함수가 무언가를 반환해야 하므로 void를 사용할 수 없습니다.  
대신 Unit이라는 특수한 내장 타입을 사용하며, Unit 타입에는 특별한 값 하나만 있습니다.

> TypeScript를 위한 변(辨): void, never, undefined  
> void는 말 그대로 없다는 뜻입니다. 그렇다면 대체 뭐가 없다는 것일까요? 만약 값이 없다는 뜻이라면 void가 아니라 never입니다.  
> 아무런 값이 없는 공집합을 TypeScript에서는 never라 부릅니다. 그렇다면 void 대신 never를 써서 어떤 오류가 나는지 확인해봅시다.
> ```typescript
> function test(): never { return } // Type 'undefined' is not assignable to type 'never'.
> ```
> 아무 값도 반환하지 않는데 타입이 undefined로 잡힌 것을 볼 수 있습니다. 그리고 이는 never 타입의 하위 타입은 아닙니다. undefined는 JavaScript에서 받은 유산입니다.  
> **아무것도 반환하지 않더라도 ‘변수 공간은 잡았는데 값이 들어 있지 않아 타입이 없는 상태’를 말하는 undefined 타입을 반환합니다. 이는 함수형 언어에는 존재하지 않는 개념입니다.**  
> TypeScript는 undefined를 함수 반환이 없다는 뜻의 void 타입의 하위 타입으로 받아들여서 이 문제를 해결한 것으로 보입니다.


```typescript
type SaveCustomer = (i: Customer) => void;
```

```kotlin
typealias SaveCustomer = (Customer) -> Unit
```

<br>

또는 입력은 없지만 출력은 있는 함수는 다음과 같이 입력을 생략합니다.

```typescript
type NextRandom = () => number;
```

```kotlin
typealias NextRandom = () -> Int
```

**시그니처에 입력이 없거나 출력이 void, Unit 타입이라면 명백히 부수효과(side effect)가 있다는 뜻입니다.**

### 리스트 및 컬렉션 모델링

```typescript
type Order = {
    orderId: OrderId;
    lines: OrderLine[];
}
```

```kotlin
class Order(
    val orderId: OrderId,
    val lines: List<OrderLine>,
)
```

<br>

리스트 요소에 액세스하기 위해 패턴 매칭을 다음과 같이 활용할 수 있습니다.

```typescript
import { match, P } from 'ts-pattern';

const printList1 = (aList: number[]) => {
  const message = match(aList)
    .with([], () => 'list is empty')
    .with([P.any], elem => `list has one element: ${elem}`)
    .with([P.any, P.any], elems => `list has two elements: ${elems}`)
    .otherwise(() => 'list has more than two elements');

  console.log(message);
}

const printList2 = (aList: number[]) => {
  const message = match(aList)
    .with([], () => 'list is empty')
    .with([P.any, ...P.array()], ([first, rest]) => `list is non-empty with the first element: ${first}`)
    .exhaustive();
  console.log(message);
}
```

```kotlin
fun printList1(aList: List<Int>) {
    val message = when (aList.size) {
        0 -> "list is empty"
        1 -> "list has one element: ${aList}"
        2 -> "list has two elements: ${aList}"
        else -> "list has more than two elements"
    }
    println(message)
}

fun printList2(i: List<Int>) {
    when {
        i.isEmpty() -> println("list is empty")
        else -> {
            val (first) = i
            println("list is non-empty with first element: $first")
        }
    }
}
```












