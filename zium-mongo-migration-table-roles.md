# 데이터 통합 솔루션 ERD 테이블 역할 정리

## 1. 기획서 요소와 데이터 모델 연결

기획서의 핵심 흐름은 RDB 데이터를 MongoDB로 이관하는 1차 마이그레이션 기능입니다. 화면에서는 "커넥션"이라는 표현을 사용하지만, 실제 데이터 모델 관점에서는 소스 연결, 타겟 연결, 작업 정의, 대상 선택, 매핑, 실행, 로그가 결합된 마이그레이션 작업으로 보는 것이 적절합니다.

| 기획서 요소 | 화면/기능 의미 | 관련 테이블 |
|---|---|---|
| 대시보드 | 전체 커넥션 수, 상태별 통계, 최근 동기화 이력, 오류 알림 표시 | `migration_job`, `migration_execution`, `migration_execution_log` |
| 커넥션 관리 | 커넥션 목록 조회, 실행, 중지, 삭제, 상세 보기. p.10의 `커넥션 ID`는 `migration_job.job_code`에 해당합니다. | `migration_job`, `migration_source`, `migration_target`, `migration_execution` |
| Source 설정 | MySQL 등 RDB 접속 정보 입력, 연결 테스트 | `migration_source` |
| Target 설정 | MongoDB URI, DB, 컬렉션, 인증/TLS 옵션 설정 | `migration_target` |
| 동기화 주기 설정 | 수동, 정기, Cron 방식 선택 | `migration_job.schedule_type`, `migration_job.schedule_expr` |
| 동기화 방식 선택 | Full Load, Incremental Load 선택 | `migration_job.sync_mode`, `migration_source_dataset` |
| 소스 데이터 선택 | 이관 대상 테이블과 컬럼 선택 | `migration_source_dataset` |
| MongoDB 저장 방식 | 타겟 DB/컬렉션 선택 또는 생성 | `migration_target`, `migration_mapping.target_collection_name` |
| 데이터 변환 및 필드 매핑 | RDB 컬럼을 MongoDB 문서 필드 경로에 매핑, 변환 규칙 설정 | `migration_mapping` |
| 인덱스 생성 | MongoDB 컬렉션 인덱스 설정 | `migration_target_index` |
| 적재 후 쿼리 실행 | MongoDB updateMany, aggregate 등 후처리 작업 설정 | `migration_post_action` |
| 최종 설정 확인 | 생성 전 전체 설정 요약, 저장 또는 저장 후 실행 | `migration_job` 및 하위 설정 테이블 |
| 커넥션 상세 개요 | 소스/타겟/동기화 설정/최근 상태 조회 | `migration_job`, `migration_source`, `migration_target`, `migration_execution` |
| 변경 이력 | 커넥션 생성, 설정 변경, 동기화 실행 이력 | `migration_audit_history`, `migration_execution` |
| 모니터링 | 처리 건수, 응답 시간, 성공/실패/경고 통계 | `migration_execution`, `migration_execution_log` |
| 로그 및 트레이스 | INFO/WARN/ERROR/DEBUG 로그 검색 | `migration_execution_log` |

## 2. 각 테이블의 역할

| 테이블 | 역할 | 주요 저장 내용 |
|---|---|---|
| `migration_source` | RDB 소스 연결정보를 저장합니다. 1차 범위에서는 MySQL 중심이며, 연결 테스트 결과도 함께 보관합니다. | source type, connection name, host, port, database, username, password secret, JDBC params, last test status |
| `migration_target` | MongoDB 타겟 연결정보를 저장합니다. URI에는 인증 정보가 포함될 수 있으므로 시크릿 참조 방식으로 관리합니다. | target type, connection name, URI secret, scheme, host, port, database, default collection, auth options, last test status |
| `migration_job` | 화면에서 말하는 커넥션 또는 마이그레이션 작업의 마스터 테이블입니다. 소스와 타겟을 연결하고 실행 가능한 작업 단위를 정의합니다. | job code, job name, source id, target id, sync mode, schedule type, schedule expression, status, latest execution status |
| `migration_source_dataset` | 작업별로 이관할 소스 테이블과 컬럼을 저장합니다. 기획서의 "소스 데이터 선택" 화면에 대응합니다. | source schema, table, column, data type, selected flag, PK flag, incremental key column |
| `migration_mapping` | RDB 컬럼을 MongoDB 문서 필드로 매핑하는 정의를 저장합니다. 변환 규칙과 null 처리 정책도 포함합니다. | source table, source column, target collection, target field/path, target type, key flag, transform rule JSON, null handling |
| `migration_target_index` | 마이그레이션 후 MongoDB 컬렉션에 생성할 인덱스를 정의합니다. | collection name, index name, index keys JSON, unique flag, options JSON |
| `migration_post_action` | 적재 완료 후 MongoDB에서 실행할 후처리 쿼리나 명령을 저장합니다. | collection name, action type, query JSON, enabled flag, sort order |
| `migration_execution` | 작업 실행 단위의 요약 이력을 저장합니다. 실행 목록, 최근 실행 상태, 모니터링 통계의 기준이 됩니다. | execution code, run mode, requested by, result status, read count, written count, failed count, progress, response time, start/end time |
| `migration_execution_log` | 실행 중 발생한 상세 로그를 저장합니다. 모니터링, 오류 원인 분석, 로그 검색 화면에 대응합니다. | execution id, log level, issue type, message, context JSON, logged time |
| `migration_audit_history` | 설정 변경과 운영 이벤트의 이력을 저장합니다. 실행 로그와 달리 "누가 어떤 설정을 바꿨는지"를 남기는 목적입니다. | job id, history type, event status, description, actor, before JSON, after JSON, occurred time |

## 3. 테이블 간 관계 요약

| 관계 | 의미 |
|---|---|
| `migration_source` 1 : N `migration_job` | 하나의 소스 연결정보를 여러 마이그레이션 작업에서 재사용할 수 있습니다. |
| `migration_target` 1 : N `migration_job` | 하나의 MongoDB 타겟을 여러 작업에서 사용할 수 있습니다. |
| `migration_job` 1 : N `migration_source_dataset` | 하나의 작업은 여러 테이블/컬럼을 이관 대상으로 선택할 수 있습니다. |
| `migration_job` 1 : N `migration_mapping` | 하나의 작업은 여러 컬럼-필드 매핑을 가집니다. |
| `migration_job` 1 : N `migration_target_index` | 하나의 작업은 여러 MongoDB 인덱스 생성을 정의할 수 있습니다. |
| `migration_job` 1 : N `migration_post_action` | 하나의 작업은 여러 후처리 쿼리를 순서대로 실행할 수 있습니다. |
| `migration_job` 1 : N `migration_execution` | 하나의 작업은 수동/예약 실행을 통해 여러 실행 이력을 가질 수 있습니다. |
| `migration_execution` 1 : N `migration_execution_log` | 하나의 실행은 여러 상세 로그를 남깁니다. |
| `migration_job` 1 : N `migration_audit_history` | 하나의 작업은 생성, 수정, 중지, 실행 요청 등 여러 운영 이력을 가집니다. |

## 4. 설계 의도

현재 ERD는 1차 RDB to MongoDB 구현을 빠르게 만들 수 있도록 구성하면서도, 기획서에 있는 모니터링, 로그, 인덱스 생성, 후처리 쿼리, 변경 이력까지 담을 수 있게 확장했습니다.

핵심 기준은 다음과 같습니다.

| 기준 | 설명 |
|---|---|
| 설정과 실행 분리 | 작업 정의는 `migration_job`과 설정 테이블에, 실행 결과는 `migration_execution`에 저장합니다. |
| 연결정보와 작업 분리 | 소스/타겟 연결정보는 재사용 가능하도록 `migration_source`, `migration_target`으로 분리합니다. |
| 매핑과 실행 분리 | 필드 매핑은 실행마다 바뀌지 않는 설정이므로 `migration_mapping`에 저장하고, 실행 결과와 분리합니다. |
| 운영 추적 가능성 | 실행 로그는 `migration_execution_log`, 설정 변경 이력은 `migration_audit_history`에 분리 저장합니다. |
| MongoDB 특화 기능 반영 | 인덱스 생성과 후처리 쿼리를 `migration_target_index`, `migration_post_action`으로 별도 관리합니다. |

## 5. 후속 개선 방향

장기적으로 파일, API, Kafka 소스까지 확장하려면 `migration_source`와 `migration_target`을 더 일반화한 `migration_connection` 구조로 바꾸는 것이 좋습니다.

또한 증분 동기화가 본격화되면 다음 테이블을 추가하는 방향이 적절합니다.

| 추가 후보 테이블 | 목적 |
|---|---|
| `migration_sync_option` | cursor column, cursor type, overlap window 등 증분 동기화 설정 저장 |
| `migration_checkpoint` | 마지막 성공 실행 기준 cursor value 저장 |
| `migration_execution_step` | CONNECT, EXTRACT, TRANSFORM, LOAD, VERIFY 등 실행 단계별 상태 저장 |
| `connection_test_history` | 연결 테스트 성공/실패 이력을 누적 저장 |
