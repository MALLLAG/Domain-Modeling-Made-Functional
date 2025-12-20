# 직렬화

지금까지 명령을 입력으로 받아 이벤트를 출력하는 작업 흐름 함수를 구현했습니다.  
그렇다면 입력받은 명령은 어디에서 오고, 출력한 이벤트트 어디로 가는걸까요 ?  
이들은 경계진 맥락 바깥의 메시지 큐나 웹 요청 등의 인프라에서 오갑니다. 인프라는 도메인을 알지 못하므로, 도메인 모델을 인프라가 이해하는 JSON, XML 또는 바이너리 형식인 protobuf로 변환해야 합니다.  
또한 주문의 현재 상태처럼 작업 흐름에서 필요한 내부 상태를 추적할 방법이 필요하므로, 데이터베이스같은 외부 서비스를 사용할 가능성이 큽니다.  
외부 인프라와 함께 작업할때는 도메인 모델을 쉽게 직렬화하고 역직렬화할 수 있는 형식으로 변환하는 것이 중요합니다.

## 영속화와 직렬화

**영속화**는 생성된 프로세스가 종료된 이후에도 상태를 유지하는 것을 말합니다.  
**직렬화**는 도메인 특화 표현을 인프라가 받기 쉬운 바이너리, JSON, 또는 XML과 같은 형식으로 변환하는 과정을 말합니다.

## 직렬화를 위한 디자인

여러 타입이 선택 타입을 이루며 제약 조건까지 복잡하게 섞인 도메인 타입들은 직렬화하기 적합하지 않습니다.  
직렬화를 더 쉽게 하기 위해서는, 도메인 객체를 직렬화하기 위한 별도의 타입인 DTO로 변환하고, 도메인 타입이 아닌 이 DTO를 직렬화하는 것이 중요합니다.

<img width="745" height="511" alt="Image" src="https://github.com/user-attachments/assets/eb3918be-e9d8-4c28-8dfd-ad427438a723" />

## 작업 흐름에 직렬화 코드 연결하기

직렬화 과정은 작업 흐름 파이프라인에 추가할 수 있는 컴포넌트입니다.  
작업 흐름 시작 부분에는 역직렬화 단계를, 끝부분에는 직렬화 단계를 추가합니다.

```typescript
const workflowWithSerialization = flow(
  deserializeInputDto,     // JSON에서 DTO로 변환
  inputDtoToDomain,        // DTO에서 도메인 객체로 변환
  workflow,                // 도메인의 핵심 작업 흐름 실행
  outputDtoFromDomain,     // 도메인 객체에서 DTO로 변환
  serializeOutputDto,      // DTO에서 JSON으로 변환
)                          // 최종 출력은 JsonString
```

```kotlin
val workflowWithSerialization: JsonString.() -> JsonString = {
    this.deserializeInputDto()  // JSON에서 DTO로 변환
        .inputDtoToDomain()     // DTO에서 도메인 객체로 변환
        .workflow()             // 도메인의 핵심 작업 흐름 실행
        .outputDtoFromDomain()  // 도메인 객체에서 DTO로 변환
        .serializeOutputDto()   // DTO에서 JSON으로 변환
}                               // 최종 출력은 JsonString
```

이제 workflowWithSerialization 함수를 인프라에 노출합니다.  
이 함수의 입력과 출력은 JsonString 같은 단순한 형식이므로 인프라가 도메인에 영향받지 않도록 격리되었습니다.

## 완전한 직렬화 예제

```typescript
class String50 {
  [string50]!: never;
  constructor(readonly value: string) { ... }
}

class Birthday {
  [birthday]!: never;
  constructor(readonly value: Date) { ... }
}

class Person {
  constructor(
    readonly first: String50,
    readonly last: String50,
    readonly birthdate: Birthdate,
  ) {}
}
```

```kotlin
@JvmInline
value class String50(val value: string) { ... }

@JvmInline
value class Birthdate(val value: LocalDate) { ... }

data class Person (
    val first: String50,
    val last: String50,
    val birthdate: Birthdate,
)
```

<br>

String50과 Birthdate 타입은 직접 직렬화할 수 없기에, 모든 필드가 원시 자료형으로 구성된 DTO 타입 Dto.Person을 정의합니다.  
그리고 도메인 객체를 DTO로 변환하는 fromDomain 함수와 DTO를 도메인 객체로 변환하는 toDomain 함수를 정의합니다.  
도메인은 DTO에 대한 정보를 가지지 않으므로, 이러한 함수들은 DTO 모듈 내에서 정의하는 것이 좋습니다.

```typescript
class PersonDto {
    constructor(
        readonly first: string,
        readonly last: string,
        readonly birthdate: Date,
    ) { }
    
    toDomain(): Either<ErrPrimitiveConstraints, Person> { ... }
    static fromDomain(person: Person): PersonDto { ... }
}
```

```kotlin
data class PersonDto(
    val first: string,
    val last: string,
    val birthdate: LocalDate,
) {
    context(_: Raise<...>)
    fun toDomain(): = Person(...)
}

fun Person.toDto(): PersonDto(...)
```

### JSON 직렬화 라이브러리 매핑하기

TypeScript는 ECMAScript 라이브러리의 JSON.stringify, JSON.parse를 활용합니다.  
다만 역직렬화 라이브러리에서 발생하는 예외를 Either 형태로 바꾸고 일반 객체에 프로토타입을 세팅하는 작업이 필요합니다.

```typescript
namespace Json {
  export const serialize = JSON.stringify
  export const deserialize = <T extends object>(cls: { prototype: T }) => E.tryCatchK(
    flow(
      JSON.parse,
      obj => Object.setPrototypeOf(obj, cls.prototype) as T,
    ),
    e => e,
  )
}
```

<br>

Kotlin은 org.jetbrains.kotlinx:kotlinx-serialization-json 라이브러리가 JSON 직렬화/역직렬화를 담당합니다.

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import kotlinx.serialization.encodeToString

context(r: Raise<DeserializationError>)
inline fun <reified T> JsonString.toDto(): T = try {
    Json.decodeFromString<T>(this)
} catch (e: IllegalArgumentException) {
    r.raise(DeserializationError(e.toString()))
}

context(r: Raise<SerializationError>)
inline fun <reified T> T.toJson(): JsonString = try {
    Json.encodeToString(this)
} catch (e: IllegalArgumentException) {
    r.raise(SerializationError(e.toString()))
}
```

### 완전한 직렬화 파이프라인

이제 DTO-도메인 변환기와 직렬화 함수로 도메인 타입인 Person을 JSON 문자열로 변환할 수 있습니다.

```typescript
const jsonFromPerson = flow(
    Dto.Person.fromDomain,
    Json.serialize,
);
```

```kotlin
val person = Domain.Person(
    first = String50("John"),
    last = String50("Doe"),
    birthdate = LocalDate.of(1980,1,1),
)

val json = person.toDto().toJSON()
```

<br>

역직렬화 파이프라인에서는 Either가 반환될 수 있으므로 다소 복잡합니다.  
Either.mapLeft로 오류를 공통 타입으로 변환하고, 결과 표현식을 통해 오류를 처리합니다.

```typescript
type DtoError = ValidationError | DeserializationError;

const jsonToDomain = (jsonString: JsonString): Either<DtoError, Domain.Person> => pipe(
  E.Do,
  E.bind('deserializedValue', () => pipe(
    jsonString,
    Json.deserialize,
    E.mapLeft(DeserializationError.from),
  )),
  E.flatMap(({deserializedValue}) => pipe(
    deserializedValue,
    Dto.Person.toDomain,
    E.mapLeft(ValidationError.from),
  )),
);
```

```kotlin
val jsonPerson = """{
    |"firstName":"John",
    |"lastName":"Doe",
    |"birthdate":"1980-01-01"
}""".trimMargin()

val person = either { CUstomerInfoDto.fromJSON(jsonPerson).toDomain() }
```

> 역직렬화 오류는 오류를 아예 처리하지 않고 예외를 발생할 수도 있습니다.  
> 어느 방식을 취할지는 역직렬화 오류를 예상 가능한 상황으로 다룰지, 아니면 전체 파이프라인을 중단하는 패닉 상황으로 다룰지에 따라 달라집니다.  
> 이는 API가 얼마나 공개되어 있는지, 호출자를 얼마나 신뢰할 수 있는지, 이와 같은 오류에 대해 호출자에게 얼마나 많은 정보를 제공할지에 따라 결정합니다.


### 직렬화 타입의 여러 버전 관리하기

디자인이 발전해감에 따라 도메인 타입에 필드가 추가 및 삭제되거나 이름이 변경될 수 있습니다.  
그에 따라 DTO 타입 역시 영향을 받을 수 있습니다. DTO 타입은 경계진 맥락간의 계약이므로 계약을 꺠지 않는것이 중요합니다.



















