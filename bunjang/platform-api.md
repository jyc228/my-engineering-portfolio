<!-- TOC -->
* [페이징 모델 재정의](#페이징-모델-재정의)
* [조직 표준 응답 데이터 모델 구조화](#조직-표준-응답-데이터-모델-구조화)
<!-- TOC -->

# 페이징 모델 재정의

***Problem***

기존에 쓰던 Spring Pageable은 데이터 계층의 모델이 API 계층까지 노출되고, 정렬이 필요 없는 API에도 sort 파라미터가 노출되는 문제가 있었습니다.

***Solution***

cursor 페이징 지원, sort 파라미터 분리, 타입 안전한 sort 파라미터를 지원하는 `ApiPageParameter`를 만들었습니다.

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

정렬이 필요 없는 메서드에 정렬 파라미터가 넘어가는 실수는 컴파일 단계에서 잡힙니다.
`ApiOffsetWithTypeSortParameter<E: Enum<E>>`는 Enum 타입 파라미터를 이용해서 Spring 파라미터 리졸버에서 sort 값을 검증하고,
springdoc 문서에 허용 값을 표시하며, 다른 객체로 변환할 때 sort key를 Enum으로 넘겨줍니다.

# 조직 표준 응답 데이터 모델 구조화

***Problem***

사내 문서에 표준 응답 스펙은 정의되어 있었지만 이를 위한 라이브러리는 없었습니다. 그래서 프로젝트마다 응답 모델을 만드는 방법이 제각각이었고,
가장 자주 보는 코드인데도 프로젝트를 옮길 때마다 다시 파악해야 했습니다.

***Solution***

표준 응답 모델과 이를 만드는 확장 함수를 제공합니다. 리시버 타입에 따라 알맞은 응답 구조로 변환됩니다.

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

모든 함수 이름은 `toResult`로 통일했습니다. 프로젝트마다 응답 모델을 따로 만들 필요가 없어졌고, 페이지 정보를 잘못 채우는 것 같은 실수도 줄일 수 있었습니다.
