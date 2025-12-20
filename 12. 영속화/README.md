# 영속화

대다수 애플리케이션은 프로세스나 작업 흐름이 끝나더라도 더 길게 상태를 영속시켜야 합니다.  
따라서 파일시스템이나 데이터베이스와 같이 상태를 영속시키는 어떤 메커니즘에 의존해야 합니다.  
완벽하게 순수한 도메인 모델에서 복잡하고 지저분한 인프라 세계로 이동할때에는 어느정도의 불일치가 발생합니다.

## 영속화 코드를 가장자리로 밀어내기

이상적으로는 모든 함수가 순수하길 원합니다. 이렇게 하면 함수를 이해하고 테스트하기에 더 쉬워집니다.  
외부 세계에서 데이터를 읽거나 쓰는 함수는 순수할 수 없기에, 작업 흐름 안에 입출력이나 영속화 로직을 섞지 않는 것이 좋습니다. 따라서 작업 흐름을 두 부분으로 분리합니다.

- 비즈니스 로직을 포함하는 도메인 중심 부분
- 입출력 관련 코드를 포함하는 가장자리 부분

```typescript
type InvoicePaymentResult =
  | "FullyPaid"
  | PartiallyPaid

// 도메인 작업 흐름: 순수 함수
const applyPayment = (payment: Payment) => (unpaidInvoice: UnpaidInvoice): InvoicePaymentResult => {
  // 지불 적용
  const updatedInvoice = unpaidInvoice.applyPayment(payment);

  // 결과 처리
  return isFullyPaid(updatedInvoice)
    ? "FullyPaid"
    : PartiallyPaid(updatedInvoice);
}
```

```kotlin
sealed class InvoicePaymentResult {
    object FullyPaid: InvoicePaymentResult()
    class PartiallyPaid( ... ): InvoicePaymentResult()
}

// 도메인 작업 흐름: 순수 함수
val applyPayment: (UnpaidInvoice, Payment) -> InvoicePaymentResult = { unpaidInvoice, payment ->
    // 지불 적용
    val updatedInvoice = unpaidInvoice.applyPayment(payment)

    // 결과 처리
    if (updatedInvoice.isFullyPaid)
        InvoicePaymentResult.FullyPaid
    else
        InvoicePaymentResult.PartiallyPaid(updatedInvoice)
}
```

이 함수는 완전히 순수합니다. 데이터를 불러오거나 저장하지 않고 필요한 데이터는 모두 매개변수로 받습니다.  
의사결정을 내린 결과를 선택 타입 형태로 반환할 뿐 그에 따른 입출력 로직을 즉각 수행하지 않습니다.  
그렇기에 함수가 기대한대로 수행하는지 테스트하기가 쉽습니다.  
순수 함수 applyPayment를 입출력이 허용되는 맥락 경계의 명령 핸들러에서 활용하면 다음과 같습니다.

```typescript
class PayInvoiceCommand {
  ...
  readonly invoiceId : ...
  readonly payment : ...
  ...
}

// 경계 진 맥락 가장자리의 명령 처리기
const payInvoice = async ({invoiceId, payment}: PayInvoiceCommand) => {
  // 데이터베이스에서 읽어 들임
  const unpaidInvoice = await loadInvoiceFromDatabase(invoiceId); // 입출력

  // 순수 도메인 호출
  const paymentResult = pipe(unpaidInvoice, applyPayment(payment)) // 순수

  // 결과 처리
  switch(paymentResult) {
    case "FullyPaid":
      await markAsFullyPaidInDb(invoiceId); // 입출력
      await postInvoicePaidEvent(invoiceId); // 입출력
    default:
      await updateInvoiceInDb(paymentResult.updatedInvoice); // 입출력
  }
}
```

```kotlin
class PayInvoiceCommand (
    val invoiceId : ...
    val payment : ...
)

// 경계 진 맥락 가장자리의 명령 처리기
suspend fun payInvoice(cmd: PayInvoiceCommand): Unit {
    // 데이터베이스에서 읽어 들임
    val unpaidInvoice = loadInvoiceFromDatabase(cmd.invoiceId) // 입출력

    // 순수 도메인 함수 호출
    val paymentResult = applyPayment(unpaidInvoice, cmd.payment) // 순수

    // 결과 처리
    when (paymentResult) {
        InvoicePaymentResult.FullyPaid -> {
            markAsFullyPaidInDb(cmd.invoiceId) // 입출력
            postInvoicePaidEvent(cmd.invoiceId) // 입출력
        }
        is InvoicePaymentResult.PartiallyPaid ->
            updateInvoiceInDb(paymentResult.updatedInvoice) // 입출력
    }
}
```

### 입력값을 토대로 한 의사 결정

만약 데이터베이스에서 읽어들인 내용을 토대로 순수한 코드 한가운데에서 결정을 내려야 한다면 어떻게 할까요 ?  
이럴때는 의사 결정 로직들은 순수 함수로 유지하되, 입출력 효과를 이들 사이에 끼워넣습니다.  
**순수 함수는 비즈니스 로직을 포함하며 결정을 내리고, 입출력 함수는 데이터를 읽고 씁니다.**

```text
--- 입출력 ---
DB에서 송장 불러오기

--- 순수 ---
지불 로직 처리

--- 입출력 ---
결과 선택 타입 패턴 매칭:
    "FullyPaid"일 경우 -> DB에서 송장을 '완납'으로 표시
    "PartiallyPaid"일 경우 -> 업데이터된 송장을 DB에 저장
    
--- 입출력 ---
DB에서 모든 미납 송장의 금액 불러오기

--- 순수 ---
금액 합산 후 금액이 너무 큰지 여부 결정

--- 입출력 ---
결과 선택 타입 패턴 매칭:
    "OverdueWarningNeeded"일 경우 -> 고객에게 메시지 전송
    "NoActionNeeded"일 경우 -> 아무 작업도 하지 않음
```

### 리포지터리 패턴은 어디에 있을까 ?

리포지터리 패턴은 가변성을 활용하는 객체지향 디자인에서 영속화를 추상화하는 훌륭한 방법입니다.  
하지만 모든것을 함수로 모델링하고 영속화 코드를 가장자리로 밀어내면, 리포지터리 패턴은 더이상 필요하지 않습니다.

## 갱신 명령과 조회 질의 분리하기

다음은 명령-질의 분리(CQRS)입니다. 함수형 도메인 모델링에서는 모든 대상을 불변으로 디자인합니다.  
함수형 스타일로 레코드를 삽입하려면 삽입 함수가 삽입할 데이터와 데이터 저장소의 기존 상태를 매개변수로 받고, 삽입 완료 후에는 데이터가 추가된 새로운 데이터 저장소를 반환한다고 생각할 수 있습니다.  
코드로 표현하면 다음과 같은 타입 시그니처로 모델링할 수 있습니다.

```typescript
type InsertData = (d: Data) => (s: DataStoreState) => NewDataStoreState;
type ReadData = (q: Query) => (s: DataStoreState) => Data;
type UpdateData = (d: Data) => (s: DataStoreState) => NewDataStoreState;
type DeleteData = (k: Key) => (s: DataStoreState) => NewDataStoreState;
```

```kotlin
typealias InsertData = DataStoreState.(Data) -> NewDataStoreState
typealias ReadData = DataStoreState.(Query) -> Data
typealias UpdateData = DataStoreState.(Data) -> NewDataStoreState
typealias DeleteData = DataStoreState.(Key) -> NewDataStoreState
```

위 연산에서 하나는 나머지와 다릅니다.

- 생성, 수정, 삭제는 데이터베이스의 상태를 변경하며 유용한 데이터를 반환하지 않습니다.
- 조회는 데이터베이스의 상태를 변경하지 않으며, 네가지 중 유일하게 유용한 결과를 반환합니다.

<br>

CQS 원칙을 함수형 프로그래밍에 적용한다면 다음과 같습니다.

- 데이터를 반환하는 함수는 저장소의 상태를 변경하는 부수효과를 가지면 안됩니다.
- 상태를 변경하는 부수 효과가 있는 함수는 데이터를 반환해서는 안됩니다. 즉, 이 함수는 Unit을 반환해야 합니다.



```typescript
type DbError = ...
type DbResult<T> = TaskEither<DbError, T>;

type InsertData = (i: Data) => DbResult<Unit>;
type ReadData = (i: Query) => DbResult<Data>;
type UpdateData = (i: Data) => DbResult<Unit>;
type DeleteData = (i: Key) => DbResult<Unit>;
```

```kotlin
class DbException { ... }

typealias InsertData = suspend context(Raise<DbException>) (Data) -> Unit
typealias ReadData = suspend context(Raise<DbException>) (Query) -> Data
typealias UpdateData = suspend context(Raise<DbException>) (Data) -> Unit
typealias DeleteData = suspend context(Raise<DbException>) (Key) -> Unit
```

### 명령-질의 책임 분리

저장소에 저장하는 객체와 읽는 객체를 똑같이 맞추려는 유혹이 자주 발생합니다.  
예를 들어 Customer 레코드를 저장소에 저장하고 불러올때 다음과 같은 부수효과로 처리할 수 있습니다.

```typescript
type SaveCustomer = (i: Customer) => DbResult<Unit>;
type LoadCustomer = (i: CustomerId) => DbResult<Customer>;
```

```kotlin
typealias SaveCustomer = suspend context(Raise<DbException>) (Customer) -> Unit
typealias LoadCustomer = suspend context(Raise<DbException>) (CustomerId) -> Customer
```

1. 읽는 데이터는 쓰여질때 필요한 데이터와 종종 다릅니다. 하나의 데이터 타입으로 여러 목적을 충족시키려 하기보다는, 각 데이터 타입을 특정 용도에 맞게 디자인하는것이 좋습니다.
2. 질의와 명령은 독립적으로 진화하는 경향이 있으므로 결합되어서는 안됩니다.
3. 일부 질의는 성능상의 이유로 한번에 여러 엔티티를 반환해야할 수 있습니다.

이러한 관찰을 바탕으로 질의와 명령은 도메인 모델링 관점에서 거의 항상 다르며, 다른 타입으로 모델링해야 합니다.  
질의와 명령 타입의 분리는 자연스럽게 이들이 서로 독립정으로 진화할 수 있도록 다른 모듈로 분리하는 디자인으로 이어집니다.

### CQRS와 데이터베이스 분리

<img width="857" height="258" alt="Image" src="https://github.com/user-attachments/assets/cac93ed9-f7af-4b28-ae3c-2de214292aaa" />

CQRS 원칙은 데이터베이스에도 적용할 수 있습니다.  

### 이벤트 소싱

이벤트 소싱 접근법은 상태를 단일 객체로 저장하지 않습니다.  
대신 상태 변경이 일어날때마다 상태 변경을 나타내는 이벤트를 저장합니다.  
이렇게 하면 이전 상태와 새로운 상태 간의 모든 차이가 캡처되며, 이는 버전 관리 시스템과 유사한 방식입니다.  
이 방식은 많은 장점이 있는데, 특히 모든것이 감사(audit) 대상인 도메인을 모델링할때 잘 맞습니다.

## 경계진 맥락마다 독자 데이터 저장소 소유하기

영속화에 관한 또 다른 중요한 지침은 각 경계진 맥락마다 자신만의 데이터 저장소를 소유하라는 것입니다.

- 경계진 맥락은 자신의 데이터 저장소와 관련 스키마를 소유해야 하며, 다른 경계진 맥락과 조율없이 언제든 이를 변경할 수 있어야 합니다.
- 다른 시스템은 경계진 맥락이 소유한 데이터를 직접 액세스할 수 없습니다. 대신 클라이언트는 경계진 맥락의 공개 API를 이용하거나 데이터 저장소의 복사본을 사용해야 합니다.

### 여러 도메인이 데이터를 사용하는 방법

보고 및 비즈니스 분석 시스템의 경우는 어떨까요 ?  
이런 시스템은 경계진 맥락들의 데이터를 액세스해야 하는데, 리포팅이나 비즈니스 인텔리전스를 별도의 도메인으로 취급하여 다른 경계진 맥락이 소유한 데이터를 리포팅 전용으로 디자인된 별도의 시스템으로 복사하여 사용합니다.  
이 방식은 추가 작업이 필요하지만 소스 시스템과 리포팅 시스템이 서로 독립적으로 진화해가며 각자 관심사에 맞게 최적화합니다.  
다른 경계진 맥락에서 비즈니스 인텔리전스 맥락으로 데이터를 가져오는 방법에는 여러가지가 있는데, 순수한 방법은 다른 시스템에서 발생한 이벤트를 구독하는 것입니다.  
다른 방법은 ETL 프로세스로 소스 시스템에서 BI 시스템으로 데이터를 복사하는 것입니다.

<img width="707" height="362" alt="Image" src="https://github.com/user-attachments/assets/bb50e3f7-6dea-4228-aa90-ff18f6e79bfa" />

## 트랜잭션

일부 데이터 스토어는 API로 트랜잭션을 지원합니다. 여러 서비스 호출을 동일 트랜잭션으로 등록할 수 있습니다.

```typescript
const tx = <A>(pool: Pool, task: (c: PoolClient) => Promise<A>): TE.TaskEither<Error, A> =>
  async () => {
    const client = await pool.connect();
    try {
      await client.query("BEGIN");
      const ret = await task(client);
      await client.query("COMMIT");
      return E.right(ret);
    } catch (e) {
      await client.query("ROLLBACK");
      return E.left(Error(`cause: ${e}`));
    } finally {
      client.release();
    }
  };

// 사용 예시
tx(pool, async client => {
  // 동일한 트랜잭션 내에서 데이터베이스에 두 번 별도로 호출
  await markAsFullyPaid(client, invoiceId);
  await markPaymentCompleted(client, paymentId);
});
```

```kotlin
transaction(db) {
    // 동일한 트랜잭션 내에서 데이터베이스에 두 번 별도로 호출
    Invoice.markAsFullyPaid(invoiceId)
    InvoicePayment.markAsCompleted(paymentId)
}   
```






























