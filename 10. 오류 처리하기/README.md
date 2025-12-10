# 구현: 오류 처리하기

어떤 시스템에서도 오류는 발생하기 마련이며 이를 어떻게 처리하는지가 중요합니다.  
실부하를 받는 시스템은 일관되고 투명하게 오류를 처리해야 합니다.

## Either 타입으로 오류 드러내기

함수형 프로그래밍은 가능한 모든 것을 명황하게 드러내는 것을 중요하게 생각하는데, 오류 처리도 그렇습니다.  
함수의 성공 여부와 실패시 발생할 오류를 명확하게 함수에 드러내야 합니다. 오류 상황을 포함한 모든 가능한 결과를 타입 시그니처에 드러내는 완전 함수를 바라는 것입니다.  
Either 타입으로 함수의 성공, 실패를 명확히 드러낸 시그니처는 다음과 같습니다.

```typescript
type AddressValidationError = "InvalidFormat" | "AddressNotFound";

type CheckAddressExists = (i: UnvalidatedAddress) => Either<AddressValidationError, CheckedAddress>;
```

```kotlin
sealed interface AddressValidationError
object InvalidFormat : AddressValidationError
object AddressNotFound : AddressValidationError

typealias CheckAddressExists = UnvalidatedAddress.() -> Either<AddressValidationError, CheckedAddress>
```

이 함수 시그니처는 다음과 같은 중요한 사항들을 알려줍니다.

- 입력값은 `UnvalidatedAddress`입니다.
- 검증 성공시 출력은 `CheckedAddress`입니다.
- 형식이 잘못되었거나 주소를 찾을 수 없을때 검증은 실패합니다.

**이처럼 함수 시그니처가 문서 역할을 할 수 있습니다.**

## 도메인 오류 다루기

소프트웨어 시스템은 꽤나 복잡해서 모든 발생 가능 오류를 타입으로 처리할 수는 없습니다.  
그러니 먼저 오류를 분류하고, 그 중 특정 오류들만 처리하는 일관된 원칙을 세워볼 수 있습니다.

- 도메인 오류: 비즈니스 프로세스가 발생시킨 오류. 비즈니스 측에서는 이미 이들 오류를 처리할 절차가 마련되어 있으므로 코드도 이 프로세스를 반영해야 합니다. (ex. 결제팀에서 거부한 주문, 잘못된 제품 코드가 포함된 주문)
- 패닉: 시스템이 알 수 없는 상태에 빠지는 오류. (ex. 메모리 부족, 0으로 나누기, 널 참조 등)
- 인프라 오류: 아키텍처에서 발생한 오류. (ex. 네트워크 타임아웃, 인증 실패)

오류는 유형마다 각기 다른 방식으로 구현해야 합니다.  
도메인 오류는 도메인의 일부로서 도메인 모델링에 포함해야 합니다.  
패닉은 예외를 발생시켜 작업 흐름을 중단하고 애플리케이션의 메인 함수같은 최상단 함수에서 예외를 포착하도록 처리하는 것이 좋습니다.

```typescript
const workflowPart2 = (input: Input) => {
  if (input === 0)
    throw Error("DivideByZeroError");
  // ...
}

function main() {
  try {
    const result1 = workflowPart1();
    const result2 = workflowPart2(result1);
    console.log(`the result is ${result2}`);
  } catch(e) {
    match(e)
      .with(P.instanceOf(OutOfMemoryException), () => console.log("exited with OutOfMemoryException"))
      .with(P.instanceOf(DivideByZeroException), () => console.log("exited with DivideByZeroException"))
      .otherwise(() => console.log("exited with " + e.message));
  }
}
```

```kotlin
val workflowPart2 = {
    input: Input ->
    if (input == 0)
        throw DivideByZeroException()
    // ...
}

fun main() {
    try {
        val result1 = workflowPart1()
        val result2 = workflowPart2(result1)
        println("the result is $result2")
    } catch(e: OutOfMemoryException) {
        println("exited with OutOfMemoryException")
    } catch(e: DivideByZeroException) {
        println("exited with DivideByZeroException ")
    } catch(e: Exception) {
        println("exited with " + e.message)
    }
}
```

<br>

인프라 오류는 이 두 방식 중 하나로 처리할 수 있습니다.  
여러 작은 서비스로 구성된 코드라면 예외 처리가 더 깔끔하고, 모놀리식한 앱이라면 명시적인 오류를 처리하는 것이 좋을 수 있습니다.

### 타입으로 도메인 오류 모델링하기

일반적으로 오류는 선택 타입으로 모델링하고 오류 유형별로 개별 래퍼를 생성합니다.

```typescript
type PlaceOrderError =
  | ValidationError
  | ProductOutOfStock
  | RemoteServiceError;

class ValidationError extends Error {
  constructor(readonly details: string, message?: string) {
    super(message);
  }
}

class ProductOutOfStock extends Error {
  constructor(readonly details: ProductCode, message?: string) {
    super(message);
  }
}

class RemoteServiceError extends ValueObject {
  constructor(
    readonly service: ServiceInfo,
    readonly exception: Error,
  ) { super() }
}
```

```kotlin
sealed interface PlaceOrderError

@JvmInline
value class ValidationError(val value: String) : PlaceOrderError

@JvmInline
value class ProductOutOfStock(val value: String) : PlaceOrderError

data class RemoteServiceError(
    val service: ServiceInfo,
    val exception: Throwable,
) : PlaceOrderError
```

이렇게 선택 타입으로 코드 내에서 발생할 수 있는 모든 오류를 문서화할 수 있습니다. 또한 요구사항 변경시 쉽고 안전하게 선택 타입을 확장 및 축소할 수 있습니다.  
컴파일러가 이 선택 타입을 패턴매칭하는 코드에서 모든 케이스를 커버하지 못하면 경고나 오류를 발생해주기 때문입니다.

### 코드를 어지럽히는 오류 처리

예외의 좋은 점 중 하나는 행복한 경로(happy path) 코드를 깔끔하게 유지할 수 있다는 것입니다.  
각 단계에서 오류를 반환하면 코드가 복잡해집니다. 보통 각 잠재적 오류 뒤에 조건문을 두거나, 예외를 포착하는 try-catch 블록을 추가해야 합니다.

## Either 타입을 출력하는 함수 연결하기

<img width="484" height="183" alt="Image" src="https://github.com/user-attachments/assets/b9f8c6ed-bf35-4aed-9459-d2ce0dcdc957" />

Either 값을 출력하는 함수는 두 갈래로 나뉜 철로로 나타낼 수 있습니다.  
모든 단계들을 파이프라인으로 연결하는 이 방식을 `이중 선로 오류 처리 모델` 또는 `철로 지향 프로그래밍`이라고 부릅니다.  
이 방식에서는 성공적으로 시작하면 끝까지 성공 경로를 유지하고, 오류가 발생하면 실패 경로로 전환하여 나머지 단계를 지나갑니다.  
이 방식은 문제가 하나 있는데, Either를 출력하는 함수를 조합할 수 없습니다.   
이 문제를 해결하기 위해, 스위치 함수를 이중 선로 함수르 변환해야 합니다. 이를 위해 **어댑터 블록**이라는 특별한 블록을 만들어 스위치 함수를 이중 선로 함수로 변환합니다.

<img width="686" height="396" alt="Image" src="https://github.com/user-attachments/assets/7487848a-9ab8-4918-af50-ad364402cc1f" />

결과적으로 성공 경로와 실패 경로가 있는 이중 선로 파이프라인이 완성됩니다.

<br>

### 어댑터 블록 구현

**스위치 함수를 이중 선로 함수로 변환하는 어댑터는 함수형 프로그래밍에서 매우 중요하며, 보통 `flatMap`이라고 불립니다.**

- 입력은 스위치 함수입니다. 출력은 이중 선로 입력과 이중 선로 출력을 가진 새로운 이중 선로 전용 함수로 나타냅니다.
- 성공 선로로 입력이 들어오면 그 입력을 스위치 함수로 전달합니다. 스위치 함수의 출력은 이중 선로의 입력 값이므로 추가 작업이 필요없습니다.
- 실패 선로로 입력이 들어오면 스위치 함수를 건너뛰고 실패값을 반환합니다.

```typescript
const flatMap = <T, E, U>(switchFn: (i: T) => E.Either<E, U>) => (twoTrackInput: E.Either<E, T>) =>
  match(twoTrackInput)
    .with({ _tag: 'Right' }, i => switchFn(i.right))
    .with({ _tag: 'Left' }, i => i)
    .exhaustive();
```

```kotlin
fun <A, B, C> Either<A, B>.flatMap(switchFn: B.() -> Either<A, C>): Either<A, C> =
    when(this) {
        is Either.Right -> this.value.switchFn()
        is Either.Left -> this
    }
```

<br>

**단일 선로 함수를 이중 선로 함수로 변환하는 다른 유용한 어댑터 블록도 있습니다. 이는 map이라 부릅니다.**  

- 입력은 단일 선로 함수와 이중 선로의 값(Either)입니다.
- 입력 Either가 Right인 경우, 그 값을 단일 선로 함수에 넣어 반환하는 출력을 다시 Right로 래핑하면 이중 선로의 위 선로가 됩니다.
- 입력 Either가 실패할 경우, 함수 실행을 우회하여 아래 선로가 됩니다.

```typescript
const map = <T, E, U>(f: (i: T) => U) => (twoTrackInput: E.Either<E, T>) =>
  match(twoTrackInput)
    .with({ _tag: 'Right' }, i => pipe(i.right, f, right))
    .with({ _tag: 'Left' }, i => i)
    .exhaustive();
```

```kotlin
fun <A, B, C> Either<A, B>.map(f: B.() -> C): Either<A, C> =
    when(this) {
        is Either.Right -> this.value.f().right()
        is Either.Left -> this
    }
```

<br>

> Either 타입 관련 함수들은 도메인 전체에서 사용되기 떄문에 새로운 유틸리티 모듈을 생성하고 도메인 타입들보다 상단에 배치하는 것이 일반적입니다.  
> 실무에서는 직접 flatMap, map을 구현하기보다는 `fp-ts / arrow-kt`에 정의되어 있는 함수들과 타입들을 불러와서 활용합니다.


### 함수 합성과 타입 검사

지금까지 함수들의 형태가 일치하도록 변환하는데 중점을 두었습니다. 하지만 함수 합성이 원활하려면 타입도 일치해야 합니다.  
**성공 경로에서는 단계별로 타입이 변할 수 있지만, 각 단계의 출력 타입이 다음 단계의 입력 타입과 일치해야 합니다.**

```typescript
type FunctionA = (i: Apple) => Either<..., Bananas>;
type FunctionB = (i: Bananas) => Either<..., Cherries>;
type FunctionC = (i: Cherries) => Either<..., Lemon>;

const functionA : FunctionA = ...
const functionB : FunctionB = ...
const functionC : FunctionC = ...

const functionABC = flow(
  functionA,
  Either.flatMap(functionB),
  Either.flatMap(functionC),
);
```

```kotlin
typealias FunctionA = Apple.() -> Either<..., Bananas>
typealias FunctionB = Bananas.() -> Either<..., Cherries>
typealias FunctionC = Cherries.() -> Either<..., Lemon>

val functionA : FunctionA = ...
val functionB : FunctionB = ...
val functionC : FunctionC = ...

fun Apple.functionABC(): Lemon = this
    .functionA()
    .flatMap(functionB)
    .flatMap(functionC)
```

### 공통 오류 타입으로 변환

성공 경로와 달리, 오류 경로에는 변환 함수가 없는 우회로이므로 각 단계에서 타입이 변하지 않고 동일한 타입이 유지됩니다.  
**즉, 파이프라인에 있는 모든 함수가 동일한 오류 타입을 가져야 합니다.**  
많은 경우, 이 오류 타입들이 서로 호환하도록 조정해야 하는데, 이를 위해 실패 트랙에 있는 값을 처리하는 함수(mapLeft)를 사용합니다.

```typescript
export const mapLeft: <E, G>(f: (e: E) => G) => <A>(fa: Either<E, A>) => Either<G, A> = (f) => (fa) =>
    isLeft(fa) ? left(f(fa.left)) : fa
```

```kotlin
public inline fun <C> mapLeft(f: (A) -> C): Either<C, B> {
    return when (this) {
        is Left -> Left(f(value))
        is Right -> Right(value)
    }
}
```

<br>

TypeScript의 유니언 타입과 Kotlin의 인터페이스 구현체만으로도 오류를 통일할 수 있습니다.  
하지만 선택 타입을 만들지 않고 새로운 오류를 변환하고 싶을때, 다음과 같이 할 수 있습니다.

```typescript
type FruitError = AppleError | BananaError;

const functionA : FunctionA = ...
const functionB : FunctionB = ...

// functionA를 "FruitError"를 사용하도록 변환
const functionAWithFruitError = flow(
    functionA,
    E.mapLeft(e => new FruitError(e.message)),
);

// functionB를 "FruitError"를 사용하도록 변환
const functionBWithFruitError = flow(
    functionB,
    E.mapLeft(e => new FruitError(e.message)),
);

// 이제 이 두 함수를 "flatMap"으로 합성할 수 있습니다
const functionAB = flow(
    functionAWithFruitError,
    E.flatMap(functionBWithFruitError),
);

// 결과적으로 functionAB의 시그니처는 다음과 같습니다.
const functionAB : (i: Apple) => Either<FruitError, Cherries>
```

```kotlin
sealed interface FruitError
object AppleError : FruitError
object BananaError : FruitError

val functionA : FunctionA = ...
val functionB : FunctionB = ...

// functionA를 "FruitError"를 사용하도록 변환
fun Apple.functionAWithFruitError() = this
    .functionA()
    .mapLeft { FruitError(it.toString()) }

// functionB를 "FruitError"를 사용하도록 변환
fun Bananas.functionBWithFruitError() = this
    .functionB()
    .mapLeft { FruitError(it.toString()) }

// 이제 이 두 함수를 "flatMap"으로 합성할 수 있습니다
fun Apple.functionAB() = this
    .functionAWithFruitError()
    .flatMap { it.functionBWithFruitError() }

// 결과적으로 functionAB의 시그니처는 다음과 같습니다.
val functionAB : Apple.() -> Either<FruitError, Cherries>
```

## flatMap과 map으로 파이프라인 조립하기

개념을 이해했으니, 다음은 실제 적용입니다.  
오류가 발생할 수 있는 함수들을 조합하여 작업 흐름 파이프라인을 구성하고, 각 함수가 잘 연결되도록 조정합니다.

```typescript
type PlaceOrderError = ValidationError | PrigingError;
```

```kotlin
@JvmInline
value class ValidationError(val value: String) : PlaceOrderError

@JvmInline
value class PrigingError(val value: String) : PlaceOrderError
```

<br>

```typescript
const placeOrder = flow(
    validateOrder,
    Either.flatMap(priceOrder),
    Either.map(acknowledgeOrder),
    Either.map(createEvents),
)
```

```kotlin
val placeOrder: UnvalidatedOrder.() -> Either<PlaceOrderError, List<PlaceOrderEvent>> = {
    this.validateOrder()
        .flatMap { it.priceOrder() }
        .map { it.acknowledgeOrder() }
        .map { it.createEvents() }
}
```


- 파이프라인의 각 함수는 오류가 발생할 수 있으며, 발생 가능한 오류는 함수의 시그니처에 드러납니다. 각 함수를 개별적으로 테스트할 수 있고, 전체로 조립하더라도 예상치 못한 동작이 발생하지 않음을 확인할 수 있습니다.
- 함수들이 여전히 체인으로 연결되어 있지만 이제 이중 선로 모델을 사용합니다. 하나의 단계에서 오류가 발생하면 나머지 단계는 건너뜁니다.
- 최상위의 placeOrder 함수의 전체 흐름은 여전히 깔끔하며 조건문이나 try-catch 블록이 필요하지 않습니다.

## 다른 유형의 함수들 이중 선로 모델에 적응시키기

- 예외를 발생시키는 함수
- 아무것도 반환하지 않는 막다른 길 함수

### 예외 처리

라이브러리나 외부 서비스같이 통제할 수 없는 코드에서 예외를 발생시키면 어떻게 해야 할까요 ?  
예외의 상당수는 도메인 디자인의 일부가 아니므로 최상위 함수에서만 예외를 잡아서 처리하면 됩니다.  
하지만 예외를 도메인의 일부로 취급하고자 할때는, 예외를 발생시키는 함수를 Either를 반환하는 함수로 변환하는 `어댑터 블록` 함수로 만들면 됩니다.  
다음은 원격 서비스에서의 타임아웃을 RemoteServiceError로 변환하는 예시입니다.


```typescript
class ServiceInfo {
    constructor (
        readonly name: string,
        readonly endpoint: string,
    ) {}
}

class RemoteServiceError extends Error {
  constructor(readonly service: ServiceInfo, readonly cause: unknown, message?: string) {
    super(message, { cause });
  }
}

const serviceExceptionAdapter = <T>(info: ServiceInfo, fn: (i: T) => U) =>
  (i: T): Either<RemoteServiceError, U> => {
  try {
    return pipe(i, fn, E.right);
  } catch(e) {
    return match(e)
      .with(P.instanceOf(TimeoutException), () => RemoteServiceError(info, e))
      .with(P.instanceOf(AuthenticationException), () => RemoteServiceError(info, e))
      .run();
  }
}
```

```kotlin
class ServiceInfo (
    val name: String,
    val endpoint: String,
)

data class RemoteServiceError(
    val service: ServiceInfo,
    val cause: Throwable,
) : PlaceOrderError

fun <A, C> serviceExceptionAdapter(serviceInfo: ServiceInfo, fn: (A) -> C) = { i: A ->
    try {
        fn(i).right()
    } catch (e: TimeoutException) {
        RemoteServiceError(serviceInfo, e).left()
    } catch (e: AuthenticationException) {
        RemoteServiceError(serviceInfo, e).left()
    }
}
```

### 막다른 길 함수 처리

또 다른 일반적인 유형의 함수는 `막다른 길` 또는 `실행 후 무시` 함수입니다. 이 함수는 입력을 받지만 출력을 반환하지 않습니다.  
이와 같은 막다른 길 함수를 이중 선로 파이프라인에 포함시키려면 또 다른 어댑터 블록이 필요합니다. 이를 구성하려면 받은 입력으로 막다른 길 함수를 호출하고, 그 입력을 그대로 반환하는 통과 함수를 작성해야 합니다.

```typescript
const tee = <T>(f: (i: T) => void) => (x: T) => {
  f(x);
  return x;
}

const adaptDeadEnd = <T, E>(f: (i: T) => void) => (x: Either<E, T>): Either<E, T> =>
  pipe(f, tee, Either.map);
```

```kotlin
fun <T> T.tee(f: (T) -> Unit): T {
    f(this)
    return this
}

fun <T, E> Either<E, T>.adaptDeadEnd(f: (T) -> Unit): Either<E, T> =
    this.tee({ it.map(f) })
```

## 복잡한 파이프라인 다루기

조건문이나 루프 안에서 작업해야 하거나, 깊게 중첩된 Either 생성 함수들과 작업할때는 작업 흐름 로직이 복잡합니다.  
이처럼 복잡한 상황을 다루기 위해 fp-ts는 **do 표기법**을 제공하고, arrow-kt는 **계산 블록**과 DSL을 제공합니다.  
이런 방식은 flatMap으로 파이프라인을 이어가는 방식보다 코드가 깔끔합니다.

### fp-ts의 do 표기법

```typescript
const toCustomerInfo = ({
  firstName,
  lastName,
  emailAddress,
}: CustomerInfoDto): E.Either<ErrPrimitiveConstraints, CustomerInfo> => pipe(
  E.Do,
  E.bind('first', () => String50.create(firstName)),
  E.bind('last', () => String50.create(lastName)),
  E.bind('email', () => EmailAddress.create(emailAddress)),
  E.let('name', ({ first, last }) => new PersonalName(first, last)),
  E.map(scope => new CustomerInfo(scope.name, scope.email)),
);
```

### arrow-kt의 either 블록

```kotlin
val toCustomerInfo: CustomerInfoDto.() -> Either<ErrPrimitiveConstraints, CustomerInfo> = either {
    val first = String50(this.firstName).bind()
    val last = String50(this.lastName).bind()
    val email = EmailAddress(this.emailAddress).bind()
    val name = PersonalName(first, last)
    return CustomerInfo(name, email)
}
```

### arrow-kt의 Raise 콘텍스트

```kotlin
context(_: Raise<ErrPrimitiveConstraints>)
fun CustomerInfoDto.toCustomerInfo(): CustomerInfo {
    val first = String50(this.firstName)
    val last = String50(this.lastName)
    val email = EmailAddress(this.emailAddress)
    val name = PersonalName(first, last)
    return CustomerInfo(name, email)
}
```

함수 본문뿐만 아니라 시그니처마저도 throw하던 코드와 완전히 동일해졌습니다.  
콘텍스트 매개변수가 Raise<ErrPrimitiveConstraints>를 콘텍스트로 주입받고 있습니다.
ErrPrimitiveConstraints라는 오류 정보를 명시하고는 있지만, 콘텍스트를 주입받은것만으로 어떻게 오류를 처리할까요 ?  
다음은 오류를 발생시키는 함수입니다.

```kotlin
@JvmInline
value class String50 private constructor(val value: String) {
    companion object {
        // Create a String50 from a string
        // Raise ErrPrimitiveConstraints if input is null, empty, or length > 50
        context(_: Raise<ErrPrimitiveConstraints>)
        operator fun invoke(str: String): String50 =
            ConstrainedType.ensureStringMaxLen(50, str) { String50(str) }
    }
}

object ConstrainedType {
    context(r: Raise<ErrPrimitiveConstraints>)
    fun <T> ensureStringMaxLen(
        maxLen: Int,
        i: String,
        ctor: () -> T,
    ): T {
        r.ensure(i.isNotEmpty()) { ErrEmptyString }
        r.ensure(i.length <= maxLen) { ErrStringTooLong(maxLen) }
        return ctor()
    }
}
```

따라 내려가보면 모든 호출 함수들이 Raise<ErrPrimitiveConstraints>와 같이 해당 함수에서 발생하는 오류를 컨텍스트 매개변수로 명시합니다.  
**ensure는 조건에 부합하지 않으면 throw가 아니라 raise하는 함수입니다. raise하는 오류와 Raise 콘텍스트에 표시해둔 오류 타입은 일치해야 합니다.**  
하위 함수가 필요한 콘텍스트는 직접 바인딩해주거나 동일한 콘텍스트를 매개변수로 명시해야 합니다.  
Raise<Error> 콘텍스트 매개변수에 대한 내용을 정리하면 다음과 같습니다.

- Raise는 함수의 출력에 부수효과를 격리하는 대신 raise, ensuer같은 DSL로 효과를 선언합니다. 함수 시그니처의 입출력 부분은 Either 도입 전과 동일해졌고, 출력에 드러나던 오류 부수효과는 콘텍스트 매개변수로 나타납니다.
- Raise 콘텍스트에 명시한 오류와 raise로 올리는 오류 타입이 다르면 컴파일 오류가 발생합니다.
- 바인딩된 Raise<Error> 콘텍스트는 raise, ensuer같은 DSL 메시지를 수신합니다. DSL 호출로 효과를 선언하면 이를 처리하는 곳에 전달하여 발생한 예외를 처리합니다.

> Either, Raise 모두 오류 부수 효과가 발생하는 함수임을 드러냅니다. 그리고 둘 다 오류가 발생하면 어떻게 처리할지를 명세합니다.    
> Either가 출력 타입과 map, flatMap, bind 등으로 철도를 놓듯이 성공, 실패 경로를 정적으로 연결하는 방식이라면,   
> Raise는 try-catch처럼 성공 케이스만 코드에 남기고 오류가 발생하면 동적으로 핸들러에 명세한 오류 처리 로직으로 프로그램 실행 흐름을 변경(비지역적 제어 흐름non-local control flow)합니다.
> 전통적으로 오류 효과에 한정하여 쓰였던 try-catch를 임의 효과로 확장하고 상위 스택에 바인딩한 핸들러를 통해 미루지 않고 즉각 처리하는 방식을 대수적 효과 처리기(algebraic effect handler)라 부릅니다.  
> TypeScript도 효과 시스템을 구현한 라이브러리 effection, effect-ts가 있습니다. 오늘날에는 부수 효과를 격리하는 방식으로 대수적 효과 처리기가 더 인기를 얻고 있습니다.

### Either 타입으로 주문 검사하기

OrderId.create같이 ValidationError가 아닌 오류를 발생하는 경우 mapLeft를 사용해 ValidationError로 변환해야 합니다.

```typescript
const validateOrder: ValidateOrder = (checkProductCodeExists, checkAddressExists) => ({
  orderId,
  customerInfo,
  lines,
  shippingAddress,
  billingAddress,
}: UnvalidatedOrder) => pipe(
  E.Do,
  E.bind('validId', () => pipe(orderId, OrderId.create, E.mapLeft(ValidationError.from))),
  E.bind('validInfo', () => toCustomerInfo(customerInfo)),
  E.bind('validLines', () => lines.map(toValidatedOrderLine(checkProductCodeExists))),
  E.bind('validShipAdr', () => pipe(shippingAddress, toAddress(checkAddressExists))),
  E.bind('validBillingAdr', () => pipe(billingAddress, toAddress(checkAddressExists))),
  E.map((scope) => new ValidatedOrder(scope.validId, scope.validInfo, scope.validShipAdr, scope.validBillingAdr, scope.validLines)),
);
```

```kotlin
context(_: Raise<ValidatoinError>, _: AddressValidator)
fun UnvalidatedOrder.validateOrder(checkCodeExists: CheckProductCodeExists) = ValidatedOrder(
    orderId = r.withError({ ValidationError(it.toString()) }) {
        OrderId.create(this.orderId)
    },
    customerInfo = this.customerInfo.toCustomerInfo(),
    shippingAddress = this.shippingAddress.toCheckedAddress().toAddress(),
    billingAddress = this.billingAddress.toCheckedAddress().toAddress(),
    lines = this.lines.map { it.toValidatedOrderLine(checkCodeExists) },
)
```

### Either 리스트의 유효성 검사

toValidatedOrderLine 함수가 Either 타입을 반환하면 Array.map 결과가 Array<Either<..., ValidateOrderLine>>가 됩니다.  
이로써 ValidateOrder.Lines에 필요한 Array<ValidatedOrderLine>과 불일치가 발생합니다.  
**이같이 Array<Either<..., a>>를 Either<..., Array<a>>로 변환하는 함수를 시퀀스라 하는데, fp-ts도 Either.sequenceArray 함수를 제공합니다.**  
**Kotlin은 오류 효과를 출력 타입에 드러내지 않으므로 별다른 조치가 필요하지 않습니다.**  
이 점이 효과 처리를 미루지 않고 즉시 위임하는 방식의 장점입니다.

## 모나드와 기타 개념

모나드는 단순히 모나딕 함수들을 줄지어 꼬리물게 해주는 프로그래밍 패턴입니다.  
모나딕 함수는 일반 값을 받아서 출력을 확장한 값을 반환하는 함수입니다.  
오류 처리에서는 Either로 출력을 감싼 것이 확장한 값이므로, EIther를 생성하는 스위치 함수가 정확히 모나딕 함수에 해당합니다.

기술적으로 모나드는 세 가지 요소로 이루어져 있습니다.

- 데이터 구조
- 관련 함수
- 관련 함수가 어떻게 동작해야만 한다는 규칙

<br>

애플리케이티브는 모나드와 유사하지만, 모나딕 함수를 줄지어 잇기보다는 모나딕 값을 병렬로 결합하게 해줍니다.  
예를 들어 특정 연산에서 발생한 모든 오류를 수집해야 하는 상황에서는, 첫번쨰 오류만 유지하는것이 아니라 발생한 모든 오류를 취합하기 위해 애플리케이티브 방식을 사용할 가능성이 큽니다.

## 비동기 효과 추가하기

효과에 단순히 오류 효과만 있지는 않고, 대부분의 파이프라인에서는 비동기 효과도 같이 있습니다.  
fp-ts의 Task 정의를 보면 다음과 같습니다.

```typescript
export interface Task<A> {
    (): Promise<A>
}
```

호출시 Promise를 반환하는 성크(thunk), 즉 지연 함수입니다.

<br>

```typescript
async function printNowAsync(): Promise<void> {
  console.log("Evaluated at ", Date.now());
}

const thinkFor = async (ms: number) =>
  new Promise<void>((resolve) =>
    setTimeout(() => {
      resolve();
    }, ms)
  );

async function lazyEval() {
  console.log("Promise: ", Date.now());
  const promise = printNowAsync();   // Promise의 경우 now()는 호출 즉시 평가
  await thinkFor(1000);
  await promise;
  console.log("Promise: ", Date.now());

  console.log("Thunk: ", Date.now());
  const thunk = () => printNowAsync();
  await thinkFor(1000);
  await thunk();                     // Thunk의 경우 1초 지난 thunk 평가 시점에 now()도 평가
  console.log("Thunk: ", Date.now());
}
```

Promise를 반환하는 함수는 호출 즉시 함수를 수행하여 Promise를 반환하게 되고, 성크 스타일은 아직 Promise 반환 함수는 호출하지 않고 있다가 1초 뒤 await하는 시점에 평가를 시작합니다.  
따라서 함수 호출과 즉시 평가하는 async 함수를 지연 평가하기 위한 의도로 fp-ts가 성크 스타일을 활용합니다.  
fp-ts는 파이프라인을 pipe, flow 함수로 구축하는데 이들은 동기 함수이므로 전달하는 개별 파이프들 또한 async 함수일 수 없습니다.  
따라서 파이프라인을 합성할 수 있게 async 함수를 성크로 감쌌을 것입니다.

**Either와 마찬가지로, 밑에서 올라온 Task 효과를 부수효과를 일으켜 값으로 변환하지 않고, 경계진 맥락을 벗어나는 최상단 호출 지점까지 Task로 감싸서 쭉 전파해야 합니다.**    
다음은 validateOrder에 비동기 효과를 포함하여 TaskEither를 적용한 예시입니다.

```typescript
const validateOrder: ValidateOrder = (checkProductCodeExists, checkAddressExists) => ({
  orderId,
  customerInfo,
  lines,
  shippingAddress,
  billingAddress,
}: UnvalidatedOrder) => pipe(
  E.Do,
  E.bind('validId', () => toOrderId(orderId)),
  E.bind('validInfo', () => toCustomerInfo(customerInfo)),
  E.bind('validLines', () => pipe(lines, A.map(toValidatedOrderLine(checkProductCodeExists)), E.sequenceArray)),
  // 이 위는 Either<ValidationError, Scope> 입니다.
  // 비동기 작업을 위해 TaskEither로 전환합니다.
  TE.fromEither,
  // 아래부터는 TaskEither<ValidationError, Scope> 입니다.
  TE.bind('checkedShippingAddress', () => pipe(shippingAddress, toCheckedAddress(checkAddressExists))),
  TE.bind('validShipAdr', ({ checkedShippingAddress }) => pipe(checkedShippingAddress, toAddress, TE.fromEither)),
  TE.bind('checkedBillingAddress', () => pipe(billingAddress, toCheckedAddress(checkAddressExists))),
  TE.bind('validBillingAdr', ({ checkedBillingAddress }) => pipe(checkedBillingAddress, toAddress, TE.fromEither)),
  TE.map(scope => new ValidatedOrder(scope.validId, scope.validInfo, scope.validShipAdr, scope.validBillingAdr, scope.validLines)),
);
```

do 표기법이 내려보내는 scope는 한가지 효과 타입으로 흘러가므로 최종 효과가 TaskEither라면 모든 bind의 콜백 함수들은 TaskEither를 출력해야 합니다.  
그 경우 비동기가 아닌 Either 출력 함수는 TaskEither.fromEither로 감싸서 출력 타입을 TaskEither로 맞춰야 합니다.  
이 예시에서는 그것이 번잡하다고 생각하여 Either.Do로 시작하여 Either.bind로 먼저 비동기 효과가 없는 함수들의 결과를 바인딩합니다.  
그리고 나서 do 표기법의 효과를 Either에서 TaskEither로 바꾸어 진행합니다.

<br>

```kotlin
context(_: Raise<ValidationError>, _: AddressValidator)
suspend fun UnvalidatedOrder.validateOrder(checkCodeExists: CheckProductCodeExists) =
    ValidatedOrder(
        orderId = this.orderId.toOrderId(),
        customerInfo = this.customerInfo.toCustomerInfo(),
        shippingAddress = this.shippingAddress.toCheckedAddress().toAddress(),
        billingAddress = this.billingAddress.toCheckedAddress().toAddress(),
        lines = this.lines.map { it.toValidatedOrderLine(checkCodeExists) },
    )
```

Kotlin은 fun 앞에 suspend가 붙은 것이 전부입니다.  
효과를 출력에 드러내지 않으니 달라진 효과들을 연결하기 위해 이곳저곳 코드를 바꾸지 않아도 됩니다.



















