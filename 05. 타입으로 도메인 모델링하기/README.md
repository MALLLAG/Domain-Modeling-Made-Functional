# 타입으로 도메인 모델링하기

소스 코드 자체가 문서 역할도 하는 것이 이상적인데, 이는 도메인 전문가와 비개발자가 코드를 검토하고 디자인을 확인할 수 있어야 한다는 뜻입니다.  
이 장에서는 TypeScript와 Kotlin 타입 시스템으로 도메인 전문가를 포함한 비개발자가 읽고 이해할 만큼 정확하게 도메인 모델을 코데으 반영하는 방법을 알아봅니다.

## 도메인 모델 다시보기

```text
context: Order-Taking

// ----------------------
// Simple types
// ----------------------

// Product codes
data ProductCode = WidgetCode OR GizmoCode
data WidgetCode = string starting with "W" then 4 digits
data GizmoCode = ...

// Order Quantity
data OrderQuantity = UnitQuantity OR KilogramQuantity
data UnitQuantity = ...
data KilogramQuantity = ...

// ----------------------
// Order life cycle
// ----------------------

// ----- unvalidated state -----
data UnvalidatedOrder = 
    UnvalidatedCustomerInfo
    AND UnvalidatedShippingAddress
    AND UnvalidatedBillingAddress
    AND list of UnvalidatedOrderLine

data UnvalidatedOrderLine = 
    UnvalidatedProductCode
    AND UnvalidatedOrderQuantity

// ----- validated state -----
data ValidatedOrder = ...
data ValidatedOrderLine = ...

// ----- priced state -----
data PricedOrder = ...
data PricedOrderLine = ...

// ----- output events -----
data OrderAcknowledgmentSent = ...
data OrderPlaced = ...
data BillableOrderPlaced = ...

// ----------------------
// Workflows
// ----------------------

workflow "Place Order" =
    input: UnvalidatedOrder
    output (on success):
        OrderAcknowledgmentSent
        AND OrderPlaced (to send to shipping)
        AND BillableOrderPlaced (to send to billing)
    output (on error):
        InvalidOrder

// etc
```

이번 장의 목표는 이 모델을 코드로 변환해내는 것입니다.

## 도메인 모델 속 패턴 찾기

도메인 모델은 다양하지만 그 속에서 여러 패턴이 반복적으로 나타납니다.

- 단순값: 기본 빌딩 블록으로서 문자열과 정수같은 원시 타입을 갖지만 원시 타입 그 자체는 아닙니다. (OrderId, ProductCode)
- 값의 조합(AND): 밀접하게 연관된 데이터 그룹입니다. 종이 기반의 세계에서는 문서나 이름, 주소, 주문 등의 문서 내 하위 요소에 해당합니다.
- 선택(OR): 우리 도메인에는 선택하는 것들이 있습니다. (Order, Quote, UnitQUantity, KilogramQuantity)
- 작업 흐름: 입력과 출력을 가지는 비즈니스 프로세스가 있습니다.

## 단순값 모델링

도메인 전문가들은 단순값을 int, string같은 데이터 타입 대신 OrderId, ProductCode같은 도메인 개념으로 생각합니다.  
이때 OrderId와 ProductCode 호환을 막는 것이 중요합니다. 두 값이 모두 int로 표현된다고 해서 서로 교환 가능한 것은 아닙니다.  
**따라서 이런 타입들을 명확히 구분하고자 원시 타입을 감싸는 래퍼 타입을 생성합니다.**

```typescript
declare const customerId: unique symbol;
class CustomerId {
  [customerId]!: never; // 브랜드
  constructor(readonly value: number) {}
}
```

```kotlin
@JvmInline
value class CustomerId(val value: Int)
```

### 래퍼 타입 활용하기

이렇게 단순 타입을 생성하면 서로 다른 타입을 혼동하지 않도록 보장할 수 있습니다.  
예를 들어 CustomerId와 OrderId를 생성하고 이들을 비교하면 TypeScript는 경고 메시지가, Kotlin은 컴파일 에러가 발생합니다.  
래퍼 타입이 감싸고 있는 내부 타입을 추출하려면 TypeScript는 패턴 매칭으로 래퍼 타입을 해체할 수 있고, Kotlin은 멤버를 직접 참조합니다.

```typescript
const { value } = new OrderId(42);
```

```kotlin
@JvmInline
value class OrderId(val value: Int)

val rawId = OrderId(42).value
```

### 단순 타입의 성능 문제 완화하기

단순 타입으로 원시 타입을 감싸는 것은 타입 안정성을 보장하고 컴파일 시점에 많은 오류를 방지하는 방법입니다.  
하지만 메모리를 더 쓰는데다가 성능은 낮아진다는 단점이 있습니다. 이 정도의 성능 저하는 일반적인 비즈니스 애플리케이션에는 영향을 주지 않지만 과학, 실시간 도메인 같은 분야에서는 주의해야 합니다.  
예를 들어 UnitQuantity 값을 포함한 큰 배열을 순회하는 것은 int 배열을 순회하는 것보다 느립니다.

단순 타입 대신 타입 별칭(type alias)으로 도메인을 문서화할 수 있습니다.  
이 방법은 성능 부담은 없지만 타입 안정성을 포기합니다.

```typescript
type UnitQuantity = number;
```

```kotlin
typealias UnitQuantity = Int
```

<br>

Kotlin은 inline value class를 활용하면 런타임 오버헤드 없이 타입 안정성을 보장합니다.

```kotlin
@JvmInline
value class UnitQuantity(val value: Int)
```

<br>

TypeScript로 큰 배열을 다룰때, 단순 타입 배열 대신 원시 타입 배열 전체를 단일 타입으로 정의해봅시다.  
이렇게 하면 행렬 곱 같은 고성능을 요구하는 작업에서는 원시 데이터를 효율적으로 처리하면서도 고수준에서는 타입 안정성을 유지할 수 있습니다.

```typescript
declare const unitQuantities: unique symbol;
class UnitQuantities {
  [unitQuantities]!: never;
  constructor(readonly value: number[]) {}
}
```

## 복잡한 데이터 모델링

이제 대수적 타입 시스템으로 도메인을 모델링해봅니다.

### 레코드 모델링

우리 도메인 데이터는 상당수 AND 관계입니다.

```typescript
class Order {
  constructor(
    readonly customerInfo: CustomerInfo,
    readonly shippingAddress: ShippingAddress,
    readonly billingAddress: BillingAddress,
    readonly orderLines: OrderLine[],
    readonly amountToBill: ...,
  ) {}
}
```

```kotlin
class Order (
  val customerInfo: CustomerInfo,
  val shippingAddress: ShippingAddress,
  val billingAddress: BillingAddress,
  val orderLines: List<OrderLine>,
  val amountToBill: ...,
)
```

### 잘 모르는 타입 모델링

도메인의 공용어를 정의했기에 모델링할 타입명은 알지만 내부 구조는 아직 모를 수 있습니다. 이것은 큰 문제는 아닙니다.  
잘 모르는 타입을 일종의 플레이스 홀더로 명시적으로 모델링하면 좋습니다.

```typescript
type Undefined = never;

type CustomerInfo = Undefined;
type ShippingAddress = Undefined;
type BillingAddress = Undefined;
type OrderLine = Undefined;
type BillingAmount = Undefined;

class Order {
    constructor(
        readonly customerInfo: CustomerInfo,
        readonly shippingAddress: ShippingAddress,
        readonly billingAddress: BillingAddress,
        readonly orderLines: OrderLine[],
        readonly amountToBill: BillingAmount,
    ) {}
}
```

```kotlin
interface Undefined
typealias CustomerInfo = Undefined
typealias ShippingAddress = Undefined
typealias BillingAddress = Undefined
typealias OrderLine = Undefined
typealias BillingAmount = Undefined

class Order (
    val customerInfo: CustomerInfo,
    val shippingAddress: ShippingAddress,
    val billingAddress: BillingAddress,
    val orderLines: List<OrderLine>,
    val amountToBill: BillingAmount,
)
```

### 선택 타입 모델링

```text
data ProductCode = 
    WidgetCode
     OR GizmoCode
     
data OrderQuantity = 
    UnitQuantity
     OR KilogramQuantity
```

우리 도메인에는 선택지가 많이 존재합니다.  
이 선택지들을 선택 타입으로 표현할 수 있습니다.

```typescript
type ProductCode = WidgetCode | GizmoCode;
type OrderQuantity = UnitQuantity | KilogramQuantity;
```

```kotlin
sealed interface ProductCode {
    @JvmInline
    value class Widget(...): ProductCode

    @JvmInline
    value class Gizmo(...): ProductCode
}

sealed interface OrderQuantity {
    @JvmInline
    value class Unit(...): OrderQuantity

    @JvmInline
    value class Kilogram(...): OrderQuantity
}
```

## 함수로 작업 흐름 모델링하기

비즈니스 프로세스는 어떻게 모델링해야 할까요 ? 여기서는 작업 흐름과 기타 프로세스를 함수 타입으로 모델링합니다.  
예를 들어 주문 양식을 검증하는 작업 흐름 단계를 가지고 있다면, 다음과 같이 문서화합니다.  
**ValidateOrder의 함수 시그니처를 통해 검증되지 않은 주문을 검증된 주문으로 변환하는 과정입니다.**

```typescript
type ValidateOrder = (i: UnvalidatedOrder) => ValidatedOrder;
```

```kotlin
typealias ValidateOrder = (i: UnvalidatedOrder) -> ValidatedOrder
```

### 복잡한 입력 및 출력 처리

함수형 프로그래밍에서는 모든 함수가 하나의 입력과 하나의 출력만 가집니다. 그러나 일부 작업 흐름은 여러 입력과 출력을 가질 수 있습니다.  
만약 출력이 `outputA`와 `outputB`가 있다면 이를 하나의 레코드로 만들 수 있습니다.  
예를 들어 주문 접수 작업 흐름은 세 가지 서로 다른 이벤트를 출력하므로 레코드 하나로 묶을 수 있습니다.

```typescript
class PlaceOrderEvents {
  constructor (
    readonly acknowledgmentSent: AcknowledgmentSent,
    readonly orderPlaced: OrderPlaced,
    readonly billableOrderPlaced: BillableOrderPlaced,
  ) {}
}
```

```kotlin
data class PlaceOrderEvents (
  val acknowledgmentSent: AcknowledgmentSent,
  val orderPlaced: OrderPlaced,
  val billableOrderPlaced: BillableOrderPlaced,
)
```

<br>

한편 작업 흐름이 `outputA`나 `outputB`를 출력한다면 선택 타입으로 둘을 포괄할 수 있습니다.  

```typescript
declare const envelopeContents: unique symbol;
class EnvelopeContents {
  [envelopeContents]!: never;
  constructor ( readonly value: string ) {}
}

type CategorizedMail = QuoteForm | OrderForm;
type CategorizeInboundMail = (i: EnvelopeContents) => CategorizedMail;
```

```kotlin
@JvmInline
value class EnvelopeContents ( val value: String )

sealed interface CategorizedMail {
    data class QuoteForm(...): CategorizedMail
    data class OrderForm(...): CategorizedMail
}

typealias CategorizeInboundMail = (EnvelopeContents) -> CategorizedMail
```

<br>

다음은 입력을 모델링하는 방법입니다. 만약 작업 흐름이 여러 입력을 받는다면, 두 가지 방법 중 하나를 선택할 수 있습니다.  

1. 각 입력들을 개별 매개변수로 전달
2. 여러 입력들을 모두 포함하는 새로운 레코드를 매개변수로 전달

두 입력이 항상 필요하고 서로 밀접하게 관련되어 있다면, 레코드로 명확하게 나타내는 것이 좋습니다.  


### 함수 시그니처에 효과 문서화하기

ValidateOrder는 실제로 실패할 수도 있으므로, 함수 시그니처에 Either 타입을 사용해 실패 가능성을 나타내는 것이 더 좋습니다.  
함수형 프로그래밍에서 효과는 함수가 기본 출력 외에 수행하는 다른 작업을 말합니다.  
여기서는 Either로 ValidateOrder 함수가 오류 효과가 생길 수 있음을 드러냅니다.

```typescript
type ValidateOrder = (i: UnvalidatedOrder) => Either<ValidationError[], ValidatedOrder>;
```

```kotlin
typealias ValidateOrder = (UnvalidatedOrder) -> Either<List<ValidationError>, ValidatedOrder>
```

<br>

만약 비동기로 실행되는 프로세스를 문서화하고 싶을때라도, 적절히 타입으로 드러낼 수 있습니다.  
TypeScript의 Task 타입으로 비동기 효과를 나타내고, Kotlin의 suspend로 비동기 효과를 나타낼 수 있습니다.  
이 함수 시그니처는 함수가 즉시 반환하지 않으며 오류도 반환할 수 있음을 드러냅니다.

```typescript
type ValidateOrder = (i: UnvalidatedOrder) => Task<Either<ValidationError[], ValidatedOrder>>;
```

```kotlin
typealias ValidateOrder = suspend (UnvalidatedOrder) -> Either<List<ValidationError>, ValidatedOrder>
```

## 정체성에 관하여: 값 객체

DDD 용어로 값이 변해도 정체성을 지속하는 데이터를 entity라 하고, 정체성이 없는 데이터를 value object라고 합니다.  
많은 경우 우리가 다루는 데이터 객체는 정체성이 없으며, 서로 교환 가능(interchangeable)합니다.  
예를 들어 값이 "W1234"인 WidgetCode의 한 인스턴스는 값이 "W1234"인 다른 WidgetCode와 동일합니다.

### 값 객체의 같음

두 값 객체가 같은지 비교하려면 모든 하위 속성들의 값이 같은지 비교해야 하는데, 이를 **구조적 동등**이라고 합니다.  
Kotlin은 == 오퍼레이터가 암묵적으로 equals를 호출하여 객체를 비교하고, 값 객체를 위한 data class가 별도로 있습니다.  
반면 TypeScript는 객체를 비교하는 특수 메서드가 없습니다. 그래서 equals 인터페이스를 도입하고 ValueObject 추상 클래스를 통해 구현할 수 있습니다.

## 정체성에 관하여: 엔터티

우리는 현실 세계에서 고유한 정체성을 지닌 것을 종종 모델링합니다.  
고유한 정체성이랑 그 구성 요소가 변해도 동일한 대상으로 인식한다는 것입니다.  
예를 들면 사람의 이름이나 주소를 바꿔도 여전히 동일한 사람입니다.  
이들은 생애 주기를 가지며, 다양한 프로세스에 의해 한 상태에서 다른 상태로 변해갑니다.

### 엔터티의 ID

엔터티는 값이 변경되더라도 안정적인 정체성을 유지해야 합니다. 따라서 이를 모델링할때 고유 ID, 주문 ID, 고객 ID 등의 식별자를 부여해야 합니다.  
이런 ID는 현실 세계 도메인에서 ID를 제공할 수도 있고, UUID, 자동 증가 데이터베이스 테이블 등을 이용할 수도 있습니다.

### 데이터 정의에 ID 포함하기

어떤 도메인 모델을 엔터티로 식별했다면 ID를 어떻게 데이터 정의에 포함시킬까요 ?  
레코드에 ID를 포함시키는 것은 간단합니다. 필드를 추가하기만 하면 됩니다.  
하지만 선택 타입에 ID를 포함시킬때는, ID를 선택 타입 개별 경우들 속에 포함시켜야 할까요 ? 혹은 선택 타입 바깥에 두어야 할까요 ?

예를 들어 Invoice에 `지불 완료`와 `미지불`이라는 두 가지 선택이 있다고 가정해봅시다.  
이를 외부 방식으로 모델링하면 인보이스 ID를 포함하는 레코드가 있고, 그 레코드 안에 있는 선택 타입 InvoiceInfo에 각 유형의 인보이스에 대한 정보를 포함시킵니다.

```typescript
class UnpaidInvoiceInfo { ... }
class PaidInvoiceInfo { ... }
type InvoiceInfo = UnpaidInvoiceInfo | PaidInvoiceInfo;

class InvoiceId { ... }

class Invoice {
  constructor(
    invoiceId: InvoiceId,
    invoiceInfo: InvoiceInfo,
  ) {}
}
```

```kotlin
sealed interface InvoiceInfo {
    data class UnpaidInvoiceInfo( ... ): InvoiceInfo
    data class PaidInvoiceInfo( ... ): InvoiceInfo
}

@JvmInline
value class InvoiceId ( ... )

class Invoice (
    val invoiceId: InvoiceId,
    val invoiceInfo: InvoiceInfo,
)
```

이 방식은 Invoice 자체가 선택 타입이 아니므로 InvoiceInfo 선택 타입 안과 밖에 Invoice 데이터가 퍼져있습니다.  
**그래서 패턴 매칭을 하여 데이터를 다루기가 어렵다는 문제가 있습니다. 따라서 ID를 각 선택 타입의 케이스 안에 포함시키는 것이 더 일반적입니다.**  
내부에 ID를 두면 두 개의 별도 타입(UnpaidInvoice와 PaidInvoice)을 만들고 두 타입 모두 고유한 InvoiceId를 갖게 됩니다.

```typescript
type UnpaidInvoice = {
  invoiceId: InvoiceId,
  // ...
}
type PaidInvoice = {
  invoiceId: InvoiceId,
  // ...
}
type Invoice = UnpaidInvoice | PaidInvoice;
```

```kotlin
sealed interface Invoice {
    class Unpaid(
        val invoiceId: InvoiceId,
        // ...,
    ): Invoice

    class Paid(
        val invoiceId: InvoiceId,
        // ...,
    ): Invoice
}
```

이런 방식은 패턴 매칭을 하고 나서 ID를 포함한 모든 데이터에 접근할 수 있다는 장점이 있습니다.

```typescript
const invoice = new PaidInvoice(invoiceId, ...);

match(invoice)
  .with(P.instanceOf(UnpaidInvoice), ({invoiceId}) => console.log(`The unpaid invoiceId is ${invoiceId}`))
  .with(P.instanceOf(PaidInvoice), ({invoiceId}) => console.log(`The paid invoiceId is ${invoiceId}`))
  .exhaustive();
```

```kotlin
val invoice = Invoice.Paid(invoiceId, ...)

when (invoice) {
    is Invoice.Unpaid -> println("The unpaid invoiceId is ${invoice.invoiceId}")
    is Invoice.Paid -> println("The paid invoiceId is ${invoice.invoiceId}")
}
```

### 엔터티의 같음

엔터티가 같은지 확인하기 위해선 오로지 ID만 비교해야 합니다.

### 불변성과 정체성

함수형 프로그래밍에서 값은 기본적으로 불변입니다.

- 값 객체는 불변이어야 합니다.
- 엔터티는 데이터가 시간에 따라 변할 것으로 기대합니다. 이것이 엔터티에 식별자가 있는 이유입니다.  
  그렇다면 데이터를 변경할 수 없는 함수형 프로그래밍에서는 어떻게 엔터티를 변경할 수 있을까요 ?
  변경된 데이터로 엔터티의 복사본을 만들때 식별자를 유지하면 됩니다.

```typescript
import { produce } from "immer"
const updatedPerson = produce(initialPerson, draft => { draft.name = "Joe" });
```

```kotlin
import arrow.optics.copy

val updatedPerson = initialPerson.copy {
  Person.name set "Joe"
}
```

## 집합체

**OrderLine을 수정하면 해당 항목을 품고 있는 Order도 수정한 것일까요 ? 명백히 그렇습니다.**  
불변 데이터 구조에서는 한 OrderLine을 수정하면 Order 수정을 피할 수 없습니다.  
OrderLine 수정본을 만든다고 해서 Order에 반영되지는 않습니다. Order에 포함된 OrderLine을 변경하고자 한다면, Order 수준에서 변경해야 합니다.

불변 데이터 구조에서는 한 계층 아래 요소를 변경하면 그 상위 요소도 변경해야만 파급 효과가 생깁니다.  
따라서 하위 엔터티인 OrderLine만 변경하더라도 항상 Order 수준에서 작업해야 합니다.

### 집합체 참조

Order에 관련 고객 정보를 추가해야 한다고 가정해봅시다.  
이런 경우 Customer 자체가 아닌 참조만 저장합니다. 즉, Order 타입에 CustomerId만 저장하는 방식입니다.

```typescript
class Order {
  constructor(
    readonly orderId: OrderId,
    readonly customerId: CustomerId,
    readonly orderLines: OrderLine[],
  ) {}
}
```

```kotlin
class Order (
    val orderId: OrderId,
    val customerId: CustomerId,
    val orderLines: List<OrderLine>,
)
```

이 방식은 Order 관련 고객 정보가 필요할때 Order에서 CustomerId를 얻어서 관련 고객 데이터를 별도로 조회합니다.  
즉, Customer와 Order는 별도의 집합체입니다.  
집합체가 도메인 모델에서 중요한 역할을 하는 이유는 다음과 같습니다.

- 집합체는 도메인 객체들의 모음으로 최상위 엔터티가 `루트`인 단일 구성 요소로 취급합니다.
- 집합체 내 객체의 모든 변경 사항은 집합체 루트에 적용해야 하며, 집합체는 내부 데이터가 일관성을 유지하도록 동시에 업데이트합니다.
- 집합체는 데이터 저장, 데이터베이스 트랜잭션, 데이터 전송의 원자적 단위입니다.

































