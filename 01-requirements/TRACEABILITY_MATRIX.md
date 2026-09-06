# Traceability Matrix — taskq-api

> Requirements Traceability Matrix(RTM)
> Framework: harness-methodology · Version: v1.0 · Created: 2026-09-07 · Phase: 1
> 需求來源:repo 根 SPEC.md v1.0.0(單一事實來源)→ 01-requirements/SRS.md(逐字轉錄)
> 測試命名權威:TEST_INVENTORY.yaml(P1 Naming Authority)→ 02-architecture/TEST_SPEC.md(P2 單一事實來源)

---

## 1. Overview

本矩陣建立 **SPEC → SRS → Design(SAD/ADR)→ Test(TEST_INVENTORY/TEST_SPEC)→ TEST_PLAN** 之雙向追蹤,
支援 ASPICE SWE.3 / SYS.4 追溯要求。雙向鏈:

1. **正向(Spec → Test)**:SPEC.md 條款 → SRS.md FR/NFR 與 AC → 02-architecture/SAD.md(FR 設計)/ 02-architecture/ADR.md(NFR 決策)→ TEST_INVENTORY.yaml 已命名測試 或 02-architecture/TEST_SPEC.md P2 派生 → 04-testing/TEST_PLAN.md 執行。
2. **反向(Test → Spec)**:每個已命名測試必須可回溯至單一 FR/NFR 與其 AC(§5);每個設計模組必須註記其滿足之 FR/NFR(NFR-05 AC-N5.1)。

### 1.1 Status Legend(與 SPEC_TRACKING.md 一致;VERIFIED 依 SRS NFR-09 AC-N9.6 加嚴)

| Status | Meaning |
|--------|---------|
| DRAFT | 需求已轉錄,尚未有對應程式碼(本階段初始狀態) |
| IN_PROGRESS | 程式碼/模組已存在 — build_traceability 掃描設定 |
| VERIFIED | 測試實際執行且通過(依 SRS NFR-09 AC-N9.6:僅在實測通過後給出,禁止預先標記) |

本檔全表 Status 目前一律 **DRAFT**:03-development/src 與 03-development/tests 均為空目錄(Phase 1),無任何測試已執行。

---

## 2. FR / NFR ↔ Spec Mapping

### 2.1 Functional Requirements(10)

| FR ID | Functional Requirement | SRS Section | Intent Class | Status |
|-------|------------------------|-------------|--------------|--------|
| FR-01 | 任務資源 CRUD API(POST /v1/tasks、GET /v1/tasks/{id}、GET /v1/tasks 分頁、DELETE /v1/tasks/{id}) | SRS.md §3 FR-01 | API — CRUD | DRAFT |
| FR-02 | 任務執行端點(POST /v1/tasks/{id}/run → 202+run_id;狀態機;結果寫 task_results;GET .../runs) | SRS.md §3 FR-02 | API — Execution | DRAFT |
| FR-03 | API Key 認證(X-API-Key;SHA-256 雜湊;hmac.compare_digest;revoked_at) | SRS.md §3 FR-03 | Security — Authentication | DRAFT |
| FR-04 | Scope 授權(read < write < admin;403 不洩漏存在性;單一 dependency) | SRS.md §3 FR-04 | Security — Authorization | DRAFT |
| FR-05 | 流量控制(per-token 令牌桶;429+Retry-After;DB 持久化 row-level lock) | SRS.md §3 FR-05 | Traffic Control | DRAFT |
| FR-06 | 持久化層與交易邊界(repository/ 層;commit/rollback context manager;N+1 防護) | SRS.md §3 FR-06 | Persistence — Transaction | DRAFT |
| FR-07 | Schema Migration(Alembic v1→v2→v3 三步演進,每步可逆 downgrade,含資料搬遷) | SRS.md §3 FR-07 | Schema Migration | DRAFT |
| FR-08 | 非同步執行器(asyncio.TaskGroup;graceful drain;併發上限;timeout 殺子進程) | SRS.md §3 FR-08 | Concurrency — Async | DRAFT |
| FR-09 | 健康檢查與可觀測性(/healthz;/readyz fail closed;/v1/metrics) | SRS.md §3 FR-09 | Observability | DRAFT |
| FR-10 | 錯誤契約(RFC 7807 problem+json;correlation_id;錯誤碼對照) | SRS.md §3 FR-10 | Error Contract | DRAFT |

### 2.2 Non-Functional Requirements(12)

| NFR ID | Non-Functional Requirement | SRS Section | Dimension | Status |
|--------|---------------------------|-------------|-----------|--------|
| NFR-01 | 效能與查詢效率(p95 < 30/80ms;SQL 陳述數常數) | SRS.md §4 NFR-01 | performance | DRAFT |
| NFR-02 | HTTP 與資料層安全(禁 shell=True/eval/exec;禁 SQL 拼接;bandit 0/0) | SRS.md §4 NFR-02 | security | DRAFT |
| NFR-03 | 錯誤處理、交易與非同步正確性(CancelledError 不吞;孤兒進程防護) | SRS.md §4 NFR-03 | reliability | DRAFT |
| NFR-04 | 敏感資料遮蔽([REDACTED] regex;DB 連線字串不落盤) | SRS.md §4 NFR-04 | security | DRAFT |
| NFR-05 | 文件覆蓋(docstring [FR-XX] 引用 100%;OpenAPI summary/description) | SRS.md §4 NFR-05 | documentation | DRAFT |
| NFR-06 | 架構分層契約(.importlinter;repository 外禁 import sqlalchemy) | SRS.md §4 NFR-06 | layering | DRAFT |
| NFR-07 | 依賴與授權合規(== 釘版;license allowlist;SBOM) | SRS.md §4 NFR-07 | licensing | DRAFT |
| NFR-08 | 變異測試(mutation score ≥ 70;範圍 service/ + repository/) | SRS.md §4 NFR-08 | mutation | DRAFT |
| NFR-09 | 驗證真實性(零 skip 鐵律;zero_assert==0;真實 DB 測 migration) | SRS.md §4 NFR-09 | testability | DRAFT |
| NFR-10 | 整合覆蓋(integration ≥ 80%;ASGITransport 驅動;錯誤碼各一例) | SRS.md §4 NFR-10 | integration | DRAFT |
| NFR-11 | 可讀性(MI ≥ 80;CC ≤ 10;檔案 ≤ 400 行;handler ≤ 40 行) | SRS.md §4 NFR-11 | maintainability | DRAFT |
| NFR-12 | 系統驗證目標(make verify-system;.env.example 12 變數) | SRS.md §4 NFR-12 | verifiability | DRAFT |

---

## 3. Spec ↔ Design Mapping

Design 歸屬遵循 SPEC_TRACKING.md 所有權慣例:FR 實現設計 → 02-architecture/SAD.md;NFR 決策 → 02-architecture/ADR.md。Planned Module 取自 SRS.md FR Block(`implementation_functions`),為 Phase 2 實作目標,非既有程式碼。

| Req ID | Design Owner | Planned Module(s)(P2) | Code Status |
|--------|--------------|------------------------|-------------|
| FR-01 | 02-architecture/SAD.md | taskq_api.api.routes.tasks、taskq_api.service.task_service、taskq_api.repository.task_repo | NOT_STARTED |
| FR-02 | 02-architecture/SAD.md | taskq_api.api.routes.tasks、taskq_api.service.runner | NOT_STARTED |
| FR-03 | 02-architecture/SAD.md | taskq_api.service.auth、taskq_api.api.dependencies | NOT_STARTED |
| FR-04 | 02-architecture/SAD.md | taskq_api.api.dependencies | NOT_STARTED |
| FR-05 | 02-architecture/SAD.md | taskq_api.api.dependencies、taskq_api.repository.rate_buckets | NOT_STARTED |
| FR-06 | 02-architecture/SAD.md | taskq_api.repository.session、taskq_api.repository.task_repo | NOT_STARTED |
| FR-07 | 02-architecture/SAD.md | migrations/versions/v1_init.py、v2_tags.py、v3_split_results.py | NOT_STARTED |
| FR-08 | 02-architecture/SAD.md | taskq_api.service.runner | NOT_STARTED |
| FR-09 | 02-architecture/SAD.md | taskq_api.api.routes.health、taskq_api.api.routes.metrics | NOT_STARTED |
| FR-10 | 02-architecture/SAD.md | taskq_api.errors | NOT_STARTED |
| NFR-01 | 02-architecture/ADR.md | SQLAlchemy event listener(測試側)、pytest-benchmark | NOT_STARTED |
| NFR-02 | 02-architecture/ADR.md | grep gates(§8 #16/#17)、bandit、CORS middleware | NOT_STARTED |
| NFR-03 | 02-architecture/ADR.md | context manager 交易、runner 終止流程 | NOT_STARTED |
| NFR-04 | 02-architecture/ADR.md | 遮蔽函式(整行 [REDACTED] 取代) | NOT_STARTED |
| NFR-05 | 02-architecture/ADR.md | docstring 規範、FastAPI route 中繼資料 | NOT_STARTED |
| NFR-06 | 02-architecture/ADR.md | .importlinter(根目錄,layers + forbidden contract) | NOT_STARTED |
| NFR-07 | 02-architecture/ADR.md | requirements.txt/requirements.lock、08-config/SBOM.json | NOT_STARTED |
| NFR-08 | 02-architecture/ADR.md | .methodology/harness_config.json(features.mutation_testing) | NOT_STARTED |
| NFR-09 | 02-architecture/ADR.md | 測試設計規範(反造假條款) | NOT_STARTED |
| NFR-10 | 02-architecture/ADR.md | httpx.AsyncClient(transport=ASGITransport(app))測試驅動 | NOT_STARTED |
| NFR-11 | 02-architecture/ADR.md | 行數/複雜度紀律(radon mi/cc/raw) | NOT_STARTED |
| NFR-12 | 02-architecture/ADR.md | Makefile `verify-system` target | NOT_STARTED |

反向約束:每個 SAD.md/ADR.md 設計元素必須標註其滿足之 `[FR-XX]`/`[NFR-XX]`(NFR-05 AC-N5.1);repository 以外任何層不得 import sqlalchemy(NFR-06 AC-N6.2)。

---

## 4. Spec ↔ Test Mapping(正向)

已命名測試 = TEST_INVENTORY.yaml `test_inventory.tests`(13 筆,P1 Naming Authority,權威名稱)。
其餘 10 個 NFR 需求之測試名稱依 TEST_INVENTORY.yaml 前言(「finer per-AC granularity 留待 P2」)由 P2 derive_test_cases.md 派生並寫入 02-architecture/TEST_SPEC.md —— 本檔不發明測試名稱。

### 4.1 Functional Requirements → Tests

| FR ID | ACs | Named Test(s) | Layer / 預期位置 | Verification Method(SRS) | Status |
|-------|-----|---------------|------------------|--------------------------|--------|
| FR-01 | AC-1.1–AC-1.7 | TC-FR01-01、TC-FR01-02 | integration / 03-development/tests/integration/test_fr01.py;unit / 03-development/tests/unit/test_fr01.py | pytest integration(SPEC.md §8 #4/#7/#8;NFR-10 CRUD 全鏈) | DRAFT |
| FR-02 | AC-2.1–AC-2.5 | TC-FR02-01 | integration / 03-development/tests/integration/test_fr02.py | pytest integration + grep gate(SPEC.md §8 #16/#25;§11 孤兒進程) | DRAFT |
| FR-03 | AC-3.1–AC-3.5 | TC-FR03-01 | integration / 03-development/tests/integration/test_fr03.py | pytest(SPEC.md §8 #5/#18;查 api_keys 表斷言) | DRAFT |
| FR-04 | AC-4.1–AC-4.3 | TC-FR04-01 | integration / 03-development/tests/integration/test_fr04.py | pytest integration(SPEC.md §8 #6;單一 dependency 斷言) | DRAFT |
| FR-05 | AC-5.1–AC-5.4 | TC-FR05-01 | integration / 03-development/tests/integration/test_fr05.py | pytest integration(SPEC.md §8 #9;429+Retry-After) | DRAFT |
| FR-06 | AC-6.1–AC-6.5 | TC-FR06-01 | unit / 03-development/tests/unit/test_fr06.py | code review + SQLAlchemy event listener 計數(SPEC.md §8 #14) | DRAFT |
| FR-07 | AC-7.1–AC-7.6 | TC-FR07-01 | integration / 03-development/tests/integration/test_fr07.py | 真實 SQLite 檔案往返 + alembic offline SQL 斷言(SPEC.md §8 #12/#13;NFR-09 AC-N9.5) | DRAFT |
| FR-08 | AC-8.1–AC-8.4 | TC-FR08-01 | integration / 03-development/tests/integration/test_fr08.py | pytest integration(SPEC.md §8 #25;§11 孤兒進程) | DRAFT |
| FR-09 | AC-9.1–AC-9.4 | TC-FR09-01 | integration / 03-development/tests/integration/test_fr09.py | pytest(SPEC.md §8 #10/#11)+ 服務啟動冒煙(§8 #27 第 3 步) | DRAFT |
| FR-10 | AC-10.1–AC-10.5 | TC-FR10-01 | integration / 03-development/tests/integration/test_fr10.py | pytest integration(SPEC.md §8 #19;§11 錯誤 body 洩漏) | DRAFT |

### 4.2 Non-Functional Requirements → Tests

| NFR ID | ACs | Named Test(s) | Layer / 預期位置 | Verification Method(SRS) | Status |
|--------|-----|---------------|------------------|--------------------------|--------|
| NFR-01 | AC-N1.1–AC-N1.3 | (P2 派生) | benchmark+integration | pytest-benchmark + event listener 計數(SPEC.md §8 #14/#15);p95 門檻需 Phase 3 專屬 benchmark 測試(SRS coverage note) | DRAFT |
| NFR-02 | AC-N2.1–AC-N2.7 | TC-N02-01 | static / 03-development/tests/static/test_nfr02.py | bandit + grep gates + code review(SPEC.md §8 #16/#17/#23) | DRAFT |
| NFR-03 | AC-N3.1–AC-N3.6 | (P2 派生) | unit+integration | ast-error-handling 掃描 + integration(SPEC.md §8 #10/#25);交易/readyz/孤兒進程需 Phase 3 專屬測試(SRS coverage note) | DRAFT |
| NFR-04 | AC-N4.1–AC-N4.3 | (P2 派生) | unit / 03-development/tests/unit/ | unit + integration(SPEC.md §8 #20;§11 DB 連線字串) | DRAFT |
| NFR-05 | AC-N5.1–AC-N5.2 | (P2 派生) | static | ast-docstrings 掃描 + /openapi.json 斷言測試 | DRAFT |
| NFR-06 | AC-N6.1–AC-N6.4 | (P2 派生) | static | lint-imports(SPEC.md §8 #21)+ .importlinter 內容斷言測試 | DRAFT |
| NFR-07 | AC-N7.1–AC-N7.4 | (P2 派生) | static+e2e | pip-licenses --format=json --with-system + SBOM 檢查(SPEC.md §8 #22) | DRAFT |
| NFR-08 | AC-N8.1–AC-N8.4 | (P2 派生) | mutation | mutation-test-score / mutmut results(SPEC.md §8 #24);config 內容需 Phase 3 專屬檢查 | DRAFT |
| NFR-09 | AC-N9.1–AC-N9.6 | TC-N09-01 | integration / 03-development/tests/integration/test_nfr09.py | pytest -q skipped 計數 + ast-assertions(SPEC.md §8 #1);反造假 audit 需 Phase 3 專屬檢查 | DRAFT |
| NFR-10 | AC-N10.1–AC-N10.4 | (P2 派生) | integration | pytest --cov(SPEC.md §8 #2/#3);ASGI 驅動方式與情境清單需 Phase 3 專屬測試 | DRAFT |
| NFR-11 | AC-N11.1–AC-N11.4 | (P2 派生) | static | radon mi/cc/raw(SPEC.md §11);CC/行數上限需 Phase 3 專屬檢查 | DRAFT |
| NFR-12 | AC-N12.1–AC-N12.3 | (P2 派生) | e2e | make verify-system(SPEC.md §8 #27)+ grep -c ^TASKQ_ .env.example(§8 #26) | DRAFT |

---

## 5. Test ↔ Spec Mapping(反向:已命名測試)

| TC ID | Layer | Test Function(權威名稱) | Test File | Verifies | AC | Cross Refs |
|-------|-------|-------------------------|-----------|----------|-----|------------|
| TC-FR01-01 | integration | test_fr01_example_integration | 03-development/tests/integration/test_fr01.py | FR-01 | AC-1.1 | — |
| TC-FR01-02 | unit | test_fr01_example_unit | 03-development/tests/unit/test_fr01.py | FR-01 | AC-1.2 | NFR-02 |
| TC-FR02-01 | integration | test_fr02_run_execution | 03-development/tests/integration/test_fr02.py | FR-02 | AC-2.1 | — |
| TC-FR03-01 | integration | test_fr03_api_key_auth | 03-development/tests/integration/test_fr03.py | FR-03 | AC-3.1 | — |
| TC-FR04-01 | integration | test_fr04_scope_authorization | 03-development/tests/integration/test_fr04.py | FR-04 | AC-4.1 | — |
| TC-FR05-01 | integration | test_fr05_rate_limiting | 03-development/tests/integration/test_fr05.py | FR-05 | AC-5.1 | — |
| TC-FR06-01 | unit | test_fr06_transaction_boundaries | 03-development/tests/unit/test_fr06.py | FR-06 | AC-6.1 | — |
| TC-FR07-01 | integration | test_fr07_migration_roundtrip | 03-development/tests/integration/test_fr07.py | FR-07 | AC-7.1 | — |
| TC-FR08-01 | integration | test_fr08_async_runner | 03-development/tests/integration/test_fr08.py | FR-08 | AC-8.1 | — |
| TC-FR09-01 | integration | test_fr09_health_checks | 03-development/tests/integration/test_fr09.py | FR-09 | AC-9.1 | — |
| TC-FR10-01 | integration | test_fr10_problem_details | 03-development/tests/integration/test_fr10.py | FR-10 | AC-10.1 | — |
| TC-N02-01 | static | test_security_example | 03-development/tests/static/test_nfr02.py | NFR-02 | AC-N2.1 | — |
| TC-N09-01 | integration | test_deployment_example | 03-development/tests/integration/test_nfr09.py | NFR-09 | AC-N9.1 | — |

對照 TEST_INVENTORY.yaml `fr_tests` / `cross_cutting` 區塊:上表 13 個函式名稱與其一致(FR-01~FR-10: unit×2 + integration×9;cross_cutting: security×1 + deployment×1)。其餘 10 個 NFR(NFR-01、NFR-03~NFR-08、NFR-10~NFR-12)之測試名稱留待 P2 derive_test_cases.md 依 7-Question Protocol 派生,並同步回填 TEST_INVENTORY.yaml 與 02-architecture/TEST_SPEC.md(該檔為 P2 起之單一事實來源)。

---

## 6. Error-Code Cross-Cutting Coverage

NFR-10 AC-N10.3 要求 401/403/404/409/422/429/503 各至少一例整合測試。正向來源對照(SRS.md AC-10.5):

| HTTP Code | 觸發情境 | 主 AC | 相關 AC |
|-----------|----------|-------|---------|
| 422 | 驗證失敗 / limit 超上限 | AC-1.2、AC-1.5 | — |
| 401 | 缺/無效 X-API-Key | AC-3.1 | AC-3.4(revoked) |
| 403 | scope 不足,body 不洩漏存在性 | AC-4.2 | AC-N2.4 |
| 404 | 未知 id | AC-1.7 | — |
| 409 | 重複 name | AC-1.2 | — |
| 429 | 超過 TASKQ_RATE_BURST | AC-5.2 | — |
| 503 | /readyz DB 不可用 / migration 未到 head | AC-9.2、AC-9.3、AC-N3.4 | — |

每個錯誤碼之測試名稱由 P2 派生(TEST_SPEC.md);NFR-10 另要求 rate limit 觸發與恢復、graceful drain、migration 往返情境。

---

## 7. Risk Traceability

風險矩陣取自 SRS.md §8(SPEC.md §9 L387-402;R13 為 DERIVED,源自 SPEC.md L429 §10 async 條款):

| Risk | 描述 | 關聯需求 | 緩解測試位置 |
|------|------|----------|--------------|
| R1 | v3 資料搬遷遺失資料 | FR-07 | 往返可逆性真實 DB 測試(§8 #12) |
| R2 | SQL injection | NFR-02、FR-06 | 禁拼接 + ORM/參數化 + grep gate(§8 #17) |
| R3 | API key 洩漏 | FR-03 | 雜湊儲存 + 常數時間比對(§8 #18) |
| R4 | 403 洩漏資源存在性 | FR-04 | 授權判定在資源查詢之前(§8 #6) |
| R5 | N+1 查詢大表崩潰 | NFR-01、FR-06 | 顯式預載 + SQL 計數斷言(§8 #14) |
| R6 | 錯誤 body 洩漏內部結構 | FR-10 | RFC 7807 固定欄位 + detail 白名單(§8 #19) |
| R7 | CancelledError 被吞 | NFR-03 | 明文禁令 + 測試斷言 re-raise |
| R8 | timeout 孤兒進程 | FR-08 | kill() + await wait()(§8 #25) |
| R9 | 部署後忘記 migration | FR-09 | /readyz fail closed(§8 #11) |
| R10 | 連線池耗盡 | FR-06、FR-08 | pool_pre_ping + 併發上限 |
| R11 | transitive license 不合規 | NFR-07 | lock 檔 + 全樹掃描(§8 #22) |
| R12 | rate bucket 競態超放行 | FR-05 | 單一交易 row-level lock(§8 #9) |
| R13 | async 使 framework 掃描器誤判(DERIVED:SPEC.md L429) | NFR-99-2 | 記入 Phase 4 bug hunt,不得靜默繞過 |

---

## 8. Completeness Verification

| Check | Target | Actual | Status |
|-------|--------|--------|--------|
| FR → SRS mapping | 100% | 10/10 FR、12/12 NFR 已映射(SRS.md FR Block) | COMPLETE(Phase 1) |
| Spec → Design mapping | 100% | 22/22 已指定 owner(SAD.md/ADR.md);模組清單已列 | PLANNED(設計文件仍為模板,待 P2 填實) |
| Spec → Test naming | 100% | 12/22 有已命名測試(FR-01~FR-10、NFR-02、NFR-09);10/22(NFR-01、NFR-03~NFR-08、NFR-10~NFR-12)依 TEST_INVENTORY.yaml 前言由 P2 派生 | PARTIAL(依 P1 Naming Authority 設計) |
| Test execution / VERIFIED | 全部通過 | 0/22 — 03-development/src、tests 均為空;無任何測試已執行 | NOT STARTED(Phase 3) |
| Test coverage(integration) | ≥ 80%(P3: ≥ 70%) | 未量測(無程式碼) | PENDING |
| Test coverage(TOTAL) | 100% | 未量測(無程式碼) | PENDING |
| VERIFIED 標記合規 | AC-N9.6 | 本檔 0 個 VERIFIED(符合:測試未執行) | COMPLIANT |

---

## 9. ASPICE Compliance

| ASPICE Capability | Status | 證據 |
|-------------------|--------|------|
| SWE.3.B.SP1 Task-to-work-product traceability | IN_PROGRESS | SPEC_TRACKING.md(已 APPROVED)+ 本檔 §2/§4 建立需求→工作產物鏈 |
| SWE.3.B.SP2 Bidirectional traceability | IN_PROGRESS | 本檔 §4(正向)+ §5(反向)+ §3(設計反向約束) |
| SWE.3.B.SP3 Traceability consistency | PLANNED | 一致性驗證需 P3 測試實測後由 05-verification/BASELINE.md 對照;現階段以 DRAFT 狀態避免虛假 VERIFIED |

---

## 10. Gap Analysis

- **G-1** 已命名測試 13 筆(FR-01~FR-10 共 11 筆、NFR-02×1、NFR-09×1)。其餘 10 個需求(NFR-01、NFR-03~NFR-08、NFR-10~NFR-12)之測試名稱須由 P2 derive_test_cases.md 派生後回填 02-architecture/TEST_SPEC.md 與本檔 §4/§5。
- **G-2** SRS coverage notes 標記多項 AC 無法由 harness 維度直接驗證,需 Phase 3 專屬測試:p95 門檻(NFR-01)、grep gates 與 SQL 拼接掃描(NFR-02)、交易邊界/readyz 503/孤兒進程/migration rollback(NFR-03)、遮蔽 regex(NFR-04)、docstring 引用內容與 OpenAPI(NFR-05)、.importlinter 內容(NFR-06)、SBOM(NFR-07)、config 內容(NFR-08)、反造假 audit(NFR-09)、ASGI 驅動方式(NFR-10)、CC/行數(NFR-11)、verify-system 內容(NFR-12)。
- **G-3** Open Issues:AC-1.2 之「注入字元黑名單」字元集待 NFR-99-1 解決後方可定稿對應測試;NFR-99-2 影響 async 掃描器誤判之處理。
- **G-4** 02-architecture/SAD.md、TEST_SPEC.md 目前為模板;其模組設計與測試名稱派生完成後,本檔 §3/§4 之 Planned 欄位須同步更新。

---

## 11. Machine-Readable Block

<!-- RTM:START -->
```json
{
  "version": "1.0",
  "phase": 1,
  "project": "taskq-api",
  "sources": {"spec": "SPEC.md v1.0.0", "srs": "01-requirements/SRS.md", "test_names": "TEST_INVENTORY.yaml", "tracking": "01-requirements/SPEC_TRACKING.md"},
  "totals": {"fr": 10, "nfr": 12, "named_tests": 13, "verified": 0},
  "traceability": {
    "fr_design_owner": "02-architecture/SAD.md",
    "nfr_design_owner": "02-architecture/ADR.md",
    "test_plan_owner": "04-testing/TEST_PLAN.md",
    "test_spec_owner": "02-architecture/TEST_SPEC.md"
  },
  "named_test_cases": [
    {"tc_id": "TC-FR01-01", "req": "FR-01", "ac": "AC-1.1", "layer": "integration"},
    {"tc_id": "TC-FR01-02", "req": "FR-01", "ac": "AC-1.2", "layer": "unit", "cross_ref_nfrs": ["NFR-02"]},
    {"tc_id": "TC-FR02-01", "req": "FR-02", "ac": "AC-2.1", "layer": "integration"},
    {"tc_id": "TC-FR03-01", "req": "FR-03", "ac": "AC-3.1", "layer": "integration"},
    {"tc_id": "TC-FR04-01", "req": "FR-04", "ac": "AC-4.1", "layer": "integration"},
    {"tc_id": "TC-FR05-01", "req": "FR-05", "ac": "AC-5.1", "layer": "integration"},
    {"tc_id": "TC-FR06-01", "req": "FR-06", "ac": "AC-6.1", "layer": "unit"},
    {"tc_id": "TC-FR07-01", "req": "FR-07", "ac": "AC-7.1", "layer": "integration"},
    {"tc_id": "TC-FR08-01", "req": "FR-08", "ac": "AC-8.1", "layer": "integration"},
    {"tc_id": "TC-FR09-01", "req": "FR-09", "ac": "AC-9.1", "layer": "integration"},
    {"tc_id": "TC-FR10-01", "req": "FR-10", "ac": "AC-10.1", "layer": "integration"},
    {"tc_id": "TC-N02-01", "req": "NFR-02", "ac": "AC-N2.1", "layer": "static"},
    {"tc_id": "TC-N09-01", "req": "NFR-09", "ac": "AC-N9.1", "layer": "integration"}
  ]
}
```
<!-- RTM:END -->
