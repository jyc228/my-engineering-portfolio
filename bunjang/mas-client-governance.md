# 전사 공용 라이브러리 구조 개편 및 마이그레이션

## 프로젝트 개요

- 기간: 2025.6 - 2025.10 (약 4개월)
- 인원: 백엔드 1명
- 핵심 기술: Kotlin, Spring Boot

## 한줄 요약

전사 서비스가 의존하는 라이브러리에 하위호환이 깨지는 변경을 하면서, 단계별 배포와 마이그레이션 스크립트를 제공해 직접 문의 약 3건으로 마무리

## 배경 & 도전

첫 재직 당시 팀장(현 `CTO`)이 마이크로서비스 간 통신 코드를 단일 저장소에서 관리하는 라이브러리(`client-kotlin`)를 요청했고, Spring 자동 구성 등 내부 구조의 설계와 구현, 유지보수를 제가 맡았었습니다.

재입사 후 보니 그 구조가 3년간 유지되면서 문제가 쌓여 있었고, 구조 개편을 제안해서 진행했습니다.

## 주요 해결 과제

### 핵심 데이터 모듈 별도 라이브러리로 분리 및 프로젝트 구조 개선

프로젝트는 서비스간 통신을 위한 코드 뿐만 아니라, 인증 인가 시스템을 위한 코드, 스프링 자동 구성 설정 코드까지 있었습니다.

처음 만들 때는 성격이 다른 코드가 섞여 있어서 3개의 서브 모듈로 나눴는데, 운영해 보니 다음과 같은 문제가 있었습니다.

- 서브 모듈별 릴리즈 방식이 익숙하지 않아 버전이 제대로 관리되지 않음
- 멀티 모듈 프로젝트 구조로 인하여 빌드 스크립트 복잡도와 빌드 시간 증가

이 문제를 해결하기 위하여, 핵심 데이터 모듈은 별도 프로젝트로 분리하고, 프로젝트 구조를 단일 모듈 프로젝트로 개선했습니다.

```
client-kotlin
   |- client-spring-starter (클라이언트들의 스프링 자동구성 설정 프로젝트) -> client-core 와 병합
   |- client-core           (클라이언트 구현) -> 제거
   |- common-model          (인증 인가 시스템 등, 스프링과, 특정 서비스에 종속적이지 않은 코드 모음) -> 별도 프로젝트 분리 (core-kotlin)
```

### 하위호환이 안되는 변경 관리

개선안에는 하위호환이 안 되는 변경이 포함되어 있었고, 조직의 모든 백엔드 프로젝트가 이 라이브러리를 쓰기 때문에 모두가 한 번씩은 겪게 되는 변경이었습니다.

- 패키지 이름 변경
- 클래스 이름 변경
- deprecated 기능 삭제
- 스프링 자동 구성 설정 방식 변경

동료들이 한 번에 몰아서 대응하지 않도록 변경 사항을 나눠서 일정 간격으로 배포했습니다.

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

패키지명 수정 같은 단순 반복 작업은 스크립트로 처리할 수 있도록 마이그레이션 스크립트를 작성해서 배포했습니다.

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

프로젝트 이름 변경 후 일부 프로젝트에서 prod 빌드가 실패했습니다. 원인을 분석해서 해결 방법과 함께 공지했습니다.

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
- 빌드 스크립트 단순화 및 빌드 시간 단축
- 서버 url 설정 코드 일원화

마이그레이션 관련해서 저에게 직접 들어온 문의는 약 3건이었습니다.

또한 이 프로젝트는 [platform-api-client.md](platform-api-client.md) 를 수행하게 되는 계기가 되었습니다.