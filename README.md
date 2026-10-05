# 장영철 | 백엔드 엔지니어

Kotlin/Spring 기반 백엔드 개발자입니다. 커머스 서비스 백엔드와 사내 공통 라이브러리 개발을 주로 했고, 블록체인 코어(go-ethereum) 유지보수 경험도 있습니다.
라이브러리나 프레임워크가 내부에서 어떻게 동작하는지 직접 확인해보는 편이고, 그 과정에서 얻은 이해가 실무 문제를 푸는 데 도움이 된 경우가 많았습니다.

- 외부 광고 지면 중 약 20%가 비어 있는 것을 발견해 개선 작업 제안 및 진행 → 일 추가 수익 약 50만원 (목표 100만원)
- 백엔드 공통 라이브러리(API/Batch/Data 계층) 개발 → 신규 프로젝트 설정 보일러플레이트 90% 이상 감소
- 상품/인증 도메인 Python → Kotlin/Spring 재작성, 피크 5500+ rps 운영
- 블록체인 코어 자료구조(zktrie) 재구현으로 성능 문제 해결 (트래픽 증가로 체인이 약 6시간 중단됐던 장애의 원인)
- 3년차부터 팀 리드 및 멘토링 담당

---

## 주요 프로젝트

| 프로젝트                                        | 한 줄 요약                          |
|---------------------------------------------|---------------------------------|
| [복권 시스템](bunjang/lottery.md)                | Redis List 기반 당첨 처리, 중복 당첨 0건    |
| [광고 최적화](bunjang/advertisement.md)          | 광고 미송출 비율 개선, 일 추가 수익 약 50만원    |
| [백엔드 공통 라이브러리](bunjang/platform.md)         | API/Batch/Data 계층 공통 라이브러리 개발    |
| [금지어 탐지](bunjang/keyword-policy.md)         | 탐지 파이프라인 재설계, 처리량 2배 이상 향상      |
| [블록체인 코어 최적화](lightscale/zktrie.md)         | zktrie 재구현, 처리량 300% 향상          |
| [keth-client (개인)](personal/keth-client.md) | Kotlin Ethereum SDK             |
| [keth (개인)](personal/keth.md)               | Kotlin으로 EVM, StateDB, MPT 구현    |

> 전체 프로젝트 목록은 [all-projects.md](all-projects.md)에서 확인하실 수 있습니다.

---

## Contact

dudcjf89@gmail.com
