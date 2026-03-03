# Troubleshooting - QueryDSL 프레임워크 버그 분석 및 전사 핫픽스

## 프로젝트 개요

- 기간: 2025.12 중 오전 약 4시간
- 인원: 1명
- 핵심 기술: Kotlin, JPA, QueryDSL

## 배경

전사 표준 `QueryDSL` 라이브러리를 `OpenFeign` 포크 버전(6.12+)으로 업그레이드한 후, 특정 쿼리에서 런타임 에러가 발생했습니다.
동료들은 이 문제를 2주이상 해결하지 못하고 있었으며, 업그레이드를 일시정지 하거나, `JPA` 계층의 핵심 스펙 수정을 검토 중이었습니다.

## 문제 정의

`CaseBuilder` 에서 `AttributeConverter` 가 구현된 `Enum` 을 사용할 때, `Enum.name` 만 렌더링 되고 있었습니다.

## 분석 및 해결

### 버그 발생을 로컬에서 재현

- 소요시간: 약 2시간

정확한 문제 해결을 위하여, 로컬에서 그대로 버그가 재현되는 환경 구축을 먼저 수행했습니다.

### 6.12 전후 차이 파악

- 소요시간: 약 1시간

위 버그가 `JPA` 버그인지, `QueryDSL` 버그인지 명확히 할 필요가 있었습니다.
디버거를 통해 QueryDSL이 SQL을 렌더링하는 콜 스택을 역추적하여, 다음과 같은 사실을 파악했습니다

- `@Query` 어노테이션으로 작성된 쿼리문 문제 없음.
- `QueryDSL 6.12` 이전에는 `fqcn` 출력됨. `Enum` 의 전체 경로가 출력되어, `jpql` 로 변환후, `AttributeConverter` 적용되고 있었습니다.
- `QueryDSL 6.12` 부터 `Enum.name` 만 출력되었습니다. db type 이 문자열인 경우, 런타임에 에러가 발생하지 않아, 테스트코드가 없었더라면 문제 인식도 못하는 상태였습니다.

### 적합한 해결책 선정

- 소요시간: 약 30분

`QueryDSL` 버그이므로, `JPA` 의 스펙인 `AttributeConverter` 를 수정하는건, 또 다른 예측할수 없는 에러를 만드는 행위라고 생각했습니다.
그러므로 `QueryDSL` 의 구현을 변경하는 방식으로 오픈소스 라이브러리의 버그를 해결했습니다.

`EnumColumn` 에 대한 자세한 설명은 [EnumColumn](platform-data.md#상황별-최적화를-제공하는-enumcolumn-시스템) 를 참고해 주세요.

```Kotlin
// JPAQuery 생성 시 커스텀 템플릿 주입
JPAQuery(entityManager, object : JPQLTemplates() {
    override fun asLiteral(constant: Any): String? {
        if (constant is EnumColumn<*>) { // EnumColumn 은 AttributeConverter 를 통해 변환되는 타입입니다. 
            return if (constant.value is String) "'${constant.value}'"
            else constant.value?.toString()
        }
        return super.asLiteral(constant)
    }
})
```

## 결과

이후 슬랙에 공지를 올려 마무리 했습니다.

```
Querydsl Case When 임시 해결책 안내
AttributeConverter 가 적용된 Enum 클래스 인 경우, querydsl 로 case when ~ otherwise end 구문 사용시 다음과 같이 적용됩니다.

6.12 이전 -> enum class 의 fqcn(패키지 + 클래스 이름) 출력, 쿼리 실행됨,
6.12 이후 -> enum class 이름만 출력, 에러 발생
querydsl 이 jpa 스펙인 AttributeConverter 를 제대로 처리하지 않아서 발생한 문제입니다.
하단은 임시 해결책으로, 타입이 EnumColumn 인 경우, 최우선으로 dbValue 를 사용하도록 강제합니다.

JPAQuery(entityManager, object : JPQLTemplates() {
    override fun asLiteral(constant: Any): String? {
        if (constant is EnumColumn<*>) {
            if (constant.value is String) return "'${constant.value}'"
            return constant.value?.toString()
        }
        return super.asLiteral(constant)
    }
})
jd 가 올리신 해결책 : String -> String 변환인 경우에만 동작합니다. (AttributeConverter<String, String>)
AttributeConverter 하위 구현체 변경 : querydsl 의 버그이므로 jpa 와 연관된 스펙 변경은 추천하지 않습니다.
```

저 코드 블록을 올린 뒤 관련된 스레드는 더이상 대화가 없어졌으며, 친한 동료 개발자에겐 잘 동작한다고 피드백 받았습니다.
