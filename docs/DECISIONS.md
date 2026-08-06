# 설계 결정

## 의사결정 요약

| ID | 문제 | 선택 | 포기한 것 | 비공개 원본의 확인 근거 |
|---|---|---|---|---|
| D1 | document, vector, cache의 commit 경계가 다름 | DB에 영속화되는 상태를 PostgreSQL과 pgvector transaction 경계로 통합 | 기존 분리 저장소 유지와 보상 로직 | transaction rollback 테스트 |
| D2 | 같은 document의 동시 갱신이 version을 충돌시킬 수 있음 | document identity 기반 transaction advisory lock | lock-free 처리와 요청 도착 순서 보장 | 동일 document 동시성 테스트 |
| D3 | 전역 advisory lock은 관계없는 document까지 막음 | document별 lock과 atomic corpus revision row update | 가장 단순한 전체 직렬화 | 서로 다른 document 동시성 테스트 |
| D4 | LLM 생성 중 corpus revision이 바뀔 수 있음 | LLM 호출 중 DB lock을 풀고 cache write에 revision guard 적용 | 생성 동안 상태 고정과 latest-at-return 보장 | 순차 revision drift 뒤 cache write 생략 테스트 |
| D5 | TTL과 질문 문자열만으로 의미가 달라진 cache를 막을 수 없음 | config version 확인, targeted invalidation, late write guard 분리 | 단순 cache key와 전체 revision별 cache key | version mismatch와 invalidation 테스트 |
| D6 | evidence가 없어도 모델을 호출하면 그럴듯한 답이 생김 | evidence가 없거나 prompt budget에 담기지 않으면 fail closed | 답변율 최대화 | no-evidence와 prompt budget 경로 테스트 |

## D1. DB 상태 정본 통합

### 배경

초기 구조는 metadata와 cache를 SQLite에, vector를 별도 저장소에 보관했습니다. Metadata commit 뒤 vector write가 실패하거나 cache invalidation과 vector 교체 순서가 어긋나면 partial state가 남을 수 있었어요.

### 선택

DB에 저장되는 document, chunk, pgvector 값, cache invalidation, corpus revision을 PostgreSQL transaction에 묶었습니다.

### 이유

보상 로직은 실패 조합이 늘어날수록 복구 상태와 테스트 경로가 함께 증가합니다. 단일 transaction은 DB에 저장되는 상태에 대해 전부 반영되거나 전부 rollback된다는 규칙을 제공합니다.

### 비용과 경계

- migration과 repository 재작성이 필요합니다.
- PostgreSQL과 pgvector에 대한 결합도가 높아집니다.
- Parsing과 embedding 생성은 DB transaction 밖에서 별도로 실패할 수 있습니다.
- 외부 object storage가 authoritative state로 추가되는 경우 이 결정만으로 원자성을 보장할 수 없습니다.

## D2. Document별 transaction advisory lock

### 배경

동일 document에 두 요청이 동시에 도착하면 둘 다 같은 current version을 보고 update를 준비할 수 있습니다.

### 선택

정규화한 document identity에서 lock key를 만들고 PostgreSQL transaction advisory lock을 획득합니다. Lock을 얻은 뒤 current ready document와 indexing version을 다시 확인해요.

### 이유

같은 document의 DB commit만 직렬화하고, lock 수명을 transaction과 맞추기 위해서입니다. 예외 경로에서 별도 release를 빠뜨릴 가능성도 줄입니다.

### 비용과 경계

- Lock key 정규화 규칙이 안정적이어야 합니다.
- Hash collision은 불필요한 직렬화를 만들 수 있습니다.
- 여러 document를 한 요청에서 처리하려면 lock order 계약이 추가로 필요합니다.
- Embedding 준비는 lock 전에 끝나므로 요청 도착 순서를 보장하지 않습니다.
- 분산 DB와 multi-region으로 일반화한 설계가 아닙니다.

## D3. Document별 lock과 global revision row

### 배경

모든 document update에 하나의 전역 advisory lock을 사용하면 관계없는 document의 준비와 write까지 같은 임계 구역에 들어갑니다. 반대로 document별 lock만 사용하면 global corpus revision 증가를 잃지 않는 별도 경계가 필요합니다.

### 선택

Document write에는 document별 advisory lock을 사용하고, global revision은 singleton row에 `revision = revision + 1`로 갱신합니다.

### 이유

Document state의 임계 구역은 분리하면서, 서로 다른 transaction의 revision 증가를 DB row update에서 보존할 수 있습니다.

### 비용과 경계

- 전역 advisory lock은 없지만 revision row update 구간에는 짧은 global serialization이 생깁니다.
- Corpus update 빈도가 매우 높을 때 revision row가 병목이 되는지는 측정하지 않았습니다.
- 성능 우위를 입증한 결정이 아니라 correctness와 설명 가능성을 위한 선택입니다.

## D4. LLM 호출과 revision guard

### 배경

LLM 호출은 DB query보다 오래 걸릴 수 있습니다. 그동안 corpus가 바뀌면 retrieval 시점의 evidence는 응답 완료 시점의 최신 corpus와 다를 수 있어요.

### 선택

Retrieval transaction에서 evidence와 expected corpus revision의 경계를 잡고 transaction을 닫습니다. LLM delta는 DB lock 없이 스트리밍합니다. 생성 후 cache write transaction에서 revision row를 잠그고 current revision과 expected revision을 비교합니다.

### 이유

느린 외부 호출 동안 DB connection과 lock을 유지하지 않으면서, revision 변경이 먼저 완료된 경우 변경 전 evidence로 만든 답변의 cache 저장을 생략하기 위해서입니다.

### 비용과 경계

- Revision이 바뀌면 이미 수행한 LLM 호출 결과를 cache에 저장하지 못합니다.
- 답변 delta는 guard 전에 사용자에게 전달되므로 stale snapshot 답변이 반환될 수 있습니다.
- latest-at-return consistency와 single-flight는 해결하지 않습니다.
- 후속 검토에서 ingestion invalidation 이후 revision row update 전에 cache writer가 끼어드는 경쟁 조건을 찾았습니다. 제출본의 lock 순서만으로는 모든 late write를 막지 못합니다.

## D5. Config version, invalidation, write guard 분리

### 배경

TTL이나 질문 문자열만으로는 prompt, embedding, retrieval 정책과 document 변경에 따른 의미 차이를 표현할 수 없습니다.

### 선택

세 가지 경계를 분리했습니다.

1. Cache lookup은 prompt, embedding, retrieval version과 active 상태를 확인합니다.
2. 새 document와 chunking 또는 indexing 변경은 전체 cache를, 기존 document update는 dependency와 새 chunk의 question similarity를 기준으로 cache를 무효화합니다.
3. LLM 생성 뒤에는 corpus revision guard로 이미 가시화된 revision drift의 cache write를 생략합니다.

Global corpus revision과 document dependency를 cache lookup key로 직접 사용하지 않습니다.

### 이유

설정 불일치, document 변경, 늦은 cache write는 서로 다른 race이므로 하나의 key나 TTL로 뭉치지 않기 위해서입니다.

### 비용과 경계

- 모든 cache writer와 mutation 경로가 세 계약을 지켜야 합니다.
- Version을 올리는 배포 규칙이 부정확하면 호환되지 않는 cache나 vector를 사용할 수 있습니다.
- Embedding version 변경 시 전체 corpus 재색인과 atomic cutover는 이 사례에서 입증하지 않았습니다.
- Targeted invalidation은 단순한 전체 cache 무효화보다 구현과 검증이 복잡합니다.
- 새 chunk와 cached question embedding의 cosine similarity가 retrieval threshold 이상인지를 사용하는 정책이며, 모든 미래 의미 변화를 수학적으로 포괄한다고 주장하지 않습니다.

## D6. Evidence가 없을 때 fail closed

### 배경

관련 evidence가 없거나 prompt budget에 근거를 담을 수 없는데 모델을 호출하면 일반 지식이나 추정으로 빈칸을 채울 수 있습니다.

### 선택

Relevant evidence가 없거나 근거가 prompt budget에 들어가지 못하면 LLM 호출과 cache 저장을 생략합니다.

### 이유

답변율보다 근거 기반 질의응답이라는 제품 계약을 우선했습니다.

### 비용과 경계

- 답변을 보류하는 비율이 늘 수 있습니다.
- Retrieval threshold와 prompt packing 정책이 fail-closed 의미에 영향을 주므로 version 관리가 필요합니다.
- Evidence가 존재한다는 사실만으로 생성된 모든 문장의 entailment를 보장하지 않습니다.

## 기록 시점의 경계

최종 구현과 회귀 테스트에서 위 선택에 대응하는 상태와 경로를 확인했습니다. 다만 모든 대안이 구현 당시 같은 형식의 ADR로 기록된 것은 아닙니다.

이 문서는 제출본 구현을 설명하기 위한 사후 구조화 문서입니다. 보존되지 않은 당시 발언, 최초 red 실행과 검토 순서를 새로 만들어 인용하지 않습니다.

## 후속 검토에서 보류한 결정

공개 문서를 다시 검토하면서 ingestion의 cache invalidation과 revision update 사이에 이전 revision의 cache writer가 새 active row를 insert할 수 있는 interleaving을 확인했습니다.

현재 보정 후보는 ingestion transaction이 document advisory lock을 얻은 뒤, cache invalidation보다 먼저 `corpus_state` row를 update lock으로 획득하고 commit까지 유지하는 것입니다. 그러면 이전 revision의 cache writer는 ingestion commit 뒤 revision mismatch를 확인하게 됩니다.

이 방안은 제출본에 반영되지 않았고 barrier 기반 회귀 테스트도 아직 없습니다. 따라서 채택된 결정이나 검증된 수정으로 기록하지 않습니다.
