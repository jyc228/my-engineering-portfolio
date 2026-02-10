<!-- TOC -->
* [배치잡 정의 방식 개선](#배치잡-정의-방식-개선)
* [airflow 연동 - 작업 실행 구간 정의](#airflow-연동---작업-실행-구간-정의)
* [airflow 연동 - 작업 결과 기록](#airflow-연동---작업-결과-기록)
<!-- TOC -->

# 배치잡 정의 방식 개선

***Problem***

기존은 인터페이스 상속 방식으로 배치잡을 정의하고, 통합된 라이브러리도 없어서 프로젝트마다 미세하게 다르며,
보통 복사 붙여넣기 해서 쓰기 때문에 테스트 부재문제가 있다고 정의했습니다.

***Solution***

스프링의 `RestController`, `RequestMapping` 방식을 본따 `JobContainer`, `Job` 인터페이스를 정의했습니다.

```kotlin
/**
 * # 배치 작업([Job]) 함수들을 포함하는 클래스임을 나타내는 마커 어노테이션입니다.
 *
 * 이 어노테이션은 [org.springframework.stereotype.Component]를 메타 어노테이션으로 포함하고 있으므로, 어노테이션이 적용된 클래스는 자동으로 스프링 빈으로 등록됩니다.
 *
 * 시스템은 이 어노테이션이 부착된 빈(Bean) 내부에서만 [Job] 어노테이션이 달린 함수를 스캔합니다.
 * 따라서 배치 작업을 정의하는 클래스에는 반드시 이 어노테이션을 붙여야 합니다.
 *
 * @see Job
 */
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@org.springframework.stereotype.Component
annotation class JobContainer

/**
 * # 배치 작업을 정의합니다.
 *
 * Command Line 으로 실행시 job 파라미터로 지정된 텍스트와 매칭되는 경우, 해당하는 함수를 실행합니다.
 * 추가로 하단 타입을 함수 파라미터로 지정할 수 있습니다.
 * - [java.time.LocalDateTime] : 잡 실행 시간 (KST)
 * - [org.springframework.boot.ApplicationArguments] : Command Line 파라미터
 * - [Job] : 실행중인 함수에 적용된 어노테이션
 * - [JobInterval] : 오케스트레이터로 부터 주입받은 데이터, 해당 클래스 주석 참고
 * - [JobReporter] : 개발자가 원하는 텍스트를 오케스트레이터로 보고하고자 할 때 사용합니다. 해당 클래스 주석 참고
 *
 * 어노테이션이 적용된 함수마다 `DAG`, `Airflow` URL 주석을 작성하시는걸 추천합니다. 하단은 추천 포맷입니다.
 * 
 * @param name 잡 이름, Unique 해야 하지만, 중복된 이름이 있더라도 시스템은 실패하지 않습니다.
 */
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Job(val name: String)
```

# airflow 연동 - 작업 실행 구간 정의

***Problem***

연속된 작업 구간을 정확하게 스케쥴링 하기 위하여, 각 프로젝트마다 (혹은 팀 단위로) 독자적으로 db 를 정의하고, 사용하고 있습니다.

***Solution***

airflow 의 개념을 받아들여, 작업 구간을 별도 디펜던시 없이 사용할 수 있도록 지원합니다.

```kotlin

/**
 * # 배치 실행 구간 정보
 *
 * `Airflow`의 `Data Interval` 시점을 주입받아 데이터 처리 범위([start], [end])를 정의합니다.
 * 별도의 DB 상태 관리 없이, 오케스트레이터가 제공하는 시간만을 신뢰하여 멱등성을 보장합니다.
 *
 * **권장 처리 범위: [start] <= target < [end]**
 *
 * ## 주입 예시
 * - **Cron**: `0-59/30 * * * *` (매시 0분, 30분 실행)
 * - **실행 시각**: 14:30 (Data Interval End 시점, airflow 에선 `run id` 와 동일)
 * - **주입 결과**: `start: 14:00`, `end: 14:30`
 *
 * @param start 처리 범위의 시작 (Inclusive). 오케스트레이터가 할당한 논리적 구간의 시작 시점입니다.
 * @param end 처리 범위의 끝 (Exclusive). 이 시점 `미만`까지의 데이터를 처리하는 것을 권장합니다.
 * @param period 실행 구간, [start] inclusive, [end] exclusive
 */
data class JobInterval(
    val start: java.time.OffsetDateTime,
    val end: java.time.OffsetDateTime
) {
    val period = start..<end
    fun period(offset: java.time.ZoneOffset) = LocalDateTimeRange(start, end, offset)
}
```

# airflow 연동 - 작업 결과 기록

***Problem***

작업 기록을 요약해서 보는 페이지가 없어서 슬랙 채널로 보고 하고 있었습니다. 이는 접근성이 떨어지고, 과거 실행 결과 파악도 어렵다 라고 정의했습니다.

***Solution***

airflow 의 note 에 데이터를 기록할 수 있는 기능을 제공했습니다.

```kotlin

/**
 * # Job 결과 보고
 *
 * 이 기능을 사용하시려면 airflow api 를 사용할 수 있어야 합니다. devops 에 요청해 주세요.
 * 테스트 등 기타 이유로 객체 생성을 직접 해야 한다면 `JobReporter()` 함수를 사용해 주세요.
 * 
 * 잡 종료시 [flush] 를 호출하며, [flush] 가 실패해도 잡은 실패로 기록되지 않습니다.
 */
interface JobReporter {
    /**
     * 기존 기록을 보존하며 새로운 내용을 순차적으로 추가합니다.
     *
     * 너무 많은 내용을 작성하실 경우, 에러가 발생하진 않고 뒤에 내용이 삭제됩니다.
     */
    fun report(text: String)
    fun clear()

    /** 결과를 airflow 로 내보냅니다. job 종료시 자동으로 호출됩니다. */
    fun flush()

    class FlushFailedException(val reported: List<String>, message: String, e: Exception) : RuntimeException(message, e)
}
```