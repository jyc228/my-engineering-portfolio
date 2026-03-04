# 전사 공용 라이브러리 운영 및 마이그레이션 전략

## 프로젝트 개요

- 기간: 2025.6 - 2025.10 (약 4개월)
- 인원: 백엔드 1명
- 핵심 기술: Kotlin, Spring Boo

퇴사전에 `CTO`의 지시로 전사 모든 마이크로서비스 간 통신을 담당하는 단일 레포지토리 프로젝트(`client-kotlin`)의 초기 설계 및 전담 유지보수를 수행했었습니다.

재입사 후, 해당 구조가 3년 동안 유지되며 기술 부채가 누적된 것을 확인하고, 플랫폼의 유연성과 유지보수성 향상을 위한 전면 개편안을 제안 및 주도했습니다.

## 주요 해결 과제

### 핵심 데이터 모듈 별도 라이브러리로 분리 및 프로젝트 구조 개선

프로젝트는 서비스간 통신을 위한 코드 뿐만 아니라, 인증 인가 시스템을 위한 코드, 스프링 자동 구성 설정 코드까지 있었습니다.

최초 개발 당시, 서로 다른 도메인 코드로 인하여 복잡도가 너무 크다고 판단하여 3개의 서브 모듈 프로젝트로 관리했으나 다음과 같은 문제가 발생했다고 정의했습니다.

- 동료 개발자들의 서브 모듈 프로젝트의 라이브러리 릴리즈에 대한 이해 부족으로 버저닝을 제대로 하지 않음
- 멀티 모듈 프로젝트 구조로 인하여 빌드 스크립트 복잡도와 빌드 시간 증가

이 문제를 해결하기 위하여, 핵심 데이터 모듈은 별도 프로젝트로 분리하고, 프로젝트 구조를 단일 모듈 프로젝트로 개선했습니다.

```
client-kotlin
   |- client-spring-starter (클라이언트들의 스프링 자동구성 설정 프로젝트) -> client-core 와 병합
   |- client-core           (클라이언트 구현) -> 제거
   |- common-model          (인증 인가 시스템 등, 스프링과, 특정 서비스에 종속적이지 않은 코드 모음) -> 별도 프로젝트 분리 (core-kotlin)
```

### 하위호환이 안되는 변경 관리

개선안엔 하위호환이 안되는 변경 내역이 있었으며, 해당 프로젝트는 조직의 모든 백엔드 프로젝트가 사용하기 때문에, 반드시 마주치게 되는 에러였습니다.

- 패키지 이름 변경
- 클래스 이름 변경
- deprecated 기능 삭제
- 스프링 자동 구성 설정 방식 변경

동료 개발자들의 업무 피로도를 낮추기 위하여, 변경 사항을 나누고 일정 간격으로 배포했습니다.

또한 배포 마다 바뀐 내역, 미조치시 예상되는 결과, 마이그레이션 가이드를 제공하였습니다.

하단은 당시에 했던 슬랙 공지를 적당히 압축했습니다. 또한 해당 변경 내역을 별도 릴리즈 파일로 프로젝트에 포함시켰습니다.

> 안녕하세요. client-kotlin 의 예정된 변경사항들을 알려드립니다. 변경 사항들은 나눠져서 배포 됩니다.
>
> * 첫번째 배포 - 대상이 되는 서버 : 적음 (오래된 서버 담당자 확인 필요), 미조치 영향 : 컴파일 실패
> * 두번째 배포 - 대상이 되는 서버 : 없음, 미조치 영향 : 없음
> * 세번째 배포 - 대상이 되는 서버 : 전체, 미조치 영향 : 컴파일 실패
> * 네번째 배포 - 대상이 되는 서버 : 전체, 미조치 영향 : 서버 실행 실패 (4.b 참고)

### 자동 마이그레이션 도구 제공

`common-model` 서브 프로젝트를 별도 프로젝트로 분리하면서, 패키지 변경 등 하위호환이 안되는 대규모 코드 변경이 발생했습니다.

동료들의 원할한 마이그레이션을 위하여, 마이그레이션 스크립트를 직접 작성 및 배포 했습니다.

```kotlin
// root directory 에 main.kts 로 만들어서 넣으시고 실행
Files.walkFileTree(Paths.get(System.getProperty("user.dir")), fileVisitor {
    onPreVisitDirectory { f, attr ->
        if (f.name.startsWith(".") || f.name in setOf("resources", "build")) FileVisitResult.SKIP_SUBTREE
        else FileVisitResult.CONTINUE
    }
    onVisitFile { f, attr ->
        if (f.name.endsWith(".kt")) migration(f)
        FileVisitResult.CONTINUE
    }
})
fun migration(path: Path) {
    val code = StringBuilder(path.readText())
    val updated = listOf(
        code.ifExist("import com.bunjang.client.auth.model.BanReason") { i, l ->
            replace(i, i + l, "import com.bunjang.core.auth.PermissionBan")
            replaceAll("BanReason", "PermissionBan")
        }
    )
    // 이하 생략..
}
```

### 전이적 의존성 버그 해결

프로젝트 이름 변경후 몇몇 프로젝트에서 prod 빌드 실패가 관측되어, 원인을 분석하고 명쾌한 해결책을 제시하여 혼선을 최소화 했습니다.

하단은 그때 당시 했던 공지내용 입니다.

> @channel [안내] `client-kotlin` 5.14+ 버전 사용 시 `admin-logger` 업데이트 필요
>
> `admin-logger` 를 사용하고 계신 경우, `client-kotlin` 을 5.14 이상 사용하시게 될 경우 `prod` 빌드 실패합니다.
> 원인은 디펜던시를 처리하지 못했기 때문이고 해결책은 둘 중 하나를 선택하시면 됩니다.
>
> client-common-model 을 직접 지정한다. 마지막으로 성공한 빌드에서 사용한 버전을 사용하는것을 권장 (임시해결책)
> `ex) implementation("client-common-model:5.0")`
>
> `admin-logger` 2.1.1 로 올린다.
>
> 하단은 빌드 실패 분석 내용입니다. 감사합니다.
> ```
> client-kotlin 에서 common-module 이름 변경 패치 및 버전 초기화를 진행함.
> admin-logger 사용중인 프로젝에서 prod 빌드 실패하며 원인은 common-module 이 prod 빌드에서도 스냅샷 버전으로 지정되어 있어 다운로드 받지 못했기 때문임.
>
> 5.14 이전
> client-kotlin:5.13 -> client-common-model:5.0
> admin-logger:2.1 -> client-common-model:4.0-SNAPSHOT (5.0 으로 대체 됩니다.)
>
> 5.14 이후
> client-kotlin:5.14 -> core-kotlin:0.1
> admin-logger:2.1 -> client-common-model:4.0-SNAPSHOT (모듈 충돌이 없으므로 2.1 에 지정된 버전 다운로드, prod 빌드시 snapshot
> repository 없어서 빌드 실패)
> ```

## 결과

- 라이브러리 릴리즈시 잘못된 버전 지정 으로 인한 버그 가능성 차단
- 빌드 파이프라인의 복잡도 개선 및 빌드 속도 최적화
- 서버 url 설정 코드 일원화

와 같은 효과가 발생하였습니다. 또한 마이그레이션에 대하여 저에게 직접적으로 인인됩 문의는 약 3건에 불과했습니다.

또한 이 프로젝트는 [platform-api-client.md](platform-api-client.md) 를 수행하게 되는 계기가 되었습니다.