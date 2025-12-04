# 도메인의 무결성과 일관성

도메인은 모든 데이터가 유효하고 일관되도록 몇가지 예방 조치가 필요합니다.  
신뢰할 수 없는 외부 세계와 구분하여 경계진 맥락 내에서는 신뢰하는 데이터만 있게 해야합니다.  
맥락 내 모든 데이터가 항상 유효하다고 확신할 수 있다면 깔끔한 구현은 물론이고, 불필요한 방어 코드를 피할 수 있습니다.

### 신뢰할 수 있는 도메인의 두 가지 측면

- 무결성: 어떤 데이터가 비즈니스 규칙을 따른다는 의미
  - UnitQuantity는 1 ~ 1000 사이여야 합니다.
  - 주문에는 항상 하나 이상의 주문 항목이 있어야 합니다.
  - 주문을 배송팀으로 보내기 전에 주문에 올바른 배송 주소를 포함해야 합니다.
- 일관성: 도메인 모델의 여러 부분을 아울러 규칙을 따른다는 의미
  - 주문의 청구 총액은 주문에 포함된 모든 주문 항목 액수의 총합입니다.
  - 주문을 접수하면 청구서가 생성되어야 합니다.
  - 주문에 할인 바우처 코드를 사용하면 바우처 코드를 다시 사용하지 못하게 표시를 해야합니다.


## 단순값의 무결성

거의 모든 단순값은 제약이 붙습니다. OrderQuantity를 일반 정수로 표현할 수 있지만 비즈니스에서 주문 수량이 음수이거나 40억개일 가능성은 매우 낮습니다.  
CustomerName을 문자열로 표현할 수는 있지만 탭 문자 또는 줄 바꿈을 포함할 수 있다는 의미는 아닙니다.

생성 시 제약 조건을 따른다는 것을 보장하기 위해서는 어떻게 해야 할까요 ?  
먼저 기본 생성자를 비공개로 만들고, 값이 유효할때만 생성자를 호출해주고, 그렇지 않으면 오류를 반환하는 함수를 두면 됩니다.  
함수형 프로그래밍에서는 이를 **스마트 생성자**라고 합니다.

```typescript
declare const unitQuantity: unique symbol;

class UnitQuantity {
    [unitQuantity]!: never;

    private constructor(readonly value: number) { }

    static create(i: number): E.Either<ErrPrimitiveConstraints, UnitQuantity> {
        if (i < 1) {
            return E.left(new ErrNumberLessThanMin(1));
        }
        if (1000 < i) {
            return E.left(new ErrNumberGreaterThanMax(1000));
        }
        return E.right(new UnitQuantity(i));
    }
}
```

```kotlin
@JvmInline
value class UnitQuantity private constructor(val value: Int) {
    companion object {
        operator fun invoke(i: Int): Either<ErrPrimitiveConstraints, UnitQuantity> =
            if (i < 1) {
                ErrNumberLessThanMin(1).left()
            } else if (1000 < i) {
                ErrNumberGreaterThanMax(1000).left()
            } else {
                UnitQuantity(i).right()
            }
    }
}
```

<br>

이를 다음과 같이 패턴 매치하여 사용할 수 있습니다.

```typescript
const unitQtyEither = UnitQuantity.create(1);
match(unitQtyEither)
  .with(P.when(E.isLeft), (e)=>console.log(`Failure, Message is ${e}`))
  .with(P.when(E.isRight), (unitQty)=>console.log(`Success, Value is ${unitQty}, inner value is ${unitQty.right.value}`))
  .exhaustive();
```

```kotlin
val unitQtyEither = UnitQuantity.create(1) // 이전 예제의 invoke를 사용한다면 UnitQuantity(1)이 될 수도 있습니다.
when (unitQtyEither) {
  is Left -> println("Failure, Message is ${unitQtyEither.value}")
  is Right -> println("Success, Value is ${unitQtyEither.value}, inner value is ${unitQtyEither.value.value}")
}
```

## 측정 단위

수치를 모델링할때 타입 안정성을 보장하면서 요구사항을 문서화하는 또 다른 방식으로는 원시 타입에 측정 단위를 달아두는 것입니다.  
우리 도메인에서는 KilogramQuantity의 양이 킬로그램인것을 보장하기 위해 다음과 같이 측정 단위를 활용할 수 있습니다.

```typescript
class KilogramQuantity { constructor(readonly value: Kilogram) { } }
```

```kotlin
@JvmInline
value class KilogramQuantity(val value: Kilogram)
```

이를 통해 Kilogram 측정 단위에서 수치가 kilogram 단위를 갖는 것을 확인하고, KilogramQuantity를 통해서 최소 최대 범주 안에 들어온다는 것을 확인합니다.  
측정 단위는 꼭 물리적 단위에만 사용할 필요는 없고, 시간을 초와 밀리초로 헷갈리지 않도록 사용하거나, x축과 y축을 혼동하지 않도록 사용하거나, 또는 통화에 대해서도 사용할 수 있습니다.  
**측정 단위는 TypeScript, Kotlin 컴파일러만 인식하며 런타임에 오버헤드나 성능상 손해는 없습니다.**

## 타입 시스템으로 불변성 강제하기

**불변성은 무슨 일이 일어나든 항상 참인 명제를 말합니다.** UnitQuantity가 항상 1 ~ 1000 사이여야 한다는 것이 불변성의 예시입니다.  
Order에는 언제나 최소 하나 이상의 OrderLine이 있어야 하는데, 이를 위해 다음과 같이 할 수 있습니다.

```typescript
import { NonEmptyArray } from 'fp-ts/NonEmptyArray'

class Order {
  readonly orderLines : NonEmptyArray<OrderLine>,
  // ...
}
```

```kotlin
import arrow.core.NonEmptyList

class Order (
    val orderLines : NonEmptyList<OrderLine>,
    // ...
)
```

## 타입 시스템에 비즈니스 규칙 녹이기

타입 시스템만으로 비즈니스 규칙을 문서화할 수 있을까요 ?  
즉 데이터의 옳고 그름을 타입 시스템으로 나타내서 컴파일러가 정적 분석때 이를 검사한다면 런타임 검사나 코드 주석에 의존하지 않고도 규칙을 유지하게 만들 수 있을까요 ?

> ex)  
> 위젯이라는 회사가 고객의 이메일 주소를 저장합니다. 하지만 모든 이메일 주소를 동일하게 처리하지 않습니다.  
> 일부 이메일 주소는 고객이 확인 이메일을 받고 링크를 클릭하여 확인을 거친 것입니다.  
> 반면 미확인 이메일 주소는 유효한 주소인지 확실하지 않습니다.  
> 그리고 이메일 주소의 검증 여부에 기반한 비즈니스 규칙이 있습니다.  
> - 기존 고객에게 스팸을 보내지 않기 위해 미확인 이메일 주소에만 확인 이메일을 보낸다.
> - 보안 침해를 방지하기 위해 확인한 이메일 주소에만 비밀번호 재설정 이메일을 보낸다.

```typescript
type CustomerEmail = EmailAddress | VerifiedEmailAddress;
```

```kotlin
sealed interface CustomerEmail {
    data class EmailAddress : CustomerEmail
    data class VerifiedEmailAddress : CustomerEmail
}
```

**확인된 이메일**과 **미확인 이메일**을 별개의 항목으로 모델링합니다.  
여기서 중요한 점은 VerifiedEmailAddress의 생성자를 private으로 막아 아무데서나 이 타입의 값을 생성하지 못하게 하는 것입니다.  
**이 방식을 제대로 수행하면 단위 테스트를 작성할 필요도 없습니다. 단위 테스트를 대신할 컴파일 타임 정적 분석이 있습니다.**

<br>

> ex)  
> 고객에게 연락할 방법이 필요하다는 비즈니스 규칙이 있습니다.  
> 고객은 이메일 또는 우편 주소를 가져야 합니다. 규칙에 따르면 고객은 다음 경우에 해당합니다.  
> - 이메일 주소만 있는 경우
> - 우편 주소만 있는 경우
> - 이메일 주소와 우편 주소 둘 다 있는 경우

```typescript
class BothContactMethods {
  constructor(
    readonly email: EmailContactInfo,
    readonly address : PostalContactInfo,
  ) {}
}

type ContactInfo =
  | EmailContactInfo
  | PostalContactInfo
  | BothContactMethods;

class Contact {
  constructor(
    readonly name: Name,
    readonly contactInfo : ContactInfo,
  ) {}
}
```

```kotlin
sealed interface ContactInfo {
    data class BothContactMethods (
        val email: EmailContactInfo,
        val address : PostalContactInfo,
    ) : ContactInfo

    @JvmInline
    value class EmailContactInfo : ContactInfo

    @JvmInline
    value class PostalContactInfo : ContactInfo
}

data class Contact (
    val name: Name,
    val contactInfo : ContactInfo,
)
```

## 일관성

일관성은 기술적 용어가 아니라 비즈니스 용어이며, 일관성의 의미는 항상 상황에 따라 달라집니다.  
일관성은 데이터를 저장할때 원자성과 밀접하게 연결되어 있습니다.

### 단일 집합체 내의 일관성

주문 총액은 개별 항목들의 합이어야 한다는 요구사항이 있다고 가정합시다.  
일관성을 보장하는 가장 쉬운 방법은 데이터를 저장하지 않고 매번 원시 데이터로부터 계산하는 것입니다.  
만약 우리가 상위의 주문 총액 AmountToBill을 저장해야 한다면, 이 데이터를 일관되게 유지해야 합니다.  
이 상황에서 일관성을 유지할 수 있는 유일한 구성 요소는 상위의 주문입니다. 그렇기에 모든 변경을 항목 수준이 아닌 주문 수준에서 수행해야 합니다.

```typescript
import {produce} from "immer"

const changeOrderLinePrice = (order: Order, orderLineId: OrderLineId, newPrice: number) => {
    // 1. orderLineId로 변경할 orderLine을 찾는다.
    const orderLine = pipe(order.OrderLines, findOrderLine(orderLineId));

    // 2. 새 가격을 반영한 새 OrderLine을 만든다.
    const newOrderLine = produce(orderLine, draft => { draft.price = newPrice; });

    // 3. 이전 항목을 새 항목으로 교체한 새 항목 리스트를 만든다.
    const newOrderLines = pipe(order.OrderLines, replaceOrderLine(orderLineId, newOrderLine));

    // 4. 새 AmountToBill을 만든다.
    const newAmountToBill = newOrderLines.reduce((acc, cur) => acc + cur.price, 0);

    // 5. 새 리스트로 교체한 새 Order를 반환한다.
    return produce(order, draft => {
        draft.orderLines = newOrderLines;
        draft.amountToBill = newAmountToBill;
    });
}
```

```kotlin
fun Order.changeOrderLinePrice(orderLineId: OrderLineId, newPrice: number): Order {
    // 1. orderLineId로 변경할 orderLine을 찾는다.
    val orderLine = findOrderLine(orderLineId, this.orderLines)

    // 2. 새 가격을 반영한 새 OrderLine을 만든다.
    val newOrderLine = orderLine.copy {
        OrderLine.price set newPrice
    }

    // 3. 이전 항목을 새 항목으로 교체한 새 항목 리스트를 만든다.
    val newOrderLines = replaceOrderLine(orderLineId, newOrderLine, this.orderLines)

    // 4. 새 AmountToBill을 만든다.
    val newAmountToBill = newOrderLines.sumOf { it.price }

    // 5. 새 리스트로 교체한 새 Order를 반환한다.
    return this.copy {
        Order.orderLines set newOrderLines
        Order.amountToBill set newAmountToBill
    }
}
```

### 다른 맥락 간의 일관성

주문을 접수하면 청구서가 생성되어야 한다는 요구사항이 있다고 가정합시다.  
청구서 발행은 청구 도메인이 수행하므로 주문 접수 도메인의 일부가 아닙니다.

비즈니스는 메시지를 사용해 비동기적으로 조정합니다.  
때로 오류가 발생할 수 있지만, 드물게 발생하는 오류를 처리하는 비용이 모든것을 동기화하는 비용보다 훨씬 적습니다.  
만약 메시지가 손실되어 청구서가 생성되지 않으면 어떻게 해야 할까요 ?

- 아무것ㅈ도 하지 않습니다. 이런 경우 고객은 무료로 제품을 받게 되고, 비즈니스는 그 비용을 생각해야 합니다.
- 메시지가 손실되었는지 확인하고 다시 보냅니다. 이것은 보통 조정 프로세스가 수행합니다.
- 일부 수행한 작업을 되돌리거나 오류를 수정하는 보상 작업을 만듭니다.

### 같은 맥락의 집합체들 간 일관성

같은 경계진 맥락의 서로 다른 집합체들 간의 일관성은 어떻게 보장할 수 있을까요 ?  
상황에 따라 다르긴 하지만, 일반적으로 한 트랜잭션당 하나의 집합체만 업데이트하고, 만약 하나 이상의 집합체가 관련되어 있다면 메시지와 결과적 일관성을 사용해야 합니다.  
떄로는 비즈니스에서 해당 작업 흐름을 단일 트랜잭션으로 간주하는 경우에는 관련 엔터티를 트랜잭션에 포함시키는 것이 좋을 수 있습니다. 두 계좌간 돈을 이체하는 것이 고전적 예시입니다.

계좌를 Account 집합체로 다루면 우리는 두 개의 다른 집합체를 같은 트랜잭션으로 업데이트해야 합니다.  
이것이 꼭 문제는 아니지만, 도메인에 대해 더 깊은 통찰력을 얻기 위해 리팩토링할 힌트가 될 수 있습니다.   
변경 후에도 Account 엔터티는 여전히 존재하지만, 직접적으로 돈을 추가하거나 제거하는 책임을 지지 않습니다.

```typescript
class MoneyTransfer {
  constructor (
    readonly id: MoneyTransferId,
    readonly toAccount: AccountId,
    readonly fromAccount: AccountId,
    readonly amount: Money,
  ) {}
}
```

```kotlin
class MoneyTransfer (
    val id: MoneyTransferId,
    val toAccount: AccountId,
    val fromAccount: AccountId,
    val amount: Money,
)
```

### 동일한 데이터를 다루는 여러 집합체

집합체는 무결성 제약을 강제해야 합니다. 그렇다면 동일한 데이터를 다루는 여러 집합체가 있을때 무결성을 일관되게 유지하려면 어떻게 해야할까요 ?  
예를 들어 Account, MoneyTransfer 둘 다 계좌 잔액을 다루고, 둘 다 잔액이 음수가 되지 않도록 해야 할 수 있습니다.  
**이런 경우 계좌 잔액이 0 미만이 될 수 없다는 요구사항은 NonNegativeMoney 타입으로 모델링할 수 있습니다.**  
만약 이것으로 부족하다면 공통 검증 함수를 사용할 수 있습니다.  
검증 함수는 특정 객체에 연결되지 않고 전역 상태에 의존하지 않으므로, 다양한 작업 흐름에서 쉽게 재사용이 가능합니다.



































