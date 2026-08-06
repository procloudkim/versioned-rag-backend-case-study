# AI 사용과 책임

## 사용 방식

Codex를 코드와 테스트 초안을 만드는 구현 도구로 사용했습니다. 이 프로젝트에서 개인 프로젝트라는 표현은 사람 팀원이 없었다는 뜻이며, 모든 코드를 AI 도움 없이 손으로 작성했다는 뜻이 아닙니다.

AI 사용 사실만으로 구현 품질을 주장하지 않습니다. 요구사항을 어떤 실패 조건으로 바꿨는지, 어떤 대안을 선택했는지, 생성된 결과를 어떤 테스트와 상태 확인으로 수용했는지를 설명 범위로 삼습니다.

## 책임 분리

| 단계 | 사람이 책임진 범위 | AI가 도운 범위 |
|---|---|---|
| 요구사항 | 모호한 요구를 기능, 상태와 실패 조건으로 해석 | 누락 조건과 질문 후보 제안 |
| 설계 | 저장소 통합, lock scope, revision과 version 정책 선택 | 대안과 위험 목록 작성 보조 |
| 구현 | 채택 범위, 코드 구조와 최종 변경 승인 | 코드와 테스트 초안 작성 |
| 실패 정의 | stale cache, partial state와 unsupported answer를 실패로 규정 | 경계 조건 후보 제안 |
| 수용 기준 | 통과해야 할 불변식과 제외 범위 결정 | fixture와 test skeleton 작성 보조 |
| 검증 | 명령, diff, DB 상태와 결과를 확인하고 최종 판정 | 로그 요약과 검토 보조 |
| 공개 범위 | 원본 요구사항, 조직 식별 정보와 전체 코드를 제외 | 문서 구조와 표현 후보 제안 |

## 사람이 선택한 핵심 경계

- Cache invalidation 호출 순서만 고치는 것으로는 분리 저장소의 commit 문제를 해결할 수 없다고 판단했습니다.
- DB에 저장되는 document, chunk, vector, invalidation과 revision을 PostgreSQL transaction에 모았습니다.
- 같은 document commit에는 transaction advisory lock을 사용하고, global revision 증가는 별도 row update로 보존했습니다.
- LLM 호출 동안 DB lock을 유지하지 않고 cache write에 expected revision guard를 적용했습니다. 후속 검토에서는 이 guard가 닫지 못한 ingestion lock-order overlap을 별도로 기록했습니다.
- Cache lookup, document 변경 invalidation, late cache write guard를 서로 다른 계약으로 분리했습니다.
- Evidence가 없거나 prompt budget에 근거를 넣을 수 없으면 LLM 호출을 생략했습니다.
- 실제 모델 공급자, 프로덕션 운영과 성능을 검증한 것처럼 확대하지 않았습니다.

## 증거 상태

| 주장 | 상태 | 경계 |
|---|---|---|
| 개인 프로젝트이며 Codex를 사용함 | 본인 확인 | 사람 팀원이 없었다는 뜻, 순수 손코딩 주장이 아님 |
| Cache invalidation 수정과 PostgreSQL 통합 | 비공개 원본의 Git 이력에서 확인 | 전체 코드는 공개하지 않음 |
| Revision guard, rollback과 version 테스트 | 비공개 고정 커밋과 CI에서 확인 | 공개 독자는 독립 재실행 불가 |
| Invalidation 이후 revision update 전 cache writer overlap | 후속 코드와 문서 검토에서 확인 | 수정과 barrier 기반 회귀 테스트는 아직 없음 |
| 특정 AI 제안을 수정하거나 거절한 대화 원문 | 공개 증거로 보존하지 않음 | 사례를 기억으로 복원하지 않음 |
| Prompt부터 최종 코드까지의 완전한 provenance | 구성하지 않음 | 함수별 작성 기원을 추정하지 않음 |

## 검증 방법

AI가 만든 초안을 수용하는 기준은 생성 속도나 코드 양이 아니었습니다.

1. 요구사항을 상태와 실패 조건으로 바꿉니다.
2. 선택한 설계가 지켜야 할 불변식을 정의합니다.
3. 실패 injection, rollback, version drift와 concurrency 경로를 테스트합니다.
4. 실제 PostgreSQL 상태와 CI 결과를 확인합니다.
5. 확인되지 않은 provider, 운영과 성능 주장은 제외합니다.
6. 공개 설명의 transaction 순서를 맥락 없는 기술 검토에 다시 노출하고, 반례가 나오면 기존 성과와 미해결 상태를 분리합니다.

이 문서는 AI가 프로젝트를 설계하거나 책임졌다는 증거가 아닙니다. 사람이 AI 초안을 어떤 기준으로 검토하고 수용했는지를 공개 가능한 범위에서 설명합니다.
