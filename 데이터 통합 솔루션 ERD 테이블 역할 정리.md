# 데이터 통합 솔루션 ERD 테이블 역할 정리

## 1. 모델 개요

이 문서는 `final-data-integration-platform.erd`의 10개 테이블을 설명한다. 모델의 목적은 RDB·파일·API·Kafka에서 데이터를 읽어 MongoDB로 변환·적재하는 기본 흐름을 학습하는 것이다. 운영 환경의 모든 기능을 담기보다 다음 흐름이 한눈에 보이도록 단순화했다.

> 연결정보 → 동기화 작업 → 수집 대상·필드 매핑 → 증분 위치 → MongoDB 적재 → 실행·로그·감사

## 2. 핵심 테이블 역할

### 연결과 작업

| 테이블 | 역할 | 주요 컬럼 |
|---|---|---|
| `data_connection` | Source와 Target의 재사용 가능한 연결정보 | `direction`, `connection_type`, `provider`, `endpoint`, `secret_ref`, `options_json` |
| `sync_job` | 어떤 Source 데이터를 언제 어떤 Target으로 보낼지 정의하는 작업 | `source_connection_id`, `target_connection_id`, `sync_mode`, `schedule_type`, `schedule_expr`, `status` |

`data_connection` 하나로 RDB, 파일, API, Kafka, MongoDB를 표현한다. 공통 정보는 일반 컬럼으로 관리하고 유형별 상세 설정은 `options_json`에 저장한다. 비밀번호나 토큰은 직접 저장하지 않고 `secret_ref`로 외부 비밀 저장소를 참조한다.

연결정보와 작업을 분리한 이유는 하나의 연결을 여러 작업에서 재사용하기 위해서다.

### 데이터 선택과 변환

| 테이블 | 역할 | 주요 컬럼 |
|---|---|---|
| `source_dataset` | 작업별 수집 대상과 선택 필드 정의 | `object_type`, `object_name`, `selected_columns_json`, `reader_options_json`, `incremental_key` |
| `field_mapping` | Source 필드를 MongoDB 문서 필드에 연결하고 변환 규칙 정의 | `source_field`, `target_collection`, `target_path`, `target_type`, `is_key`, `transform_json` |

학습용 모델에서는 테이블·파일·API 리소스·Kafka Topic을 모두 `source_dataset`으로 추상화한다. 컬럼 선택과 Reader 옵션은 JSON으로 단순화했다. 규모가 커지면 데이터셋과 필드를 별도 테이블로 나눌 수 있다.

### 증분 적재와 MongoDB 후처리

| 테이블 | 역할 | 주요 컬럼 |
|---|---|---|
| `sync_checkpoint` | 마지막 성공 처리 위치 저장 | `cursor_key`, `cursor_value`, `updated_at` |
| `target_index` | MongoDB Collection에 생성할 인덱스 정의 | `collection_name`, `index_name`, `index_keys_json`, `unique_yn` |
| `post_load_action` | 적재 완료 후 실행할 MongoDB 작업 정의 | `collection_name`, `action_type`, `query_json`, `sort_order` |

`source_dataset.incremental_key`는 증분 처리의 기준을, `sync_checkpoint.cursor_value`는 실제로 처리 완료한 위치를 나타낸다. 설정과 실행 상태를 분리한 예다.

### 실행과 이력

| 테이블 | 역할 | 주요 컬럼 |
|---|---|---|
| `job_execution` | 작업 실행 1회의 결과 요약 | `run_type`, `status`, `read_count`, `written_count`, `failed_count`, `started_at`, `ended_at` |
| `execution_log` | 실행 중 발생한 상세 로그 | `log_level`, `message`, `context_json`, `logged_at` |
| `audit_history` | 작업 설정과 운영 행위의 변경 이력 | `event_type`, `description`, `before_json`, `after_json`, `actor_name` |

`job_execution`은 실행 결과를 빠르게 조회하기 위한 요약이고, `execution_log`는 장애 분석을 위한 상세 기록이다. `audit_history`는 누가 작업 설정을 변경하거나 실행했는지를 기록한다.

## 3. 핵심 관계와 데이터 흐름

| 관계 | 의미 |
|---|---|
| `data_connection` 1:N `sync_job` | 하나의 연결정보를 여러 작업에서 Source 또는 Target으로 재사용 |
| `sync_job` 1:N `source_dataset` | 한 작업에서 여러 수집 대상 정의 |
| `sync_job` 1:N `field_mapping` | 한 작업에서 여러 필드 매핑 정의 |
| `sync_job` 1:N `sync_checkpoint` | 대상 또는 Partition별 증분 위치 관리 |
| `sync_job` 1:N `target_index` | 한 작업이 여러 MongoDB 인덱스 생성 가능 |
| `sync_job` 1:N `post_load_action` | 적재 후 여러 작업을 순서대로 실행 가능 |
| `sync_job` 1:N `job_execution` | 하나의 작업 설정을 반복 실행 |
| `job_execution` 1:N `execution_log` | 실행 1회에 여러 로그 기록 |
| `sync_job` 1:N `audit_history` | 작업별 변경·운영 이력 기록 |

기본 실행 흐름은 다음과 같다.

1. `data_connection`에서 Source와 MongoDB Target 연결을 선택한다.
2. `sync_job`의 동기화 방식과 스케줄에 따라 작업을 시작한다.
3. `source_dataset`에서 수집 대상을 읽는다.
4. `field_mapping`의 매핑과 변환 규칙을 적용한다.
5. MongoDB에 적재하고 `target_index`, `post_load_action`을 적용한다.
6. `job_execution`과 `execution_log`에 결과를 기록한다.
7. 성공한 증분 실행이면 `sync_checkpoint`를 갱신한다.

## 4. 기획서 요구사항 대응

| 요구사항 | 대응 테이블 |
|---|---|
| RDB·파일·API·Kafka 연결 | `data_connection` |
| 수동·주기·Cron 실행 | `sync_job` |
| 전체·증분 동기화 | `sync_job`, `source_dataset`, `sync_checkpoint` |
| 필드 선택·MongoDB 매핑·변환 | `source_dataset`, `field_mapping` |
| MongoDB 인덱스·적재 후 쿼리 | `target_index`, `post_load_action` |
| 실행 결과와 처리 건수 | `job_execution` |
| 로그와 변경 이력 | `execution_log`, `audit_history` |

## 5. 추가할 테이블

아래 테이블은 기획서상 필요하지만 학습용 핵심 ERD에는 포함하지 않았다. 구현 범위가 확장될 때 목적에 따라 추가한다.

| 추가 테이블 | 해결하는 부족 사항 | 핵심 역할 |
|---|---|---|
| `app_user`, `app_role`, `app_user_role` | 사용자·역할·권한 | 사용자 인증정보와 RBAC의 N:M 관계 관리 |
| `job_permission` | 사용자별 작업 접근권한 | 특정 작업에 대한 조회·수정·실행 권한 관리 |
| `alert_channel`, `alert_rule`, `notification_event` | 알림 규칙·채널·중복 억제 | Email·Slack·SMS 설정, 조건, 발송 이력 관리 |
| `retry_policy` | 재시도 횟수·간격·백오프 | 작업 또는 단계별 재시도 정책 재사용 |
| `dead_letter_record` | 실패 레코드와 DLQ | 실패 Payload·원인·재처리 상태 관리 |
| `batch`, `batch_job` | 배치와 작업 실행 순서 | 여러 동기화 작업의 순서·의존성 관리 |
| `schema_snapshot`, `schema_change_event` | 스키마 변경 감지 | Source 스키마 버전과 변경 내역 관리 |
| `delete_sync_policy` | 삭제 데이터 반영 정책 | Source 삭제를 Target 삭제·Soft Delete 등으로 매핑 |
| `job_version` | 설정 버전·승인·배포 | 작업 설정의 버전, 승인 상태, 활성 버전 관리 |
| `connection_test_history` | 연결 테스트 상세 이력 | 테스트 시각·응답시간·오류 사유 기록 |
| `quality_rule`, `quality_result` | 데이터 검증 규칙과 결과 | 필수값·형식·범위 검사 및 실패 결과 관리 |
| `oauth_token` | OAuth 토큰 수명주기 | 발급·갱신·만료 상태와 Secret 참조 관리 |
| `execution_control` | 실행 중지·취소 상태 | 취소 요청자·시각·처리 결과 관리 |
| `retention_policy` | 로그·이력 보존 기간 | 데이터 종류별 보존·삭제·마스킹 정책 관리 |
| `idempotency_record` | 중복 방지와 멱등성 | Source Key와 처리 결과를 기준으로 중복 적재 방지 |

이 테이블들을 처음부터 모두 추가하면 핵심 흐름보다 운영 예외가 더 크게 보인다. 학습 단계에서는 현재 10개 테이블로 관계를 이해하고, 이후 요구사항이 생기는 순서대로 확장하는 것이 적절하다.

## 6. 학습 시 확인할 설계 포인트

- 화면 하나가 반드시 테이블 하나가 되는 것은 아니다.
- 연결 설정처럼 재사용되는 정보와 실행 결과처럼 계속 누적되는 정보를 분리한다.
- 증분 기준과 마지막 처리 위치는 서로 다른 데이터다.
- 실행 로그와 변경 감사 이력은 기록 목적이 다르다.
- JSON은 다양한 커넥터 설정을 단순화하지만 타입·필수값 검증은 애플리케이션에서 보완해야 한다.
- 실제 구현 전에는 재시도, 중복 방지, 삭제 반영, 보존 기간 정책을 반드시 확정해야 한다.
