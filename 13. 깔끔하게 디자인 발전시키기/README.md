# 깔끔하게 디자인 발전시키기

도메인 모델이 처음에는 깨끗하고 우아한 모습이지만, 요구사항이 변해가며 모델이 점점 복잡해지고 여러 하류 맥락이 얽혀서 테스트하기 어려워지는 경우가 많습니다.  
따라서 우리의 마지막 도전 과제는 다음과 같습니다. 어떻게 하면 모델을 big ball of mud로 만들지 않고 발전시켜나갈 수 있을까요 ?

## 첫번째 변경: 배송비 추가하기

첫번째로 배송 및 배달 요금을 계산하는 방법을 추가합니다. 회사가 배송비를 계산하여 고객에게 청구하고 싶다고 가정합시다.  
우선 배송비를 계산하는 함수를 만들어야 합니다.

```typescript
// Calculate the shipping cost for an order
const calculateShippingCost = ({shippingAddress}: ValidatedOrder): number => {
  if (shippingAddress.country === "US") {
    switch(shippingAddress.state) {
      case "CA": case "OR": case "AZ": case "NV":   // local
        return 5.0;
      default:                                     // remote
        return 10.0;
    }
  }
  else {
    return 20.0;                                   // shipping outside USA
  }
}
```

```kotlin
// Calculate the shipping cost for an order
fun ValidatedOrder.calculateShippingCost(): Double {
    return if (this.shippingAddress.country == "US") {
        // shipping inside USA
        when(this.shippingAddress.state) {
            "CA", "OR", "AZ", "NV" -> 5.0 // local
            else -> 10.0 // remote
        }
    } else 20.0    // shipping outside USA
}
```

이러한 조건문은 특정 조건에 따라 여러 분기로 나뉘어 이해하기 어렵고 유지 관리도 까다롭습니다.

### 관심사 분리로 비즈니스 로직 단순하게 만들기

이러한 로직을 더 쉽게 유지, 관리하는 방법은 도메인 중심의 분류 작업을 실제 가격 책정 로직에서 분리하는 것입니다.  
우선 각 배송 카테고리에 맞는 패턴 집합을 정의합니다.

```typescript
type CostType = "UsLocalState" | "UsRemoteState" | "International";

const costTypeOf = (i: ShippingAddress) => match(i)
  .with({country: "US", state: "CA" | "OR" | "AZ" | "NV"}, () => "UsLocalState")
  .with({country: "US"}, () => "UsRemoteState")
  .otherwise(() => "International");
```

```kotlin
enum class CostType { UsLocalState, UsRemoteState, International }

fun ShippingAddress.costType(): CostType = when {
    this.country == "US" && (
        this.state == "CA"
        || this.state == "OR"
        || this.state == "AZ"
        || this.state == "NV"
    ) -> CostType.UsLocalState
    this.country == "US" -> CostType.UsRemoteState
    else -> CostType.International
}
```

<br>

그 다음 배송 계산 로직에서 카테고리를 패턴 매칭하여 가격을 책정합니다.

```typescript
const calculateShippingCost = flow(
  costTypeOf,
  t => match(t)
    .with("UsLocalState", () => 5.0)
    .with("UsRemoteState", () => 10.0)
    .with("International", () => 20.0)
    .exhaustive(),
);
```

```kotlin
fun ShippingAddress.calculateShippingCost() = when(this.costType()) {
    CostType.UsLocalState -> 5.0
    CostType.UsRemoteState -> 10.0
    CostType.International -> 20.0
}
```

### 작업 흐름에 새 단계 추가하기

다음은 이 배송비 계산을 주문 처리 작업 흐름에 통합하는 것입니다.  
기존 코드를 손대지 않고 대신 파이프 합성으로 작업 흐름에 새로운 단계를 추가해봅시다.

```typescript
type AddShippingInfoToOrder = (i: PricedOrder) => PricedOrderWithShippingInfo;
```

```kotlin
typealias AddShippingInfoToOrder = PricedOrder -> PricedOrderWithShippingInfo
```

이 새로운 작업 흐름 단계는 PricedOrder 단계와 AcknowledgeOrder 단계 사이에 삽입할 수 있습니다.  
보통 디자인을 발전시켜나가다 보면 다시 들여다볼 세부 사항들을 더 많이 발견합니다.  
간단한 예시로, 고객이 가격뿐 아니라 배송 방식을 알고 싶어할 수도 있습니다.

```typescript
enum ShippingMethod {
  PostalService = "PostalService",
  Fedex24 = "Fedex24",
  Fedex48 = "Fedex48",
  Ups48 = "Ups48",
}

class ShippingInfo {
  constructor(
    readonly shippingMethod : ShippingMethod,
    readonly shippingCost : Price,
  ) {}
}

class PricedOrderWithShippingMethod {
  constructor(
    readonly shippingInfo : ShippingInfo,
    readonly pricedOrder : PricedOrder,
  ) {}
}
```

```kotlin
enum class ShippingMethod {
    PostalService, Fedex24, Fedex48, Ups48
}

data class ShippingInfo (
    val shippingMethod : ShippingMethod,
    val shippingCost : Price,
)

data class PricedOrderWithShippingMethod (
    val shippingInfo : ShippingInfo,
    val pricedOrder : PricedOrder
)
```

기존 PricedOrder 타입에 ShippingInfo 필드만 추가하면 될 일을 새로운 타입을 도입하는것이 과잉 디자인이라고 생각할 수도 있습니다.  
하지만 이 방식에는 몇가지 장점이 있습니다.

- AckowledgeOrder 단계가 PricedOrderWithShippingInfo를 입력값으로 받도록 수정하면 파이프라인 단계들의 순서가 꼬이지 않습니다.
- ShippingInfo를 PricedOrder의 필드로 추가한다면, 배송비를 계산하기 전에는 해당 필드를 어떤 값으로 초기화할지 고민해야 합니다.

<br>

구현 코드는 다음과 같습니다.

```typescript
type CalculateShippingCost = (i: PricedOrder) => ShippingCost;
type AddShippingInfoToOrder = (dep: CalculateShippingCost) => (i: PricedOrder) =>
PricedOrderWithShippingInfo;

const addShippingInfoToOrder: AddShippingInfoToOrder = (calculateShippingCost) =>
(pricedOrder) => pipe(
    pricedOrder,
    calculateShippingCost,
    (shippingCost) => new ShippingInfo(shippingMethod, shippingCost),
    (shippingInfo) => new PricedOrderWithShippingInfo(
        pricedOrder.OrderId,
        ...
        shippingInfo,
    ),
)
```

```kotlin
fun calculateShippingCost(pricedOrder: PricedOrder): ShippingCost = ...

fun addShippingInfoToOrder(pricedOrder: PricedOrder): PricedOrderWithShippingInfo {
    // create the shipping info
    val shippingInfo = ShippingInfo(
        shippingMethod = ...
        shippingCost = calculateShippingCost(pricedOrder),
    )

    // add it to the order
    return PricedOrderWithShippingInfo (
        orderId = pricedOrder.OrderId,
        ...
        shippingInfo = shippingInfo,
    )
}
```

<br>

이 로직을 최상위 작업 흐름에 끼워넣습니다.

```typescript
// 파이프라인 단계의 로컬 버전 설정
// 부분 적용을 사용하여 의존성을 미리 주입(bake in)함
const addShippingInfo = addShippingInfoToOrder(calculateShippingCost);

// 새로운 단일 매개변수 함수들로 파이프라인 구성
flow(
  unvalidatedOrder,
  validateOrder,
  priceOrder,
  addShippingInfo,
  ...
)
```

```kotlin
unvalidatedOrder
    .validateOrder()
    .priceOrder()
    .addShippingInfo(calculateShippingCost)
    ...
```

## 두번쨰 변경: VIP 고객 지원 추가하기

이번엔 전체 작업 흐름의 입력에 영향을 주는 변경의 예시입니다.  
VIP 고객을 지원하게 되었고, 이 고객에게는 무료 배송이나 익일 배송같은 특별한 혜텍을 줘야합니다.  
그렇다면 VIP 상태는 어떻게 모델링해야 할까요 ?  CustomerInfo에 플래그를 추가해야 할까요, 고객 상태 중 하나로 모델링해야 할까요 ? 

```typescript
class CustomerInfo {
  ...
  readonly isVip : boolean;
  ...
}
```

```kotlin
class CustomerInfo (
    ...
    val isVip : Boolean,
    ...
)
```

```typescript
type Normal = Wrapper<CustomerInfo, "Normal">;
type Vip = Wrapper<CustomerInfo, "Vip">;
type CustomerStatus = Normal | Vip;

type Order = {
  customerStatus : CustomerStatus
  ...
}
```

```kotlin
sealed interface CustomerStatus {
    @JvmInline
    value class Normal(val value: CustomerInfo): CustomerStatus

    @JvmInline
    value class Vip(val value: CustomerInfo): CustomerStatus
}

class Order (
    val customerStatus : CustomerStatus,
    ...
)
```

고객 상태를 CustomerStatus로 모델링하는 방식의 단점은 신규 고객과 기존 고객, 멤버십 카드 보유 고객 등과 같이 VIP 상태와 독립적인 다른 고객 상태가 존재할 수 있다는 점입니다.  
가장 좋은 접근법은 VIP 상태를 나타내는 선택 타입을 사용하는 것입니다. 이렇게 하면 다른 고객 정보와는 독립적으로 상태를 표현할 수 있습니다.

```typescript
type VipStatus = "Normal" | "Vip";

class Order {
  readonly vipStatus : VipStatus;
  ...
}
```

```kotlin
enum class VipStatus { Normal, Vip }

class Order (
    val vipStatus : VipStatus,
    ...
)
```

### 작업 흐름에 새로운 입력 추가하기

VipStatus 필드를 사용한다고 가정합시다.  
생성자를 호출하는 모든 곳에서 새롭게 추가한 필드를 위한 값을 채워야 합니다.

```typescript
class UnvalidatedCustomerInfo {
  ...
  readonly vipStatus : string;
}

class CustomerInfo {
  ...
  readonly vipStatus : string | null = null;
}

const validateCustomerInfo = (unvalidatedCustomerInfo) => pipe(
  ...
  TE.bind("vipStatus", () => VipStatus.create(unvalidatedCustomerInfo.vipStatus)),
  TE.map(scope => new CustomerInfo(
    ...
    scope.vipStatus,
  )),
);
```

```kotlin
class UnvalidatedCustomerInfo (
    ...
    val vipStatus : String,
)

class CustomerInfo (
    ...
    val vipStatus : String? = null,
)


context(_: Raise<...>, ...)
suspend fun validateCustomerInfo(unvalidatedCustomerInfo: UnvalidatedCustomerInfo): CustomerInfo {
    ...
    // 새로운 필드 생성 및 검증
    val vipStatus = VipStatus(unvalidatedCustomerInfo.vipStatus)

    return CustomerInfo(
        ...
        vipStatus = vipStatus,
    )
}
```

### 작업 흐름에 무료 배송 규칙 추가하기

안정적인 코드를 수정하는 대신에 파이프라인에 새로운 단계를 추가하는 방식을 사용할 수 있습니다.

```typescript
type FreeVipShipping = 
  (i: PricedOrderWithShippingMethod) => PricedOrderWithShippingMethod;
```

```kotlin
typealias FreeVipShipping = 
  (PricedOrderWithShippingMethod) -> PricedOrderWithShippingMethod
```

## 세번째 변경: 프로모션 코드 지원 추가

영업팀이 프로모션을 진행하고 싶어하며, 주문시 프로모션 코드를 제공하면 할인을 받도록 요청했습니다.  
영업팀과의 논의 끝에 반영할 새 요구사항은 다음과 같습니다.

- 주문시 고객이 프로모션 코드를 제공할 수 있다.
- 코드가 있으면 특정 제품의 가격을 할인가로 제공한다.
- 주문서에 프로모션 할인이 적용되었음을 표시한다.

### 도메인 모델에 프로모션 코드 추가하기

먼저 프로모션 코드를 위한 타입을 정의하고 주문에 옵셔널 필드로 추가해봅시다.

```typescript
class PromotionCode {
  ...
  constructor( readonly value: string ) { }
}

class ValidatedOrder {
  ...
  promotionCode : Option<PromotionCode>;
}
```

```kotlin
@JvmInline
value class PromotionCode(val value: String)

class ValidatedOrder (
    ...
    val promotionCode : PromotionCode?
)
```

<br>

이번 경우에는 UnvalidatedOrder와 DTO에도 promotionCode를 추가해야 합니다.  
**참고로 ValidatedOrder에서 이 필드를 명시적으로 선택사항으로 표시했더라도, DTO에서는 nullable 문자열로 사용할 수 있습니다.**

```typescript
class OrderDto {
  ...
  readonly promotionCode : string | null = null,
  ...
}

class UnvalidatedOrder {
  ...
  readonly promotionCode : string | null,
  ...
}
```

```kotlin
class OrderDto (
    ...
    val promotionCode : String? = null,
    ...
)

class UnvalidatedOrder (
    ...
    val promotionCode : String?,
    ...
)
```

### 가격 책정 로직 변경하기

프로모션 코드 여부에 따라 다른 GetProductPrice 함수를 제공해야 합니다. 로직은 다음과 같습니다.

- 프로모션 코드가 존재하면, 해당 코드를 적용한 할인가를 반환하는 GetProductPrice 함수를 제공한다.
- 프로모션 코드가 존재하지 않으면, 기존의 GetProductPrice 함수를 제공한다.

**따라서 옵셔널 프로모션 코드에 따라 적절한 GetProductPrice 함수를 반환하는 팩토리 함수가 필요합니다.**

```typescript
type PricingMethod =
  | "Standard"
  | PromotionCode
```

```kotlin
sealed interface PricingMethod {
    object Standard : PricingMethod

    @JvmInline
    value class Promotion(val value: PromotionCode) : PricingMethod
}
```

<br>

이제 ValidatedOrder 타입은 다음과 같습니다.

```typescript
class ValidatedOrder {
  ... // 이전과 동일
  readonly pricingMethod : PricingMethod;
}
```

```kotlin
class ValidatedOrder (
    ... // 이전과 동일
    val pricingMethod : PricingMethod
)
```

<br>

GetPricingFuntion도 다음과 같이 변경됩니다.

```typescript
type GetPricingFunction = (i: PricingMethod) => GetProductPrice;
```

```kotlin
typealias GetPricingFunction = (PricingMethod) -> GetProductPrice
```

<br>

이제 GetPricingFuntion 팩토리 함수를 가격 책정 단계의 의존으로 주입해야 합니다.

```typescript
type PriceOrder =
  (dep: GetPricingFunction) // 새 의존
  => (i: ValidatedOrder)    // 입력
  => PricedOrder;           // 출력
```

```kotlin
typealias PriceOrder = ValidatedOrder.(GetPricingFunction) -> PricedOrder
```

### GetPricingFunction 구현하기

```typescript
type GetStandardPriceTable = 
  // 입력 없음 -> 표준 가격 반환
  () => Map<ProductCode, Price>;

type GetPromotionPriceTable = 
  // 프로모션 입력 -> 프로모션 가격 반환
  (i: PromotionCode) => Map<ProductCode, Price>;

const getPricingFunction = 
  (standardPrices: GetStandardPriceTable) => 
  (promoPrices: GetPromotionPriceTable): GetPricingFunction => {

    // 표준 가격 함수
    const getStandardPrice: GetProductPrice = (productCode) => 
      standardPrices().get(productCode);

    // 프로모션 가격 함수
    const getPromotionPrice = (promoCode: PromotionCode): GetProductPrice => 
      (productCode) => 
        promoPrices(promoCode).get(productCode) ?? getStandardPrice(productCode);

    // GetPricingFunction에 맞는 함수 반환
    return (i: PricingMethod) => match(i)
      .with("Standard", () => getStandardPrice)
      .with(P.instanceOf(Promotion), ({value: promoCode}) => getPromotionPrice(promoCode))
      .exhaustive();
  }
```

```kotlin
typealias GetStandardPriceTable = 
    // 입력 없음 -> 표준 가격 반환
    () -> Map<ProductCode, Price>

typealias GetPromotionPriceTable = 
    // 프로모션 입력 -> 프로모션 가격 반환
    (PromotionCode) -> Map<ProductCode, Price>

fun getPricingFunction(
    standardPrices: GetStandardPriceTable,
    promoPrices: GetPromotionPriceTable,
): GetPricingFunction {
    // 표준 가격 함수
    val getStandardPrice: GetProductPrice = { standardPrices()[it] }

    // 프로모션 가격 함수
    fun getPromotionPrice(promoCode: PromotionCode): GetProductPrice = {
        val promotionPrices = promoPrices(promoCode)
        promotionPrices[it] ?: getStandardPrice(it)
    }

    // GetPricingFunction에 맞는 함수 반환
    return { pricingMethod ->
        when(pricingMethod) {
            is PricingMethod.Standard -> getStandardPrice
            is PricingMethod.Promotion -> getPromotionPrice(pricingMethod.value)
        }
    }
}
```

### 주문 항목에 할인 문서화하기

먼저 하류 맥락에 프로모션에 대한 정보가 필요한지를 알아야 합니다.  
정보가 필요하지 않다면 주문 항목 목록에 주석 항목을 추가하면 됩니다.

```typescript
class CommentLine { constructor(readonly value: string) {} ... }

type PricedOrderLine =
  | PricedOrderProductLine
  | CommentLine;
```

```kotlin
sealed interface PricedOrderLine {
    class Product(...): PricedOrderLine { ... }

    @JvmInline
    value class Comment( val value: String): PricedOrderLine
}
```

<br>

주석 항목을 추가하려면 priceOrder 함수를 변경해야 합니다.

- 먼저 GetPricingFunction 팩토리에서 가격 함수를 가져옵니다.
- 다음으로 각 항목에 대해 해당 가격 함수로 가격을 설정합니다.
- 마지막으로 프로모션 코드가 사용된 경우 항목 목록에 특별 주석 항목을 추가합니다.

```typescript
const priceOrder : PriceOrder = (getPricingFunction) => ({
  pricingMethod,
  orderLines,
}: ValidatedOrder) => {
  // 1. 가격 책정 방식에 맞는 함수 가져오기
  const getProductPrice = getPricingFunction(pricingMethod);

  // 2. 각 주문 항목에 가격 매기기
  const productOrderLines = orderLines.map(toPricedOrderLine(getProductPrice));

  // 3. 프로모션 적용 시 주석 라인 추가 여부 결정
  const finalOrderLines = match(pricingMethod)
    .with("Standard", () => productOrderLines)
    .with(P.instanceOf(Promotion), ({value: promoCode}) => {
      const commentLine = CommentLine.create(`프로모션 ${promoCode} 적용됨`);
      return productOrderLines.concat([commentLine]);
    })
    .exhaustive();

  // 4. 최종 가격 책정 주문 반환
  return new PricedOrder(..., finalOrderLines, ...);
}
```

```kotlin
val priceOrder: PriceOrder = { getPricingFunction ->
    // 1. 가격 책정 방식에 맞는 함수 가져오기
    val getProductPrice = getPricingFunction(this.pricingMethod)

    // 2. 각 주문 항목에 가격 매기기
    val productOrderLines = this.orderLines.map { it.toPricedOrderLine(getProductPrice) }

    // 3. 프로모션 적용 시 주석 라인 추가 여부 결정
    val orderLines = when(val method = this.pricingMethod) {
        PricingMethod.Standard -> productOrderLines
        is PricingMethod.Promotion -> productOrderLines + CommentLine("프로모션 ${method.value} 적용됨")
    }

    // 4. 최종 가격 책정 주문 반환
    PricedOrder(... orderLines = orderLines, ...)
}
```

### 경계진 맥락 간 계약 수정하기

앞서 도입한 CommentLine 항목을 배송 시스템이 알아야 주문서를 올바르게 인쇄할 것입니다.  
따라서 하류 맥락으로 전송할 OrderPlaced 이벤트도 변경해야 합니다.

이제 주문 접수 맥락과 배송 맥락 간의 계약이 깨졌습니다. 이와 같이 맥락 간 이벤트를 수정하는 방식은 지속 가능한가요 ?  
이 문제에 대한 좋은 해결책은 **소비자 주도 계약**을 사용하는 것입니다.  
이 방식은 하류의 소비자가 필요한 것을 결정하면, 상류의 생산자는 요구한 데이터를 가감없이 그대로 제공해야 합니다.

## 네번째 변경: 영업시간 제약 추가

새 비즈니스 규칙은 주문은 영업시간 중에만 접수할 수 있다는 것입니다.  
이를 구현하기 위해 어댑터 함수를 만들 수 있습니다.  
이 경우 임의의 함수를 입력으로 받아서 영업시간 외에 호출하면 오류를 발생시키는 래퍼 또는 프록시 함수를 출력하는 영업시간 전용 함수를 만들 것입니다.

```typescript
// 영업시간 결정
const isBusinessHour = (hour: number) => 9 <= hour && hour <= 17;

// 변환기
const businessHoursOnly = (getHour: () => number) =>
    (onError: () => void) =>
        (onSuccess: () => void) =>
            isBusinessHour(getHour()) ? onSuccess() : onError();
```

```kotlin
// 영업시간 결정
fun isBusinessHour(hour: Int) = 9 <= hour && hour <= 17

// 변환기
fun businessHoursOnly(
    getHour: () -> Int,
    onError: () -> Unit,
    onSuccess: () -> Unit
) = when (isBusinessHour(getHour())) {
    true -> onSuccess()
    false -> onError()
}
```

















