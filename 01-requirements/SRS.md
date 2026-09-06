# Software Requirements Specification (SRS) — taskq-api

> 本文件為 `taskq-api` 的軟體需求規格(Phase 1 產物),內容 100% 轉錄自專案根目錄
> `SPEC.md`(v1.0.0,單一事實來源,10 FR / 12 NFR / 12 env)。每條 AC 均標註
> 出處行號(`SPEC.md L<行號>`)或驗收命令編號(`SPEC.md §8 #<n>`),無本檔發明之條款。

## 1. Introduction

- **專案名稱**:`taskq-api`
- **目的**:任務佇列的 HTTP 服務化 — 以 REST API 提交、查詢、執行任務;資料持久化於關聯式資料庫;schema 隨版本演進;支援認證、授權與流量控制(SPEC.md L52)
- **語言**:Python 3.11(SPEC.md L53)
- **形態**:ASGI 服務,`uvicorn taskq_api.app:app` 啟動;另提供 `python -m taskq_api` 管理入口(migrate / seed / healthcheck)(SPEC.md L54)
- **驗證輪次**:harness-methodology 漸進式驗證測床第 2 輪 / 共 3 輪(SPEC.md L19)
- **本輪設計意圖**:相對第 1 輪(單進程 CLI)新增 HTTP 層、真實資料庫、真 schema migration、async 四個面向(SPEC.md L4-5、L26-39):

| 未覆蓋面向 | 本輪對策 | 條款 |
|---|---|---|
| 無 HTTP 層 | REST API + API key 認證 + per-token scope 授權 + rate limit | FR-03/04/05、NFR-02 |
| 無資料庫 | SQLAlchemy ORM + 明確交易邊界 + N+1 防護斷言 | FR-06、NFR-01 |
| schema migration 是自製 JSON 且測試全 skip | Alembic 三步真實演進 + 含資料搬遷 + downgrade 可逆 | FR-07、NFR-03 |
| 無 async | async 端點 + asyncio 背景執行器 | FR-08、NFR-03 |
| 依賴樹淺 | fastapi / sqlalchemy / alembic / uvicorn + transitive deps | NFR-07 |
| 整合測試只有 CLI 子進程 | `httpx.ASGITransport` 端到端,涵蓋認證與錯誤契約 | NFR-10 |

## 2. Constraints

### 2.1 技術架構(SPEC.md L58-73)

| 元件 | 技術 |
|------|------|
| HTTP 框架 | FastAPI(ASGI) |
| 資料驗證 | `pydantic` v2 request/response 模型 |
| ORM | SQLAlchemy 2.x(declarative + `Session` 明確交易邊界) |
| 資料庫 | SQLite(開發/測試)、PostgreSQL(生產)——同一份 ORM 模型 |
| Migration | Alembic(v1 → v2 → v3,每步都有 `downgrade`) |
| 非同步 | `async def` 端點 + `asyncio.TaskGroup` 背景執行器 |
| 認證 | `X-API-Key` header,金鑰雜湊後比對(不存明文) |
| 授權 | per-token scope:`read` / `write` / `admin` |
| 流量控制 | per-token 令牌桶(token bucket) |
| 錯誤契約 | RFC 7807 `application/problem+json` |
| 任務執行 | `asyncio.create_subprocess_exec`(禁 `shell=True`) |
| 分層約束 | `import-linter` layers contract(見 NFR-06) |

### 2.2 環境變數(`config.py` 讀取;`.env.example` 完整宣告)(SPEC.md L287-302)

| 變數 | 預設 | 說明 |
|------|------|------|
| `TASKQ_DB_URL` | `sqlite:///./taskq.db` | 資料庫連線字串(不得出現在日誌 — NFR-04) |
| `TASKQ_DB_POOL_SIZE` | `5` | 連線池大小(FR-06) |
| `TASKQ_TASK_TIMEOUT` | `10.0` | 單任務 subprocess timeout(秒) |
| `TASKQ_MAX_CONCURRENT` | `8` | 背景執行併發上限(FR-08) |
| `TASKQ_DRAIN_TIMEOUT` | `30.0` | 關閉時 graceful drain 上限(秒) |
| `TASKQ_RATE_BURST` | `20` | 令牌桶容量(FR-05) |
| `TASKQ_RATE_PER_SEC` | `5.0` | 令牌補充速率(FR-05) |
| `TASKQ_CORS_ORIGINS` | (空字串) | CORS 允許來源,逗號分隔;空 = 全拒(NFR-02) |
| `TASKQ_LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `TASKQ_LOG_FORMAT` | `json` | `json` / `text` |
| `TASKQ_HOST` | `127.0.0.1` | 監聽位址(預設不對外) |
| `TASKQ_PORT` | `8000` | 監聽埠 |

### 2.3 資料庫 Schema(由 FR-07 的 Alembic revision 定義)(SPEC.md L304-315)

| 表 | revision | 主要欄位 |
|---|---|---|
| `tasks` | v1 | `id`(uuid)、`command`、`name`、`status`、`created_at` |
| `api_keys` | v1 | `id`、`key_hash`(sha256)、`scope`、`created_at`、`revoked_at` |
| `tags` | v2 | `id`、`label` |
| `task_tags` | v2 | `task_id`、`tag_id`(複合主鍵) |
| `task_results` | v3 | `id`、`task_id`(FK)、`exit_code`、`stdout_tail`、`stderr_tail`、`duration_ms`、`finished_at` |
| `rate_buckets` | v1 | `key_id`(FK)、`tokens`、`updated_at` |

`tasks.result_json` 在 v1 建立、v3 移除(資料搬遷至 `task_results`)。這一步是 FR-07 往返可逆性驗收的重點(SPEC.md L315)。

### 2.4 專案側必備設定檔(非可選)(SPEC.md L317-327)

| 檔案 | 用途 | 對應 |
|------|------|------|
| `.importlinter` | 分層契約 + `sqlalchemy` 禁令 | NFR-06 |
| `requirements.txt` + `requirements.lock` | 釘版 + transitive 鎖定 | NFR-07 |
| `requirements-dev.txt` | `import-linter` / `pip-licenses` / `mutmut` / `pytest-benchmark` / `httpx` | NFR-06/07/08/10 |
| `alembic.ini` + `migrations/versions/` | 三個 revision(FR-07) | FR-07 |
| `.env.example` | 全部 12 個 `TASKQ_*` 逐一宣告並附註解 | §2.2 |
| `.methodology/harness_config.json` | `features.mutation_testing: true`;不得調降 `crg_cohesion_healthy` | NFR-08 |
| `Makefile` | `verify-system`(含 migration 往返) | NFR-12 |

### 2.5 framework 預設門檻(SPEC.md L406-425)

- `linting` / `type_safety` / `test_coverage`:ruff / pyright / pytest-cov,對應 SPEC.md §8 #1 / #2 + 框架預設門檻
- `architecture`(CRG):code-review-graph,NFR-06 四層 + FR-07 migrations;`crg_cohesion_healthy` 保持預設值,不得為了讓專案通過而調降(SPEC.md L425)
- `secrets_scanning`:gitleaks,框架預設門檻 100
- 高風險模組:`taskq_api.service.runner`(async 子進程)、`taskq_api.service.auth`(認證授權)、`taskq_api.repository.session`(交易邊界)、`migrations/versions/v3_split_results.py`(資料搬遷)。四者需 per-module TDD 覆蓋(SPEC.md L427)

## 3. Functional Requirements

### FR-01: 任務資源 CRUD API

來源:SPEC.md L79-91。

| 方法 | 路徑 | scope | 行為 |
|------|------|-------|------|
| `POST` | `/v1/tasks` | `write` | 建立任務;body 由 `TaskCreate` pydantic 模型驗證 |
| `GET` | `/v1/tasks/{id}` | `read` | 取得單一任務全欄位 |
| `GET` | `/v1/tasks` | `read` | 分頁列表,支援 `?status=`、`?limit=`、`?cursor=` |
| `DELETE` | `/v1/tasks/{id}` | `admin` | 刪除任務(連同結果列,同一交易) |

**Acceptance criteria (FR-01)**

- **AC-1.1**: `POST /v1/tasks`(scope `write`)建立任務,body 由 `TaskCreate` pydantic 模型驗證;有效 write key → 201 + task id(SPEC.md L83、§8 #4)。驗證者:`pytest 03-development/tests/integration`(NFR-10 之 CRUD 全鏈)
- **AC-1.2**: 驗證規則同第 1 輪 FR-01(非空 / ≤1000 字元 / 注入字元黑名單 / 名稱唯一);違反 → HTTP 422 + problem+json;重複 name → 409(SPEC.md L88、§8 #8)
  - 「注入字元黑名單」之字元集未於 SPEC.md 定義(引用外部「第 1 輪 FR-01」)→ 見 §7 Open Issues NFR-99-1
- **AC-1.3**: `GET /v1/tasks/{id}`(scope `read`)→ 200,取得單一任務全欄位(SPEC.md L84)
- **AC-1.4**: `GET /v1/tasks`(scope `read`)→ 200,分頁列表,支援 `?status=`、`?limit=`、`?cursor=`(SPEC.md L85)
- **AC-1.5**: 分頁為 cursor-based(不得用 offset —— 大表 offset 掃描是 N+1 的親戚);列表端點的預設 `limit` 為 50,上限 200;超過上限 → 422(SPEC.md L90-91)
- **AC-1.6**: `DELETE /v1/tasks/{id}`(scope `admin`)刪除任務(連同結果列,同一交易)(SPEC.md L86)
- **AC-1.7**: 未知 id → HTTP 404 + problem+json(SPEC.md L89、§8 #7)

### FR-02: 任務執行端點

來源:SPEC.md L93-99。

**Acceptance criteria (FR-02)**

- **AC-2.1**: `POST /v1/tasks/{id}/run`(scope `write`)→ HTTP 202 Accepted,body 含 `run_id`(SPEC.md L95)
- **AC-2.2**: 實際執行以 `asyncio.create_subprocess_exec(*shlex.split(command))` 進行,禁 `shell=True`,timeout 為 `TASKQ_TASK_TIMEOUT`(SPEC.md L96)。驗證者:grep gate(§8 #16)+ integration test
- **AC-2.3**: 狀態機:`pending → running → done | failed | timeout`(SPEC.md L97)
- **AC-2.4**: 執行結果寫入 `task_results` 表(FR-07 的 v3 schema),欄位:`exit_code` / `stdout_tail` / `stderr_tail` / `duration_ms` / `finished_at`(SPEC.md L98)
- **AC-2.5**: `GET /v1/tasks/{id}/runs`(scope `read`)→ 200,該任務的歷史執行紀錄,新到舊排序(SPEC.md L99)

### FR-03: API Key 認證

來源:SPEC.md L101-107。

**Acceptance criteria (FR-03)**

- **AC-3.1**: 全部 `/v1/*` 端點要求 `X-API-Key` header;缺少或無效 → HTTP 401 + problem+json(SPEC.md L103、§8 #5)
- **AC-3.2**: 金鑰以 SHA-256 雜湊儲存於 `api_keys` 表,不得存明文;比對用 `hmac.compare_digest`(常數時間);查 `api_keys` 表無明文金鑰且 `key_hash` 為 64 hex(SPEC.md L104、§8 #18)
- **AC-3.3**: 金鑰由 `python -m taskq_api key create --scope <scope>` 產生,明文只在建立當下印出一次(SPEC.md L105)
- **AC-3.4**: 停用金鑰:`revoked_at` 非空的金鑰一律視為無效(SPEC.md L106)
- **AC-3.5**: `/healthz`、`/readyz` 不要求認證(FR-09)(SPEC.md L107)

### FR-04: Scope 授權

來源:SPEC.md L109-113。

**Acceptance criteria (FR-04)**

- **AC-4.1**: 每把金鑰帶一個 scope:`read` < `write` < `admin`(階層包含)(SPEC.md L111)
- **AC-4.2**: 端點所需 scope 見 FR-01/02 表;不足 → HTTP 403 + problem+json,且 body 不得洩漏該資源是否存在(SPEC.md L112、§8 #6)
- **AC-4.3**: 授權判定必須在單一中介層(dependency)完成,不得散落於各 handler —— 以測試斷言「每個 `/v1` 路由都經過同一個 dependency」(SPEC.md L113)

### FR-05: 流量控制

來源:SPEC.md L115-120。

**Acceptance criteria (FR-05)**

- **AC-5.1**: per-token 令牌桶:容量 `TASKQ_RATE_BURST`,補充速率 `TASKQ_RATE_PER_SEC`(SPEC.md L117)
- **AC-5.2**: 超限 → HTTP 429 + problem+json + `Retry-After` header(秒)(SPEC.md L118、§8 #9)
- **AC-5.3**: 令牌桶狀態存於資料庫(跨 worker 一致),更新必須在單一交易內以 row-level lock 進行(SPEC.md L119;schema 見 §2.3 `rate_buckets`)
- **AC-5.4**: `/healthz`、`/readyz` 不受限(SPEC.md L120)

### FR-06: 持久化層與交易邊界

來源:SPEC.md L122-128。

**Acceptance criteria (FR-06)**

- **AC-6.1**: 全部資料存取經由 `repository/` 層,業務層不得直接持有 `Session`(SPEC.md L124)
- **AC-6.2**: 每個 API 請求一個 `Session`,交易邊界明確:成功 commit、例外 rollback(以 context manager 保證)(SPEC.md L125)
- **AC-6.3**: 禁止字串拼接 SQL;一律使用 ORM 或參數化查詢(NFR-02)(SPEC.md L126)
- **AC-6.4**: 關聯查詢必須用 `selectinload` / `joinedload` 顯式預載 —— N+1 為驗收失敗條件(NFR-01)(SPEC.md L127、§8 #14)
- **AC-6.5**: 連線池:`pool_size=TASKQ_DB_POOL_SIZE`,`pool_pre_ping=True`(SPEC.md L128)

### FR-07: Schema Migration(Alembic 三步演進)

來源:SPEC.md L130-143。

**Acceptance criteria (FR-07)**

- **AC-7.1**: 三個 revision,每一步都必須有可運作的 `downgrade`(SPEC.md L132)
- **AC-7.2**: revision 內容(SPEC.md L134-138):
  - v1:建立 `tasks`、`api_keys` 兩表;downgrade 為 drop 兩表
  - v2:新增 `tags`、`task_tags`(多對多)+ `tasks.name` 唯一索引;downgrade 為 drop 新表與索引,不影響 v1 資料
  - v3:含資料搬遷:把 `tasks.result_json` 拆為獨立的 `task_results` 表,搬遷既有資料後移除原欄位;downgrade 為反向搬遷回 `tasks.result_json` 後 drop `task_results`,資料不得遺失
- **AC-7.3**: `alembic upgrade head` 與 `alembic downgrade base` 必須都成功;`alembic downgrade base` exit 0,無殘留表(SPEC.md L140、§8 #13)
- **AC-7.4**: 往返可逆性驗收:`upgrade head` → 寫入樣本資料 → `downgrade -1` → `upgrade head`,樣本資料的欄位值必須逐欄相同(v3 的資料搬遷是本條的重點)(SPEC.md L141、§8 #12)
- **AC-7.5**: 禁止以 `op.execute("DROP TABLE ...")` 之類的破壞性捷徑取代真正的 downgrade(SPEC.md L142)
- **AC-7.6**: migration 檔本身納入測試覆蓋(以 `alembic` 的 offline SQL 產生 + 斷言)(SPEC.md L143)

### FR-08: 非同步執行器

來源:SPEC.md L145-150。

**Acceptance criteria (FR-08)**

- **AC-8.1**: 背景執行以 `asyncio.TaskGroup` 管理;服務關閉時必須 graceful drain(等待進行中的任務至 `TASKQ_DRAIN_TIMEOUT`,逾時則標記 `interrupted`)(SPEC.md L147、§8 #25)
- **AC-8.2**: 併發上限 `TASKQ_MAX_CONCURRENT`;超過時新任務排隊,不得無限制生成 coroutine(SPEC.md L148)
- **AC-8.3**: 任務 timeout 以 `asyncio.wait_for` 實作;逾時必須確實終止子進程(`process.kill()` 後 `await process.wait()`),不得留下孤兒進程(SPEC.md L149;§11:孤兒子進程 | 0 | integration test)
- **AC-8.4**: 取消語意:`asyncio.CancelledError` 必須向上傳播,不得被 `except Exception` 吞掉(NFR-03)(SPEC.md L150)

### FR-09: 健康檢查與可觀測性

來源:SPEC.md L152-160。

| 端點 | 認證 | 行為 |
|------|------|------|
| `GET /healthz` | 無 | 進程存活 → 200 `{"status":"ok"}` |
| `GET /readyz` | 無 | DB 連線可用 且 `alembic current` == head → 200;否則 503 並在 body 說明哪一項失敗 |
| `GET /v1/metrics` | `admin` | 任務計數(按狀態)、執行延遲分位數、rate-limit 拒絕數 |

**Acceptance criteria (FR-09)**

- **AC-9.1**: `GET /healthz` 無認證;進程存活 → 200 `{"status":"ok"}`(SPEC.md L156)
- **AC-9.2**: `GET /readyz` 無認證;DB 連線可用 且 `alembic current` == head → 200;否則 503 並在 body 說明哪一項失敗(SPEC.md L157、§8 #10/#11)
- **AC-9.3**: `/readyz` 的「migration 未到 head」判定是關鍵:部署了新程式碼但忘記跑 migration 時必須 fail closed(SPEC.md L160)
- **AC-9.4**: `GET /v1/metrics`(scope `admin`)回報任務計數(按狀態)、執行延遲分位數、rate-limit 拒絕數(SPEC.md L158)

### FR-10: 錯誤契約(RFC 7807)

來源:SPEC.md L162-168。

**Acceptance criteria (FR-10)**

- **AC-10.1**: 全部非 2xx 回應的 `Content-Type` 為 `application/problem+json`(SPEC.md L164)
- **AC-10.2**: body 欄位:`type`(URI)、`title`、`status`、`detail`、`instance`、`correlation_id`(SPEC.md L165)
- **AC-10.3**: `detail` 不得洩漏內部細節:不得含 SQL 陳述、堆疊追蹤、檔案路徑、資料庫結構描述(SPEC.md L166、§8 #19)
- **AC-10.4**: `correlation_id` 同時出現在回應 header `X-Correlation-Id` 與伺服器日誌,可用於串接(SPEC.md L167)
- **AC-10.5**: 錯誤碼對照:422 驗證 / 401 未認證 / 403 scope 不足 / 404 未知資源 / 409 名稱衝突 / 429 超限 / 503 未就緒 / 500 其他(SPEC.md L168、§7 錯誤處理表 L335-345)

## 4. Non-Functional Requirements

### NFR-01: 效能與查詢效率

- **dimension**:`performance`
- 來源:SPEC.md L177-183。

**Acceptance criteria (NFR-01)**

- **AC-N1.1**: `GET /v1/tasks/{id}` 在 10,000 筆資料下 p95 < 30ms(不含網路,以 ASGI transport 量測)—— 量測方式:`pytest-benchmark`(SPEC.md L180、L183、§8 #15)
  - Coverage note:harness `performance` 維度只對 mean > 1000ms 扣分(evaluate_dimension.md `### performance`),不驗證 p95 < 30ms 門檻 — 此 AC 需 Phase 3 專屬 benchmark 測試實作
- **AC-N1.2**: `GET /v1/tasks?limit=50` 在 10,000 筆資料下 p95 < 80ms,量測方式:`pytest-benchmark`(SPEC.md L181、L183、§11)
  - Coverage note:同上,harness 維度不驗證 p95 < 80ms 門檻 — 需專屬 benchmark 測試
- **AC-N1.3**: N+1 為失敗條件:列表端點回應一次請求所發出的 SQL 陳述數必須是常數(與回傳筆數無關),以 SQLAlchemy event listener 計數斷言(SPEC.md L182、§8 #14)
  - Coverage note:harness `performance` 維度不量測 SQL 陳述計數 — 需 Phase 3 專屬 event-listener 測試實作

### NFR-02: HTTP 與資料層安全

- **dimension**:`security`
- 來源:SPEC.md L185-194。

**Acceptance criteria (NFR-02)**

- **AC-N2.1**: 全 codebase 禁用 `shell=True`、`eval(`、`exec(`(grep 0 命中)(SPEC.md L188、§8 #16)
  - Coverage note:harness `security` 維度(僅 `bandit` 掃描)不驗證 grep gate — 需專屬 CI grep gate 或測試
- **AC-N2.2**: 禁止字串拼接 SQL:不得出現 f-string / `%` / `+` 組成的 SQL;一律 ORM 或參數化(以 grep + code review 雙重驗證)(SPEC.md L189、§8 #17)
  - Coverage note:bandit 不驗證 SQL 字串拼接掃描 — 需專屬 grep gate + code review
- **AC-N2.3**: API key 雜湊儲存,比對用 `hmac.compare_digest`(FR-03)(SPEC.md L190)
- **AC-N2.4**: 403 回應不得洩漏資源存在性(FR-04)(SPEC.md L191)
- **AC-N2.5**: 錯誤 body 不得含堆疊/SQL/路徑(FR-10)(SPEC.md L192)
- **AC-N2.6**: CORS 預設拒絕所有來源;允許清單由 `TASKQ_CORS_ORIGINS` 明示(SPEC.md L193)
- **AC-N2.7**: `bandit -r 03-development/src/`:0 HIGH、0 MEDIUM(SPEC.md L194、§8 #23)
  - Coverage note:此 AC 即 harness `security` 維度之量測本身(0 HIGH 0 MEDIUM);其餘 AC-N2.1~N2.6 非該維度所驗

### NFR-03: 錯誤處理、交易與非同步正確性

- **dimension**:`error_handling`
- 來源:SPEC.md L196-204。

**Acceptance criteria (NFR-03)**

- **AC-N3.1**: 每個請求的交易邊界明確:成功 commit、例外 rollback,以 context manager 保證(FR-06)(SPEC.md L199)
- **AC-N3.2**: 不得出現裸 `except:`、`except Exception: pass`(SPEC.md L200)
- **AC-N3.3**: `asyncio.CancelledError` 不得被吞掉 —— 必須重新拋出(async 專屬的吞噬陷阱)(SPEC.md L201)
- **AC-N3.4**: 資料庫連線失敗 → `/readyz` 503 + 明確 detail;不得靜默重試至無限(SPEC.md L202、§8 #10)
- **AC-N3.5**: 任務 timeout 必須確實終止子進程,不留孤兒(FR-08)(SPEC.md L203、§8 #25)
- **AC-N3.6**: migration 失敗 → 交易 rollback,資料庫維持在前一個 revision(FR-07)(SPEC.md L204)
  - Coverage note:harness `error_handling` 維度只量測「檔案層級 try/except 存在率 + 4 種反模式扣分」,不驗證交易邊界、readyz 503、孤兒進程、migration rollback 等行為 — AC-N3.1/N3.4/N3.5/N3.6 需 Phase 3 專屬測試實作;AC-N3.2/N3.3 之「被吞」樣態由維度反模式(bare_except / except_base_exception)部分覆蓋,仍需專屬測試斷言 re-raise

### NFR-04: 敏感資料遮蔽

- **dimension**:`security`
- 來源:SPEC.md L206-212。

**Acceptance criteria (NFR-04)**

- **AC-N4.1**: `stdout_tail` / `stderr_tail` / 日誌 / 錯誤 body 落盤或送出前,匹配 `(sk-[A-Za-z0-9_-]{8,}|token=\S+|Bearer\s+\S+|postgres(ql)?://[^\s]+)` 的行整行以 `[REDACTED]` 取代(SPEC.md L209-210)
- **AC-N4.2**: 資料庫連線字串(含密碼)不得出現在任何日誌、錯誤訊息或 `/v1/metrics` 回應中(SPEC.md L211、§8 #20;§11:DB 連線字串出現於日誌 | 0 | unit test)
- **AC-N4.3**: API key 明文只在 `key create` 當下輸出一次,不得寫入任何持久化位置(SPEC.md L212)
  - Coverage note:harness `security`(bandit)與 `secrets_scanning`(gitleaks)維度皆不驗證遮蔽 regex 與 DB 連線字串落盤 — 需 Phase 3 專屬 unit/integration 測試實作

### NFR-05: 文件覆蓋

- **dimension**:`documentation`
- 來源:SPEC.md L214-218。

**Acceptance criteria (NFR-05)**

- **AC-N5.1**: 全部公開函式/類別有 docstring 且含 `[FR-XX]` 或 `[NFR-XX]` 引用,覆蓋率 100%(SPEC.md L217)
  - Coverage note:harness `documentation` 維度(ast-docstrings)只計 docstring「存在性」,不驗證 `[FR-XX]`/`[NFR-XX]` 引用內容 — 引用內容需 Phase 3 專屬測試實作
- **AC-N5.2**: 每個 API 端點在 OpenAPI schema 中有 `summary` 與 `description`(FastAPI 自動產生的 `/openapi.json` 以測試斷言)(SPEC.md L218)
  - Coverage note:harness `documentation` 維度不檢查 OpenAPI — 需專屬 `/openapi.json` 測試

### NFR-06: 架構分層契約

- **dimension**:`architecture_constraints`
- 來源:SPEC.md L220-232。

**Acceptance criteria (NFR-06)**

- **AC-N6.1**: 專案根目錄必須存在 `.importlinter`,宣告 layers contract:`api > service > repository > models`;上層可 import 下層,下層不得 import 上層;`config` 與 `errors` 為 independence 模組(SPEC.md L223-229)
- **AC-N6.2**: 額外禁令(forbidden contract):`repository` 以外的任何層不得 import `sqlalchemy` —— ORM 洩漏到業務層是本輪要防的具體反模式(SPEC.md L230)
- **AC-N6.3**: `lint-imports` 必須 exit 0(SPEC.md L231、§8 #21)
- **AC-N6.4**: 禁止以刪除 `.importlinter`、萬用字元 `ignore_imports`、或降級 contract 的方式取得通過(SPEC.md L232)
  - Coverage note:harness `architecture_constraints` 維度只驗證 `lint-imports` exit 0(且 `.importlinter` 缺失時為 UNSCOREABLE);契約「內容」是否符合 AC-N6.1/N6.2 之宣告由專案自身測試斷言(讀取 `.importlinter` 內容比對)—— 需 Phase 3 專屬測試實作

### NFR-07: 依賴與授權合規

- **dimension**:`license_compliance`
- 來源:SPEC.md L234-240。

**Acceptance criteria (NFR-07)**

- **AC-N7.1**: 全部 runtime 依賴在 `requirements.txt` 以 `==` 釘版;transitive 依賴以 lock 檔(`requirements.lock`)完整鎖定(SPEC.md L237)
- **AC-N7.2**: 允許的 license:MIT / BSD-2-Clause / BSD-3-Clause / Apache-2.0 / PSF;出現其他 → 該依賴不得使用(SPEC.md L238)
- **AC-N7.3**: 掃描範圍必須包含完整依賴樹(直接 + transitive),證據命令:`pip-licenses --format=json --with-system`(SPEC.md L239、§8 #22)
- **AC-N7.4**: 產出 SBOM 於 `08-config/SBOM.json`,含每個依賴的 `name` / `version` / `license` / `direct|transitive`(SPEC.md L240)
  - Coverage note:harness `license_compliance` 維度只執行 `scancode --license ... src/`(掃原始碼檔案標頭),不驗證 pip 依賴樹 license 與 SBOM artifact — AC-N7.1/N7.3/N7.4 需 Phase 3 專屬實作任務(pip-licenses 掃描 + SBOM 產生與斷言測試)

### NFR-08: 變異測試

- **dimension**:`mutation_testing`
- 來源:SPEC.md L242-247。

**Acceptance criteria (NFR-08)**

- **AC-N8.1**: `.methodology/harness_config.json` 設 `features.mutation_testing: true`(SPEC.md L245、§2.4)
- **AC-N8.2**: mutation score ≥ 70(SPEC.md L246、§8 #24)
- **AC-N8.3**: 範圍限定於 `service/` 與 `repository/` 兩層,並在 `harness_config.json` 註記限定理由(執行時間預算)(SPEC.md L247)
DERIVED: SPEC.md L326 — `crg_cohesion_healthy` 不調降條款出自 §5.3 必備設定檔表與 §10 CRG 校準鐵律,因同屬 `harness_config.json` 配置而聚合至 NFR-08
- **AC-N8.4**: `.methodology/harness_config.json` 不得調降 `crg_cohesion_healthy`(SPEC.md L326、L425)
  - Coverage note:harness `mutation_testing` 維度只驗證 score;AC-N8.1/N8.3/N8.4 之配置內容需 Phase 3 專屬檢查

### NFR-09: 驗證真實性(零 skip 鐵律)

- **dimension**:`test_assertion_quality`
- 來源:SPEC.md L249-257。

**Acceptance criteria (NFR-09)**

- **AC-N9.1**: 任何 FR / NFR 的驗證測試不得是 `pytest.skip` / `skipif` / `xfail` / 無斷言的 stub(SPEC.md L252)
- **AC-N9.2**: `pytest 03-development/tests -q` 的 skipped 計數必須為 0(SPEC.md L253、§8 #1、§11)
- **AC-N9.3**: 每個測試函式至少一個 `assert`(`zero_assert == 0`)(SPEC.md L254、§11:零斷言測試函式數 | 0 | ast-assertions)
- **AC-N9.4**: 反造假條款:不得以 `--ignore` / `-k` / `--deselect` / `collect_ignore` / 從 `testpaths` 移除目錄的方式排除測試(SPEC.md L255)
- **AC-N9.5**: 本輪特別條款:`FR-07` 的三步 migration 必須以真實資料庫測試(SQLite 檔案,非 in-memory mock),往返可逆性以實際資料比對驗證。不得以「migration 邏輯太難測」為由降級為 skip(SPEC.md L256)
- **AC-N9.6**: `TRACEABILITY_MATRIX.md` 的 `VERIFIED` 只能在測試實際執行並通過時給出(SPEC.md L257)
  - Coverage note:harness `test_assertion_quality` 維度只驗證 zero-assert(asserted/total);skip 計數 0、反造假條款、真實 DB migration 測試、TRACEABILITY_MATRIX 狀態需 Phase 3 專屬檢查(pytest -q 輸出解析 + 測試設計審查)

### NFR-10: 整合覆蓋

- **dimension**:`integration_coverage`
- 來源:SPEC.md L259-264。

**Acceptance criteria (NFR-10)**

- **AC-N10.1**: `03-development/tests/integration/` 行覆蓋 ≥ 80%(SPEC.md L262、§8 #3、§11)
- **AC-N10.2**: 整合測試以 `httpx.AsyncClient(transport=ASGITransport(app))` 驅動,不得直接呼叫 handler 函式(SPEC.md L263)
- **AC-N10.3**: 至少涵蓋:CRUD 全鏈、401/403/404/409/422/429/503 每個錯誤碼各一例、migration 往返、rate limit 觸發與恢復、graceful drain(SPEC.md L264)
DERIVED: SPEC.md L358 — §8 #2 之全量行覆蓋 100% 在 canonical 中歸屬 framework 預設 `test_coverage` 門檻(§10 L421)而非任何 NFR,因屬覆蓋率類而歸入 NFR-10(整合覆蓋)
- **AC-N10.4**: `pytest 03-development/tests --cov=03-development/src --cov-report=term` 之 TOTAL 行覆蓋 100%(SPEC.md §8 #2、§11:行覆蓋率 | 100% | pytest-cov;§10 `test_coverage` framework 預設門檻)
  - Coverage note:harness `integration_coverage` 維度只量測整合套件之行覆蓋率(AC-N10.1 即該維度);ASGI 驅動方式(AC-N10.2)、情境清單(AC-N10.3)、全量覆蓋 100%(AC-N10.4,framework `test_coverage` 預設 gate)需 Phase 3 專屬測試與檢查

### NFR-11: 可讀性

- **dimension**:`readability`
- 來源:SPEC.md L266-271。

**Acceptance criteria (NFR-11)**

- **AC-N11.1**: 專案 MI(LLOC 加權)≥ 80(SPEC.md L269、§11:專案 MI | ≥ 80 | readability-v2)
- **AC-N11.2**: 單一函式 CC ≤ 10(SPEC.md L269)
- **AC-N11.3**: 單一檔案 ≤ 400 行;單一目錄 ≤ 15 檔(SPEC.md L270)
- **AC-N11.4**: 每個 API handler ≤ 40 行(業務邏輯必須下沉到 `service/`)(SPEC.md L271)
  - Coverage note:harness `readability` 維度只量測「全部檔案 MI 平均值」(radon mi);CC ≤ 10、檔案/目錄行數檔數上限、handler ≤ 40 行需 Phase 3 專屬檢查(radon cc/raw 等)

### NFR-12: 系統驗證目標

- **dimension**:`execute_verification_target`
- 來源:SPEC.md L273-281。

**Acceptance criteria (NFR-12)**

- **AC-N12.1**: `Makefile` 的 `verify-system` target 必須串接:1. `alembic upgrade head`、2. 全套測試、3. 服務啟動 + `/healthz`、`/readyz` 冒煙、4. `alembic downgrade base` 後再 `upgrade head`(往返驗證)(SPEC.md L276-280)
- **AC-N12.2**: `make verify-system` 必須 exit 0 並在 stdout 印出 `verify-system: PASS`(SPEC.md L281、§8 #27)
DERIVED: SPEC.md L302 — §8 #26 之 `.env.example` 12 變數檢查在 canonical 中未繫結任何 FR/NFR,因屬系統級配置完整性而歸入 NFR-12(系統驗證目標)
- **AC-N12.3**: `grep -c "^TASKQ_" .env.example` == 12(§2.2 全部宣告)(SPEC.md §5.1、§8 #26)
  - Coverage note:harness `execute_verification_target` 維度只驗證 `make verify-system` exit 0,「不讀取 target 內容」;AC-N12.1 之四步驟內容、stdout 訊息(AC-N12.2)、`.env.example` 12 變數(AC-N12.3)需 Phase 3 專屬測試/檢查實作(evaluate_dimension.md 明示:內容由專案自身 `# NFR-XX` 測試強制)

## 5. Acceptance Criteria Summary

轉錄自 SPEC.md §8 驗收標準(27 條命令 + 期望輸出,SPEC.md L351-383)。

| # | 命令 / 情境 | 期望 | AC 對應 |
|---|-------------|------|---------|
| 1 | `pytest 03-development/tests -q` | 全綠,skipped 計數為 0(NFR-09) | AC-N9.1、AC-N9.2 |
| 2 | `pytest 03-development/tests --cov=03-development/src --cov-report=term` | TOTAL 100% | AC-N10.4 |
| 3 | `pytest 03-development/tests/integration --cov=03-development/src --cov-report=term` | TOTAL ≥ 80%(NFR-10) | AC-N10.1 |
| 4 | `POST /v1/tasks`(有效 write key) | 201 + task id | AC-1.1 |
| 5 | `POST /v1/tasks`(無 `X-API-Key`) | 401 + problem+json | AC-3.1 |
| 6 | `DELETE /v1/tasks/{id}`(write key,非 admin) | 403,body 不透露該 id 是否存在 | AC-4.2 |
| 7 | `GET /v1/tasks/{unknown}` | 404 + problem+json | AC-1.7 |
| 8 | `POST /v1/tasks` 重複 name | 409 | AC-1.2 |
| 9 | 連續請求超過 `TASKQ_RATE_BURST` | 429 + `Retry-After` header | AC-5.2 |
| 10 | 停掉 DB 後 `GET /readyz` | 503,detail 指明 DB 不可用 | AC-9.2、AC-N3.4 |
| 11 | `alembic downgrade -1` 後 `GET /readyz` | 503,detail 指明 migration 未到 head | AC-9.2、AC-9.3 |
| 12 | `alembic upgrade head` → 寫樣本 → `downgrade -1` → `upgrade head` | 樣本資料逐欄相同(v3 資料搬遷可逆 — FR-07) | AC-7.4 |
| 13 | `alembic downgrade base` | exit 0,無殘留表 | AC-7.3 |
| 14 | `GET /v1/tasks?limit=50`(10,000 筆)的 SQL 陳述計數 | 常數(與筆數無關 — N+1 防護,NFR-01) | AC-N1.3 |
| 15 | `GET /v1/tasks/{id}` p95(10,000 筆) | < 30ms(NFR-01) | AC-N1.1 |
| 16 | `grep -rn "shell=True\|eval(\|exec(" 03-development/src/` | 0 命中 | AC-N2.1 |
| 17 | 掃描 SQL 字串拼接(f-string / `%` / `+` 組 SQL) | 0 命中(NFR-02) | AC-N2.2 |
| 18 | 查 `api_keys` 表 | 無明文金鑰;`key_hash` 為 64 hex(NFR-02) | AC-3.2 |
| 19 | 觸發 500 後檢查回應 body | 不含堆疊 / SQL / 檔案路徑(FR-10 / NFR-02) | AC-10.3、AC-N2.5 |
| 20 | 日誌與 `/v1/metrics` 全文 | 不含 `TASKQ_DB_URL` 的密碼片段(NFR-04) | AC-N4.2 |
| 21 | `lint-imports` | exit 0,且 `service`/`api` 層 import `sqlalchemy` 會被擋(NFR-06) | AC-N6.2、AC-N6.3 |
| 22 | `pip-licenses --format=json --with-system` | 每個依賴 license ∈ allowlist(NFR-07) | AC-N7.3 |
| 23 | `bandit -r 03-development/src/` | 0 HIGH,0 MEDIUM | AC-N2.7 |
| 24 | `mutmut run` 後 `mutmut results` | mutation score ≥ 70(NFR-08) | AC-N8.2 |
| 25 | 服務關閉時有進行中的任務 | graceful drain;逾時者標記 `interrupted`,無孤兒進程(FR-08) | AC-8.1、AC-N3.5 |
| 26 | `grep -c "^TASKQ_" .env.example` | 12(§5.1 全部宣告) | AC-N12.3 |
| 27 | `make verify-system` | exit 0 且 stdout 含 `verify-system: PASS`(NFR-12) | AC-N12.2 |

## 6. Out-of-Scope

以下項目不在本輪規格範圍(逐項皆為 SPEC.md 明文之排除或未要求,非本檔推斷):

| 項目 | 依據 |
|------|------|
| `shell=True` 子進程執行 | 明文禁用(SPEC.md L96、L188) |
| offset 分頁 | 明文禁用,分頁為 cursor-based(SPEC.md L90) |
| API key 明文儲存 | 明文禁止,僅 SHA-256 雜湊(SPEC.md L104) |
| 未認證的 `/v1/*` 存取 | 全部 `/v1/*` 要求 `X-API-Key`(SPEC.md L103) |
| 非 RFC 7807 的錯誤格式 | 全部非 2xx 回應為 `application/problem+json`(SPEC.md L164) |
| 字串拼接 SQL | 明文禁止(SPEC.md L126、L189) |
| 單進程 CLI 形態 | 第 1 輪測床範圍;本輪為 ASGI HTTP 服務(SPEC.md L4-5、L28-29) |
| 外部任務 broker / 分散式佇列 | 本輪執行模型為 `asyncio.TaskGroup` 進程內執行器(SPEC.md L67),規格未要求外部 broker |
| 使用者帳戶 / 登入會話體系 | 規格僅定義 `X-API-Key` 認證與 per-token scope(SPEC.md L68-69),未定義帳戶體系 |
| in-memory mock 測 migration | 明文禁止,NFR-09 特別條款要求真實 SQLite 檔案(SPEC.md L256) |
| `op.execute("DROP TABLE ...")` 式 downgrade 捷徑 | 明文禁止(SPEC.md L142) |
| 降級 `.importlinter` contract / 刪檔 / 萬用字元 ignore | 明文禁止(SPEC.md L232) |
| 以 `--ignore` / `-k` / `--deselect` / `collect_ignore` / `testpaths` 移除排除測試 | 明文禁止(SPEC.md L255) |

## 7. Open Issues

- **NFR-99-1**(ambiguity resolution):Resolve FR-01「注入字元黑名單」之字元集定義 — SPEC.md L88 引用「第 1 輪 FR-01」(外部測床規格),本檔未列出黑名單字元清單;當前 SPEC phrasing 在 (A) 沿用第 1 輪測床之黑名單字元集 與 (B) 本輪另定 之間存在歧義 — test harness 與 stakeholder 確認後回填 AC-1.2。// @rule R-CANONICAL-INTERP-001
- **NFR-99-2**(ambiguity resolution):Resolve SPEC.md §10 L429「async 為本輪新變數」之處理 — 若 framework 的 `ast-error-handling` / `ast-assertions` 掃描器在 async 語法上出現誤判或漏判,記入 Phase 4 bug hunt,不得靜默繞過(此為測床發現導向條款,非 FR/NFR 功能需求)
- Prompt-injection scan:SPEC.md 全文掃描 0 命中(無嵌入指令、無 override/disregard 條款),無 FR-XX-deferred 項目。

## 8. Risks

轉錄自 SPEC.md §9 風險矩陣(SPEC.md L387-402);R13 另源自 SPEC.md L429,見下方 DERIVED 註記。

| ID | 風險 | 影響 | 可能性 | 緩解 |
|----|------|------|--------|------|
| R1 | v3 資料搬遷遺失資料 | 高 | 中 | 往返可逆性測試以真實 DB 逐欄比對(FR-07 / §8 #12) |
| R2 | SQL injection | 高 | 低 | 禁字串拼接 + ORM/參數化 + grep gate(NFR-02) |
| R3 | API key 洩漏 | 高 | 中 | 雜湊儲存 + 常數時間比對 + 明文只印一次(FR-03) |
| R4 | 403 洩漏資源存在性 | 中 | 中 | 授權判定在資源查詢之前(FR-04 / §8 #6) |
| R5 | N+1 查詢在大表上崩潰 | 高 | 高 | 顯式預載 + SQL 計數斷言(NFR-01 / §8 #14) |
| R6 | 錯誤 body 洩漏內部結構 | 中 | 高 | RFC 7807 固定欄位 + detail 白名單(FR-10) |
| R7 | `CancelledError` 被吞 → 關閉時卡死 | 中 | 中 | 明文禁令 + 測試斷言(NFR-03) |
| R8 | 任務 timeout 留下孤兒進程 | 中 | 中 | `kill()` + `await wait()`(FR-08 / §8 #25) |
| R9 | 部署後忘記跑 migration | 高 | 中 | `/readyz` fail closed(FR-09 / §8 #11) |
| R10 | 連線池耗盡 | 中 | 中 | `pool_pre_ping` + 併發上限(FR-06/08) |
| R11 | transitive 依賴引入不相容 license | 中 | 中 | lock 檔 + 全樹掃描(NFR-07) |
| R12 | rate bucket 競態導致超放行 | 低 | 中 | 單一交易 + row-level lock(FR-05) |
| R13 | async 語法使 framework 掃描器誤判/漏判 | 中 | 中 | 記入 Phase 4 bug hunt,不得靜默繞過(SPEC.md L429,見 NFR-99-2) |

DERIVED: SPEC.md L429 — R13 源自 §10「async 為本輪新變數」條款(框架掃描器誤判/漏判之處理),非 §9 風險矩陣既有列;因屬本輪測床特有風險而納入本表。

## 9. Glossary

| 術語 | 定義 |
|------|------|
| RFC 7807 / problem+json | 機器可讀錯誤格式 `application/problem+json`,欄位 `type` / `title` / `status` / `detail` / `instance` / `correlation_id`(SPEC.md L164-165) |
| cursor-based pagination | 以 cursor 分頁,不得用 offset(SPEC.md L90) |
| token bucket(令牌桶) | per-token 流量控制:容量 `TASKQ_RATE_BURST`、補充速率 `TASKQ_RATE_PER_SEC`(SPEC.md L117) |
| scope 階層 | `read` < `write` < `admin`,階層包含(SPEC.md L111) |
| graceful drain | 服務關閉時等待進行中任務至 `TASKQ_DRAIN_TIMEOUT`,逾時標記 `interrupted`(SPEC.md L147) |
| N+1 | 列表回應逐筆再查詢之反模式;本規格以 SQLAlchemy event listener 計數斷言 SQL 陳述數為常數(SPEC.md L127、L182) |
| `selectinload` / `joinedload` | SQLAlchemy 顯式預載機制,防止 N+1(SPEC.md L127) |
| Alembic revision | migration 版本;本輪 v1 → v2 → v3,每步必有可運作 downgrade(SPEC.md L132) |
| SBOM | Software Bill of Materials,`08-config/SBOM.json`,含 `name` / `version` / `license` / `direct\|transitive`(SPEC.md L240) |
| transitive dependency | 間接依賴;以 `requirements.lock` 完整鎖定並納入 license 掃描(SPEC.md L237、L239) |
| MI(LLOC 加權) | Maintainability Index,以 LLOC 加權,門檻 ≥ 80(SPEC.md L269) |
| CC | Cyclomatic Complexity,單一函式 ≤ 10(SPEC.md L269) |
| p95 | 第 95 百分位延遲(SPEC.md L180) |
| ASGI transport | `httpx.AsyncClient(transport=ASGITransport(app))` 之端到端測試驅動,不得直接呼叫 handler(SPEC.md L263) |
| independence 模組 | import-linter 契約中不屬於分層的模組(本輪:`config`、`errors`)(SPEC.md L229) |
| correlation_id | RFC 7807 body 欄位,同時出現於 `X-Correlation-Id` header 與伺服器日誌(SPEC.md L167) |
| NFR-99 | 歧義解決條款:canonical 用語歧義時,由 test harness 與 stakeholder 確認後回填(§7 Open Issues) |
| FR-XX-deferred | 因 prompt-injection 等安全理由暫不轉錄之 canonical 條款(本輪無此項目,見 §7) |

## FR Block (machine-readable)

<!-- FR:START -->
```json
{
  "version": "1.0",
  "created_at": "2026-09-07",
  "phase": 1,
  "project": "taskq-api",
  "functional_requirements": [
    {"id": "FR-01", "description": "任務資源 CRUD API:POST /v1/tasks(write)、GET /v1/tasks/{id}(read)、GET /v1/tasks 分頁列表(read,cursor-based)、DELETE /v1/tasks/{id}(admin);驗證失敗 422、未知 id 404、預設 limit 50 上限 200(SPEC.md L79-91)", "implementation_functions": ["taskq_api.api.routes.tasks", "taskq_api.service.task_service", "taskq_api.repository.task_repo"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #4/#7/#8;NFR-10 CRUD 全鏈)"},
    {"id": "FR-02", "description": "任務執行端點:POST /v1/tasks/{id}/run(write)→202+run_id;asyncio.create_subprocess_exec(*shlex.split(command)) 禁 shell=True,timeout=TASKQ_TASK_TIMEOUT;狀態機 pending→running→done|failed|timeout;結果寫入 task_results;GET /v1/tasks/{id}/runs(read) 新到舊(SPEC.md L93-99)", "implementation_functions": ["taskq_api.api.routes.tasks", "taskq_api.service.runner"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #25、§11 孤兒進程)"},
    {"id": "FR-03", "description": "API Key 認證:全部 /v1/* 要求 X-API-Key,缺/無效→401;SHA-256 雜湊儲存不存明文,hmac.compare_digest 比對;key create 明文只印一次;revoked_at 非空視為無效;/healthz、/readyz 不要求認證(SPEC.md L101-107)", "implementation_functions": ["taskq_api.service.auth", "taskq_api.api.dependencies"], "verification_method": "pytest 03-development/tests(SPEC.md §8 #5/#18)"},
    {"id": "FR-04", "description": "Scope 授權:read < write < admin 階層包含;不足→403 且 body 不洩漏資源存在性;授權判定於單一 dependency 中介層(SPEC.md L109-113)", "implementation_functions": ["taskq_api.api.dependencies"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #6;單一 dependency 斷言測試)"},
    {"id": "FR-05", "description": "流量控制:per-token 令牌桶 TASKQ_RATE_BURST/TASKQ_RATE_PER_SEC;超限→429+Retry-After;狀態存資料庫,單一交易 row-level lock;/healthz、/readyz 不受限(SPEC.md L115-120)", "implementation_functions": ["taskq_api.api.dependencies", "taskq_api.repository.rate_buckets"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #9)"},
    {"id": "FR-06", "description": "持久化層與交易邊界:資料存取經 repository/ 層;每請求一 Session,context manager 保證 commit/rollback;禁字串拼接 SQL;selectinload/joinedload 顯式預載,N+1 為失敗條件;pool_size=TASKQ_DB_POOL_SIZE、pool_pre_ping=True(SPEC.md L122-128)", "implementation_functions": ["taskq_api.repository.session", "taskq_api.repository.task_repo"], "verification_method": "code review + SQLAlchemy event listener 計數測試(SPEC.md §8 #14、NFR-01)"},
    {"id": "FR-07", "description": "Schema Migration:Alembic 三步演進 v1(建 tasks/api_keys)→v2(tags/task_tags+name 唯一索引)→v3(result_json 拆至 task_results 含資料搬遷),每步有可運作 downgrade;upgrade head/downgrade base 成功;往返可逆性逐欄相同;禁 op.execute DROP TABLE 捷徑;migration 檔納入 offline SQL 測試(SPEC.md L130-143)", "implementation_functions": ["migrations/versions/v1_init.py", "migrations/versions/v2_tags.py", "migrations/versions/v3_split_results.py"], "verification_method": "真實 SQLite 檔案往返測試 + alembic offline SQL 斷言(SPEC.md §8 #12/#13、NFR-09)"},
    {"id": "FR-08", "description": "非同步執行器:asyncio.TaskGroup 管理,關閉時 graceful drain 至 TASKQ_DRAIN_TIMEOUT 逾時標記 interrupted;併發上限 TASKQ_MAX_CONCURRENT;wait_for timeout 後 kill()+await wait() 不留孤兒;CancelledError 必須向上傳播(SPEC.md L145-150)", "implementation_functions": ["taskq_api.service.runner"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #25、§11 孤兒進程)"},
    {"id": "FR-09", "description": "健康檢查與可觀測性:/healthz 無認證→200; /readyz 無認證,DB 可用且 alembic current==head→200 否則 503 說明失敗項;migration 未到 head 時 fail closed;/v1/metrics(admin)任務計數、延遲分位數、rate-limit 拒絕數(SPEC.md L152-160)", "implementation_functions": ["taskq_api.api.routes.health", "taskq_api.api.routes.metrics"], "verification_method": "pytest(SPEC.md §8 #10/#11)+ 服務啟動冒煙(§8 #27 之第 3 步)"},
    {"id": "FR-10", "description": "錯誤契約(RFC 7807):非 2xx 一律 application/problem+json;欄位 type/title/status/detail/instance/correlation_id;detail 不得含 SQL/堆疊/路徑/DB 結構;correlation_id 出現於 X-Correlation-Id header 與日誌;錯誤碼對照 422/401/403/404/409/429/503/500(SPEC.md L162-168、§7)", "implementation_functions": ["taskq_api.errors"], "verification_method": "pytest 03-development/tests/integration(SPEC.md §8 #19、§11 錯誤 body 洩漏)"}
  ],
  "non_functional_requirements": [
    {"id": "NFR-01", "type": "performance", "description": "效能與查詢效率:GET /v1/tasks/{id} p95<30ms、GET /v1/tasks?limit=50 p95<80ms(10,000 筆,ASGI transport,pytest-benchmark);列表端點 SQL 陳述數為常數(SQLAlchemy event listener 計數斷言)(SPEC.md L177-183)", "test_method": "pytest-benchmark + SQLAlchemy event listener 計數(SPEC.md §8 #14/#15)"},
    {"id": "NFR-02", "type": "security", "description": "HTTP 與資料層安全:禁 shell=True/eval(/exec(;禁字串拼接 SQL(grep+code review);金鑰雜湊儲存+hmac.compare_digest;403 不洩漏存在性;錯誤 body 不含堆疊/SQL/路徑;CORS 預設全拒、TASKQ_CORS_ORIGINS 明示;bandit 0 HIGH 0 MEDIUM(SPEC.md L185-194)", "test_method": "bandit -r 03-development/src/ + grep gates + code review(SPEC.md §8 #16/#17/#23)"},
    {"id": "NFR-03", "type": "reliability", "description": "錯誤處理、交易與非同步正確性:交易 commit/rollback context manager;禁裸 except: 與 except Exception: pass;CancelledError 不得被吞;DB 失敗→readyz 503 明確 detail 不無限重試;timeout 確實終止子進程;migration 失敗 rollback 維持前 revision(SPEC.md L196-204)", "test_method": "ast-error-handling 掃描(反模式)+ integration tests(SPEC.md §8 #10/#25)"},
    {"id": "NFR-04", "type": "security", "description": "敏感資料遮蔽:stdout_tail/stderr_tail/日誌/錯誤 body 匹配敏感 regex 之整行以 [REDACTED] 取代;DB 連線字串(含密碼)不得出現於日誌/錯誤/度量;API key 明文只印一次不落盤(SPEC.md L206-212)", "test_method": "unit test(§11 DB 連線字串)+ integration test(SPEC.md §8 #20)"},
    {"id": "NFR-05", "type": "documentation", "description": "文件覆蓋:全部公開函式/類別 docstring 含 [FR-XX]/[NFR-XX] 引用,覆蓋率 100%;每個端點 OpenAPI 有 summary 與 description(/openapi.json 斷言)(SPEC.md L214-218)", "test_method": "ast-docstrings 掃描 + /openapi.json 斷言測試"},
    {"id": "NFR-06", "type": "layering", "description": "架構分層契約:.importlinter 宣告 api > service > repository > models,config/errors 為 independence;repository 以外任何層不得 import sqlalchemy;lint-imports exit 0;禁刪檔/萬用字元 ignore_imports/降級 contract(SPEC.md L220-232)", "test_method": "lint-imports(SPEC.md §8 #21)+ .importlinter 內容斷言測試"},
    {"id": "NFR-07", "type": "licensing", "description": "依賴與授權合規:requirements.txt == 釘版、requirements.lock 完整鎖定;allowlist MIT/BSD-2-Clause/BSD-3-Clause/Apache-2.0/PSF;掃描完整依賴樹 pip-licenses --format=json --with-system;SBOM 產出於 08-config/SBOM.json(SPEC.md L234-240)", "test_method": "pip-licenses --format=json --with-system + SBOM 檔案檢查(SPEC.md §8 #22)"},
    {"id": "NFR-08", "type": "mutation", "description": "變異測試:features.mutation_testing: true;mutation score ≥ 70;範圍限 service/ 與 repository/ 並註記理由;不得調降 crg_cohesion_healthy(SPEC.md L242-247、L326)", "test_method": "mutation-test-score framework 命令 / mutmut results(SPEC.md §8 #24)"},
    {"id": "NFR-09", "type": "testability", "description": "驗證真實性(零 skip 鐵律):禁 skip/skipif/xfail/無斷言 stub;pytest -q skipped 計數 0;每測試函式 ≥1 assert;禁 --ignore/-k/--deselect/collect_ignore/testpaths 移除;FR-07 migration 以真實 SQLite 檔案測試;TRACEABILITY_MATRIX VERIFIED 僅在實測通過後給出(SPEC.md L249-257)", "test_method": "pytest 03-development/tests -q(skipped 計數)+ ast-assertions(zero_assert)+ 反造假 audit(SPEC.md §8 #1)"},
    {"id": "NFR-10", "type": "integration", "description": "整合覆蓋:03-development/tests/integration/ 行覆蓋 ≥ 80%;httpx.AsyncClient(transport=ASGITransport(app)) 驅動不得直呼 handler;至少涵蓋 CRUD 全鏈、401/403/404/409/422/429/503 各一例、migration 往返、rate limit 觸發與恢復、graceful drain(SPEC.md L259-264)", "test_method": "pytest 03-development/tests/integration --cov(SPEC.md §8 #3)"},
    {"id": "NFR-11", "type": "maintainability", "description": "可讀性:專案 MI(LLOC 加權)≥ 80;單一函式 CC ≤ 10;單一檔案 ≤ 400 行、單一目錄 ≤ 15 檔;每個 API handler ≤ 40 行(SPEC.md L266-271)", "test_method": "readability-v2 / radon mi、cc、raw(SPEC.md §11)"},
    {"id": "NFR-12", "type": "verifiability", "description": "系統驗證目標:Makefile verify-system 串接 alembic upgrade head、全套測試、服務啟動+/healthz、/readyz 冒煙、downgrade base 後再 upgrade head(往返);exit 0 且 stdout 印 verify-system: PASS;.env.example 宣告全部 12 個 TASKQ_*(SPEC.md L273-281、§5.1)", "test_method": "make verify-system(SPEC.md §8 #27)+ grep -c ^TASKQ_ .env.example(§8 #26)"}
  ]
}
```
<!-- FR:END -->
