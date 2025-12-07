# 파이프라인으로 작업 흐름 모델링하기

<img width="689" height="287" alt="Image" src="https://github.com/user-attachments/assets/f6871e81-af91-4366-9fee-1c46bd5e6a73" />

주문 접수 작업 흐름은 다음과 같습니다.  
대다수 비즈니스 프로세스는 일련의 문서 변환 작업으로 간주할 수 있으며, 주문 접수 작업 흐름도 같은 방식으로 모델링합니다.  
비즈니스 프로세스를 표현하는 파이프라인은 다시 작은 파이프들로 이뤄집니다. 개별적인 작은 파이프는 하나의 변환을 수행하며, 이 작은 파이프들을 이어붙여 더 큰 파이프라인이 만들어집니다.  
함수형 프로그래밍의 원칙에 따라, 파이프라인의 각 단계를 상태나 부수효과가 없도록 디자인해야 합니다. 이는 각 단계를 독립적으로 테스트하고 이해할 수 있다는 뜻입니다.

## 작업 흐름 입력

작업 흐름의 입력은 항상 도메인 객체여야 합니다.

```typescript
class UnvalidatedOrder {
    ...
    readonly orderId : string,
    readonly customerInfo : UnvalidatedCustomerInfo,
    readonly shippingAddress : UnvalidatedAddress,
    ...
}
```

```kotlin
data class UnvalidatedOrder (
    val orderId : String,
    val customerInfo : UnvalidatedCustomerInfo,
    val shippingAddress : UnvalidatedAddress,
    ...
)
```

### 명령을 입력으로 사용하기

주문 처리 작업 흐름을 시작하는 명령을 PlaceOrder라고 합시다. 이 명령은 작업 흐름이 요청을 처리하는데 필요한 모든 것을 포함해야 합니다.  
PlaceOrder가 필요한 정보는 UnvalidatedOrder입니다. 또한 누가 언제 명령했는지 나타내는 시간 기록 로그와 감사(audit)를 위한 메타데이터를 추적하고 싶을 수 있습니다.

```typescript
class PlaceOrder {
    constructor(
        readonly orderForm : UnvalidatedOrder,
        readonly timeStamp : DateTime,
        readonly userId: string,
    ) {
    }
}
```

```kotlin
data class PlaceOrder (
    val orderForm : UnvalidatedOrder,
    val timeStamp : LocalDateTime,
    val userId: String,
)
```

### 공통 구조 일반화하기

각 명령은 개별 작업 흐름에 필요한 데이터를 갖지만 UserId와 Timestamp같이 모두가 필요한 공통 필드를 공유할 것입니다.  
객체지향 디자인을 한다면 공통 필드를 포함한 base class를 정의하고 개별 명령에 이를 상속하여 해결하려 할 것입니다.  
**함수형 프로그래밍에서는 제네릭 타입으로 같은 목표를 달성할 수 있습니다.**

```typescript
type Command<D> = {
  readonly data : D;
  readonly timestamp: DateTime;
  readonly userId: string;
};

class PlaceOrder implements Command<UnvalidatedOrder> {
  constructor(
    readonly data : UnvalidatedOrder,
    readonly timestamp: DateTime,
    readonly userId: string,
  ) {}
}
```

```kotlin
interface Command<D> {
  val data : D
  val timestamp: DateTime
  val userId: String
  // 기타 정보
}

data class PlaceOrder(
  override val data: UnvalidatedOrder,
  override val timestamp: DateTime,
  override val userId: String,
) : Command<UnvalidatedOrder>
```

### 여러 명령을 단일 타입으로 묶기

경계 진 맥락 내의 모든 명령이 같은 입력 채널을 공유하기도 하므로, 이들을 단일 데이터 구조로 통합할 방법이 필요합니다.  
해결책은 모든 명령을 아우르는 선택 타입을 만드는 것입니다.  
양파 아키텍처의 `인프라` 영역에 새로운 라우팅 OR 디스패칭 입력 단계를 추가하면 됩니다.

```typescript
type OrderTakingCommand = PlaceOrder | ChangeOrder | CancelOrder;
```

```kotlin
sealed interface OrderTakingCommand {
    data class PlaceOrder(...) : OrderTakingCommand
    data class ChangeOrder(...) : OrderTakingCommand
    data class CancelOrder(...) : OrderTakingCommand
}
```

## 상태 집합으로 주문 모델링하기

주문 접수 작업 흐름에 따르면, 주문은 단순한 정적 문서가 아니라 실제로 여러 상태를 전이하는 것(상태 전이)입니다.  
도메인을 모델링할때는 주문 상태별로 새로운 타입을 만드는 것이 좋습니다. 이렇게 하면 상태를 명시적으로 드러내고 조건부 필드를 제거할 수 있습니다.

```typescript
class ValidatedOrder {
  constructor(
    readonly orderId : OrderId,
    readonly customerInfo : CustomerInfo,
    readonly shippingAddress : Address,
    readonly billingAddress : Address,
    readonly orderLines : ValidatedOrderLine[],
  ) {}
}

class PricedOrder {
  constructor(
    readonly orderId : OrderId,
    readonly customerInfo : CustomerInfo,
    readonly shippingAddress : Address,
    readonly billingAddress : Address,
    readonly orderLines : PricedOrderLine[],
    readonly amountToBill : BillingAmount,
  ) {}
}
```

```kotlin
class ValidatedOrder(
    val orderId : OrderId,
    val customerInfo : CustomerInfo,
    val shippingAddress : Address,
    val billingAddress : Address,
    val orderLines : List<ValidatedOrderLine>,
)

class PricedOrder(
    val orderId : OrderId,
    val customerInfo : CustomerInfo,
    val shippingAddress : Address,
    val billingAddress : Address,
    val orderLines : List<PricedOrderLine>,
    val amountToBill : BillingAmount,
)
```

<br>

마지막으로 모든 상태를 포괄한 최상위 타입을 구현할 수 있습니다.

```typescript
type Order = UnvalidatedOrder | ValidatedOrder | PricedOrder;
```

```kotlin
sealed interface Order
class UnvalidatedOrder(...) : Order
class ValidatedOrder(...) : Order
class PricedOrder(...) : Order
```

<br>

> 각 상태별 타입을 사용하면 기존 코드를 깨뜨리지 않고 새로운 상태를 추가할 수 있다는 장점이 있습니다.  
> 예를 들어 환불을 지원해야 한다는 요구사항이 생기면, 환불 정보를 포함하여 새로운 `RefundedOrder` 상태를 추가해야 할 수도 있습니다.  
> 개별 상태들이 따로 정의되어 있으므로 상태별 수행 코드는 새 상태를 추가하여도 영향받지 않습니다.

## 상태 기계

상태를 도메인 모델링 도구로 사용하는 방법론을 살펴봅시다.  
보통 문서나 레코드는 하나 이상의 상태에 있을 수 있으며, 명령에 따라 상태가 전이됩니다. 이를 상태 기계(state machine)이라고 부릅니다.  
아래는 몇가지 예시입니다.

1. 이메일 주소는 `미확인` 상태와 `확인됨` 상태가 있으며, 사용자가 확인 이메일의 링크를 클릭하면 `미확인`상태에서 `확인됨`상태로 전이됩니다.
2. 쇼핑 카트는 `비었음`, `쇼핑중`, `결제함` 상태가 있으며, 아이템을 카트에 추가하면 `비었음` 상태에서 `쇼핑중` 상태로 전이되고, 결제하면 `결제함` 상태로 전이됩니다.
3. 택배는 `배송전`, `배송중`, `배송완료` 상태가 있으며, 택배개 배송 트럭에 실리면 `배송전`상태에서 `배송중`상태로 전이됩니다.

### 왜 상태 기계를 사용할까요 ?

- 각 상태별로 허용 가능한 동작이 구분됩니다.
- 모든 상태를 타입으로 드러냅니다.
- 모든 가능성을 따져보게 만드는 디자인 도구가 되어줍니다.

### TypeScript와 Kotlin으로 간단한 상태 기계를 구현하는 방법

```typescript
class Item { ... }
class ActiveCart { unpaidItems: Item[] }
class PaidCart { paidItems: Item[]; payment: number }
class EmptyCart {}

type ShoppingCart =
  | EmptyCart // 데이터 없음
  | ActiveCart
  | PaidCart;
```

```kotlin
class Item(...)

sealed interface ShoppingCart {
    data class Active ( val unpaidItems: List<Item> ): ShoppingCart
    data class Paid ( val paidItems: List<Item>, val payment: Double ): ShoppingCart
    object Empty: ShoppingCart
}
```

<br>

명령 핸들러는 전이 가능한 상태들의 선택 타입을 입력으로 받아 같은 선택 타입을 출력하는 함수로 표현합니다.  
결과는 새로운 ShoppingCard이며, 이 카트는 새로운 상태를 전이했을 수도 있고 그대로 머무를 수도 있습니다.  


```typescript
const addItem = (item: Item) => (cart: ShoppingCart) =>
  match (cart)
    .with( P.instanceOf(EmptyCart), _ =>
      new ActiveCart([item]))
    .with( P.instanceOf(ActiveCart), ({unpaidItems: existingItems}) =>
      new ActiveCart([item].concat(existingItems)))
    .with( P.instanceOf(PaidCart), i => i)
    .exhaustive();
```

```kotlin
fun ShoppingCart.addItem(item: Item): ShoppingCart =
  when (this) {
    is ShoppingCart.Empty -> Shopping.Cart(listOf(item))
    is ShoppingCart.Active -> ShoppingCart.Active(listOf(item) + this.unpaidItems)
    is ShoppingCart.Paid -> this
  }
```

<br>

카트를 결제하려면 makePayment 함수에 ShoppingCard 매개변수와 결제 정보를 전달하여 처리합니다.  
결과는 새로운 Paid 상태의 ShoppingCard이거나 종전 상태에 머무를 수 있습니다.

```typescript
const makePayment = (payment: Payment) => (cart: ShoppingCart) =>
  match (cart)
    .with(P.instanceOf(ActiveCart), ({unpaidItems: existingItems}) =>
      new PaidCart(existingItems, payment))   // 결제 정보와 함께 새로운 PaidCart를 생성
    .otherwise( i => i );                     // Empty, Paid는 bypass
```

```kotlin
fun ShoppingCart.makePayment(payment: Payment): ShoppingCart =
  when (this) {
    is ShoppingCart.Active -> ShoppingCart.Paid(this.unpaidItems, payment)   // 결제 정보와 함께 새로운 PaidCart를 생성
    else -> this                              // Empty, Paid는 bypass
  }
```

## 타입으로 작업 흐름의 개별 단계 모델링하기

### 검증 단계

<img width="723" height="281" alt="Image" src="https://github.com/user-attachments/assets/89085486-9722-4d5d-9caf-ae44ec998b5f" />

프로세스는 입력과 출력이 있는 함수로 모델링합니다. 그렇다면 이 의존들을 어떻게 타입으로 모델링할 수 있을까요 ?  
이들도 함수로 모델링하면 됩니다. 함수 시그니처가 우리가 나중에 구현할 `인터페이스`가 됩니다.

다음은 제품 코드가 존재하는지 확인하는 함수입니다.

```typescript
type CheckProductCodeExists = (i: ProductCode) => boolean;
```

```kotlin
type CheckProductCodeExists = (ProductCode) -> Boolean
```

<br>

다음은 UnvalidatedAddress를 받아서 유효한 경우 확인한 주소를 반환하고, 유효하지 않으면 검증 오류를 반환하는 함수입니다.

```typescript
declare const checkedAddress: unique symbol;

class CheckedAddress {
  [checkedAddress]!: never;
  constructor(readonly value: UnvalidatedAddress) {}
}

class AddressValidationError {
  constructor(readonly message: string) {}
}

type CheckAddressExists = (i: UnvalidatedAddress) => Either<AddressValidationError, CheckedAddress>;
```

```kotlin
@JvmInline
value class CheckedAddress(val value: UnvalidatedAddress)

sealed interface AddressValidationError

typealias CheckAddressExists = (UnvalidatedAddress) -> Either<AddressValidationError, CheckedAddress>
```

<br>

이제 의존들을 정리했으니 ValidateOrder 단계를 정의할 수 있습니다.

```typescript
type ValidateOrder =
  (d1: CheckProductCodeExists, d2: CheckAddressExists) // 의존
    => (i: UnvalidatedOrder)                           // 입력
    => Either<ValidationError, ValidatedOrder>;        // 출력
```

```kotlin
typealias ValidateOrder = UnvalidatedOrder.           // 입력
  (CheckProductCodeExists, CheckAddressExists)        // 의존
  -> ValidatedOrder                                   // 출력
```

> 전체 함수의 반환값은 Either여야 합니다.  
> 의존 중 하나인 CheckAddressExists 함수가 Either를 반환하기 때문에, 오류를 처리하는 함수까지의 모든 연산에 전파됩니다.

### 가격 책정 단계

<img width="686" height="255" alt="Image" src="https://github.com/user-attachments/assets/2b933007-3c5f-427f-afc6-7d68d3513023" />

주문에 가격을 매기는 함수도 제품 코드를 입력받아 해당 제품의 가격을 반환하는 함수가 필요합니다.

```typescript
type GetProductPrice = (i: ProductCode) => Price;
```

```kotlin
typealias GetProductPrice = (ProductCode) -> Price
```

<br>

가격 책정 함수는 다음과 같이 정의합니다.

```typescript
type PriceOrder =
    (dep: GetProductPrice)
        => (i: ValidatedOrder)
        => PricedOrder;
```

```kotlin
typealias PriceOrder = ValidatedOrder.(GetProductPrice) -> PricedOrder
```

### 주문 확인 단계

다음은 주문 확인 메일을 작성하고 고객에게 발송하는 것입니다.  
주문 확인 메일은 HTML 문자열을 전송하고, 이는 단순 타입으로 모델링하여 OrderAcknowledgment를 이메일 주소화 메일 내용을 포함한 레코드로 정의할 수 있습니다.  
작업 흐름에 메일 생성 로직을 포함하지 않고, 서비스 함수가 생성한다고 가정합시다.

```typescript
type CreateOrderAcknowledgmentLetter = (i: PricedOrder) => HtmlString;
```

```kotlin
typealias CreateOrderAcknowledgmentLetter = (PricedOrder) -> HtmlString
```

<br>

다음은 메일을 발송할 차례입니다.  


```typescript
enum SendResult { 
    SENT = "Sent",
    FAILED = "NotSent",
}
type SendOrderAcknowledgment = (i: OrderAcknowledgment) => SendResult;
```

```kotlin
enum class SendResult { SENT, FAILED }
typealias SendOrderAcknowledgment = (OrderAcknowledgment) -> SendResult
```

<br>

마지막으로 이것들을 결합한 주문 확인 단계의 함수 타입을 정의할 수 있습니다.  
확인 메일이 발송되지 않을 수 있기 때문에 옵셔널 이벤트를 반환합니다.

```typescript
type AcknowledgeOrder =
  (dep1: CreateOrderAcknowledgmentLetter, dep2: SendOrderAcknowledgment)   // 의존
    => (i: PricedOrder)                                                    // 입력
    => Option<OrderAcknowledgmentSent>;                                    // 출력
```

```kotlin
typealias AcknowledgeOrder = PricedOrder.                                  // 입력
  (CreateOrderAcknowledgmentLetter, SendOrderAcknowledgment)               // 의존
  -> OrderAcknowledgmentSent?                                              // 출력
```

### 반환할 이벤트 생성

배송용 OrderPlaced 이벤트와 청구용 BillableOrderPlaced 이벤트가 남았습니다.

```typescript
type PlaceOrderEvent = 
    | OrderPlaced
    | BillableOrderPlaced
    | OrderAcknowledgmentSent;
```

```kotlin
sealed interface PlaceOrderEvent {
    data class OrderPlaced : PlaceOrderEvent
    data class BillableOrderPlaced : PlaceOrderEvent
    data class OrderAcknowledgmentSent : PlaceOrderEvent
}
```

<br>

그 다음 최종 단계는 PlaceOrderEvent들을 반환하는 것입니다.

```typescript
type CreateEvents = (i: PricedOrder) => PlaceOrderEvent[];
```

```kotlin
typealias CreateEvents = PricedOrder -> List<PlaceOrderEvent>
```

## 효과 문서화하기

의존에 효과를 명시적으로 문서화할 필요가 있는지 알아봅시다.  
검증 단계에는 `CheckProductCodeExists`와 `CheckAddressExists` 두 가지 의존이 있습니다.  
`CheckAddressExists` 함수는 도메인 내부가 아닌 원격 서비스를 호출하므로, 오류 효과 Either와 더불어 비동기 효과 Task를 함꼐 가져야 합니다.

```typescript
type CheckAddressExists =
  (i: UnvalidatedAddress) => TaskEither<AddressValidationError, CheckedAddress>;
```

```kotlin
typealias CheckAddressExists =
  suspend (UnvalidatedAddress) -> Either<AddressValidationError, CheckedAddress>
```

이제 타입 시그니처만으로 `CheckAddressExists` 함수가 외부 입출력을 수행하다 실패할 수 있음을 알 수 있습니다.  
오류 효과와 마찬가지로 비동기 효과는 모든 코드로 전파됩니다. 따라서 ValidateOrder 단계 역시 TaskEither를 반환하도록 변경해야 합니다.

```typescript
type ValidateOrder =
  (d1: CheckProductCodeExists, d2: CheckAddressExists) // 의존
  => (i: UnvalidatedOrder)                             // 입력
  => TaskEither<ValidationError[], ValidatedOrder>;    // 출력
```

```kotlin
typealias ValidateOrder = suspend UnvalidatedOrder.    // 입력
  (CheckProductCodeExists, CheckAddressExists)         // 의존
  -> Either<List<ValidationError>, ValidatedOrder>     // 출력
```

## 개별 단계로부터 작업 흐름 합성하기

이제 모든 단계를 정의했으니, 각 단계별 구현을 마치면 한 단계의 출력을 다음 단계의 입력에 연결하여 전체 작업 흐름을 구축할 수 있습니다.  
함수들을 조합하려면 입력과 출력 타입을 호환하게 조정하고 서로 연결짓도록 해야 합니다. 

```typescript
type ValidateOrder =
  (i: UnvalidatedOrder)                               // 입력
  => TaskEither<ValidationError[], ValidatedOrder>;   // 출력

type PriceOrder =
  (i: ValidatedOrder)                                 // 입력
  => Either<PricingError, PricedOrder>;               // 출력

type AcknowledgeOrder =
  (i: PricedOrder)                                    // 입력
  => TaskOption<OrderAcknowledgmentSent>;             // 출력

type CreateEvents =
  (i: PricedOrder)                                    // 입력
  => PlaceOrderEvent[];                               // 출력
```

```kotlin
typealias ValidateOrder =
  suspend UnvalidatedOrder.()                         // 입력
  -> Either<List<ValidationError>, ValidatedOrder>    // 출력

typealias PriceOrder =
  ValidatedOrder.()                                   // 입력
  -> Either<PricingError, PricedOrder>                // 출력

typealias AcknowledgeOrder =
  suspend PricedOrder.()                              // 입력
  -> OrderAcknowledgmentSent?                         // 출력

typealias CreateEvents =
  PricedOrder.()                                      // 입력
  -> List<PlaceOrderEvent>                            // 출력
```





























