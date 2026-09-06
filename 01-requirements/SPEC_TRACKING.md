# Specification Tracking Matrix — taskq-api

> 追蹤矩陣,需求集 100% 轉錄自 SRS.md(其內容逐條溯源至 repo 根 SPEC.md v1.0.0)。
> 本檔為人類可讀視圖;Status 為機器刷新(見下),計分權威為 quality_manifest.json。

## Project Info
- Project Name: taskq-api
- Version: v1.0.0
- Created: 2026-09-07
- Canonical Spec Source: SPEC.md(repo 根,單一事實來源;SRS.md 為逐字轉錄)

## Status Legend

| Status | Meaning |
|--------|---------|
| DRAFT | 需求已轉錄,尚未有對應程式碼(本階段初始狀態,人工填寫) |
| IN_PROGRESS | 程式碼/模組已存在 — build_traceability 掃描設定 |
| VERIFIED | 程式碼與測試皆存在 — build_traceability 掃描設定 |

## Specification Status

> **The Status column is machine-refreshed** — `advance-phase` overwrites each
> FR's Status from `build_traceability`'s live code/test scan (IN_PROGRESS once
> code/module exists, VERIFIED once code+test exist). The authoritative status is
> that scan / `quality_manifest.json`, NOT this hand-filled cell. Fill the
> semantic columns (Spec Description / Intent Class / Decision Framework / Notes);
> leave Status to refresh itself (a hand-edit is overwritten on the next advance).

Ownership convention(Notes 欄):FR 之實現設計歸屬 02-architecture/SAD.md、驗證歸屬
04-testing/TEST_PLAN.md;NFR 另經 02-architecture/ADR.md 覆蓋。所有 Source 引用
一律指向 repo 根 SPEC.md(行號)與 SRS.md(章節),無其他來源。

| FR ID | Spec Description | Intent Class | Decision Framework | Status | Notes |
|-------|-----------------|--------------|-------------------|--------|-------|
| FR-01 | 任務資源 CRUD API:POST /v1/tasks(write,201)、GET /v1/tasks/{id}(read,404 未知 id)、GET /v1/tasks 分頁列表(read,cursor-based,limit 預設 50 上限 200)、DELETE /v1/tasks/{id}(admin,連同結果列同一交易);422 驗證失敗、409 重複 name | API — CRUD | AC-1.1–AC-1.7;pytest 03-development/tests/integration(SPEC.md §8 #4/#7/#8) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L79-91、SRS.md FR-01;impl: taskq_api.api.routes.tasks、taskq_api.service.task_service、taskq_api.repository.task_repo |
| FR-02 | 任務執行端點:POST /v1/tasks/{id}/run(write)→202+run_id;asyncio.create_subprocess_exec(*shlex.split(command)) 執行,禁 shell=True,timeout=TASKQ_TASK_TIMEOUT;狀態機 pending→running→done/failed/timeout;結果寫 task_results;GET /v1/tasks/{id}/runs(read) 新到舊 | API — Execution | AC-2.1–AC-2.5;pytest integration + grep gate(SPEC.md §8 #16/#25) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L93-99、SRS.md FR-02;impl: taskq_api.api.routes.tasks、taskq_api.service.runner |
| FR-03 | API Key 認證:全部 /v1/* 要求 X-API-Key,缺/無效→401;SHA-256 雜湊儲存不存明文,hmac.compare_digest 常數時間比對;key create 明文只印一次;revoked_at 非空視為無效;/healthz、/readyz 免認證 | Security — Authentication | AC-3.1–AC-3.5;pytest(SPEC.md §8 #5/#18) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L101-107、SRS.md FR-03;impl: taskq_api.service.auth、taskq_api.api.dependencies |
| FR-04 | Scope 授權:read < write < admin 階層包含;不足→403 且 body 不洩漏資源存在性;授權判定集中於單一 dependency 中介層 | Security — Authorization | AC-4.1–AC-4.3;pytest integration(SPEC.md §8 #6) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L109-113、SRS.md FR-04;impl: taskq_api.api.dependencies |
| FR-05 | 流量控制:per-token 令牌桶 TASKQ_RATE_BURST/TASKQ_RATE_PER_SEC;超限→429+Retry-After;桶狀態存資料庫,單一交易 row-level lock;/healthz、/readyz 不受限 | Traffic Control | AC-5.1–AC-5.4;pytest integration(SPEC.md §8 #9) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L115-120、SRS.md FR-05;impl: taskq_api.api.dependencies、taskq_api.repository.rate_buckets |
| FR-06 | 持久化與交易邊界:資料存取全經 repository/ 層;每請求一 Session,context manager 保證 commit/rollback;禁字串拼接 SQL;selectinload/joinedload 顯式預載(N+1 為失敗條件);pool_size=TASKQ_DB_POOL_SIZE、pool_pre_ping=True | Persistence — Transaction | AC-6.1–AC-6.5;code review + SQLAlchemy event listener 計數(SPEC.md §8 #14) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L122-128、SRS.md FR-06;impl: taskq_api.repository.session、taskq_api.repository.task_repo |
| FR-07 | Schema Migration:Alembic 三步演進 v1(建 tasks/api_keys)→v2(tags/task_tags+name 唯一索引)→v3(result_json 拆至 task_results 含資料搬遷),每步有可運作 downgrade;upgrade head/downgrade base 成功;往返可逆逐欄相同;禁 op.execute DROP TABLE 捷徑;offline SQL 測試 | Schema Migration | AC-7.1–AC-7.6;真實 SQLite 檔案往返 + offline SQL 斷言(SPEC.md §8 #12/#13) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L130-143、SRS.md FR-07;impl: migrations/versions/v1_init.py、v2_tags.py、v3_split_results.py |
| FR-08 | 非同步執行器:asyncio.TaskGroup 管理;關閉時 graceful drain 至 TASKQ_DRAIN_TIMEOUT,逾時標記 interrupted;併發上限 TASKQ_MAX_CONCURRENT;timeout 後 kill()+await wait() 不留孤兒;CancelledError 必須向上傳播 | Concurrency — Async | AC-8.1–AC-8.4;pytest integration(SPEC.md §8 #25) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L145-150、SRS.md FR-08;impl: taskq_api.service.runner |
| FR-09 | 健康檢查與可觀測性:/healthz 免認證→200;/readyz 免認證,DB 可用且 alembic current==head→200,否則 503 說明失敗項(fail closed);/v1/metrics(admin)任務計數、延遲分位數、rate-limit 拒絕數 | Observability | AC-9.1–AC-9.4;pytest + 服務啟動冒煙(SPEC.md §8 #10/#11) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L152-160、SRS.md FR-09;impl: taskq_api.api.routes.health、taskq_api.api.routes.metrics |
| FR-10 | 錯誤契約(RFC 7807):非 2xx 一律 application/problem+json;欄位 type/title/status/detail/instance/correlation_id;detail 不得含 SQL/堆疊/路徑/DB 結構;correlation_id 同現於 X-Correlation-Id header 與日誌;錯誤碼對照 422/401/403/404/409/429/503/500 | Error Contract | AC-10.1–AC-10.5;pytest integration(SPEC.md §8 #19) | DRAFT | Owner: 02-architecture/SAD.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L162-168、SRS.md FR-10;impl: taskq_api.errors |
| NFR-01 | 效能與查詢效率:GET /v1/tasks/{id} p95<30ms、GET /v1/tasks?limit=50 p95<80ms(10,000 筆,ASGI transport);列表端點 SQL 陳述數為常數(event listener 計數) | performance | AC-N1.1–AC-N1.3;pytest-benchmark + SQL 計數(SPEC.md §8 #14/#15) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L177-183、SRS.md NFR-01 |
| NFR-02 | HTTP 與資料層安全:禁 shell=True、eval(、exec(;禁字串拼接 SQL;金鑰雜湊+hmac.compare_digest;403 不洩漏存在性;錯誤 body 不含堆疊/SQL/路徑;CORS 預設全拒;bandit 0 HIGH 0 MEDIUM | security | AC-N2.1–AC-N2.7;bandit + grep gates + code review(SPEC.md §8 #16/#17/#23) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L185-194、SRS.md NFR-02 |
| NFR-03 | 錯誤處理、交易與非同步正確性:commit/rollback context manager;禁裸 except: 與 except Exception: pass;CancelledError 不得被吞;DB 失敗→readyz 503 不無限重試;timeout 確實終止子進程;migration 失敗 rollback 維持前 revision | reliability | AC-N3.1–AC-N3.6;ast-error-handling 反模式掃描 + integration(SPEC.md §8 #10/#25) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L196-204、SRS.md NFR-03 |
| NFR-04 | 敏感資料遮蔽:stdout_tail/stderr_tail/日誌/錯誤 body 匹配敏感 regex 之整行以 [REDACTED] 取代;DB 連線字串(含密碼)不得出現於日誌/錯誤/度量;API key 明文只印一次不落盤 | security | AC-N4.1–AC-N4.3;unit + integration(SPEC.md §8 #20) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L206-212、SRS.md NFR-04 |
| NFR-05 | 文件覆蓋:全部公開函式/類別 docstring 含 [FR-XX]/[NFR-XX] 引用,覆蓋率 100%;每個端點 OpenAPI 有 summary 與 description(/openapi.json 斷言) | documentation | AC-N5.1–AC-N5.2;ast-docstrings 掃描 + /openapi.json 斷言測試 | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L214-218、SRS.md NFR-05 |
| NFR-06 | 架構分層契約:.importlinter 宣告 api > service > repository > models,config/errors 為 independence;repository 以外任何層不得 import sqlalchemy;lint-imports exit 0;禁刪檔/ignore_imports/降級 contract | layering | AC-N6.1–AC-N6.4;lint-imports + .importlinter 內容斷言(SPEC.md §8 #21) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L220-232、SRS.md NFR-06 |
| NFR-07 | 依賴與授權合規:requirements.txt == 釘版、requirements.lock 完整鎖定;allowlist MIT/BSD-2-Clause/BSD-3-Clause/Apache-2.0/PSF;pip-licenses 掃描完整依賴樹;SBOM 產出(AC-N7.4) | licensing | AC-N7.1–AC-N7.4;pip-licenses + SBOM 檢查(SPEC.md §8 #22) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L234-240、SRS.md NFR-07 |
| NFR-08 | 變異測試:features.mutation_testing: true;mutation score ≥ 70;範圍限 service/ 與 repository/ 並註記理由;不得調降 crg_cohesion_healthy | mutation | AC-N8.1–AC-N8.4;mutation-test-score / mutmut results(SPEC.md §8 #24) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L242-247、L326;SRS.md NFR-08 |
| NFR-09 | 驗證真實性(零 skip 鐵律):禁 skip/skipif/xfail/無斷言 stub;pytest -q skipped 計數 0;每測試函式 ≥1 assert;禁 --ignore/-k/--deselect/collect_ignore/testpaths 移除;FR-07 migration 以真實 SQLite 檔案測試;TRACEABILITY_MATRIX.md VERIFIED 僅在實測通過後給出 | testability | AC-N9.1–AC-N9.6;pytest -q skipped 計數 + ast-assertions(SPEC.md §8 #1) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L249-257、SRS.md NFR-09 |
| NFR-10 | 整合覆蓋:03-development/tests/integration/ 行覆蓋 ≥ 80%;httpx.AsyncClient(transport=ASGITransport(app)) 驅動,不得直呼 handler;涵蓋 CRUD 全鏈、401/403/404/409/422/429/503 各一例、migration 往返、rate limit 觸發與恢復、graceful drain;全量行覆蓋 100% | integration | AC-N10.1–AC-N10.4;pytest --cov(SPEC.md §8 #2/#3) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L259-264、SRS.md NFR-10 |
| NFR-11 | 可讀性:專案 MI(LLOC 加權)≥ 80;單一函式 CC ≤ 10;單一檔案 ≤ 400 行、單一目錄 ≤ 15 檔;每個 API handler ≤ 40 行 | maintainability | AC-N11.1–AC-N11.4;radon mi/cc/raw | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L266-271、SRS.md NFR-11 |
| NFR-12 | 系統驗證目標:Makefile verify-system 串接 upgrade head、全套測試、服務啟動+/healthz、/readyz 冒煙、downgrade base 後再 upgrade head;exit 0 且 stdout 印 verify-system: PASS;.env.example 宣告全部 12 個 TASKQ_* | verifiability | AC-N12.1–AC-N12.3;make verify-system + grep -c ^TASKQ_ .env.example(SPEC.md §8 #26/#27) | DRAFT | Owner: 02-architecture/ADR.md → 04-testing/TEST_PLAN.md;Source: SPEC.md L273-281、SRS.md NFR-12 |

## Update log

| Date | Change | By |
|------|--------|----|
| 2026-09-07 | 以 SRS.md 之 FR-01–FR-10、NFR-01–NFR-12 填滿追蹤矩陣;Source 引用一律為 repo 根 SPEC.md 行號 + SRS.md 章節;Status 初始 DRAFT(待 advance-phase 機器刷新) | Agent A |
