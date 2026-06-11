# AI-ISMS Manager ERD 초안 v0.1

| 항목 | 내용 |
|---|---|
| 기준 문서 | AI_ISMS_PRD_v0.4.md (10장 데이터 모델), AI_ISMS_FSD_v0.1.md |
| DBMS | PostgreSQL 15+ (pgvector 확장) |
| 작성일 | 2026-06-11 |

---

## 1. 설계 원칙

1. **테넌트 격리 (FR-TEN-07, NFR-SCL-06)**: 모든 업무 테이블은 `tenant_id NOT NULL`을 가지며 PostgreSQL Row-Level Security(RLS) 정책으로 강제한다. 애플리케이션 계층 필터에만 의존하지 않는다.
2. **시스템 제공 콘텐츠 vs 테넌트 콘텐츠**: 기준 카탈로그(`standards` 계열)는 `tenant_id NULL` 허용 — NULL이면 시스템 제공 콘텐츠(예: ISO/IEC 27001:2022 시드)로 전 테넌트가 읽기 전용 공유하고, 테넌트 커스텀 기준은 자신의 `tenant_id`로 생성한다. `plans`, `modules`, `embedding_models`는 플랫폼 전역 테이블이다.
3. **모듈 경계 (2.4, 10.2-18)**: 취약점 모듈(VULN) 테이블은 코어(CORE) 테이블에 DB FK를 걸지 않고 **ID 논리 참조**만 사용한다 — 모듈 미구독 테넌트에는 빈 영역이고, 향후 별도 서비스/DB로 분리 배포가 가능해야 한다. 코어→VULN 참조(`gaps.vuln_finding_ref`)도 논리 참조다.
4. **증적 3원 연결 (FR-PRJ-07)**: 증적은 복사하지 않고 `evidence_links(project_id, assessment_id, evidence_id)`로 연결한다. 점검 결과는 프로젝트별 분리, 증적은 공유.
5. **승인본 보존 (NFR-AUD-04)**: 점검·개선계획 등 승인 대상은 버전 칼럼과 이력 테이블로 승인 시점 값을 보존한다.
6. **임베딩 모델 버전 (10.2-12)**: 청크는 `embedding_model_id`를 가지며, 모델 교체 시 재색인·병행 운영을 지원한다.
7. **감사로그 (FR-SEC-13)**: append-only + 해시체인(`prev_hash`, `row_hash`).
8. **ISO27001 우선 (2.4-5)**: 초기 시드 데이터는 ISO/IEC 27001:2022 — 통제 테마 4개(조직/인적/물리/기술), Annex A 93개 통제, 심사 유형 프로파일 3종(최초/사후/갱신). 원문 저작권 제약(20장)에 따라 통제 제목·요약 메타데이터 수준으로 시드하고, 원문은 고객 라이선스 보유 시 등록한다.

---

## 2. 도메인별 ERD

### 2.1 플랫폼 / SaaS (FR-TEN)

```mermaid
erDiagram
    plans ||--o{ tenants : "구독"
    tenants ||--o{ tenant_module_subscriptions : ""
    modules ||--o{ tenant_module_subscriptions : ""
    tenants ||--o{ usage_records : ""
    tenants ||--o{ tenant_settings : ""

    tenants {
        uuid id PK
        string name
        string status "active/suspended/grace/closed"
        uuid plan_id FK
        timestamptz created_at
        timestamptz closed_at "탈퇴(파기 스케줄 기준)"
    }
    plans {
        uuid id PK
        string code "FREE/STD/ENT"
        int max_users
        bigint max_storage_bytes
        int max_ai_calls_month
    }
    modules {
        uuid id PK
        string code "CORE / VULN"
        string name
        bool addon "VULN=true"
    }
    tenant_module_subscriptions {
        uuid id PK
        uuid tenant_id FK
        uuid module_id FK
        string status "active/expired/grace/cancelled"
        date started_at
        date expires_at
        string billing_ref "Phase4 결제 연동용, MVP는 수동 계약"
    }
    usage_records {
        uuid id PK
        uuid tenant_id FK
        string metric "users/storage/ai_calls/scan_runs"
        bigint value
        date period_start
        date period_end
    }
    tenant_settings {
        uuid id PK
        uuid tenant_id FK
        string key "llm_transfer_matrix/log_retention 등 (FR-TEN-09)"
        jsonb value
    }
```

- 기능 게이팅(FR-TEN-04): API 게이트웨이와 각 서비스가 `tenant_module_subscriptions`를 이중 검증한다(16장 리스크 "모듈 게이팅 우회").
- `usage_records.metric='scan_runs'`는 VULN 모듈 과금 지표.

### 2.2 조직 / 사용자 (FR-ORG, FR-USR)

```mermaid
erDiagram
    tenants ||--o{ organizations : ""
    organizations ||--o{ departments : ""
    organizations ||--o{ services : ""
    services ||--o{ systems : ""
    tenants ||--o{ users : ""
    users ||--o{ user_roles : ""
    roles ||--o{ user_roles : ""
    users ||--o{ user_project_scopes : "외부 컨설턴트 범위"

    organizations {
        uuid id PK
        uuid tenant_id FK
        string name
    }
    departments {
        uuid id PK
        uuid tenant_id FK
        uuid organization_id FK
        string name
    }
    services {
        uuid id PK
        uuid tenant_id FK
        uuid organization_id FK
        string name
        bool in_cert_scope
        string exclusion_reason
    }
    systems {
        uuid id PK
        uuid tenant_id FK
        uuid service_id FK
        string name
        string location
        bool in_cert_scope
        uuid owner_user_id
    }
    users {
        uuid id PK
        uuid tenant_id FK
        string email UK
        uuid department_id FK
        string status "invited/active/locked/dormant/disabled (FR-USR-06)"
        bool mfa_enrolled "관리자 필수 (FR-USR-04)"
        date account_expires_at "컨설턴트 계정 만료 (FR-USR-07)"
    }
    roles {
        uuid id PK
        string code "SYS_ADMIN/SEC_ADMIN/DEPT_USER/REVIEWER/AUDITOR/EXEC/CONSULTANT"
    }
    user_roles {
        uuid user_id FK
        uuid role_id FK
    }
    user_project_scopes {
        uuid user_id FK
        uuid project_id FK "허용 프로젝트 바인딩"
    }
```

- 인증 범위 변경 이력(FR-ORG-10)은 8절 `audit_logs`로 기록(승인자·사유 포함).

### 2.3 기준 / 콘텐츠 (FR-STD)

```mermaid
erDiagram
    standards ||--o{ standard_versions : ""
    standard_versions ||--o{ control_domains : ""
    control_domains ||--o{ controls : ""
    controls ||--o{ check_items : ""
    check_items ||--o{ evidence_requirements : ""
    controls ||--o{ control_mappings : "출발"

    standards {
        uuid id PK
        uuid tenant_id "NULL=시스템 제공 카탈로그(ISO27001:2022 시드)"
        string code "ISO27001/ISMS/ISMSP/INTERNAL"
        string issuer "발행 기관 (7.3 필수 기록)"
    }
    standard_versions {
        uuid id PK
        uuid standard_id FK
        string version_label "2022 / 고시 제2023-XX호"
        string status "draft/confirmed/deprecated"
    }
    control_domains {
        uuid id PK
        uuid standard_version_id FK
        string code "A.5 조직 / A.6 인적 / A.7 물리 / A.8 기술"
        string name
    }
    controls {
        uuid id PK
        uuid control_domain_id FK
        string code "A.5.1 등"
        string title
        text summary "원문 저작권 제약 시 요약 메타데이터"
        bool auto_assessable "FR-STD-07"
        string asset_types "FR-STD-08"
    }
    check_items {
        uuid id PK
        uuid control_id FK
        string code
        text question
    }
    evidence_requirements {
        uuid id PK
        uuid check_item_id FK
        string evidence_type "7.5의 16종"
        int recommended_cycle_days "FR-EVD-04 만료일 기본값 원천"
        bool mandatory "스테퍼 증적 보완 단계 집계 기준"
    }
    control_mappings {
        uuid id PK
        uuid from_control_id FK
        uuid to_control_id "타 기준 통제항목 (FR-STD-09)"
        string mapping_type "equivalent/partial/related"
    }
```

### 2.4 심사 프로젝트 / 점검 (FR-PRJ, FR-CHK)

```mermaid
erDiagram
    assessment_types ||--o{ assessment_projects : ""
    standard_versions ||--o{ assessment_types : "프로파일-기준 버전 연결"
    assessment_projects ||--o{ project_stage_progress : ""
    assessment_projects ||--o{ assessments : ""
    check_items ||--o{ assessments : ""
    assessments ||--o{ assessment_history : ""
    assessment_projects |o--o| assessment_projects : "직전 프로젝트(이어받기)"

    assessment_types {
        uuid id PK
        uuid tenant_id "NULL=시스템 제공 프로파일"
        string code "ISO27001_INITIAL/SURVEILLANCE/RENEWAL 등 11종"
        uuid standard_version_id FK
        jsonb stages "단계 구성+가중치 (FR-PRJ-03 산식)"
        jsonb report_templates
        jsonb menu_profile "SoA 노출 여부 등"
    }
    assessment_projects {
        uuid id PK
        uuid tenant_id FK
        uuid assessment_type_id FK
        string name
        string status "준비중/진행중/심사중/완료/보류 (FR-PRJ-08)"
        date target_audit_date
        uuid prev_project_id "사후·갱신 이어받기 (FR-PRJ-05)"
    }
    project_stage_progress {
        uuid id PK
        uuid project_id FK
        int stage_no
        numeric weight
        numeric completion_rate "데이터 기반 자동 계산"
        bool is_current
    }
    assessments {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        uuid check_item_id FK
        uuid assignee_id FK
        uuid reviewer_id FK
        string result "충족/부분충족/미흡/해당없음/판단불가"
        string workflow_status "작성중/검토중/승인완료/반려"
        text opinion
        string na_reason "해당없음 사유 필수"
        int version "승인 후 변경 시 증가 (NFR-AUD-04)"
    }
    assessment_history {
        uuid id PK
        uuid assessment_id FK
        int version
        jsonb snapshot "승인 시점 값 보존"
    }
```

- 인증 준비율(FR-DSH-01) = `assessments`에서 (충족·승인완료 + 0.5×부분충족·승인완료) / 평가 대상(해당없음 제외) — 프로젝트/전사 단위 집계 뷰(머티리얼라이즈드 뷰)로 구현.

### 2.5 증적 (FR-EVD)

```mermaid
erDiagram
    evidence_files ||--o{ evidence_versions : ""
    evidence_files ||--o{ evidence_links : ""
    assessment_projects ||--o{ evidence_links : ""
    assessments ||--o{ evidence_links : ""
    evidence_files ||--o{ evidence_reviews : ""

    evidence_files {
        uuid id PK
        uuid tenant_id FK
        string title
        string evidence_type "7.5의 16종"
        string status "등록/검토중/승인/반려/만료/교체필요"
        string classification "공개/내부/기밀/개인정보포함 (FR-EVD-07, LLM 매트릭스 입력)"
        date expires_at "권장 주기 자동 계산+수정 (FR-EVD-04)"
        uuid owner_id FK
        uuid current_version_id
    }
    evidence_versions {
        uuid id PK
        uuid evidence_file_id FK
        int version_no
        string storage_path "암호화 저장 (FR-SEC-12)"
        string sha256_hash "중복 탐지 (FR-EVD-12)"
        string av_scan_status "clean/quarantined (FR-EVD-14)"
        string mime_type
        bigint size_bytes
    }
    evidence_links {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        uuid assessment_id FK
        uuid evidence_file_id FK "3원 연결 — 증적 비복사 공유 (FR-PRJ-07)"
    }
    evidence_reviews {
        uuid id PK
        uuid evidence_file_id FK
        uuid reviewer_id FK
        string decision "approve/reject"
        text reject_reason "반려 사유 필수"
    }
```

- 상태 전이(승인→색인 반영, 만료·반려·교체→색인 무효화)는 이벤트로 2.6 파이프라인에 전파(FR-EVD-10, 11).

### 2.6 RAG / AI (FR-RAG, FR-AIA)

```mermaid
erDiagram
    embedding_models ||--o{ document_chunks : ""
    documents ||--o{ document_chunks : ""
    ai_answer_logs ||--o{ ai_citation_logs : ""
    document_chunks ||--o{ ai_citation_logs : ""
    ai_answer_logs ||--o{ ai_feedback : ""

    embedding_models {
        uuid id PK
        string name
        string version
        int dimension
        string status "active/reindexing/retired"
    }
    documents {
        uuid id PK
        uuid tenant_id FK
        string source_type "EVIDENCE/POLICY/STANDARD/GAP/PLAN"
        uuid source_id "원본 객체 ID (10.2-11 색인 사본)"
        int source_version
        string status "indexed/invalidated"
        jsonb acl "권한 사전 필터 메타데이터 (6.5)"
        date valid_until "만료 문서 제외 (FR-RAG-14)"
    }
    document_chunks {
        uuid id PK
        uuid tenant_id FK
        uuid document_id FK
        int seq
        text content_masked "색인 전 PII 마스킹 본 (9.1-4)"
        vector embedding
        uuid embedding_model_id FK
        tsvector keyword_index "하이브리드 검색"
    }
    retrieval_logs {
        uuid id PK
        uuid tenant_id FK
        uuid user_id FK
        text query
        jsonb applied_filters "권한 사전 필터 검증 근거 (6.5-4)"
        jsonb result_chunk_ids
        bool no_evidence "검색 실패 (FR-RAG-15)"
    }
    ai_answer_logs {
        uuid id PK
        uuid tenant_id FK
        uuid user_id FK
        uuid retrieval_log_id FK
        uuid prompt_template_id FK
        text question
        jsonb answer "7.7의 11필드 구조화 응답"
        string confidence "high/mid/low — 규칙 산정 (9.3)"
        string no_judgement_reason "근거미검색/권한부족/증적만료/증적상충"
        string user_action "approved/edited/rejected"
        string classification "로그 자체 접근통제 (FR-SEC-15)"
    }
    ai_citation_logs {
        uuid id PK
        uuid answer_log_id FK
        uuid chunk_id FK
        uuid document_id
        int document_version "근거 포함률 측정 원천 (13.2-1)"
    }
    prompt_templates {
        uuid id PK
        uuid tenant_id "NULL=시스템 기본"
        string name
        int version "변경 시 회귀 평가 (9.5-3)"
        text body
    }
    ai_feedback {
        uuid id PK
        uuid answer_log_id FK
        uuid user_id FK
        string rating
        text comment "평가 데이터 환류 (9.5-5)"
    }
```

### 2.7 미흡사항 / 개선계획 / SoA / 보고서 (FR-GAP, FR-PLN, FR-SOA, FR-RPT)

```mermaid
erDiagram
    assessment_projects ||--o{ gaps : ""
    controls ||--o{ gaps : ""
    gaps ||--o{ action_plans : ""
    action_plans ||--o{ action_items : ""
    assessment_projects ||--o{ soa_items : ""
    controls ||--o{ soa_items : ""
    assessment_projects ||--o{ reports : ""
    reports ||--|| report_snapshots : ""

    gaps {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        uuid control_id FK
        string gap_type "7.8의 12종"
        string source "assessment/ai/manual — 모두 담당자 확정 등록 (FR-GAP-01)"
        string risk_level "상/중/하 수동 입력 (MVP)"
        uuid assignee_id FK
        date due_date
        string status
        uuid recurrence_of "재발 연결 (FR-GAP-12)"
        uuid vuln_finding_ref "VULN 모듈 논리 참조 (Phase 2, FK 아님)"
    }
    action_plans {
        uuid id PK
        uuid tenant_id FK
        uuid gap_id FK
        text cause
        text improvement
        uuid assignee_id FK
        date target_date
        string priority
        text completion_criteria
        string workflow_status
        uuid ai_answer_log_id "AI 초안 출처 (FR-PLN-01)"
        int version
    }
    action_items {
        uuid id PK
        uuid action_plan_id FK
        text task
        string status
        date done_at
    }
    soa_items {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        uuid control_id FK
        string applicability "적용/미적용/부분적용/검토중"
        text justification "적용 근거/제외 사유 (MVP 기본 SoA)"
        uuid risk_ref "위험 연결 (Phase 3 고급)"
        int version
        string workflow_status
    }
    reports {
        uuid id PK
        uuid tenant_id FK
        uuid project_id FK
        string report_type "자체점검/미흡목록/보완계획/보완내역/경영진/SoA"
        string format "pdf/docx"
        uuid snapshot_id FK
        string workflow_status
    }
    report_snapshots {
        uuid id PK
        uuid tenant_id FK
        timestamptz captured_at "보고서 표기 (NFR-AUD-03)"
        jsonb data
    }
```

### 2.8 승인 / 알림 / 감사 (FR-WFL, FR-NTF, FR-LOG)

```mermaid
erDiagram
    approvals {
        uuid id PK
        uuid tenant_id FK
        string target_type "assessment/evidence/action_plan/report/soa_item"
        uuid target_id
        uuid requester_id FK
        uuid approver_id FK
        string decision "approved/rejected"
        text comment "반려 사유 필수"
        timestamptz decided_at
    }
    notifications {
        uuid id PK
        uuid tenant_id FK
        uuid user_id FK
        string ntf_type "FR-NTF-01~07"
        string channel "inapp/email (MVP)"
        string target_type
        uuid target_id
        timestamptz read_at
    }
    notification_prefs {
        uuid id PK
        uuid user_id FK
        string ntf_type
        string channel
        bool enabled "FR-NTF-08"
    }
    audit_logs {
        uuid id PK
        uuid tenant_id FK
        uuid actor_id
        string action "FR-LOG-01~16"
        string target_type
        uuid target_id
        jsonb before_value
        jsonb after_value
        string prev_hash
        string row_hash "해시체인 (FR-SEC-13)"
        timestamptz created_at
    }
```

- `approvals`는 MVP 단일 단계 — P1 다단계 확장 시 `step_no`, `delegation` 칼럼 추가 여지를 둔다(7.16 승인 범위).
- `audit_logs`는 append-only(UPDATE/DELETE 권한 회수) + 파티셔닝(보관기간 정책, 7.19).

### 2.9 취약점 진단 모듈 (VULN 애드온 — FR-AST, FR-VUL, Phase 2)

```mermaid
erDiagram
    assets ||--o{ vulnerability_scans : ""
    vulnerability_scans ||--o{ vulnerability_findings : ""
    assets ||--o{ asset_control_links : ""
    vulnerability_findings ||--o{ vulnerability_control_links : ""
    scan_consents ||--o{ vulnerability_scans : ""

    assets {
        uuid id PK
        uuid tenant_id FK
        string asset_type "7.11의 11종"
        string name
        string importance
        bool internet_exposed
        bool handles_pii
        uuid owner_user_ref "코어 users 논리 참조 (FK 아님)"
        uuid system_ref "코어 systems 논리 참조"
    }
    scan_consents {
        uuid id PK
        uuid tenant_id FK
        string target "도메인/IP 대역"
        string ownership_proof "DNS TXT/파일 검증 (7.12 SaaS 동의)"
        string status "verified/pending/revoked"
        uuid approved_by
        date expires_at
    }
    vulnerability_scans {
        uuid id PK
        uuid tenant_id FK
        uuid asset_id FK
        uuid consent_id FK "검증 없으면 실행 차단"
        string tool "nmap/openvas/zap/trivy/manual"
        string scan_window "스캔 시간대"
        string status
    }
    vulnerability_findings {
        uuid id PK
        uuid tenant_id FK
        uuid scan_id FK
        string cve_id
        numeric cvss_score
        bool kev_listed
        string severity
        int priority_score "7.12 우선순위 산정"
        string status "open/fixed/accepted/false_positive"
    }
    vulnerability_control_links {
        uuid id PK
        uuid finding_id FK
        uuid control_ref "코어 controls 논리 참조"
        string mapping_source "ai_suggested/confirmed (추천+확정)"
        uuid confirmed_by
    }
    asset_control_links {
        uuid id PK
        uuid asset_id FK
        uuid control_ref "코어 controls 논리 참조"
    }
```

- 이 도메인 전체가 VULN 구독 게이팅 대상: 미구독 테넌트는 API·메뉴·배치에서 차단(FR-TEN-04), `usage_records.scan_runs`로 사용량 과금.
- 코어와의 연결은 전부 `*_ref` 논리 참조 — 모듈 분리 배포 시 서비스 간 API 참조로 전환 가능.

---

## 3. RLS 정책 요약

| 대상 | 정책 |
|---|---|
| 모든 업무 테이블 | `tenant_id = current_setting('app.tenant_id')::uuid` 강제 |
| standards 계열, assessment_types, prompt_templates | 위 조건 OR `tenant_id IS NULL`(시스템 콘텐츠 읽기 전용) |
| plans, modules, embedding_models | 플랫폼 전역(테넌트 정책 미적용, 쓰기는 플랫폼 관리자만) |
| document_chunks 벡터 검색 | RLS + 질의 시 `tenant_id`·ACL 사전 필터 이중 적용 (6.5) |
| audit_logs | INSERT만 허용(append-only) |

## 4. PRD 10장과의 대응

PRD 10.1의 47개 테이블 중 `knowledge_sources`는 `documents.source_type/source_id`로 흡수, `embeddings`는 `document_chunks` 내장 칼럼으로 구현(임베딩 모델 버전 포함), `integrations`(P1)와 `risks/risk_treatments/exceptions`(Phase 3)는 해당 Phase 설계 시 추가한다. 본 ERD는 MVP+Phase 2(VULN) 범위의 물리 설계 초안이며, 신규 추가 테이블(`evidence_versions`, `evidence_links`, `scan_consents`, `notification_prefs`, `tenant_settings`, `user_project_scopes` 등)은 PRD 요구사항의 구현 필요에 따른 상세화다.

## 5. 개발 착수 시 권장 순서 (ISO27001 우선)

1. 플랫폼 골격: tenants/modules/subscriptions + RLS + 게이팅 미들웨어 (이후 모든 기능의 전제)
2. ISO/IEC 27001:2022 시드: standards→controls 93개→check_items→evidence_requirements, 심사 유형 프로파일 3종
3. 코어 업무 흐름: 프로젝트 생성→체크리스트→증적→승인 (스테퍼 포함)
4. RAG/AI: 색인 파이프라인→검색→응답/인용 로그
5. 미흡사항→개선계획→SoA→보고서
6. Phase 2: VULN 모듈 (scan_consents 소유권 검증부터)
