# 포트폴리오: 회사 백엔드 플랫폼

<!-- TOC -->
* [Philosophy: 왜 이 플랫폼을 만들었는가?](#philosophy-왜-이-플랫폼을-만들었는가)
* [Architecture: 플랫폼 구성 요소](#architecture-플랫폼-구성-요소)
* [Key Features & Design Decisions](#key-features--design-decisions)
  * [API 계층: bun-spring-starter-api-lib](#api-계층-bun-spring-starter-api-lib)
    * [페이징 모델 재정의](#페이징-모델-재정의)
    * [조직 표준 응답 데이터 모델 구조화](#조직-표준-응답-데이터-모델-구조화)
  * [데이터 계층: bun-spring-starter-data-lib](#데이터-계층-bun-spring-starter-data-lib)
    * [데이터 저장소 일관된 설정 구조 제공](#데이터-저장소-일관된-설정-구조-제공)
    * [조직 표준 분산락 구현](#조직-표준-분산락-구현)
      * [CohortDistributedLock](#cohortdistributedlock)
    * [querydsl 확장](#querydsl-확장)
    * [jpa 타입 확장](#jpa-타입-확장)
      * [상황별 최적화를 제공하는 EnumColumn 시스템](#상황별-최적화를-제공하는-enumcolumn-시스템)
      * [필드 레벨 암호화 SecretString](#필드-레벨-암호화-secretstring)
  * [서비스간 통신 방식 개선: bun-spring-starter-api-client-lib (개발중)](#서비스간-통신-방식-개선-bun-spring-starter-api-client-lib-개발중)
    * [ApiPayload](#apipayload)
* [백엔드 E2E 테스트를 위한 여정](#백엔드-e2e-테스트를-위한-여정)
* [Impact & Vision](#impact--vision)
<!-- TOC -->

# Philosophy: 왜 이 플랫폼을 만들었는가?

회사의 백엔드 시스템은 수십 개의 마이크로서비스로 구성되어 있으며, 30명 이상의 개발자가 동시에 협업하고 있습니다.
이러한 환경에서 저는 이런 문제가 있다고 정의했습니다.

- 프로젝트마다 일관성 없이 조직 공통 응답 데이터 모델 생성
- 통일되지 않는 데이터 접근 로직
- 신뢰할 수 없는 서비스 간 통신
- 통합테스트의 부재와 비효율적인 테스트로 소스코드의 신뢰도를 보장하지 못함

이런 문제를 근본적으로 해결하기 위하여 비기능적인 소스코드를 라이브러리로 분리하면서 플랫폼을 구축하기 시작했습니다.
플랫폼의 핵심 철학은 **개발자의 인지적 부하를 최소화하고, 실수를 컴파일 타임에 방지하며, 모든 개발자가 재미있게 비즈니스 로직에만 집중할 수 있는 잘 닦인 고속도로를 제공하는 것** 입니다.

저는 이 비전을 실현하기 위해 API, 데이터, 이벤트, MSA 통신, 테스트 등 백엔드 개발의 전체 생명주기를 아우르는 플랫폼을 0에서 1로 직접 설계하고 구축했습니다.

# Architecture: 플랫폼 구성 요소

회사 백엔드 플랫폼은 각각 명확한 책임을 가진 spring-boot-starter 라이브러리 세트로 구성되어 있으며, bun-core-kotlin이라는 핵심 약속(Contract)을 공유합니다.

- bun-core-kotlin: 플랫폼의 헌법. AuthUser, BunContext 등 모든 라이브러리가 공유하는 핵심 추상화.
- bun-spring-starter-api-lib: API 계층의 표준.
- bun-spring-starter-data-lib: 데이터 접근 계층의 안정성과 생산성.
- bun-event-publisher/subscriber: 이벤트 기반 시스템의 표준 (2년차때 만든거라서 이름이 통일되어 있지 않습니다.)
- bun-spring-starter-api-client-lib: MSA 통신의 신뢰성 개선.

# Key Features & Design Decisions

## API 계층: bun-spring-starter-api-lib

이 프로젝트는 조직 api 서버 프로젝트 에서 나오는 공통적인 문제를 해결합니다.

### 페이징 모델 재정의

***Problem***

기존 Spring Pageable은 데이터 계층의 모델이 API 계층까지 노출되는 아키텍처 문제를 가졌으며, 정렬이 필요 없는 API에도 sort 파라미터를 노출하는 등 API 계약을 모호하게 만들었습니다.

***Solution***

cursor 모델 지원, sort 파라메터 분리, 타입 안전한 sort 파라메터 등 `ApiPageParameter`를 직접 설계하여 문제를 근본적으로 해결했습니다.

```Kotlin
sealed interface ApiPageParameter {
    sealed interface OffsetBase : ApiPageParameter { /* ... */ }
    sealed interface CursorBase : ApiPageParameter { /* ... */ }
}

sealed interface ApiOffsetParameter : ApiPageParameter.OffsetBase
sealed interface ApiOffsetWithSortParameter : ApiPageParameter.OffsetBase {
    val sort: Iterable<String>
}
sealed interface ApiOffsetWithTypeSortParameter<E : Enum<E>> : ApiOffsetWithSortParameter
```

이 설계를 통해, 정렬을 원치 않는 메서드에 정렬 기능이 있는 파라미터가 넘어가는 실수를 컴파일 타임에 원천 차단했습니다.
특히, `ApiOffsetWithTypeSortParameter<E: Enum<E>>`와 같은 제네릭 인터페이스는, Enum 타입 파라메터 활용하여 Spring 파라미터 리졸버 통과 시 자동으로 유효성을 검사하고,
springdoc 문서화 시 Enum 필드를 렌더링하며, 다른 객체로 변환 시 sort key를 Enum 으로 제공하여 타입 안정성을 보장하는, 높은 수준의 개발자 경험(DX)을 제공합니다.

### 조직 표준 응답 데이터 모델 구조화

***Problem***

사내 문서에는 표준 스펙이 정의 되어 있지만 이를 위한 라이브러리는 존재하지 않았습니다. 그로 인하여, 프로젝트마다 조직 표준 응답 데이터 모델링이 다양한 방법으로 있고, 만드는 방법도 제각각 이기 때문에 가장 자주
마주치는 코드들이 프로젝트마다 일관성이 없어 개발자의 인지 부하를 일으킨다고 저는 판단했습니다.

***Solution***

조직 표준 응답 데이터를 모델링하고, 이를 쉽게 만들기 위한 확장 함수를 제공합니다. 리시버 타입에 따라 올바른 응답 데이터 구조를 알아서 처리합니다.

```kotlin
/**
 * 표준 api 응답 모델. 이 클래스가 가장 기본적인 형태이며 추가 필드에 따라 하단과 같은 모델이 있습니다.
 *
 * - [ApiOffsetResult] : offset 페이징.
 * - [ApiSliceResult] : offset 페이징.
 * - [ApiCursorResult] : 단방향 커서 페이징.
 * - [ApiBiCursorResult] : 양방향 커서 페이징.
 * - [ApiErrorResult] : 에러 응답 (일반적으론 직접 생성하지 않으며 [HttpException] 발생시 [com.bunjang.api.spring.web.BunApiWebFluxExceptionHandler] 에서 해당 객체를 생성합니다.)
 *
 * [Any.toResult] 로 간단하게 자료구조에 맞는 인스턴스로 변환할 수 있습니다. 대부분 `ApiResult<T>` 형태로 변환이 되며 특정케이스에 한해서 위에 언급한 클래스로 변환 될 수 있습니다.
 */
data class ApiResult<R>(val data: R) {
    companion object
}

/**
 * Spring의 `Page` 객체를 API 응답 모델인 [ApiOffsetResult] 로 변환합니다.
 *
 * ## 핵심 동작 및 주의사항
 *
 * 이 함수는 응답 객체의 `size` 필드에 `Page`의 실제 컨텐츠 개수(`getNumberOfElements()`)가 아닌, **요청된 페이지 크기(`getSize()`)**를 설정합니다.
 *
 * ### 예시: 마지막 페이지 조회
 * - **요청**: 페이지 크기(size) = `10`
 *
 * - **이 함수의 응답**: `size` = **10** (요청된 `size` 값)
 * - **기존 방식의 응답**: `size` = **5** (실제 컨텐츠 개수)
 *
 * 따라서 기존 코드에서 응답 객체 변환 코드를 이 함수로 교체할 경우, 클라이언트와의 API 계약을 반드시 확인해야 합니다.
 *
 * @see org.springframework.data.domain.Page.getNumberOfElements
 * @sample ApiOffsetResult
 */
fun <E, R> org.springframework.data.domain.Page<E>.toResult(
    transformer: (E) -> R
): ApiOffsetResult<R> = ApiOffsetResult(
    data = content.map(transformer),
    page = number,
    size = size, // 요청한 페이지 사이즈, content 의 사이즈가 아닙니다! getNumberOfElements 로 데이터 설정중이라면 점검 필요
    totalPages = totalPages,
    totalElements = totalElements,
)

fun <K, V, R> Map<K, V>.toResult(transformer: (Map.Entry<K, V>) -> R): ApiResult<List<R>> = ApiResult(map(transformer))

// 익스텐션 함수 문서화시 receiver 타입과 함수 순서에 따라 렌더링됨. 이 함수는 Any 에 익스텐션이 붙은것과 동일하므로 위에 두면 항상 이 문서가 나타나게됨.
// 그러므로 제일 아래에 배치
fun <R> R.toResult(): ApiResult<R> = ApiResult(this)

fun ApiErrorResult(
    errorCode: String,
    reason: String,
    init: (ApiErrorResult.Builder.() -> Unit)? = null,
): ApiErrorResult = HashMapApiErrorResult(errorCode, reason).apply { init?.invoke(this) }

interface ApiErrorResult {
    val errorCode: String
    val reason: String

    operator fun get(key: String): Any?

    interface Builder {
        var errorCode: String
        var reason: String
        operator fun get(key: String): Any?
        operator fun set(key: String, value: Any?)
    }

    companion object
}

```

모든 함수의 이름음 `toResult` 로 통합되어 있습니다. 결과적으로 프로젝트마다 일관된 api 응답 객체 생성을 유도할 수 있게 되었고,
개발자는 프로젝트마다 표준 데이터 모델링을 안해도 되는 등, 개발 생산성과 인지 부하 감소, 잘못된 데이터 설정으로 인한 실수 원천 차단이 가능해졌습니다.

## 데이터 계층: bun-spring-starter-data-lib

### 데이터 저장소 일관된 설정 구조 제공

***Problem***

일관된 설정 구조를 제공하는 라이브러리가 없어서 (특히 db) 하단과 같은 문제가 있다고 정의했습니다.

- 다른데서 쓰던 코드 복붙해가면서 설정 경로만 바꿈 : 설정 파라메터가 표준화 되지 않음 (특히 db password)
- 멀티 datasource 를 쓰게된다면 복붙해가는 코드양은 더 늘어나는데 보통 테스트 코드를 작성하지 않으니 신뢰도가 떨어지고, 매 프로젝트마다 디비 접근 방식을 학습해야함.
- application.yml 설정 구조가 통일되지 않음
- 로컬 개발시 db 환경이 일관되지 않거나 staging 에 붙어서 개발하는 경향이 있음

***Solution***

인프라를 설정 클래스로 모델링하고, 파라메터 정의, 실행 환경별 디폴트 설정 제공을 통해 해결했습니다.
하단은 이 문제를 해결한 뒤의 datasource 설정 코드 입니다.

```kotlin
@Configuration
@Import(
    ProductDataSourceConfig::class,
    UserDataSourceConfig::class,
)
@EnablePrimaryDataSource(ProductDataSourceConfig::class)
class DBConfig

/**
 * [dataSource] 에 지정한 bean 에 [org.springframework.context.annotation.Primary] 를 적용합니다.
 *
 * [javax.sql.DataSource] 인스턴스가 2대 이상일때 [jakarta.persistence.EntityManagerFactory] 생성 실패 문제를 해결하기 위해 사용됩니다.
 */
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Import(BunPrimaryDataSourceRegistrar::class)
annotation class EnablePrimaryDataSource(val dataSource: KClass<out BunDataSourceConfig>)
```

그 외에도 redis, redisson 과 같은 조직 표준 설정도 동일한 논리로 자동구성합니다. 그 결과 data 계층 설정 보일러 플레이트 코드 90% 이상 제거라는 목표를 달성했습니다.

또한 테스트 실행 환경에 따라 인메모리 or 경량 버전 로 실행하던지, testcontainer 로 운영과 유사한 환경에서 실행할지 선택할 수 있도록 했습니다.
로컬에선 빠른 피드백을, ci 에선 운영환경과 최대한 비슷한 환경에서 테스트를 실행해서 소스코드의 신뢰도를 향상시켰습니다.

### 조직 표준 분산락 구현

***Problem***

분산 락을 사용하는 로직은 테스트하기가 매우 어렵습니다. 실제 Redis와 같은 외부 의존성에 기대어 통합 테스트를 작성하는 것은 느리고 불안정하며,
이는 결국 테스트 코드 작성 자체를 기피하게 만들어 코드의 신뢰도를 떨어뜨리는 원인이 되었습니다.

***Solution***

이 문제를 해결하기 위해, 먼저 실제 분산 락처럼 동작하지만 메모리 내에서만 비동기적으로 작동하는 정교한 테스트 더블(Test Double)인 `InMemoryLock`을 직접 구현했습니다.
또한 서버 실행시 실행 환경, 런타임 클래스 여부 등을 판단하여 적합한 분산락 인스턴스를 (spin, pub/sub, inmemory) 자동으로 선택하도록 구성했습니다.

이 단순한 `InMemoryLock`을 만들고 보니, 이것을 활용하여 분산 시스템의 더 근본적인 문제인 **Thundering Herd**를 해결할 수 있다는 통찰을 얻게 되었습니다.

#### CohortDistributedLock

제가 만든 `InMemoryLock` 을 **로컬 락**으로 활용하여 이 문제를 해결했습니다.
먼저 각 서버 내부에서 로컬 락으로 **대표**를 단 하나만 선출하고, 오직 이 소수의 대표들만이 실제 **글로벌 분산 락** 경쟁에 참여하도록 설계했습니다.
만들고 나니 이게 `락 코호팅`이라는걸 알게 되었고 클래스 이름을 `CohortDistributedLock` 라고 부여하게 되었습니다.

```Kotlin
class CohortDistributedLock(
    private val global: DistributedLock, // 실제 분산 락 (예: Redis)
) : DistributedLock {
    // 테스트를 위해 만들었던 InMemoryLock을 '로컬 락'으로 재활용
    private val local = InMemoryLock(...)

    override suspend fun <R> withLock(/*...*/) {
        // 1. 먼저 서버 내부에서 '대표 코루틴'을 선출 (Local Lock)
        return local.withLock(...) {
            // 2. 오직 대표 코루틴만이 글로벌 분산 락 경쟁에 참여 (Global Lock)
            global.withLock(...)
        }
    }
}
```

이 이야기는 제가 어떻게 단순한 테스트 도구를 만드는 과정에서 시스템 전체의 안정성을 높이는 새로운 아키텍처의 영감을 얻고, 그것을 직접 설계하고 구현할 수 있는지를 보여주는 가장 대표적인 사례입니다.
이는 특정 기술의 지식을 넘어, 근본 원리에서부터 생각하고 해결책을 창조하는 저의 문제 해결 방식을 증명합니다.

### querydsl 확장

***Problem***

Querydsl은 강력하지만, JPAQueryFactory 주입, Q-Class import, selectFrom 호출 등 모든 쿼리마다 반복되는 보일러플레이트 코드는
개발 생산성을 저해하고 코드 가독성을 떨어뜨리며 무엇보다도 개발을 재미 없게 만든다고 저는 판단했습니다.

***Solution***

Kotlin의 확장 함수와 람다를 활용하여, 개발자가 마치 새로운 언어(DSL)를 쓰듯이 간결하고 타입-세이프하게 Querydsl 쿼리를 작성할 수 있는 `BunQuerydslExtension`을 직접 창조했습니다.

```kotlin
interface UserRepository : JpaRepository<User, Long>, BunQuerydslExtension<User, QUser, Long>

// 사용 사례
userRepository.update {
    set(it.name, "newName")
    where(it.id.eq(1L))
}
```

이 DSL은 불필요한 import를 피하기 위해 콜백 함수의 파라미터로 Q-Class를 주입해주어, 개발자가 오직 비즈니스 조건에만 완벽하게 집중할 수 있는 환경을 제공합니다.
저는 이 기능이 가진 강력함과 안티패턴 양산 가능성을 인지하고, 문서화를 통해 올바른 사용법을 가이드하여 그 위험을 통제하고 있습니다.

### jpa 타입 확장

***Problem***

JPA Entity에서 Enum 타입을 다루거나 민감한 개인정보를 암호화할 때, 반복적이고 실수하기 쉬운 코드가 서비스 로직에 흩어져 있었습니다.
이는 런타임 에러나 심각한 보안 사고로 이어질 수 있는 잠재적 위험이었습니다.

db 상수 값 또한, raw type 으로 엔터티에 맵핑하거나 AttributeConverter 를 제각각 스타일로 등록해서 쓰는등, 규격화가 되어 있지 않았습니다.

***Solution***

개발자가 단지 특정 타입을 필드로 선언하는 것만으로 안정성과 보안이 자동으로 보장되는 시스템을 직접 설계하고 구축했습니다.

#### 상황별 최적화를 제공하는 EnumColumn 시스템

매우 간단한 사용법으로, DB 값과 Enum 사이의 타입-세이프한 변환을 보장받습니다.
저는 여기서 한 걸음 더 나아가, 다양한 상황에 맞는 최적화된 컨버터를 함께 제공했습니다.

```Kotlin
/**
 * 데이터베이스 열 값에 매핑될 수 있는 열거형 타입의 인터페이스 입니다.
 * 구현 클래스는 열거형 항목에 대응하는 [dbValue] 속성의 구체적인 구현을 제공해야 합니다.
 *
 * 이 클래스를 구현하실 경우, 양방향 맵핑을 실행해주는 Converter 구현 및 등록이 필수 입니다. 구현된 Converter 는 다음과 같습니다.
 * - [EnumColumnConverter] : 가장 기본적인 구현. 시간 복잡도는 O(n) 입니다
 * - [MapEnumColumnConverter] : [Map] 기반 구현체 입니다. 시간 복잡도는 O(1) 입니다.
 * - [IndexedEnumColumnConverter] : [IndexedEnumColumn] 을 구현한 경우에만 사용할 수 있습니다. 시간 복잡도는 O(1) 입니다.
 * - [LowerCaseEnumColumnConverter] : [LowerCaseEnumColumn] 을 구현한 경우에만 사용할 수 있습니다. 시간 복잡도는 O(1) 입니다.
 *
 * Converter 의 두번째 파라메터는 db -> enum 변환 실패시 디폴트로 응답할 값입니다.
 * Converter 구현 및 등록은 아무렇게나 하셔도 상관없습니다만 하단과 같은 방법을 추천합니다.
 *
 * enum class Rank(override val dbValue: Int) : com.bunjang.data.database.type.enums.EnumColumn<Int> {
 *   IRON(0), BRONZE(1), SILVER(2), GOLD(3), PLATINUM(4);
 *
 *   @jakarta.persistence.Converter(autoApply = true)
 *   companion object : com.bunjang.data.database.type.enums.EnumColumnConverter<Rank, Int>(entries, fallback value)
 * }
 *
 * // 이럴 경우, 불필요한 클래스가 늘어나는것을 방지하는것과 별개로 하단과 같은 장점이 있습니다.
 * fun main() {
 *   println(Rank.convertToEntityAttribute(3)) // dbValue 로 부터 enum type 변환 함수를 static 함수로 사용 가능
 * }
 */
interface EnumColumn<T> {
    /** db 에 저장된 원본 값 */
    val dbValue: T
}

/**
 * [dbValue] 가 [Int] 이며 배열 인덱스로 매핑이 가능한 경우 사용할 수 있는 인터페이스입니다.
 *
 * 사용 가능한 컨버터: [IndexedEnumColumnConverter]
 *
 * 주의사항:
 * - 초기화 시, 필요한 경우 [List] 를 새로 생성하며, 이때 [dbValue] 의 최대값을 기준으로 [List] 의 크기를 설정합니다.
 * - 자세한 동작 방식은 [IndexedEnumColumnConverter]의 설명을 참고하세요.
 */
interface IndexedEnumColumn : EnumColumn<Int>
```

이 설계는 단순히 문제를 해결하는 것을 넘어, 성능과 편의성이라는 트레이드오프까지 고려하여 개발자에게 최적의 선택지를 제공하는, 시스템의 깊이를 보여주는 사례입니다.

#### 필드 레벨 암호화 SecretString

```kotlin
/**
 * db 암호화 컬럼을 위한 타입입니다. 동등성 비교시 [plain] 을 기준으로 동작합니다.
 *
 * 다음은 사용 예제 입니다.
 * class ExampleEntity {
 *   @Convert(converter = SecretString.DefaultAESConverter::class)
 *   val name: SecretString
 * }
 *
 * fun main() {
 *   ExampleEntity(SecretString("이름"))
 * }
 * 암호화 관련 공통 사항
 * - "defaultAesCrypto" 가 없어도 정상 동작하며, 하단과 같이 이 클래스와 상호작용 할때 에러가 발생하게 됩니다.
 *   - entity select -> 복호화 필드 읽기
 *   - entity insert
 *   - entity update (dirty checking)
 * - "defaultAesCrypto" 는 [BunDefaultAESCryptoConfig] 가 설정되어야 합니다.
 */
sealed interface SecretString {
    val plain: String

    class DefaultAESConverter : SecretStringAESConverter("defaultAesCrypto")
    companion object
}
```

서비스 로직 개발자는 암호화의 존재 자체를 신경 쓸 필요가 없습니다. SecretString 타입을 필드로 선언하는 순간,
플랫폼이 JPA 계층에서 모든 암복호화 과정을 투명하게 처리합니다. 이는 개발자의 실수를 원천 차단하고 조직의 보안 표준을 코드로 강제하는 저의 플랫폼 설계 철학을 명확히 보여줍니다.

## 서비스간 통신 방식 개선: bun-spring-starter-api-client-lib (개발중)

***Problem***

수십개의 서비스간 통신을 위한 조직 공용 라이브러리가 있습니다. 어떤 api 가 필요로 하면 개발자가 api 를 직접 추가하는 형태로 관리 되고 있습니다.
저는 이 구조에 다음과 같은 문제가 있다고 정의했습니다.

- 책임 부재
- 테스트 없음
- 코드와 api 스펙이 동기화 되지 않음
- 단일 라이브러리로 빌드 되기 때문에 수많은 클래스가 프로젝트에 노출되며 네임스페이스 오염

***Solution***

OpenAPI 명세로부터 타입 안전한 API 클라이언트를 자동 생성하는 Gradle 플러그인을 직접 설계하고 개발하여, 서비스 간 통신 오류를 컴파일 타임에 차단하는 시스템을 구축했습니다.

이 라이브러리는 단순히 코드만 생성하는 것이 아니라, 사용하는 개발자의 경험을 극한으로 끌어올리도록 설계되었습니다.

- api 스펙에 http request body 가 있는 경우, 빌더 패턴 자동 생성
- 빌더 패턴 사용시, import 없는 enum 참조 제공
- 응답 데이터에 유효하지 않는 enum 구조 지원

```kotlin
// api call 예제입니다.
client.apicall {
    name = "새로운 상품"
    saleStatus = it.SaleStatus.SELLING
}

// 응답 데이터 구조중 enum 이 있는 경우, EnumProperty 로 wrapping 됩니다.
sealed interface EnumProperty<E : Enum<out E>> {
    val value: String
    val valueOrNull: E?
    val valueOrThrow: E

    fun onFailure(action: (unknownEnumValue: String) -> Unit): EnumProperty<E>
}
```

특히 import 없는 enum 을 구현하기 위해, enum 대신 `value class`를 사용하고 `companion object`와 스코프 함수를 조합하여,
개발자가 마치 원래 언어의 일부인 것처럼 자연스럽게 코드를 작성할 수 있는 새로운 DSL(도메인 특화 언어)을 창조했습니다.

저는 이 라이브러리를 통해 api 제공자는 api-client 도 함께 제공할 의무를 가지도록 강제하고,
통합테스트 작성을 유도하여 api client, api server 두 프로젝트를 동시에 검증하는걸 의도하고 있습니다.

### ApiPayload

***Problem***

함수의 과도한 오버로딩은 다음과 같은 문제가 있다고 정의했습니다.

- 단번에 어떤 함수를 써야 하는지 혼동을 줌
- 주석이 길 경우, 오버로딩 되는 함수 및 주석 파악이 쉽지 않음.

***Solution***

빌더 패턴과 일반 DTO 전달 방식을 하나의 함수 시그니쳐로 통합하기 위해, Kotlin의 `fun interface`와 확장 람다를 활용한 `ApiPayload` 인터페이스를 설계했습니다.

```kotlin
fun interface ApiPayload<REQUEST, CONTEXT> {
    operator fun invoke(init: REQUEST.(CONTEXT) -> Unit)
}
```

하지만 저는 이 설계가 `fun interface`와 리시버를 받는 람다 등 Kotlin의 고급 기능에 대한 깊은 이해를 요구하기 때문에, 어려워 하는 동료 개발자들이 많을 수 있다고 판단했습니다.
또한 저의 사용 사례에선 오버로딩은 3개 이상 넘지 않기 때문에 이건 **오버엔지니어링** 이다. 라는 판단을 하게 되었습니다.
따라서 기술적 순수함보다는 팀 전체의 명확성과 유지보수성을 우선하여, **더 단순하고 명시적인 두 개의 오버로딩된 메서드를 제공하는 현재의 방식을 최종적으로 선택했습니다.**

# 백엔드 E2E 테스트를 위한 여정

저는 위의 모든 플랫폼 작업을 **통합테스트 작성 장벽을 낮추고, 궁극적으로는 자동화된 E2E 회귀테스트를 구현**하기 위해 수행했습니다.
제가 이전에 구축한 통합테스트 프레임워크는 퇴사 후에도 지속적으로 활용되고 있었으며, 재입사 후 **조직장으로부터 "통합테스트 덕분에 개발과 검증이 훨씬 수월해졌다"는 직접적인 피드백**을 받을 수 있었습니다.
현재는 이 경험을 바탕으로 통합테스트 프레임워크를 한층 더 발전시켜 팀 전체에 확산하고 있습니다. 다음은 현재 사용 중인 테스트 코드의 일부입니다.

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
abstract class IntegrationTest {

    @Autowired
    lateinit var userApi: UserApi

    @LocalServerPort
    var port: Int = 0

    val client by lazy { ApiClient("http://localhost:$port") }

    fun test(
        token: UserToken? = null,
        test: suspend Context.() -> Unit
    ) = runBlocking {
        val token = token ?: userApi.signUpRandom()
        Context(token, client.withUser(token), coroutineContext).test()
    }

    fun UserApi.signUpRandom(
        name: String = "user-${Random.nextLong(1, Int.MAX_VALUE.toLong())}",
    ): UserToken = this.signUp(SignUpRequest(name = name))

    class Context(
        val user: UserToken,
        val api: ApiClient,
        override val coroutineContext: CoroutineContext,
    ) : CoroutineScope
}

class AdRewardResourceTest : IntegrationTest() {
    @Test
    fun `광고 시청 및 보상 이력을 검색할 수 있다`() = test {
        api.callApi()
    }
}
```

이 테스트 프레임워크를 통해 각 API 서버는 **자신의 비즈니스 로직을 검증하는 동시에 자동 생성된 API 클라이언트의 신뢰성까지 함께 보장**할 수 있습니다.
더 나아가, **설정만으로 테스트 더블 사용 여부를 결정**할 수 있도록 구현 예정입니다. 목표는 설정 변경만으로 통합테스트를 E2E 테스트로 전환하기 위함입니다.
**이러한 설정 기반 테스트 환경 전환이야말로 궁극의 E2E 테스트를 향한 핵심 기반**이라고 생각합니다.

# Impact & Vision

이 플랫폼은 현재 회사의 일부 팀에 적용되어 개발 생산성을 높이고 있으며, 조직의 공식적인 기술 자산으로 인정받았습니다.
`event-publisher/subscriber` 와 같은 초기 모듈은 제가 필요해서 만들었지만, 제가 퇴사한 이후엔 거의 모든 팀이 사용하면서 수년간 조직의 핵심 인프라로 사용될 만큼 그 가치를 증명했습니다.

저의 최종 목표는 이 플랫폼을 더욱 발전시켜, devops 와 개발자간의 소통 비용을 줄이고, 개발 속도 증가 및 버그 하락, 신뢰성 있는 자동화 E2E 회귀 테스트를 가능하게 하는 것입니다.
각 서비스가 자체 통합 테스트를 통해 api와 api client 를 원자적으로 증명하고, 통합 테스트들이 모여 설정 변경만으로 E2E 테스트로 기동 하여 전체 시스템의 안정성을 보장하는 것.
그것이 제가 그리는 기술적 비전입니다.
