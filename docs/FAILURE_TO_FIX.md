# 실패에서 수정까지

## 증거 경계

이 문서는 비공개 원본 저장소의 Git 이력, 구현과 테스트 이름을 기준으로 재구성했습니다.

| 상태 | 내용 |
|---|---|
| 비공개 원본에서 확인 | 최초 파이프라인 시점, cache invalidation 수정, 추가된 테스트, PostgreSQL과 pgvector 통합, 최종 고정 커밋의 CI 통과 |
| 공개 독립 검증 불가 | 전체 구현 코드, private commit과 CI 원문, 실제 test output |
| 보존 자료에서 확인하지 못함 | 수정 전 red 실행 로그, 별도 red commit, 최초 재현 명령 |
| 후속 검토에서 확인 | Ingestion invalidation 이후 revision update 전에 cache writer가 끼어들 수 있는 lock-order 반례 |
| 검증 범위 밖 | 실제 모델 공급자, 프로덕션 트래픽, 성능과 HA |

수정 전 실패 로그가 없으므로 실패 테스트를 먼저 작성하고 red를 확인했다고 주장하지 않습니다. 테스트 이름은 수용 조건을 보여주는 보조 증거이며, 이 공개 저장소만으로 테스트 실행을 재현할 수는 없어요.

## 시간순 기록

| 날짜 | 확인된 변화 |
|---|---|
| 2026-07-28 | document upload, retrieval, answer cache를 포함한 첫 pipeline 구현 |
| 2026-07-29 | ingestion 실패 전에 cache를 무효화하도록 순서를 수정하고 실패 후 재시도 테스트 추가 |
| 2026-07-30 | PostgreSQL과 pgvector 통합 migration 기반 추가 |
| 2026-07-30 | config version, corpus revision, transaction 경계와 revision guard 구현 |
| 이후 고정 제출본 | 같은 bytes에서 embedding policy만 달라진 경우의 reindex 경로 보강 |
| 2026-08-07 후속 검토 | Invalidation과 revision update 사이의 cache writer overlap을 문서와 제출 코드의 transaction 순서에서 확인 |

## 1. 첫 구조의 실패 창

초기 구조는 상태를 서로 다른 저장소에 보관했습니다.

- SQLite: document metadata, answer cache, cache dependency
- 별도 vector store: chunk embedding과 retrieval index

Embedding 또는 vector 교체가 cache invalidation보다 먼저 실패하면 이전 cache가 활성 상태로 남을 수 있었습니다. 실패한 upload를 재시도하는 동안 이전 document 기준의 답변이 다시 cache에 저장될 가능성도 있었어요.

## 2. 첫 수정

Cache invalidation을 embedding과 vector 저장보다 먼저 수행하도록 순서를 바꿨습니다. 다음 회귀 테스트가 추가된 사실을 확인했습니다.

- `test_embedding_failure_invalidates_related_cache_before_retry`
- `test_vector_failure_invalidates_related_cache_before_retry`
- `test_created_embedding_failure_retry_invalidates_failure_window_recache`
- `test_created_vector_failure_retry_invalidates_failure_window_recache`
- `test_normal_same_hash_reupload_invalidates_corpus_cache`

이 수정은 실패 창을 줄였지만 metadata와 vector store의 commit을 원자적으로 만들지는 못했습니다. 한쪽 성공과 다른 쪽 실패를 보상 코드로 계속 복구해야 했습니다.

## 3. 문제를 다시 정의한 이유

핵심 문제는 cache invalidation 함수를 어디서 호출할지가 아니었습니다. Document metadata, chunk, vector, cache와 revision이 서로 다른 commit과 recovery 경계에 있다는 점이 원인이었어요.

검토한 방향은 다음과 같습니다.

| 방향 | 장점 | 한계 | 최종 판정 |
|---|---|---|---|
| 보상 로직 유지 | 기존 저장소 유지 가능 | 실패 조합마다 복구 경로와 중간 상태 증가 | 선택하지 않음 |
| 전역 advisory lock | 동시성 추론이 단순함 | 관계없는 document까지 직렬화 | 선택하지 않음 |
| LLM 호출까지 transaction 유지 | 생성 중 revision 변화 차단 | 외부 호출 동안 DB 자원과 lock 점유 | 선택하지 않음 |
| PostgreSQL과 pgvector 통합 | DB 상태 변경을 한 transaction에 포함 | migration과 query 재작성 필요 | 선택 |
| Document별 lock과 revision guard | document commit 직렬화와 stale cache write 차단 | 요청 순서와 latest response는 별도 문제 | 선택 |

이 표는 최종 구조를 설명하기 위한 사후 설계 분석입니다. 모든 대안이 당시 같은 깊이의 ADR이나 대화로 보존됐다고 주장하지 않습니다.

## 4. 제출본의 마지막 수정

### Ingestion transaction

1. Parsing, chunking, embedding을 DB transaction 밖에서 준비
2. 같은 document identity의 advisory lock 획득
3. Current ready document와 indexing version 확인
4. 새 document version, chunk와 vector 저장
5. 영향받을 수 있는 cache 비활성화
6. Ready version 전환
7. `corpus_state.revision = revision + 1`
8. Commit

DB write 중 어느 단계에서 실패해도 document, chunk, vector, invalidation과 revision update를 함께 rollback합니다. 이 경우 마지막으로 성공한 ready corpus가 계속 authoritative state이므로 기존 cache invalidation도 되돌아갑니다.

### Question과 cache write

1. Active 상태와 config version을 확인해 cache lookup
2. Cache miss이면 revision row를 shared lock으로 읽고 ready chunk retrieval
3. DB lock 없이 LLM delta를 사용자에게 스트리밍
4. Cache write transaction에서 revision row를 update lock으로 읽음
5. Current revision과 expected revision이 다르면 cache write 생략

Revision drift가 생긴 답변은 사용자에게 이미 전달될 수 있습니다. 또한 제출본의 guard는 revision 변경이 먼저 가시화된 경우에는 cache write를 생략하지만, ingestion이 invalidation을 끝내고 revision row를 잠그기 전의 overlap까지 차단하지는 않습니다.

## 5. 제출본의 회귀 조건

비공개 원본에서 다음 테스트 이름을 확인했습니다.

- `test_commit_is_atomic_and_pgvector_retrieval_tracks_ready_version`
- `test_cache_requires_all_versions_and_revision_guard_skips_stale_write`
- `test_update_invalidates_newly_relevant_non_dependency_cache`
- `test_chunk_failure_rolls_back_document_and_revision`
- `test_same_document_concurrency_serializes_to_one_revision`
- `test_failure_after_cache_invalidation_rolls_back_every_write`
- `test_distinct_document_concurrency_preserves_both_revision_increments`
- `test_same_bytes_reindex_when_only_embedding_policy_changes`

테스트가 다루는 범위도 제한해서 읽어야 합니다.

| 테스트 축 | 확인 범위 | 확대하면 안 되는 주장 |
|---|---|---|
| Atomic commit과 rollback | 실제 PostgreSQL과 pgvector 경로의 DB 상태 | 외부 embedding 호출까지 DB transaction에 포함 |
| 같은 document 동시성 | 동일 내용 동시 요청이 하나의 ready version과 revision으로 정리 | 서로 다른 내용의 요청 도착 순서 보장 |
| 서로 다른 document 동시성 | 두 revision 증가가 유실되지 않음 | global revision row의 성능 우위 |
| Config version | prompt, embedding, retrieval version mismatch cache miss | embedding version의 전체 corpus atomic cutover |
| Revision guard | Ingestion commit 뒤 revision mismatch가 된 late cache write 생략 | 사용자에게 항상 최신 답변 반환, invalidation과 revision update 사이 overlap 차단 |
| Reindex | 동일 bytes라도 indexing version 불일치 시 새 version 생성 | 모든 기존 document가 배포 시 자동으로 재색인 |

## 6. 후속 검토에서 발견한 잔여 경쟁 조건

제출본의 transaction 순서를 그대로 따르면 다음 실행이 가능합니다.

1. Ingestion이 기존 cache를 무효화하고 ready document를 전환합니다.
2. Ingestion은 아직 `corpus_state` row를 갱신하지 않았습니다.
3. 이전 revision의 cache writer가 해당 row를 먼저 update lock으로 읽습니다.
4. Writer는 revision이 같다고 판단해 새 active cache를 insert하고 commit합니다.
5. Ingestion이 revision을 증가시키고 commit합니다.

새 cache row는 ingestion의 invalidation statement 뒤에 생성되었으므로 active 상태로 남을 수 있습니다. 기존 테스트는 ingestion commit이 끝난 뒤 stale write를 시도하는 순차 경로를 확인하며, 이 interleaving을 barrier로 재현하지 않습니다.

보정 후보는 ingestion이 cache invalidation 전에 revision row를 update lock으로 획득하고 commit까지 유지하는 것입니다. 다음 테스트에서는 두 transaction을 barrier로 정지시켜 writer가 ingestion commit 뒤 `corpus_changed`로 저장을 생략하는지 확인해야 합니다.

이 보정은 아직 구현하거나 검증하지 않았습니다. 따라서 완료된 수정이나 제출본의 성과로 사용하지 않아요.

## 7. 이 사례에서 확인된 변화

처음에는 cache invalidation 호출 순서 문제로 접근했습니다. 실패 후 재시도 경로를 고정하면서 저장소 간 commit과 recovery 경계가 더 근본적인 문제임을 확인했어요.

최종 제출본에서는 DB에 저장되는 상태를 PostgreSQL과 pgvector의 한 transaction에 모으고, document commit, retrieval boundary, cache write가 사용하는 lock과 revision의 책임을 분리했습니다. 후속 검토에서는 이 분리만으로 모든 transaction overlap이 닫히지는 않는다는 점도 확인했어요.

이 과정에서 다음 한계는 그대로 남겼습니다.

- 최초 red 실행 로그와 재현 명령은 보존 자료에서 확인하지 못했습니다.
- 전체 구현과 CI 원문은 비공개이므로 공개 독자가 독립 실행할 수 없습니다.
- 실제 모델 공급자 호출은 최종 검증 범위가 아닙니다.
- Embedding version의 전체 corpus cutover는 입증하지 않았습니다.
- 프로덕션 운영, 성능, 고가용성과 single-flight는 검증하지 않았습니다.
- Ingestion과 cache writer의 lock-order overlap은 수정하거나 회귀 테스트로 닫지 않았습니다.
