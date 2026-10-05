# 수행한 모든 프로젝트 목록

시간대별로 기록했습니다.

## 번개장터

### [금지어 서비스 고도화](bunjang/keyword-policy.md) | 2025년 12월 (2개월)

- **핵심 성과**: 금지어 탐지 성능 2배 이상 향상
- **키워드**: `Kotlin`, `최적화`, `관측성 확보`

### [전사 라이브러리 파괴적 변경 수행](bunjang/mas-client-governance.md) | 2025년 6월 (4개월)

- **핵심 성과**: 전사 공용 라이브러리의 하위호환이 깨지는 변경을 단계적으로 배포, 직접 문의 약 3건으로 마무리
- **키워드**: `Kotlin`, `라이브러리`, `마이그레이션`

### [신규 복권 서비스 백엔드 리딩](bunjang/lottery.md) | 2025년 10월 (2개월)

- **핵심 성과**: 고트래픽 상황에서 초과 발행·중복 당첨이 발생하지 않는 구조 설계
- **키워드**: `Kotlin`, `팀 리드`

### [외부 광고 개선 백엔드 리딩](bunjang/advertisement.md) | 2025년 4월 (2개월)

- **핵심 성과**: 광고 미송출 비율 20% → 15%, 일 추가 수익 약 50만원
- **키워드**: `Kotlin`, `팀 리드`

### [백엔드 공통 라이브러리 개발](bunjang/platform.md) | 2025년 4월 (6개월)

- **핵심 성과**: 설정 보일러플레이트 90% 이상 감소, 신규 프로젝트 설정 시간 단축
- **키워드**: `Kotlin`, `인지부하 감소`

## 라이트스케일

### [DApp 서비스 백엔드 리딩](lightscale/dapp.md) | 2024년 10월 (4개월)

- **핵심 성과**: 인력 이탈 상황에서 백엔드 리드 역할을 맡아 QA 기간 백엔드 버그 5개 이하로 런칭
- **키워드**: `Kotlin`, `팀 리드`

### [P2P 데이터 동기화 구현](lightscale/snapsync.md) | 2024년 4월 (6개월)

- **핵심 성과**: 신규 노드 시작 시간 2주 이상에서 2일 내로 단축
- **키워드**: `golang`, `대량 데이터 동기화`

### [블록체인 코어 트리 최적화](lightscale/zktrie.md) | 2023년 10월 (4개월)

- **핵심 성과**: 처리량 300% 향상, 메인넷 성능 저하 장애 원인 해결
- **키워드**: `golang`, `최적화`

### [블록체인 서비스 유지보수](lightscale/maintenance.md) | 2023년 5월 (17개월)

- **핵심 성과**: zktrie 성능 문제 사전 제기 및 장애 대응, go-ethereum upstream 반영 2회
- **키워드**: `golang`, `오픈소스 업스트림`

## 개인 프로젝트

### [Kotlin Ethereum rpc client](personal/keth-client.md)

- **핵심 성과**: Batch Request DSL과 ABI 기반 코드 생성기를 갖춘 Kotlin Ethereum SDK
- **키워드**: `Kotlin`, `라이브러리`

### [kotlin Ethereum](personal/keth.md)

- **핵심 성과**: go-ethereum의 trie, StateDB, EVM을 Kotlin으로 재구현하며 내부 동작 학습
- **키워드**: `Kotlin`, `가상머신`

### [IntelliJ Ethereum Plugin](personal/intellij-ethereum-plugin.md)

- **핵심 성과**: IntelliJ VirtualFileSystem(VFS)을 확장한 블록체인 데이터 탐색기 (디버거는 미완성)
- **키워드**: `Kotlin`, `Plugin`