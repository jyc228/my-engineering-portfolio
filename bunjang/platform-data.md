<!-- TOC -->
* [데이터 저장소 일관된 설정 구조 제공](#데이터-저장소-일관된-설정-구조-제공)
* [조직 표준 분산락 구현](#조직-표준-분산락-구현)
  * [CohortDistributedLock](#cohortdistributedlock)
* [querydsl 확장](#querydsl-확장)
* [jpa 타입 확장](#jpa-타입-확장)
  * [상황별 최적화를 제공하는 EnumColumn 시스템](#상황별-최적화를-제공하는-enumcolumn-시스템)
  * [필드 레벨 암호화 SecretString](#필드-레벨-암호화-secretstring)
* [동일한 요소를 가지는 Enum 변환](#동일한-요소를-가지는-enum-변환)
<!-- TOC -->

# 데이터 저장소 일관된 설정 구조 제공

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

# 조직 표준 분산락 구현

***Problem***

분산 락을 사용하는 로직은 테스트하기가 매우 어렵습니다. 실제 Redis와 같은 외부 의존성에 기대어 통합 테스트를 작성하는 것은 느리고 불안정하며,
이는 결국 테스트 코드 작성 자체를 기피하게 만들어 코드의 신뢰도를 떨어뜨리는 원인이 되었습니다.

***Solution***

이 문제를 해결하기 위해, 먼저 실제 분산 락처럼 동작하지만 메모리 내에서만 비동기적으로 작동하는 정교한 테스트 더블(Test Double)인 `InMemoryLock`을 직접 구현했습니다.
또한 서버 실행시 실행 환경, 런타임 클래스 여부 등을 판단하여 적합한 분산락 인스턴스를 (spin, pub/sub, inmemory) 자동으로 선택하도록 구성했습니다.

이 단순한 `InMemoryLock`을 만들고 보니, 이것을 활용하여 분산 시스템의 더 근본적인 문제인 **Thundering Herd**를 해결할 수 있다는 통찰을 얻게 되었습니다.

## CohortDistributedLock

제가 만든 `InMemoryLock` 을 **로컬 락**으로 활용하여 이 문제를 해결했습니다.
먼저 각 서버 내부에서 로컬 락으로 **대표**를 단 하나만 선출하고, 오직 이 소수의 대표들만이 실제 **글로벌 분산 락** 경쟁에 참여하도록 설계했습니다.
만들고 나니 이게 `락 코호팅`이라는걸 알게 되었고 클래스 이름을 `CohortDistributedLock` 라고 부여하게 되었습니다.

```Kotlin
class CohortDistributedLock(
    private val global: DistributedLock, // 실제 분산 락 (예: Redis)
) : DistributedLock {
    // 테스트를 위해 만들었던 InMemoryLock을 '로컬 락'으로 재활용
    private val local = InMemoryLock()

    override suspend fun <R> withLock(key: String, action: suspend () -> R) {
        // 1. 먼저 서버 내부에서 '대표 코루틴'을 선출 (Local Lock)
        return local.withLock(key) {
            // 2. 오직 대표 코루틴만이 글로벌 분산 락 경쟁에 참여 (Global Lock)
            global.withLock(key, action)
        }
    }
}
```

이 이야기는 제가 어떻게 단순한 테스트 도구를 만드는 과정에서 시스템 전체의 안정성을 높이는 새로운 아키텍처의 영감을 얻고, 그것을 직접 설계하고 구현할 수 있는지를 보여주는 가장 대표적인 사례입니다.
이는 특정 기술의 지식을 넘어, 근본 원리에서부터 생각하고 해결책을 창조하는 저의 문제 해결 방식을 증명합니다.

# querydsl 확장

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

# jpa 타입 확장

***Problem***

JPA Entity에서 Enum 타입을 다루거나 민감한 개인정보를 암호화할 때, 반복적이고 실수하기 쉬운 코드가 서비스 로직에 흩어져 있었습니다.
이는 런타임 에러나 심각한 보안 사고로 이어질 수 있는 잠재적 위험이었습니다.

db 상수 값 또한, raw type 으로 엔터티에 맵핑하거나 AttributeConverter 를 제각각 스타일로 등록해서 쓰는등, 규격화가 되어 있지 않았습니다.

***Solution***

개발자가 단지 특정 타입을 필드로 선언하는 것만으로 안정성과 보안이 자동으로 보장되는 시스템을 직접 설계하고 구축했습니다.

## 상황별 최적화를 제공하는 EnumColumn 시스템

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

## 필드 레벨 암호화 SecretString

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

# 동일한 요소를 가지는 Enum 변환

***Problem***

계층간 `Enum` 을 분리하게 될 경우, 필연적으로 비슷한 종류의 `Enum` 클래스가 여러개 생겨나게 됩니다.

제가 지금까지 봐온 대부분의 코드들은 나눠야 한다라는 생각을 아얘 안하는 경우였습니다. `JsonIgnore` 같은 메타 어노테이션이 적용되어 있으며,
프로젝트 전역적으로 사용되고 있기 때문에 사용처, 용도, 스펙화 등 유지보수를 매우 어렵게 만들고 있다고 생각해왔습니다.

또한 계층별 `Enum` 을 추가하더라도, `when else` 를 쓰는 코드들이 있거나 (컴파일 타임 안정성 포기), `exhaustive when` 으로 풀 전개 하는 경우 였습니다.

`exhaustive when` 은 컴파일 타임 안정성은 훌륭하고, 코드가 적을경우는 괜찮다고 생각하지만 코드가 많아질수록 유지보수를 어렵게 하고,
테스트가 생략되는 경우가 자주 있으며, 개발자가 수동으로 직접 연결하기 때문에 실수할 가능성이 있다. 라는 사실은 변함이 없었습니다.

***Solution***

`Enum` 간 변환을 쉽게 등록하고 관리할 수 있는 추상 클래스를 제공합니다. 이 클래스를 확장하여 원하는 변환 규칙을 쉽게 등록하고, 편하게 사용할 수 있습니다.
또한 `Bean` 으로 등록시, `Spring Startup` 시점에 `Kotlin Reflection`을 활용하여 모든 매핑 함수의 정합성(부분집합 여부)을 전수 검사합니다.
이를 통해 휴먼 에러로 인한 런타임 익셉션을 배포 전 단계에서 원천 차단합니다.

```kotlin
/**
 * 동일한 요소를 가지는 Enum 클래스간 변환을 도와주는 유틸성 클래스 입니다.
 *
 * 이 기능은 계층간 Enum 클래스를 독립적으로 두게 되면 필연적으로 발생하는 동일한 요소를 가지는 Enum을 처리하기 위해 구현되었습니다.
 *
 * 이 클래스를 object class 에서 상속 받은 후, 변환 함수를 등록하시면 됩니다.
 *
 * 상속 받은 클래스를 spring bean 으로 등록시킬 경우, startup 시 유효성 검사를 합니다.
 *
 * ## 제약 조건
 * - 함수 이름 : enumOf, convert 만 가능
 * - 한 클래스에 동일한 input 을 받는 변환 함수 등록 불가능
 * - A -> B 로 변환 시, 원칙적으로 A 는 B 의 부분집합이어야 합니다. (A ⊆ B)
 * - 단, A 에는 존재하나 B 에는 없는 필드의 경우, 명시적으로 처리를 해야 합니다. 이 경우 결과 타입은 nullable이 됩니다.
 */
abstract class AbstractEnumMapping {
    fun throwIfInvalid() {
        // ... 구현
    }
}

// 예제.. 원본 코드에선 AbstractEnumMapping 주석에 이 코드가 포함되어 있습니다.
object EnumMapping : AbstractEnumMapping() {
    fun EnumA.convert() = this<EnumB>()
    fun enumOf(enum: EnumB) = enum<EnumA>()
    fun enumOf(enum: EnumB) = enum<EnumA>("UNKNOWN") // 지원 안하는 enum 필드 명시
}

fun test(enum: EnumA): EnumB {
    return enum.convert()
}
```