<!-- TOC -->
* [페이징 모델 재정의](#페이징-모델-재정의)
* [조직 표준 응답 데이터 모델 구조화](#조직-표준-응답-데이터-모델-구조화)
<!-- TOC -->

# 페이징 모델 재정의

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

# 조직 표준 응답 데이터 모델 구조화

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
