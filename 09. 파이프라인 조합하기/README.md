# 구현: 파이프라인 조합하기

기술적인 관점에서 살펴보면 파이프라인에서는 다음 과정이 일어납니다.

- UnvalidatedOrder를 받아서 유효성 검사 성공시 ValidatedOrder로 변환, 실패시 오류 반환
- 앞 단계 출력 ValidateOrder를 받아 추가 정보를 더해 PricedOrder 생성
- 앞 단계 출력 PricedOrder로 주문 확인 메일 작성 및 전송
- 관련 이벤트들 생성 및 반환

```typescript
const placeOrder = flow(
  validateOrder,
  priceOrder,
  acknowledgeOrder, 
  createEvents,
);
```

```kotlin
fun UnvalidatedOrder.placeOrder() = this
    .validateOrder()
    .priceOrder()
    .acknowledgeOrder()
    .createEvents()
```

파이프라인의 각 단계부터 독립적인 함수로 구현합니다. 상태나 부수효과가 없도록 함수를 구현하여 독립적으로 테스트하고 추론할 수 있게 해야합니다.  
그 다음 이 작은 함수들을 모아 하나의 큰 함수로 합성합니다.

## 단순 타입 다루기

`OrderId`와 `ProductCode` 같은 단순 타입부터 구현합니다.  
대다수 단순 타입은 어떤식으로든 제약이 있으므로 앞서 살펴봤던 제약 타입처럼 구현해봅시다. 

- String이나 Int같은 원시 타입에서 단순 타입을 생성해내는 create 함수.
- 내부 원시값을 추출하는 value 함수

```typescript
class OrderId {
  [orderId]!: never; 
  
  private constructor(readonly value: string) { }

  static create(str: string): OrderId {
    if (!str) {
      throw Error(`must not be null or empty`);
    }
    if (50 < str.length) {
      throw Error(`must not be more than 50 chars`);
    }
    return new OrderId(str);
  }
}
```

```kotlin
@JvmInline
value class OrderId private constructor(val value: String) {
    companion object {
        operator fun invoke(str: String): OrderId {
            require(str.isNotEmpty()) { ErrEmptyString }
            require(str.length <= 50) { ErrStringTooLong(maxLen) }
            return OrderId(str)
        }
    }
}
```

## 함수 타입으로 구현 가이드하기

구현이 함수 타입에 부합하는지는 어떻게 알 수 있을까요 ?  
가장 간단한 방식으로는 함수 타입 정의를 생략하고 함수 본문을 작성한뒤, 호출 시점에서야 타입 검사를 통해 오류를 확인하는 것입니다.

```typescript
const validateOrder = checkProductCodeExists => checkAddressExists => unvalidatedOrder => {
  // ...
}
```

```kotlin
fun UnvalidatedOrder.validateOrder(d1: CheckProductCodeExists, d2: CheckAddressExists): ValidatedOrder { 
    // ... 
}
```

<br>

두 언어 모두 출력 타입은 추론이 가능하므로 생략할 수 있습니다.  
그러나 특정 함수 타입을 구현한다는 것을 명확히 하고 싶다면, 함수 타입을 명시하고 람다로 함수 본문을 작성합니다.  
이 방식은 함수의 모든 매개변수와 출력 타입이 함수 타입에 따라 결정되므로, 함수 구현 과정에서 실수를 하면 함수 정의 안에서 바로 오류 확인이 가능하다는 장점이 있습니다.

```typescript
type ValidateOrder = (
  dep1: CheckProductCodeExists,
  dep2: CheckAddressExists, // 의존
) => (
  i: UnvalidatedOrder,      // 입력
) => ValidatedOrder;        // 출력

const validateOrder: ValidateOrder =
  (checkProductCodeExists, checkAddressExists) =>
  // CheckProductCodeExists와 CheckAddressExists로 타입 추론
  (unvalidatedOrder) => { // UnvalidatedOrder로 타입 추론
    // ...
    return {} as ValidatedOrder; // 출력 타입이 ValidatedOrder이어야 함
  }
```

```kotlin
typealias ValidateOrder = UnvalidatedOrder.(    // 입력
    CheckProductCodeExists,
    CheckAddressExists                          // 의존
) -> ValidatedOrder                             // 출력

val validateOrder: ValidateOrder = {
    checkProductCodeExists, checkAddressExists ->
    // ...
    // ...
    // 마지막 표현식 타입이 ValidatedOrder이어야 함
    // (여기서 this는 UnvalidatedOrder입니다)
}
```

## 유효성 검증 단계 구현

유효성 검증 단계는 검증에 필요한 원시 정보를 담고있는 UnvalidatedOrder를 받아서 모든 정보들을 검증하여 이로부터 유효한 주문 도메인 객체를 생성합니다.

- UnvalidatedOrder의 OrderId에 해당하는 문자열로부터 OrderId 도메인 타입 생성
- UnvalidatedOrder의 UnvalidatedCustomerInfo 필드로부터 CustomerInfo 도메인 타입 생성
- UnvalidatedOrder의 ShippingAddress 필드로부터 Address 도메인 타입 생성
- BillingAddress 등 나머지 속성도 동일하게 처리
- ValidatedOrder의 모든 구성 요소를 준비하면 이들로 새 레코드를 생성

```typescript
const validateOrder: ValidateOrder =
  (checkProductCodeExists, checkAddressExists) =>
  ({orderId, customerInfo, shippingAddress}) => {
    const orderId = OrderId.create(orderId);
    const customerInfo = toCustomerInfo(customerInfo);
    const shippingAddress = toAddress(shippingAddress);

    return new ValidatedOrder(
        orderIdVal,
        customerInfoVal,
        shippingAddressVal,
        // ...
    );
  }
```

```kotlin
val validateOrder: ValidateOrder = { checkProductCodeExists, checkAddressExists ->
    val orderId = this.orderId.toOrderId()
    val customerInfo = this.customerInfo.toCustomerInfo()
    val shippingAddress = this.shippingAddress.toAddress()
    
    ValidatedOrder(
        orderId = orderId,
        customerInfo = customerInfo,
        shippingAddress = shippingAddress,
        billingAddress = TODO(),
        lines = TODO()
    )
}
```

### 유효한 주소 생성

toAddress 함수는 원시 타입을 도메인 객체로 변환하는 일 외에도 CheckAddressExists 서비스로 실존하는 서비스인지 확인해야 합니다.

```typescript
const toAddress = (checkAddressExists: CheckAddressExists) => (i: UnvalidatedAddress) => {
  // 원격 서비스를 호출하여 주소 확인
  const {addressLine1, addressLine2, addressLine3, addressLine4, city, zipCode} =
    checkAddressExists(i);

  // Address 생성
  return new Address(
    String50.create(addressLine1),
    String50.createOption(addressLine2), // 값이 없을 수 있으므로 createOption 사용
    String50.createOption(addressLine3),
    String50.createOption(addressLine4),
    String50.create(city),
    ZipCode.create(zipCode),
  );
}
```

```kotlin
fun UnvalidatedAddress.toAddress(checkAddressExists: CheckAddressExists): Address {
    // 원격 서비스를 호출하여 주소 확인
    val valid = checkAddressExists(this)

    // Address 생성
    return Address(
        String50(valid.addressLine1),
        valid.addressLine2?.let { String50(it) }, // 값이 null이 아니면 String50 생성
        valid.addressLine3?.let { String50(it) },
        valid.addressLine4?.let { String50(it) },
        String50(valid.city),
        ZipCode(valid.zipCode),
    )
}
```

### 주문 항목 생성

주문 항목 목록은 UnvalidatedOrderLine 하나를 ValidatedOrderLine으로 변환하는 방법이 필요합니다.

```typescript
const toValidatedOrderLine =
  (checkProductCodeExists: CheckProductCodeExists) =>             // 의존
  ({orderLineId, productCode, quantity}: UnvalidatedOrderLine) => // 입력
  new ValidatedOrderLine(
    OrderLineId.create(orderLineId),
    pipe(productCode, toProductCode(checkProductCodeExists)),
    pipe(quantity, toOrderQuantity(productCode)),
  )
```

```kotlin
fun UnvalidatedOrderLine.toValidatedOrderLine(checkProductCodeExists: CheckProductCodeExists) =
    ValidatedOrderLine(
        OrderLineId(this.orderLineId),
        this.productCode.toProductCode(checkProductCodeExists),
        this.quantity.toOrderQuantity(this.productCode),
    )
```

이제 개별 OrderLine을 변환하는 방법이 준비되었으니, map으로 목록 전체를 변환할 수 있습니다.  
이렇게 하면 ValidatedOrderLines를 생성하여 ValidatedOrder에 사용할 수 있습니다.

<br>

다음은 toOrderQuantity 헬퍼 함수입니다. 이는 맥락 경계에서 수행하는 유효성 검사의 좋은 예시입니다.  
UnvalidatedOrderLine으로부터 원시값을 입력받지만 KilogramQuantity와 UnitQUantity의 선택 타입인 OrderQuantity를 출력합니다.

```typescript
const toOrderQuantity = (productCode: ProductCode) =>
  (quantity: number) => match(productCode)
    .with(P.instanceOf(Widget), _ => UnitQuantity.create(quantity)) 
    .with(P.instanceOf(Gizmo), _ => KilogramQuantity.create(quantity))
    .exhaustive();
```

```kotlin
fun Double.toOrderQuantity(productCode: ProductCode) =
    when(productCode) {
        is Widget -> OrderQuantity.Unit(this.toInt()) 
        is Gizmo -> OrderQuantity.Kilogram(this)      
    }
```

<br>

다음은 toProductCode 헬퍼 함수입니다.  
여기서 문제가 발생하는데, toProductCode 함수가 ProductCOde를 반환하기를 원하지만 파이프라인 마지막 단계인 checkProductCodeExists는 boolean을 반환합니다.  
따라서 전체 파이프라인 또한 boolean을 반환하므로 checkProductCodeExists가 ProductCode를 반환하도록 만들어야 합니다.

```typescript
const toProductCode = (checkProductCodeExists: CheckProductCodeExists) => flow(
  ProductCode.create,
  checkProductCodeExists,
); // 반환값은 boolean :(
```

```kotlin
fun String.toProductCode(checkProductCodeExists: CheckProductCodeExists) =
    ProductCode(this).checkProductCodeExists() // 반환값은 Boolean :(
```

### 함수 어댑터 생성

우리는 ProductCode가 올바른지 검사하여 참 거짓을 출력하는게 아니라, 적합하다면 ProductCode를 반환하기를 원합니다.  
스펙을 변경하는 대신 원래 함수를 입력받아 파이프라인에서 사용하도록 원하는 모양의 새 함수를 내보내는 어댑터 함수를 생성합니다.

```typescript
const predicateToPassthru =
  <T>(errorMsg: string, f: (i: T) => boolean) =>
  (x: T): T => {
    if (!f(x)) throw errorMsg;
    return x;
  }
```

```kotlin
fun <T> T.predicateToPassthru(errorMsg: String, f: (T) -> Boolean): T {
    require(f(this)) { errorMsg }
    return this
}
```

## 나머지 단계 구현

각 OrderLine을 PricedOrderLine으로 변환하여 새 PricedOrder를 구성하는 개요는 다음과 같습니다.

```typescript
const priceOrder : PriceOrder =
  (getProductPrice) => ({orderId, customerInfo, shippingAddress, billingAddress, lines}) => {
    const pricedLines = lines.map(toPricedOrderLine(getProductPrice));
    const amountToBill = pipe(
      pricedLines.map(line => line.linePrice),
      BillingAmount.sumPrices,
    );
    return new PricedOrder(
      orderId,
      customerInfo,
      shippingAddress,
      billingAddress,
      pricedLines,
      amountToBill,
    );
  }
```

```kotlin
val priceOrder : PriceOrder = { getProductPrice ->
    val pricedLines = this.lines.map { it.toPricedOrderLine(getProductPrice) }
    val linePriceList = pricedLines.map { it.linePrice }
    val amountToBill = BillingAmount.sumPrices(linePriceList)
    PricedOrder(
        this.orderId,
        this.customerInfo,
        this.shippingAddress,
        this.billingAddress,
        pricedLines,
        amountToBill,
    )
}
```

### 승인 단계 구현

```typescript
const acknowledgeOrder: AcknowledgeOrder =
  (createAcknowledgmentLetter, sendAcknowledgment) => pricedOrder => pipe(
    createAcknowledgmentLetter(pricedOrder),
    letter => new OrderAcknowledgement(pricedOrder.customerInfo.emailAddress, letter),
    acknowledgment => match(sendAcknowledgment(acknowledgment))
      .with(Sent, () => O.some(
        new OrderAcknowledgmentsSent(
          pricedOrder.orderId,
          pricedOrder.customerInfo.emailAddress,
        )
      ))
      .with(NotSent, () => O.none)
      .exhaustive(),
  );
```

```kotlin
val acknowledgeOrder: AcknowledgeOrder = { createAcknowledgmentLetter, sendAcknowledgment ->
    val letter = createAcknowledgmentLetter(this)
    val acknowledgment = OrderAcknowledgement(
        this.customerInfo.emailAddress,
        letter,
    )
    when(sendAcknowledgment(acknowledgment)) {
        SendResult.Sent -> OrderAcknowledgmentSent(
            this.orderId,
            this.customerInfo.emailAddress,
        )
        SendResult.NotSent -> null
    }
}
```

### 이벤트 생성

마지막으로 작업 흐름이 반환할 이벤트들을 생성해야 합니다.  
청구 가능한 금액이 0보다 클때만 청구 이벤트가 전송되도록 요구사항을 추가해봅시다.

```typescript
class OrderPlaced {
  constructor(
    readonly orderId: OrderId,
    readonly customerInfo: CustomerInfo,
    readonly shippingAddress: Address,
    readonly billingAddress: Address,
    readonly amountToBill: BillingAmount,
    readonly lines: readonly PricedOrderLine[],
  ) {}
}

class BillableOrderPlaced {
  constructor(
    readonly orderId: OrderId,
    readonly billingAddress: Address,
    readonly amountToBill: BillingAmount,
  ) {}
}

type PlaceOrderEvent =
  | OrderPlaced
  | BillableOrderPlaced
  | OrderAcknowledgmentsSent;
```

```kotlin
sealed interface PlaceOrderEvent {
    data class OrderPlaced (
        val orderId: OrderId,
        val customerInfo: CustomerInfo,
        val shippingAddress: Address,
        val billingAddress: Address,
        val amountToBill: BillingAmount,
        val lines: List<PricedOrderLine>,
    ) : PlaceOrderEvent

    data class BillableOrderPlaced (
        val orderId: OrderId,
        val billingAddress: Address,
        val amountToBill: BillingAmount,
    ) : PlaceOrderEvent

    data class OrderAcknowledgmentSent (
        val emailAddress : EmailAddress,
        val orderId : OrderId,
    ) : PlaceOrderEvent
}
```

<br>

다음은 이벤트를 생성하는 함수입니다.  
createBillingEvent는 청구 금액이 0이 아닌지 확인하여 옵셔널로 이벤트를 반환해야 합니다.

```typescript
const createOrderPlacedEvent = (i: PricedOrder) =>
  new OrderPlaced(
    i.orderId,
    i.customerInfo,
    i.shippingAddress,
    i.billingAddress,
    i.amountToBill,
    i.lines,
  );

const createBillingEvent = ({ orderId, billingAddress, amountToBill }: PricedOrder):
  O.Option<BillableOrderPlaced> =>
    amountToBill.value > 0
      ? O.some(new BillableOrderPlaced(orderId, billingAddress, amountToBill))
      : O.none;
```

```kotlin
fun PricedOrder.createOrderPlacedEvent() = OrderPlaced(
    this.orderId,
    this.customerInfo,
    this.shippingAddress,
    this.billingAddress,
    this.amountToBill,
    this.lines,
)

fun PricedOrder.createBillingEvent(): BillableOrderPlaced? =
    if (this.amountToBill.value > 0) BillableOrderPlaced(this.orderId, this.billingAddress, this.amountToBill)
    else null
```

<br>

이제 OrderPlaced, OrderAcknowledgmentSent, BillableOrderPlaced 이벤트 세 가지가 있습니다.  
이를 공통 타입으로 반환하기 위해 최소공배수 방식을 사용합니다.

```typescript
const createEvents: CreateEvents = (pricedOrder, acknowledgmentEventOpt) => {
  const event1 = createOrderPlacedEvent(pricedOrder);
  const event2Opt = pipe(
    acknowledgmentEventOpt,
    O.map(e => new OrderAcknowledgmentSent(e.orderId, e.emailAddress)),
  );
  const event3Opt = createBillingEvent(pricedOrder);
}
```

```kotlin
fun createEvents(pricedOrder: PricedOrder, acknowledgmentEventOpt: AcknowledgmentEvent?): Unit {
    val event1 = pricedOrder.createOrderPlacedEvent()
    val event2Opt = acknowledgmentEventOpt?.let {
        OrderAcknowledgmentSent(it.orderId, it.emailAddress)
    }
    val event3Opt = pricedOrder.createBillingEvent()
}
```

<br>

모두 PlaceOrderEvent 이벤트들이지만 일부 이벤트가 옵셔널입니다.  
이를 옵셔널인 이벤트와 아닌 이벤트 모두에 적합한 일반적인 타입, `리스트`로 변환합니다.

```typescript
// helper to convert an Option into a List
export const optionToList: <T>(opt: O.Option<T>) => Array<T> = O.match(
  () => [],
  (x) => [x],
);

const createEvents: CreateEvents = (pricedOrder, acknowledgmentEventOpt) => [
  pipe(
    pricedOrder,
    createOrderPlacedEvent,
  ),
  ...pipe(
    acknowledgmentEventOpt,
    O.map(e => new OrderAcknowledgmentSent(e.orderId, e.emailAddress)),
    optionToList,
  ),
  ...pipe(
    pricedOrder,
    createBillingEvent,
    optionToList,
  ),
];
```

```kotlin
fun <T> T?.toList(): List<T> = this?.let { listOf(it) } ?: emptyList()

fun createEvents(pricedOrder: PricedOrder, acknowledgmentEventOpt: AcknowledgmentEvent?) =
    pricedOrder.createOrderPlacedEvent().toList()
    + acknowledgmentEventOpt.toList().map { OrderAcknowledgmentSent(it.orderId, it.emailAddress) }
    + pricedOrder.createBillingEvent().toList()
```

## 파이프라인 단계들 모두 모으기

validateOrder 함수는 UnvalidatedOrder 입력 외에 두 가지 추가 매개변수가 필요합니다.  
현재 상태로는 입력과 출력이 일치하지 않아서 PlaceOrder 작업 흐름의 입력을 validateOrder 함수와 연결할 수 있는 방법이 없습니다.  
함수형 프로그래밍에서 이렇게 형태가 다른 함수를 조립하는 것이 주요 과제 중 하나입니다.  
이 문제의 대부분 해결책은 **모나드**를 수반합니다. 하지만 지금까지 다뤄온 비교적 간단한 방법인 **부분 적용**을 활용합니다.  
즉, validateOrder 세가지 매개변수중 두가지만 적용하여 입력 하나만 남는 새 함수를 구현합니다.

```typescript
const validate = validateOrder(checkProductCodeExists, checkAddressExists);
const price = priceOrder(getProductPrice)
const acknowledge = acknowledgeOrder(createAcknowledgmentLetter, sendAcknowledgment),
const placeOrder : PlaceOrderWorkflow = flow(
  validate,
  price,
  acknowledge,
  createEvents,
);
```

```kotlin
val placeOrder : PlaceOrderWorkflow = {
    this.validateOrder(checkProductCodeExists, checkAddressExists)
        .priceOrder(getProductPrice)
        .acknowledgeOrder(createAcknowledgmentLetter, sendAcknowledgment)
        .createEvents()
}
```

## 의존 주입

toValidProductCode가 그렇듯, 서비스 함수를 매개변수로 받는 저수준 헬퍼 함수들이 있습니다.  
객체지향 프로그래밍에서는 이를 의존 주입이나 IoC 컨테이너를 사용할 것입니다.  
하지만 함수형 프로그래밍에선느 의존 관계를 명확히 드러내야 하므로, 의존을 명시적인 매개변수로 전달하여 드러내는것이 중요합니다.  
이런 작업을 함수형 프로그래밍에서 처리하는 방법은 `리더 모나드`, `프리 모나드`같은 기법들이 있지만, 이 책에서는 간단한 방법으로 접근합니다.  
모든 의존을 최상위 함수에서 준비하여 콜 스택을 타고 필요한 곳까지 전달하는 방식입니다.

> Kotlin에서는 독특한 의존 주입 방식을 사용합니다.  
> Kotlin은 암묵적 콘텍스트를 명시적으로 다룰 방법을 콘텍스트 수신자(context receiver)라는 실험 기능을 거치며 오랫동안 고민해왔습니다.  
> 마침내 암묵적 콘텍스트를 함수 시그니처에 포함시켜 콘텍스트 매개변수(context parameter)라는 이름으로 제공하게 되었습니다.  
> context라는 키워드로 메시지를 위임할 대상을 지정하면, 콜 스택 상단에 바인딩된 수신 객체에게 메시지를 위임해주는 기능입니다.

```typescript
const validateOrder: ValidateOrder = (checkProductCodeExists, checkAddressExists) =>
    (unvalidatedOrder) => {
        // ...
        const shippingAddress = toAddress(checkAddressExists)(unvalidatedOrder.shippingAddress);
        const lines = pipe(unvalidatedOrder.lines, A.map(toValidatedOrderLine(
            checkProductCodeExists)));
        // ...
        return new ValidatedOrder(..., shippingAddress, lines);
    };
```

```kotlin
context(_: AddressValidator)
fun UnvalidatedOrder.validateOrder(checkProductCodeExists: CheckProductCodeExists) =
ValidatedOrder(
  // ...,
  shippingAddress = this.shippingAddress.toAddress(),
  lines = this.lines.map { it.toValidatedOrderLine(checkProductCodeExists) },
)
```

상위 함수로부터 필요한 의존을 전달받다 보면 결국 모든 서비스를 설정하는 최상위 함수까지 이르게 됩니다.  
객체지향 디자인에서는 이러한 최상위 함수를 **컴포지션 루트**라고 부릅니다.

<br>

그렇다면 validateOrder를 호출하는 placeOrder 작업 흐름 함수가 컴포지션 루트 역할을 해야할까요 ??  
그렇지 않습니다. 서비스를 설정하는 작업은 보통 설정을 읽어오는 등의 과정이 필요하기 때문에, placeOrder 작업 흐름 자체는 필요한 서비스들을 매개변수로 받아야 합니다.

```typescript
const placeOrder: (
  checkCode: CheckProductCodeExists, // 의존
  checkAddress: CheckAddressExists, // 의존
  getPrice: GetProductPrice,         // 의존
  createAck: CreateOrderAcknowledgmentLetter, // 의존
  sendAck: SendOrderAcknowledgment,  // 의존
) => PlaceOrderWorkflow = flow(
  command => command.data,
  validateOrder(checkCode, checkAddress),
  priceOrder(getPrice),
  (pricedOrder) => pipe(
    (pricedOrder),
    acknowledgeOrder(createAck, sendAck),
    createEvents(pricedOrder),
  )
);
```

```kotlin
context(_: AcknowledgmentSender, _: AddressValidator)
fun PlaceOrderCommand.placeOrder(
  checkCode: CheckProductCodeExists,
  getPrice: GetProductPrice,
  createAck: CreateOrderAcknowledgmentLetter,
): List<PlaceOrderEvent> {
  val validatedOrder = this.data.validateOrder(checkCode)
  val pricedOrder = validatedOrder.priceOrder(getPrice)
  val acknowledgementOption = pricedOrder.acknowledgeOrder(createAck)

  return createEvents(pricedOrder, acknowledgementOption)
}
```

### 넘쳐나는 의존

validateOrder의 의존이 만약 네다섯개 이상 필요하면 어떻게 해야할까요 ?  
그리고 다른 단계들도 여러 의존이 필요하면 상위 함수 의존이 폭발적으로 증가할 수 있습니다.  
먼저 해당 함수가 너무 많은 일을 하는지 살펴보고, 더 작은 조각으로 나눠봅시다.  
불가능하다면 의존을 단일 레코드로 묶어서 하나의 매개변수로 전달할 수 있습니다.

예를 들어 checkAddressExists 함수가 URI 엔드포인트와 자격증명이 필요하다고 가정해봅시다.  
이 두개의 매개변수를 checkAddressExists 상위 함수인 toAddress의 호출자에게도 매개변수로 받게 만들어야 할까요 ??  
그렇지 않습니다. 이러한 중간 함수들은 checkAddressExists 함수의 의존에 대해 알 필요가 없어야 합니다.  
훨씬 좋은 방법은 최상위 함수 외부에서 의존을 부분 적용하여 그 자식 함수를 내려보내는 것입니다.

```typescript
const placeOrder : PlaceOrderWorkflow = (unvalidatedOrder) => {
  const endPoint = ...
  const credentials = ...

  const checkAddressExists_ = checkAddressExists(endpoint, credentials)

  const validatedOrder = validateOrder(checkProductCodeExists, checkAddressExists_)
  (unvalidatedOrder);
  
  // ...
}
```

> 함수를 수행하기 위해 입력 외 어떤 의존이 필요한지 명확하게 함수 매개변수로 드러내는 것이 좋지만  
> 내가 필요한 것이 아니라 내가 의존하는 함수가 필요한 의존도 매개변수로 받아서, 마치 내 의존인 것처럼 함수 시그니처에 드러내는 것은 좋지 않습니다.


## 조립한 파이프라인

1. 특정 작업 흐름을 구현하는 모든 코드는 해당 작업 흐름의 이름을 따서 같은 모듈에 모아둡니다. (ex. place-order.workflow.tx, PlaceOrderWorkflow.kt)
2. 파일의 맨 위에는 타입 정의를 작성합니다.
3. 그 후 각 단계의 구현 코드를 작성합니다.
4. 파일의 맨 아래에 각 단계를 모아 주요 작업 흐름 함수로 조립합니다.

```kotlin
typealias CheckProductCodeExists = (ProductCode) -> Boolean

sealed interface AddressValidationError
object InvalidFormat : AddressValidationError
object AddressNotFound : AddressValidationError

@JvmInline
value class CheckedAddress(val value: UnvalidatedAddress)

typealias CheckAddressExists = (UnvalidatedAddress) -> CheckedAddress

interface AddressValidator {
    val checkAddressExists: CheckAddressExists
}

class ValidatedOrderLine(
    val orderLineId: OrderLineId,
    val productCode: ProductCode,
    val quantity: OrderQuantity,
) : Entity<OrderLineId>() {
    override val id: OrderLineId
        get() = orderLineId
}

class ValidatedOrder(
    val orderId: OrderId,
    val customerInfo: CustomerInfo,
    val shippingAddress: Address,
    val billingAddress: Address,
    val lines: List<ValidatedOrderLine>,
) : Entity<OrderId>() {
    override val id: OrderId
        get() = orderId
}

typealias ValidateOrder = context(AddressValidator)
UnvalidatedOrder.
    (CheckProductCodeExists)
-> ValidatedOrder

typealias GetProductPrice = (ProductCode) -> Price

typealias PriceOrder = ValidatedOrder.
    (GetProductPrice)
-> PricedOrder

@JvmInline
value class HtmlString(val value: String)

data class OrderAcknowledgment(
    val emailAddress: EmailAddress,
    val letter: HtmlString,
)

typealias CreateOrderAcknowledgmentLetter =
            (PricedOrder) -> HtmlString

sealed interface SendResult
object Sent : SendResult
object NotSent : SendResult

typealias SendOrderAcknowledgment =
            (OrderAcknowledgment) -> SendResult

interface AcknowledgmentSender {
    val sendOrderAcknowledgment: SendOrderAcknowledgment
}

typealias AcknowledgeOrder = context(AcknowledgmentSender)
PricedOrder. //input
    (CreateOrderAcknowledgmentLetter) 
-> PlaceOrderEvent.OrderAcknowledgmentSent? 

typealias CreateEvents =
            (PricedOrder, PlaceOrderEvent.OrderAcknowledgmentSent?)
        -> List<PlaceOrderEvent>              

fun <E, A> throwOnError(
    block: Raise<E>.() -> A,
): A = either { block() }
    .fold(
        ifLeft = { throw RuntimeException(it.toString()) },
        ifRight = { it }
    )

fun UnvalidatedCustomerInfo.toCustomerInfo(): CustomerInfo {
    val unvalidated = this
    val firstName = throwOnError { String50(unvalidated.firstName) }
    val lastName = throwOnError { String50(unvalidated.lastName) }
    val emailAddress = throwOnError { EmailAddress(unvalidated.emailAddress) }
    return CustomerInfo(PersonalName(firstName, lastName), emailAddress)
}


fun CheckedAddress.toAddress(): Address {
    val checked = this.value
    val addressLine1 = throwOnError { String50(checked.addressLine1) }
    val addressLine2 = checked.addressLine2?.let { throwOnError { String50(it) } }
    val addressLine3 = checked.addressLine3?.let { throwOnError { String50(it) } }
    val addressLine4 = checked.addressLine4?.let { throwOnError { String50(it) } }
    val city = throwOnError { String50(checked.city) }
    val zipCode = throwOnError { ZipCode(checked.zipCode) }
    return Address(addressLine1, addressLine2, addressLine3, addressLine4, city, zipCode)
}


context(avs: AddressValidator)
fun UnvalidatedAddress.toCheckedAddress(): CheckedAddress {
    return avs.checkAddressExists(this)
}

fun String.toOrderId(): OrderId = throwOnError { OrderId(this@toOrderId) }


fun String.toOrderLineId(): OrderLineId = throwOnError { OrderLineId(this@toOrderLineId) }


fun String.toProductCode(checkProductCodeExists: CheckProductCodeExists): ProductCode {
    val code = throwOnError { ProductCode(this@toProductCode) }
    return if (checkProductCodeExists(code)) {
        code
    } else {
        throw RuntimeException("Invalid: ${code}")
    }
}

fun Double.toOrderQuantity(productCode: ProductCode): OrderQuantity = throwOnError {
    OrderQuantity(productCode, this@toOrderQuantity)
}

fun UnvalidatedOrderLine.toValidatedOrderLine(checkProductExists: CheckProductCodeExists): ValidatedOrderLine {
    val orderLineId = this.orderLineId.toOrderLineId()
    val productCode = this.productCode.toProductCode(checkProductExists)
    val quantity = this.quantity.toOrderQuantity(productCode)
    return ValidatedOrderLine(orderLineId, productCode, quantity)
}

val validateOrder: ValidateOrder = { checkCodeExists ->
    ValidatedOrder(
        orderId = this.orderId.toOrderId(),
        customerInfo = this.customerInfo.toCustomerInfo(),
        shippingAddress = this.shippingAddress.toCheckedAddress().toAddress(),
        billingAddress = this.billingAddress.toCheckedAddress().toAddress(),
        lines = this.lines.map { it.toValidatedOrderLine(checkCodeExists) },
    )
}

fun ValidatedOrderLine.toPricedOrderLine(getProductPrice: GetProductPrice): PricedOrderLine {
    val qty = this.quantity.value()
    val linePrice = throwOnError { getProductPrice(this@toPricedOrderLine.productCode).multiply(qty) }
    return PricedOrderLine(
        this.orderLineId,
        this.productCode,
        this.quantity,
        linePrice,
    )
}

fun ValidatedOrder.priceOrder(getProductPrice: GetProductPrice): PricedOrder {
    val lines = this.lines.map { it.toPricedOrderLine(getProductPrice) }
    val linePrices = lines.map { it.linePrice }
    val amountToBill = throwOnError { BillingAmount.sumPrices(linePrices) }

    return PricedOrder(
        this.orderId,
        this.customerInfo,
        this.shippingAddress,
        this.billingAddress,
        amountToBill,
        lines,
    )
}

context(aS: AcknowledgmentSender)
fun PricedOrder.acknowledgeOrder(createAck: CreateOrderAcknowledgmentLetter) =
    OrderAcknowledgment(this.customerInfo.emailAddress, createAck(this))
        .let {
            when (aS.sendOrderAcknowledgment(it)) {
                Sent -> PlaceOrderEvent.OrderAcknowledgmentSent(
                    this.orderId,
                    this.customerInfo.emailAddress,
                )

                NotSent -> null
            }
        }


fun PricedOrder.createOrderPlacedEvent() = PlaceOrderEvent.OrderPlaced(
    this.orderId,
    this.customerInfo,
    this.shippingAddress,
    this.billingAddress,
    this.amountToBill,
    this.lines,
)

fun PricedOrder.createBillingEvent(): PlaceOrderEvent.BillableOrderPlaced? {
    val billingAmount = this.amountToBill.value
    return if (billingAmount > 0) {
        PlaceOrderEvent.BillableOrderPlaced(
            this.orderId,
            this.billingAddress,
            this.amountToBill,
        )
    } else {
        null
    }
}


val createEvents: CreateEvents = { pricedOrder, ackSent ->
    val acknowledgmentEvents = ackSent?.let { listOf(it) } ?: emptyList()
    val orderPlacedEvents = listOf(pricedOrder.createOrderPlacedEvent())
    val billingEvents = pricedOrder.createBillingEvent()?.let { listOf(it) } ?: emptyList()
    acknowledgmentEvents + orderPlacedEvents + billingEvents
}

context(_: AcknowledgmentSender)
fun PricedOrder.toPlaceOrderEvents(createAck: CreateOrderAcknowledgmentLetter): List<PlaceOrderEvent> =
    createEvents(this, this.acknowledgeOrder(createAck))

context(_: AddressValidator, _: AcknowledgmentSender)
fun UnvalidatedOrder.placeOrder(
    checkCodeExists: CheckProductCodeExists,
    getProductPrice: GetProductPrice,
    createAck: CreateOrderAcknowledgmentLetter,
) = this
    .validateOrder(checkCodeExists)
    .priceOrder(getProductPrice)
    .toPlaceOrderEvents(createAck)
```












