# 도메인 이해하기

## 도메인 전문가 인터뷰하기

보통 도메인 전문가들은 현업이 너무 바쁜 나머지 개발자들에게 많은 시간을 할애하기 어려운 것이 대부분입니다.  
이런 경우 **커맨드/이벤트** 방식은 하루종일 진행하는 인터뷰 대신 개별 작업 흐름별로 인터뷰를 짧게 진행할 수 있다는 면에서 유리합니다.  
도메인을 익힐때는 고객의 시트템 사용 방식을 넘겨짚거나, 속단하지 않도록 자제해야 합니다.  
좋은 인터뷰어가 되려면 많이 들어야 하고, 어떤 선입견도 가져서는 안됩니다.

### 비기능적 요구사항 이해하기

도메인 전문가와 하루에 들어오는 주문 수, 성수기에 발생하는 주문 수 등을 이야기하며 트래픽 급증에 대한 대비를 할 수 있습니다.  
또한 고객들이 전문가인지 아닌지 등에 따라 비전문가용 시스템, 전문가용 시스템을 나눠 생각할 수도 있습니다.  
B2B 애플리케이션의 경우 응답시간, 예측 가능성, 실패없는 데이터 처리, 이벤트의 감사 및 추적 등이 일반적으로 요구됩니다.

### 입력 생각해보기

<img width="749" height="435" alt="Image" src="https://github.com/user-attachments/assets/a28a87ce-2462-4194-aa27-d20d96be45df" />

작업 흐름의 출력은 항상 작업 흐름이 생성한 이벤트로서 다른 경계 진 맥락의 동작을 유발시켜야 합니다.  
주문 접수 작업 흐름의 출력은 '주문 접수함' 이벤트가 될 수 있으며, 이는 배송 및 청구 맥락으로 전달됩니다.

## 데이터베이스 중심 디자인 지양하기

<img width="409" height="340" alt="Image" src="https://github.com/user-attachments/assets/7d608cc5-8595-4879-b34d-2da86fabe6df" />

데이터베이스 경험을 능숙하게 다루는 개발자라면, 가장 먼저 테이블과 그들 사이의 관계에 대해 생각할 것입니다.  
하지만 이는 성급한 시도입니다. 도메인 주도 설계에서는 데이터베이스 스키마가 아니라 도메인이 디자인을 주도합니다.

특정 저장소 구현 방식과 무관한 도메인에서 작업하고 모델링하는 것이 좋습니다.  
**데이터베이스 관점에서 모든 디자인을 하다 보면 종종 데이터베이스 모델에 맞춰서 디자인을 왜곡하기 때문입니다.**  

## 클래스 중심 디자인 지양하기

<img width="575" height="460" alt="Image" src="https://github.com/user-attachments/assets/eb3c486e-70e4-4b0c-a10c-451d4eb7d5a6" />

만약 여러분이 객체지향 개발자라면 특정 데이터베이스 모델에 치우치지 않게 한다는 아이디어가 익숙할 것입니다.  
실제로 **의존 주입**과 같은 객체지향 기법은 데이터베이스 구현체를 비즈니스 로직에서 분리하도록 권장합니다.  
하지만 도메인보다는 객체 관점에서 생각한다면 여전히 편향된 디자인을 할 여지가 있습니다.

예비 디자인에서 우리는 주문과 견적을 분리했지만, 실세계에는 없는 OrderBase를 base class로 도입했습니다.  
이는 도메인을 왜곡한 것입니다. 도메인 전문가는 OrderBase가 무엇인지 모릅니다.

## 도메인 문서화

- 작업 흐름은 입력과 출력을 문서화하고 간단한 의사 코드로 비즈니스 로직을 표현합니다.
- 데이터 구조의 AND는 둘 다 모두 필요하다는 것을, OR은 둘 중 하나만 필요하다는 것을 의미합니다.

```
# 주문 배치 작업 흐름
Bounded context: Order-taking

Workflow: Place order
  triggered by:
    'Order form received' event (when Quote is not checked)
  primary input:
    An order form
  other input:
    Product catalog
  output events:
    'Order Placed' event
  side-effects:
    An acknowledgment is sent to the customer,
    along with the placed order
```

```
# 작업 흐름 관련 데이터 구조
Bounded context: Order-taking

data Order =
  CustomerInfo
  AND ShippingAddress
  AND BillingAddress
  AND list of OrderLines
  AND AmountToBill

data OrderLine =
  Product
  AND Quantity
  AND Price

data CustomerInfo = ??? // 아직 모름
data BillingAddress = ??? // 아직 모름
```

## 복잡미묘한 도메인 모델링하기

<img width="590" height="560" alt="Image" src="https://github.com/user-attachments/assets/ef65d253-8c1f-41b9-80b9-6be3b8d15370" />

### 제약 사항 표현하기

가장 먼저 기본값인 제품 코드와 수량부터 시작합니다.  
이들은 단순한 문자열과 정수가 아니라 여러 제약 조건들이 붙어있습니다.

```
Bounded context: Order-taking

data WidgetCode = string starting with "W" with 4 digits
data GizmoCode = string starting with "G" with 3 digits
data ProductCode = WidgetCode OR GizmoCode
```

도메인 전문가의 관점으로 디자인을 포착하는 것이 중요합니다. 유형병 코드를 검사하는 것은 검증 과정에서 중요한 부분입니다.  
**따라서 도메인 디자인에 제약 사항을 표시함으로써 디자인 자체로 문서 역할을 해야합니다.**  

이어서 수량에 대한 요구사항을 문서화하면 다음과 같습니다.

```
data OrderQuantity = UnitQuantity OR KilogramQuantity

data UnitQuantity = integer between 1 and 1000
data KilogramQuantity = decimal between 0.05 and 100.00
```

### 주문의 생애 주기 표현하기

주문에는 생애 주기가 있습니다. 주문을 우편함에서 꺼내면 '검증 전' 상태로 시작하여 '검증'을 거쳐 '가격 책정'이 이루어집니다.  
종이 주문서를 다룰 때에는 각 단계마다 주문서에 표시를 남겨 구분합니다. 따라서 검증되지 않은 주문을 검증된 주문과 즉시 구별할 수 있고, 검증된 주문은 가격이 책정된 주문과 구별할 수 있습니다.  
**가격이 책정되지 않은 주문은 배송팀으로 보내져서는 안된다는 것을 명확히 하기 위하여 개별 단계를 도메인 디자인에 반영해야 합니다.**

```
# 검증되지 않은 초기 주문과 견적
data UnvalidatedOrder =
  UnvalidatedCustomerInfo
  AND UnvalidatedShippingAddress
  AND UnvalidatedBillingAddress
  AND list of UnvalidatedOrderLine

data UnvalidatedOrderLine =
  UnvalidatedProductCode
```

```
# 주문을 검증하고 난 뒤 단계
data ValidatedOrder =
  ValidatedCustomerInfo
  AND ValidatedShippingAddress
  AND ValidatedBillingAddress
  AND list of ValidatedOrderLine

data ValidatedOrderLine =
  ValidatedProductCode
  AND ValidatedOrderQuantity
```

```
# 주문에 가격을 매기는 단계
data PricedOrder =
  ValidatedCustomerInfo
  AND ValidatedShippingAddress
  AND ValidatedBillingAddress
  AND list of PricedOrderLine      // ValidatedOrderLine과 다름
  AND AmountToBill                 // 새 정보

data PricedOrderLine =
  ValidatedOrderLine
  AND LinePrice // 새 정보
```

```
# 주문 알림 생성
data PlacedOrderAcknowledgment =
  PricedOrder
  AND AcknowledgementLetter
```

> 이 디자인으로 많은 비즈니스 로직을 포착했습니다. 예를 들면 다음과 같은 규칙입니다.
> - 검증하지 않은 주문에는 가격이 없습니다.
> - 검증한 주문의 모든 라인은 일부가 아니라 모두 검증해야 합니다.

### 작업 흐름의 단계 구체화하기

원래는 '주문 접수' 이벤트가 유일한 출력이었지만, 이제는 다음과 같은 것들이 작업 흐름의 결과가 될 수 있습니다.

- 배송/결제에 '주문 접수함' 이벤트를 보냄
- 주문서를 잘못된 주문 더미에 넣고 나머지 단계를 건너뜀

```
# 전체 작업 흐름 의사코드
workflow 'Place Order' =
  input: OrderForm
  output:
    OrderPlaced event (put on a pile to send to other teams)
    OR InvalidOrder (put on appropriate pile)

  // step 1
  do ValidateOrder
  If order is invalid then:
    add InvalidOrder to pile
    stop

  // step 2
  do PriceOrder

  // step 3
  do SendAcknowledgmentToCustomer

  // step 4
  return OrderPlaced event (if no errors)
```

전체 흐름을 문서화했으면 각 하위 단계에 대한 세부 정보를 추가할 수 있습니다.  
양식 검증 단계는 UnvalidatedOrder를 입력으로 받고, 출력은 ValidatedOrder 또는 ValidationError입니다.

```
substep 'ValidateOrder' =
  input: UnvalidatedOrder
  output: ValidatedOrder OR ValidationError
  dependencies: CheckProductCodeExists, CheckAddressExists

  validate the customer name
  check that the shipping and billing address exist
  for each line:
    check product code syntax
    check that product code exists in ProductCatalog

  if everything is OK, then:
    return ValidatedOrder
  else:
    return ValidationError
```

```
substep 'PriceOrder' =
  input: ValidatedOrder
  output: PricedOrder
  dependencies: GetProductPrice

  for each line:
    get the price for the product
    set the price for the line
  set the amount to bill ( = sum of the line prices)
```

```
substep 'SendAcknowledgmentToCustomer' =
  input: PricedOrder
  output: None

  create acknowledgment letter and send it
  and the priced order to the customer
```




















