# Holien Secrets Manager 規格文件

| 項目 | 內容 |
|------|------|
| 文件版本 | 0.1.0（草稿） |
| 狀態 | 待審閱 |
| 最後更新 | 2026-10-28 |
| 適用範圍 | 全專案（後端服務、CLI、Web UI） |

> 本文件為專案起始規格。專案目前為空專案，文件中所有「假設」與「待決議」項目皆需在開發前確認，詳見第 13 章與附錄 A。

---

## 目錄

1. [專案概述](#1-專案概述)
2. [名詞定義](#2-名詞定義)
3. [使用者與角色](#3-使用者與角色)
4. [功能需求](#4-功能需求)
5. [安全設計](#5-安全設計)
6. [系統架構](#6-系統架構)
7. [資料模型](#7-資料模型)
8. [API 規格](#8-api-規格)
9. [CLI 規格](#9-cli-規格)
10. [非功能需求](#10-非功能需求)
11. [建議技術選型](#11-建議技術選型)
12. [開發階段規劃](#12-開發階段規劃)
13. [待決議事項](#13-待決議事項)
- [附錄 A：假設清單](#附錄-a假設清單)
- [附錄 B：錯誤碼](#附錄-b錯誤碼)

---

## 1. 專案概述

### 1.1 目標

Holien Secrets Manager（以下簡稱 HSM，不與硬體安全模組混淆，見第 2 章）是一套自架（self-hosted）的機密資料管理系統，提供團隊集中、加密、可稽核地管理下列兩大類機密：

1. **帳號密碼（Credential）**：服務登入帳密、API Token、資料庫連線字串等。
2. **伺服器金鑰（Key Pair）**：伺服器使用的非對稱金鑰對，包含 SSH 金鑰與 TLS 憑證私鑰，涵蓋產生、匯入、儲存、取用、輪替、撤銷與綁定主機的完整生命週期。

### 1.2 要解決的問題

- 帳密與金鑰散落在個人電腦、聊天軟體、共用文件中，無法掌握誰持有、誰取用過。
- 伺服器私鑰缺乏集中保管與輪替機制，人員異動時難以確認金鑰是否已失效。
- 缺乏存取紀錄，發生事故時無法追溯。

### 1.3 範圍

**納入範圍（In Scope）**

- 機密資料的加密儲存、版本管理與存取控制。
- SSH / TLS 金鑰對的產生、匯入、匯出、指紋計算、輪替與撤銷。
- 金鑰與伺服器主機（Host）的綁定關係維護。
- 使用者帳號、角色權限（RBAC）、多因子驗證（MFA）。
- 完整稽核日誌（Audit Log）。
- REST API、CLI 工具、Web 管理介面。

**排除範圍（Out of Scope，至少第一版不做）**

- 自動將金鑰推送部署到目標伺服器（列為第三階段評估項目，見第 12 章）。
- 憑證簽發（CA）功能；系統只保管私鑰與憑證，不扮演 CA。
- 瀏覽器擴充套件、自動填表。
- 硬體安全模組（Hardware Security Module）整合；保留介面但不在第一版實作。

### 1.4 設計原則

1. **預設安全（Secure by default）**：所有機密靜態加密，私鑰永不以明文落地。
2. **最小權限**：每次取用機密值皆需明確授權並留下紀錄。
3. **可稽核**：任何讀取機密值的行為皆不可被略過紀錄。
4. **可自架**：單一二進位檔或單一容器即可啟動，不依賴外部雲端服務。
5. **API 優先**：Web UI 與 CLI 皆透過同一組 API 運作，無後門路徑。

---

## 2. 名詞定義

| 名詞 | 定義 |
|------|------|
| HSM | 本專案 Holien Secrets Manager 之簡稱。**不是**硬體安全模組。文件中提到硬體安全模組時一律使用全名。 |
| Secret（機密） | 系統中任何被加密保存的資料單位，包含帳密、Token、金鑰對等，皆有型別（type）。 |
| Credential（帳密） | Secret 的一種型別，由帳號、密碼、網址、備註等欄位組成。 |
| Key Pair（金鑰對） | Secret 的一種型別，包含私鑰、公鑰、演算法、指紋等欄位。 |
| Host（主機） | 一台伺服器或服務端點的紀錄，可與多把金鑰綁定。 |
| Vault（金鑰庫） | Secret 的邏輯容器，為權限控管的基本單位。一個 Secret 只屬於一個 Vault。 |
| Version（版本） | Secret 的值每次變更即產生一個新版本，舊版本保留供回溯。 |
| KEK | Key Encryption Key，主金鑰，用於包裝（wrap）DEK。 |
| DEK | Data Encryption Key，資料加密金鑰，每個 Secret 版本一把，用於實際加密機密值。 |
| Seal / Unseal | 服務啟動時 KEK 尚未載入的狀態稱為 sealed；完成 KEK 載入後為 unsealed，方可服務機密讀寫。 |
| Principal | 執行操作的主體，可能是使用者（User）或 API Token。 |

---

## 3. 使用者與角色

### 3.1 使用者類型

| 類型 | 說明 | 主要操作 |
|------|------|----------|
| 系統管理員（Admin） | 負責部署、備份、使用者管理 | 建立使用者、Unseal、查看全域稽核 |
| 維運人員（Ops） | 管理伺服器與金鑰 | 產生/輪替金鑰、綁定主機、取用私鑰 |
| 開發人員（Dev） | 使用帳密與 API Token | 讀取被授權的 Secret、建立個人 Token |
| 自動化系統（CI/CD、腳本） | 以 API Token 身分存取 | 讀取特定 Secret |

### 3.2 角色與權限（RBAC）

權限以 Vault 為單位授予。系統角色與 Vault 角色分開。

**系統角色**

| 角色 | 權限 |
|------|------|
| `system_admin` | 所有操作，包含使用者管理、系統設定、Unseal、全域稽核查詢。不自動擁有讀取任何 Vault 機密值的權限（需明確加入 Vault，確保稽核一致）。 |
| `user` | 一般使用者，僅能操作被授權的 Vault。 |

**Vault 角色**

| 角色 | 列出 Secret | 讀取 metadata | 讀取機密值 | 建立/修改 | 刪除 | 管理成員 |
|------|:--:|:--:|:--:|:--:|:--:|:--:|
| `owner` | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| `editor` | ✔ | ✔ | ✔ | ✔ | ✔ | ✘ |
| `reader` | ✔ | ✔ | ✔ | ✘ | ✘ | ✘ |
| `metadata_reader` | ✔ | ✔ | ✘ | ✘ | ✘ | ✘ |

- `metadata_reader` 可看到 Secret 存在、名稱、公鑰與指紋，但不可讀取私鑰或密碼。此角色供盤點與自動化檢查用。
- 第一版不做 Secret 層級的個別權限覆寫，列為第二階段（見第 12 章）。

---

## 4. 功能需求

需求編號格式：`FR-<模組>-<序號>`。優先級：**P0** 必要（MVP）、**P1** 重要、**P2** 可延後。

### 4.1 使用者與驗證（AUTH）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-AUTH-01 | P0 | 支援本地帳號，密碼以 Argon2id 雜湊儲存。 |
| FR-AUTH-02 | P0 | 登入成功後核發短效 Session Token（建議 15 分鐘）與 Refresh Token（建議 8 小時）。 |
| FR-AUTH-03 | P0 | 連續登入失敗達門檻（預設 5 次）後，帳號鎖定一段時間（預設 15 分鐘），並記錄稽核。 |
| FR-AUTH-04 | P0 | 使用者可建立 API Token；Token 可設定有效期限與可存取的 Vault 範圍，且只在建立當下顯示一次。 |
| FR-AUTH-05 | P1 | 支援 TOTP（RFC 6238）多因子驗證；管理員可設定全域強制啟用。 |
| FR-AUTH-06 | P1 | 管理員可停用使用者；停用後其所有 Session 與 API Token 立即失效。 |
| FR-AUTH-07 | P2 | 支援 OIDC 單一登入（SSO）。 |

### 4.2 Vault 管理（VAULT）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-VAULT-01 | P0 | 使用者可建立 Vault，建立者自動成為 `owner`。 |
| FR-VAULT-02 | P0 | Owner 可新增、移除成員並指派 Vault 角色。 |
| FR-VAULT-03 | P0 | 刪除 Vault 前必須先清空其中所有 Secret，或改以「封存」狀態保留。 |
| FR-VAULT-04 | P1 | 支援以群組（Group）為單位授權成員。 |

### 4.3 帳密管理（CRED）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-CRED-01 | P0 | 建立帳密型 Secret，欄位：名稱、帳號、密碼、網址、備註、標籤。密碼與備註為加密欄位。 |
| FR-CRED-02 | P0 | 修改密碼時產生新版本，舊版本保留，預設保留最近 10 個版本。 |
| FR-CRED-03 | P0 | 讀取密碼值需呼叫獨立端點，與讀取 metadata 分開，並產生稽核紀錄。 |
| FR-CRED-04 | P0 | 支援通用型 Secret（任意 key/value 或單一文字區塊），用於連線字串、設定檔等。 |
| FR-CRED-05 | P1 | 內建密碼產生器，可設定長度與字元集。 |
| FR-CRED-06 | P1 | 可設定到期日，到期前 N 天（預設 14 天）產生提醒。 |
| FR-CRED-07 | P2 | 支援附件（檔案）型 Secret，大小上限預設 1 MB。 |

### 4.4 伺服器金鑰管理（KEY）

本模組為專案核心，涵蓋伺服器私鑰與公鑰的完整生命週期。

#### 4.4.1 產生與匯入

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-01 | P0 | 在伺服器端產生金鑰對。支援演算法：`ed25519`（預設）、`rsa-4096`、`rsa-2048`、`ecdsa-p256`。 |
| FR-KEY-02 | P0 | 支援匯入既有私鑰（OpenSSH 格式、PEM / PKCS#8）。匯入時自動推導公鑰與指紋，並驗證私鑰與使用者提供的公鑰（若有）相符。 |
| FR-KEY-03 | P0 | 支援匯入 TLS 憑證與對應私鑰；系統自動解析憑證的 Subject、SAN、到期日並驗證私鑰與憑證匹配。 |
| FR-KEY-04 | P0 | 金鑰對可設定用途（`purpose`）：`ssh`、`tls`、`signing`、`other`。 |
| FR-KEY-05 | P1 | 產生金鑰時可選擇為私鑰加上 passphrase；passphrase 另存為加密欄位。 |
| FR-KEY-06 | P1 | 支援僅儲存公鑰（無私鑰）的紀錄，用於登錄外部人員或外部系統的公鑰。 |

#### 4.4.2 儲存與取用

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-10 | P0 | 私鑰以第 5 章定義的加密架構靜態加密；資料庫中不存在明文私鑰。 |
| FR-KEY-11 | P0 | 公鑰與指紋（SHA256 與 MD5 兩種格式）為非加密 metadata，`metadata_reader` 以上皆可讀取。 |
| FR-KEY-12 | P0 | 取用私鑰須呼叫獨立端點，需 `reader` 以上角色，且必須附上取用原因（`reason`，至少 1 字元）。每次取用皆寫入稽核。 |
| FR-KEY-13 | P0 | 私鑰可以 OpenSSH、PEM（PKCS#1 / PKCS#8）格式匯出；格式轉換於伺服器端記憶體內完成。 |
| FR-KEY-14 | P0 | 公鑰可以 OpenSSH `authorized_keys` 單行格式、PEM 格式輸出。 |
| FR-KEY-15 | P1 | 可依指紋反查金鑰（用於從伺服器 `authorized_keys` 盤點）。 |
| FR-KEY-16 | P2 | 支援 PuTTY `.ppk` 格式匯出。 |

#### 4.4.3 生命週期

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-20 | P0 | 金鑰狀態：`active`、`retired`（已輪替、仍可讀取）、`revoked`（已撤銷、不可再取用私鑰）。 |
| FR-KEY-21 | P0 | 輪替（rotate）：以相同演算法與設定產生新金鑰版本，舊版本自動轉為 `retired`。輪替後新舊版本並存，供維運人員完成伺服器端更換。 |
| FR-KEY-22 | P0 | 撤銷（revoke）：需填寫原因；撤銷後私鑰不可再被任何角色讀取，僅 `system_admin` 可於緊急情況下以「破窗」（break-glass）流程取回，並產生高優先稽核事件。 |
| FR-KEY-23 | P1 | 可設定金鑰到期日；TLS 憑證自動以憑證到期日為準。到期前 30 / 14 / 7 天產生提醒。 |
| FR-KEY-24 | P1 | 可設定輪替政策（例如每 90 天），到期產生「應輪替」提醒；第一版不自動執行輪替。 |
| FR-KEY-25 | P2 | 刪除金鑰採軟刪除，保留期（預設 30 天）後由排程永久清除。 |

#### 4.4.4 主機綁定

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-KEY-30 | P0 | 建立 Host 紀錄：名稱、位址（IP 或 FQDN）、環境（`prod`/`staging`/`dev`/`other`）、備註、標籤。 |
| FR-KEY-31 | P0 | 一把金鑰可綁定多台 Host，一台 Host 可綁定多把金鑰；綁定時記錄伺服器端使用的帳號（例如 `root`、`deploy`）與用途。 |
| FR-KEY-32 | P0 | 可列出某台 Host 所有綁定的金鑰，及某把金鑰綁定的所有 Host。 |
| FR-KEY-33 | P1 | 可記錄 Host 的 SSH host key 指紋，供 CLI 連線時比對。 |
| FR-KEY-34 | P2 | 透過 SSH 連線到 Host 掃描 `authorized_keys`，與系統紀錄比對並回報差異（只讀，不修改）。 |
| FR-KEY-35 | P2 | 透過 SSH 將公鑰推送到 Host 的 `authorized_keys`（需明確啟用且需二次確認）。 |

### 4.5 稽核日誌（AUDIT）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-AUDIT-01 | P0 | 下列事件必須記錄：登入成功/失敗、Token 建立/撤銷、Vault 建立/成員變更、Secret 建立/修改/刪除、**每一次機密值讀取**、金鑰產生/匯入/輪替/撤銷/私鑰取用、Unseal、系統設定變更。 |
| FR-AUDIT-02 | P0 | 每筆紀錄包含：時間（UTC）、Principal、操作類型、目標資源、來源 IP、User-Agent、結果（成功/失敗）、附加資訊（例如取用原因）。 |
| FR-AUDIT-03 | P0 | 稽核紀錄為只增不改（append-only），API 不提供修改或刪除端點。 |
| FR-AUDIT-04 | P0 | 寫入稽核失敗時，對應的機密讀取操作必須失敗（fail-closed）。 |
| FR-AUDIT-05 | P1 | 可依時間、Principal、資源、操作類型查詢與匯出（JSON Lines / CSV）。 |
| FR-AUDIT-06 | P1 | 可將稽核事件同步輸出到 syslog 或 Webhook。 |
| FR-AUDIT-07 | P2 | 稽核紀錄雜湊鏈（hash chain），可驗證未被竄改。 |

### 4.6 系統管理（SYS）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-SYS-01 | P0 | 首次啟動初始化流程：產生 KEK、建立第一位 `system_admin`、輸出 Unseal 所需資訊。 |
| FR-SYS-02 | P0 | 服務啟動後為 sealed 狀態，需 Unseal 才能存取機密；健康檢查端點回報 seal 狀態。 |
| FR-SYS-03 | P0 | 提供加密備份與還原指令；備份檔不含 KEK。 |
| FR-SYS-04 | P1 | KEK 輪替：以新 KEK 重新包裝所有 DEK，過程中服務維持可讀。 |
| FR-SYS-05 | P1 | 提供 Prometheus 格式指標端點。 |

### 4.7 介面（UI）

| 編號 | 優先 | 需求 |
|------|:--:|------|
| FR-UI-01 | P0 | CLI 覆蓋所有 P0 功能（規格見第 9 章）。 |
| FR-UI-02 | P1 | Web UI 覆蓋帳密與金鑰的日常操作、Vault 成員管理、稽核查詢。 |
| FR-UI-03 | P1 | Web UI 顯示機密值時預設遮罩，需點擊「顯示」並記錄稽核；複製到剪貼簿後 30 秒自動清除（瀏覽器允許時）。 |

---

## 5. 安全設計

### 5.1 加密架構（Envelope Encryption）

```
                 ┌──────────────────────────┐
                 │   KEK（主金鑰，256-bit）   │  ← 僅存在記憶體，來源見 5.2
                 └────────────┬─────────────┘
                              │ wrap / unwrap（AES-256-GCM）
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   DEK（Secret v1）     DEK（Secret v2）     DEK（Key Pair v1）   ← 每個版本一把，隨機 256-bit
          │                   │                   │
          ▼ AES-256-GCM       ▼                   ▼
   密文（密碼欄位）     密文（密碼欄位）      密文（私鑰 PEM）
```

- 每個 Secret 版本產生一把隨機 DEK（256-bit，CSPRNG）。
- 機密欄位以 AES-256-GCM 加密，nonce 為 96-bit 隨機值，每次加密不重複。
- AAD（Additional Authenticated Data）包含 `secret_id`、`version`、`field_name`，防止密文被搬移到其他紀錄。
- DEK 以 KEK 包裝後與密文一同存入資料庫；KEK 永不落地到資料庫。
- 資料庫中儲存的欄位：`ciphertext`、`nonce`、`wrapped_dek`、`kek_id`（供 KEK 輪替時辨識）。

### 5.2 KEK 來源與 Unseal

依部署需求擇一，由系統設定決定：

| 模式 | 說明 | 適用情境 |
|------|------|----------|
| `passphrase` | 管理員於 Unseal 時輸入通關密語，以 Argon2id（記憶體 64 MB、迭代 3、並行 4）推導 KEK。 | 小型團隊、單機 |
| `file` | KEK 存於服務主機上的受保護檔案（權限 0600），啟動時自動 Unseal。 | 容器化部署、搭配磁碟加密 |
| `kms` | 由外部 KMS（例如雲端 KMS 或硬體安全模組）包裝 KEK，啟動時請求解封。 | 第三階段 |

- Unseal 前：所有機密讀寫端點回傳 `503 SEALED`；metadata 查詢亦不開放，避免洩漏存在性。
- 第一版實作 `passphrase` 與 `file`，`kms` 保留介面。

### 5.3 私鑰處理原則

1. 私鑰明文僅在處理請求期間存在於伺服器記憶體，處理完畢立即覆寫（zeroize）。
2. 私鑰明文不得出現在任何日誌、錯誤訊息、Debug 輸出。
3. API 回傳私鑰時，回應標頭加上 `Cache-Control: no-store`，且禁止 HTTP 壓縮。
4. CLI 取得私鑰寫入檔案時，權限固定為 `0600`，並拒絕寫入到 world-readable 的目錄。
5. 已撤銷（`revoked`）的私鑰在資料層不刪除，但 API 層強制拒絕讀取；break-glass 例外見 FR-KEY-22。

### 5.4 傳輸與 API 安全

- 僅接受 HTTPS（TLS 1.2 以上，建議 1.3）；可設定由反向代理終止 TLS，此時服務需驗證 `X-Forwarded-*` 僅來自信任的代理。
- 所有回應加上 `Strict-Transport-Security`、`X-Content-Type-Options: nosniff`、`Content-Security-Policy`。
- 登入、Unseal、私鑰取用端點套用速率限制（預設每 IP 每分鐘 10 次）。
- API Token 以 `hsm_` 前綴 + 32 bytes 隨機值組成，資料庫只存其 SHA-256 雜湊。
- 請求體大小上限預設 2 MB。

### 5.5 威脅模型

| 威脅 | 防護措施 |
|------|----------|
| 資料庫檔案或備份外洩 | 所有機密欄位加密，KEK 不在資料庫與備份中。 |
| 內部人員超權讀取 | Vault RBAC、每次讀值皆稽核、私鑰取用需原因。 |
| 離職人員仍持有金鑰 | 停用帳號即失效 Token；金鑰輪替與撤銷流程；Host 綁定可盤點受影響主機。 |
| 服務主機記憶體被傾印 | KEK 與私鑰明文停留時間最短化、zeroize；此威脅無法完全消除，列為已知限制。 |
| API Token 外洩 | Token 有效期、Vault 範圍限制、可即時撤銷、稽核可追蹤來源 IP。 |
| 重放與密文搬移 | AES-GCM AAD 綁定資源識別；Session Token 短效。 |
| 暴力破解登入 | Argon2id、失敗鎖定、速率限制、MFA。 |

**已知限制**：擁有服務主機 root 權限且能取得記憶體的攻擊者可取得 KEK。系統不試圖防禦此情境，需以主機層級控管補強。

---

## 6. 系統架構

### 6.1 元件

```
┌──────────┐   ┌──────────┐   ┌─────────────┐
│  Web UI  │   │   CLI    │   │ CI/CD 腳本   │
└────┬─────┘   └────┬─────┘   └──────┬──────┘
     │ HTTPS         │ HTTPS          │ HTTPS（API Token）
     └───────────────┼────────────────┘
                     ▼
         ┌───────────────────────┐
         │   HSM Server（單一程序） │
         │  ├ API 層（REST）       │
         │  ├ 授權層（RBAC）       │
         │  ├ 加密層（KEK/DEK）    │
         │  ├ 金鑰引擎（產生/轉換）  │
         │  ├ 稽核寫入器           │
         │  └ 排程器（到期提醒等）   │
         └───────────┬───────────┘
                     ▼
         ┌───────────────────────┐
         │  資料庫                │
         │  SQLite（單機）/        │
         │  PostgreSQL（多節點）    │
         └───────────────────────┘
```

- 伺服器為單一程序，內嵌 Web UI 靜態檔，部署時為單一二進位檔或單一容器映像。
- Web UI 為純前端 SPA，只透過 REST API 與後端溝通，無伺服器端渲染機密值。
- CLI 與 Web UI 使用完全相同的 API，不存在只供其中一方使用的端點。

### 6.2 請求流程（讀取私鑰為例）

1. 用戶端帶 Bearer Token 呼叫 `POST /api/v1/keys/{id}/private`，body 含 `reason`。
2. API 層驗證 Token 有效、未過期、Principal 未被停用。
3. 授權層確認 Principal 在該金鑰所屬 Vault 擁有 `reader` 以上角色，且 Token 範圍包含該 Vault。
4. 檢查金鑰狀態非 `revoked`。
5. **先寫入稽核紀錄**（含 reason）；寫入失敗則回傳 500，不繼續。
6. 加密層以 KEK 解開 DEK，解密私鑰，依請求格式轉換。
7. 回傳私鑰，回應完成後 zeroize 記憶體。

### 6.3 部署模式

| 模式 | 說明 |
|------|------|
| 單機 | 單一程序 + SQLite，資料目錄需定期備份。適合第一階段。 |
| 容器 | 官方 Docker 映像，資料目錄掛載 volume。 |
| 多節點（第三階段） | 多個無狀態服務節點 + PostgreSQL，每個節點各自 Unseal。 |

---

## 7. 資料模型

以下為邏輯模型，實體欄位型別依選定資料庫調整。所有主鍵為 UUID v7，所有時間為 UTC。

### 7.1 `users`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| username | text | 唯一，3–64 字元 |
| email | text | 唯一 |
| password_hash | text | Argon2id 編碼字串 |
| system_role | enum | `system_admin` / `user` |
| mfa_enabled | bool | |
| mfa_secret_enc | bytes | TOTP 秘密，以 KEK 直接加密 |
| status | enum | `active` / `disabled` |
| failed_login_count | int | |
| locked_until | timestamp? | |
| created_at / updated_at | timestamp | |

### 7.2 `api_tokens`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| user_id | uuid | FK users |
| name | text | 使用者自訂名稱 |
| token_hash | text | SHA-256 |
| token_prefix | text | 前 8 字元，供辨識 |
| vault_scope | uuid[] | 可存取的 Vault；空陣列代表繼承使用者全部權限 |
| expires_at | timestamp? | |
| last_used_at | timestamp? | |
| revoked_at | timestamp? | |
| created_at | timestamp | |

### 7.3 `vaults`、`vault_members`

`vaults`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| name | text | 唯一 |
| description | text | |
| status | enum | `active` / `archived` |
| created_by | uuid | FK users |
| created_at / updated_at | timestamp | |

`vault_members`

| 欄位 | 型別 | 說明 |
|------|------|------|
| vault_id | uuid | FK，複合 PK |
| user_id | uuid | FK，複合 PK |
| role | enum | `owner` / `editor` / `reader` / `metadata_reader` |
| granted_by | uuid | FK users |
| created_at | timestamp | |

### 7.4 `secrets`、`secret_versions`

`secrets`（所有型別共用的 metadata）

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| vault_id | uuid | FK vaults |
| type | enum | `credential` / `generic` / `key_pair` / `tls_certificate` |
| name | text | 同一 Vault 內唯一 |
| description | text | |
| tags | text[] | |
| current_version | int | |
| expires_at | timestamp? | |
| status | enum | `active` / `retired` / `revoked` / `deleted` |
| revoked_reason | text? | |
| created_by | uuid | |
| created_at / updated_at / deleted_at | timestamp | |

`secret_versions`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| secret_id | uuid | FK secrets |
| version | int | 從 1 遞增，與 secret_id 複合唯一 |
| public_fields | jsonb | 非加密欄位（例如帳號、網址、公鑰、指紋） |
| encrypted_fields | jsonb | `{ field_name: { ciphertext, nonce } }` |
| wrapped_dek | bytes | |
| kek_id | text | |
| created_by | uuid | |
| created_at | timestamp | |

**各型別欄位定義**

| type | public_fields | encrypted_fields |
|------|---------------|------------------|
| `credential` | `username`, `url` | `password`, `notes` |
| `generic` | `keys`（欄位名稱列表） | 每個 key 一個項目，或單一 `content` |
| `key_pair` | `algorithm`, `purpose`, `public_key_openssh`, `public_key_pem`, `fingerprint_sha256`, `fingerprint_md5`, `comment`, `has_passphrase` | `private_key_pem`（PKCS#8）, `passphrase` |
| `tls_certificate` | `subject`, `sans`, `issuer`, `not_before`, `not_after`, `serial`, `certificate_pem`, `chain_pem`, `fingerprint_sha256` | `private_key_pem` |

### 7.5 `hosts`、`host_keys`

`hosts`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK |
| vault_id | uuid | FK，Host 隨所屬 Vault 控管權限 |
| name | text | 同一 Vault 內唯一 |
| address | text | IP 或 FQDN |
| environment | enum | `prod` / `staging` / `dev` / `other` |
| ssh_port | int | 預設 22 |
| host_key_fingerprint | text? | FR-KEY-33 |
| tags | text[] | |
| notes | text | |
| created_at / updated_at | timestamp | |

`host_keys`

| 欄位 | 型別 | 說明 |
|------|------|------|
| host_id | uuid | FK，複合 PK |
| secret_id | uuid | FK secrets（type 須為 key_pair 或 tls_certificate），複合 PK |
| remote_user | text | 伺服器端帳號，例如 `deploy` |
| usage | text | 用途說明 |
| created_by | uuid | |
| created_at | timestamp | |

### 7.6 `audit_logs`

| 欄位 | 型別 | 說明 |
|------|------|------|
| id | uuid | PK（v7，時間有序） |
| occurred_at | timestamp | |
| principal_type | enum | `user` / `api_token` / `system` |
| principal_id | uuid? | |
| action | text | 例如 `secret.value.read`、`key.private.read`、`key.rotate` |
| resource_type | text | `vault` / `secret` / `host` / `user` / `system` |
| resource_id | uuid? | |
| vault_id | uuid? | 冗餘欄位，加速查詢 |
| result | enum | `success` / `denied` / `error` |
| source_ip | inet | |
| user_agent | text | |
| details | jsonb | 例如 `{ "reason": "...", "version": 3, "format": "openssh" }` |
| prev_hash / hash | bytes? | FR-AUDIT-07 雜湊鏈，第一版可為空 |

### 7.7 `system_config`

| 欄位 | 型別 | 說明 |
|------|------|------|
| key | text | PK |
| value | jsonb | |

保存：`kek_mode`、`kek_id`、Argon2 參數與 salt（passphrase 模式）、KEK 驗證用密文（用以確認 Unseal 輸入正確）、初始化完成旗標。

---

## 8. API 規格

### 8.1 通則

- 基底路徑：`/api/v1`。
- 認證：`Authorization: Bearer <token>`，Session Token 與 API Token 皆走此標頭。
- 內容格式：`application/json; charset=utf-8`；私鑰與憑證匯出可以 `text/plain` 回傳。
- 分頁：`?limit=50&cursor=<opaque>`，回應含 `next_cursor`。
- 錯誤格式：

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "insufficient role on vault",
    "request_id": "01J..."
  }
}
```

- 錯誤碼清單見附錄 B。

### 8.2 系統與認證

| 方法 | 路徑 | 權限 | 說明 |
|------|------|------|------|
| GET | `/health` | 無 | 回傳 `{ status, sealed, version }` |
| GET | `/metrics` | 設定決定 | Prometheus 指標（FR-SYS-05） |
| POST | `/system/init` | 無（僅限未初始化） | 首次初始化 |
| POST | `/system/unseal` | 無 | body：`{ passphrase }`（passphrase 模式） |
| POST | `/system/seal` | system_admin | 手動進入 sealed 狀態 |
| POST | `/auth/login` | 無 | body：`{ username, password }`；啟用 MFA 時回傳 `mfa_required` 與暫時 token |
| POST | `/auth/mfa/verify` | 暫時 token | body：`{ code }` |
| POST | `/auth/refresh` | Refresh Token | |
| POST | `/auth/logout` | 已登入 | |
| GET | `/me` | 已登入 | 目前 Principal 資訊 |
| POST | `/me/mfa/enroll` | 已登入 | 回傳 TOTP 秘密與 QR 資料 |
| POST | `/me/mfa/confirm` | 已登入 | 確認啟用 |
| GET/POST | `/me/tokens` | 已登入 | 列出 / 建立 API Token |
| DELETE | `/me/tokens/{id}` | 已登入 | 撤銷 |

### 8.3 使用者管理（system_admin）

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | `/users` | |
| POST | `/users` | body：`{ username, email, password, system_role }` |
| GET | `/users/{id}` | |
| PATCH | `/users/{id}` | 修改 email、system_role、status |
| POST | `/users/{id}/reset-password` | 產生一次性重設 |
| POST | `/users/{id}/mfa/reset` | 清除 MFA |

### 8.4 Vault

| 方法 | 路徑 | 最低角色 | 說明 |
|------|------|------|------|
| GET | `/vaults` | 已登入 | 只列出有權限的 Vault |
| POST | `/vaults` | 已登入 | |
| GET | `/vaults/{id}` | metadata_reader | |
| PATCH | `/vaults/{id}` | owner | |
| DELETE | `/vaults/{id}` | owner | 需為空或已封存 |
| GET | `/vaults/{id}/members` | metadata_reader | |
| PUT | `/vaults/{id}/members/{user_id}` | owner | body：`{ role }` |
| DELETE | `/vaults/{id}/members/{user_id}` | owner | 不可移除最後一位 owner |

### 8.5 Secret（帳密與通用型）

| 方法 | 路徑 | 最低角色 | 說明 |
|------|------|------|------|
| GET | `/vaults/{id}/secrets` | metadata_reader | 支援 `?type=&tag=&q=` |
| POST | `/vaults/{id}/secrets` | editor | body 依型別，見下 |
| GET | `/secrets/{id}` | metadata_reader | 只回 metadata 與 public_fields |
| PATCH | `/secrets/{id}` | editor | 修改名稱、描述、標籤、到期日 |
| PUT | `/secrets/{id}/value` | editor | 寫入新值，產生新版本 |
| POST | `/secrets/{id}/value` | reader | **讀取機密值**，body：`{ version?, reason? }`，稽核 |
| GET | `/secrets/{id}/versions` | metadata_reader | 版本列表（不含值） |
| DELETE | `/secrets/{id}` | editor | 軟刪除 |

建立 credential 範例：

```json
POST /api/v1/vaults/{id}/secrets
{
  "type": "credential",
  "name": "prod-db-admin",
  "tags": ["db", "prod"],
  "fields": {
    "username": "admin",
    "password": "s3cr3t",
    "url": "postgres://db.internal:5432",
    "notes": "每季輪替"
  },
  "expires_at": "2027-01-31T00:00:00Z"
}
```

### 8.6 金鑰對

金鑰對本質是 `type=key_pair` 的 Secret，共用 8.5 的 metadata 端點，另提供專屬端點：

| 方法 | 路徑 | 最低角色 | 說明 |
|------|------|------|------|
| POST | `/vaults/{id}/keys/generate` | editor | body：`{ name, algorithm, purpose, comment?, passphrase?, tags?, expires_at? }` |
| POST | `/vaults/{id}/keys/import` | editor | body：`{ name, private_key_pem, passphrase?, public_key?, purpose, ... }` |
| POST | `/vaults/{id}/keys/import-public` | editor | 僅公鑰（FR-KEY-06） |
| GET | `/keys/{id}/public` | metadata_reader | `?format=openssh\|pem`；可 `Accept: text/plain` |
| POST | `/keys/{id}/private` | reader | **取用私鑰**，body：`{ reason, format: "openssh"\|"pkcs8"\|"pkcs1", version? }`，稽核 |
| POST | `/keys/{id}/rotate` | editor | body：`{ reason?, comment? }`；回傳新版本 metadata |
| POST | `/keys/{id}/revoke` | editor | body：`{ reason }` |
| POST | `/keys/{id}/break-glass` | system_admin | body：`{ reason, ticket_ref? }`；取回 revoked 私鑰 |
| GET | `/keys/{id}/hosts` | metadata_reader | 綁定的主機 |
| GET | `/keys/by-fingerprint/{fp}` | 已登入 | 只回有權限的結果 |

### 8.7 TLS 憑證

| 方法 | 路徑 | 最低角色 | 說明 |
|------|------|------|------|
| POST | `/vaults/{id}/certificates/import` | editor | body：`{ name, certificate_pem, chain_pem?, private_key_pem, tags? }` |
| GET | `/certificates/{id}/certificate` | metadata_reader | 回傳憑證與鏈 |
| POST | `/certificates/{id}/private` | reader | 同金鑰取用規則 |
| POST | `/certificates/{id}/renew` | editor | 以新憑證與私鑰建立新版本 |
| GET | `/certificates/expiring` | 已登入 | `?within_days=30` |

### 8.8 Host

| 方法 | 路徑 | 最低角色 | 說明 |
|------|------|------|------|
| GET/POST | `/vaults/{id}/hosts` | metadata_reader / editor | |
| GET/PATCH/DELETE | `/hosts/{id}` | metadata_reader / editor / editor | |
| GET | `/hosts/{id}/keys` | metadata_reader | 綁定金鑰列表，含 remote_user |
| PUT | `/hosts/{id}/keys/{secret_id}` | editor | body：`{ remote_user, usage }` |
| DELETE | `/hosts/{id}/keys/{secret_id}` | editor | |
| GET | `/hosts/{id}/authorized-keys` | metadata_reader | 輸出該 Host 所有 active 金鑰的 `authorized_keys` 內容（text/plain） |

### 8.9 稽核

| 方法 | 路徑 | 權限 | 說明 |
|------|------|------|------|
| GET | `/audit` | system_admin | `?from=&to=&principal_id=&action=&resource_id=&vault_id=` |
| GET | `/vaults/{id}/audit` | owner | 該 Vault 範圍 |
| GET | `/audit/export` | system_admin | `?format=jsonl\|csv` |

---

## 9. CLI 規格

### 9.1 基本

- 執行檔名稱：`hsm`（待決議，見第 13 章）。
- 設定檔：`~/.config/hsm/config.yaml`，含 `server_url`；Token 存於作業系統金鑰圈（macOS Keychain / Linux Secret Service / Windows Credential Manager），無法使用時退回到 `0600` 權限的檔案並警告。
- 環境變數：`HSM_SERVER`、`HSM_TOKEN` 優先於設定檔，供 CI 使用。
- 輸出：預設人類可讀表格，`--json` 輸出 JSON。
- 所有會讀取機密值的指令支援 `--reason`，若未提供且為互動終端則提示輸入。

### 9.2 指令

```
hsm login [--server URL]
hsm logout
hsm whoami

hsm vault list
hsm vault create <name> [--description TEXT]
hsm vault member add <vault> <username> --role ROLE
hsm vault member remove <vault> <username>

hsm secret list <vault> [--type TYPE] [--tag TAG]
hsm secret create <vault>/<name> --type credential --username U [--password P | --generate] [--url URL]
hsm secret create <vault>/<name> --type generic --from-file PATH | --field k=v ...
hsm secret get <vault>/<name> [--field password] [--version N] [--reason TEXT]
hsm secret set <vault>/<name> --field password [--value V | --generate | --stdin]
hsm secret versions <vault>/<name>
hsm secret delete <vault>/<name>

hsm key generate <vault>/<name> [--algorithm ed25519|rsa-4096|rsa-2048|ecdsa-p256] [--purpose ssh|tls|signing|other] [--comment TEXT] [--passphrase]
hsm key import <vault>/<name> --private-key PATH [--public-key PATH] [--purpose ...]
hsm key import-public <vault>/<name> --public-key PATH
hsm key show <vault>/<name>                     # metadata、公鑰、指紋、綁定主機
hsm key public <vault>/<name> [--format openssh|pem] [--out PATH]
hsm key private <vault>/<name> --reason TEXT [--format openssh|pkcs8|pkcs1] [--out PATH]   # 預設寫到檔案而非 stdout
hsm key rotate <vault>/<name> [--reason TEXT]
hsm key revoke <vault>/<name> --reason TEXT
hsm key find --fingerprint SHA256:...

hsm cert import <vault>/<name> --cert PATH --key PATH [--chain PATH]
hsm cert show <vault>/<name>
hsm cert expiring [--within-days N]

hsm host list <vault> [--env prod|staging|dev]
hsm host add <vault>/<name> --address ADDR [--env ENV] [--port N]
hsm host bind <vault>/<host> <vault>/<key> --remote-user USER [--usage TEXT]
hsm host unbind <vault>/<host> <vault>/<key>
hsm host keys <vault>/<host>
hsm host authorized-keys <vault>/<host> [--out PATH]

hsm token create --name NAME [--expires 90d] [--vault V ...]
hsm token list
hsm token revoke <id>

hsm audit list [--from T] [--to T] [--action A] [--vault V]

hsm admin init
hsm admin unseal
hsm admin user create <username> --email E [--role system_admin|user]
hsm admin backup --out PATH
hsm admin restore --in PATH
hsm admin rotate-kek
```

### 9.3 行為規範

- `hsm key private` 預設輸出到檔案（`--out`，未指定時為 `./<name>`），權限 `0600`；要輸出到 stdout 需明確加 `--stdout`，且在偵測到 stdout 為終端時再次確認。
- 任何指令都不得將機密值寫入 shell history 以外的地方；文件應提醒使用者 `--password` 參數會留在 history，建議用 `--stdin` 或 `--generate`。
- 非 0 結束碼：1 一般錯誤、2 參數錯誤、3 認證失敗、4 權限不足、5 服務 sealed。

---

## 10. 非功能需求

| 類別 | 需求 |
|------|------|
| 效能 | 單機 SQLite 模式下，讀取機密值 P95 < 100 ms（不含網路）；金鑰產生 rsa-4096 < 3 s。 |
| 容量 | 第一版目標：100 位使用者、50 個 Vault、10,000 個 Secret、100 萬筆稽核紀錄。 |
| 可用性 | 單機模式不要求高可用；需在 5 分鐘內可由備份還原。 |
| 備份 | 備份檔為加密資料庫快照，不含 KEK；還原後需以原 KEK Unseal。 |
| 可觀測性 | 結構化日誌（JSON），日誌中絕不包含機密值、Token、密碼；Prometheus 指標。 |
| 相容性 | 伺服器：Linux x86_64 / arm64。CLI：Linux、macOS、Windows。瀏覽器：近兩年主流版本。 |
| 測試 | 加密層與授權層單元測試覆蓋率 ≥ 90%；每個 API 端點至少一個整合測試；金鑰匯入匯出需與 OpenSSH `ssh-keygen` 互通測試。 |
| 安全驗證 | 發布前執行相依套件弱點掃描；私鑰取用端點需有「未寫入稽核即拒絕」的自動化測試。 |
| 國際化 | 第一版 UI 與 CLI 訊息以繁體中文與英文雙語提供，預設依系統語系。 |

---

## 11. 建議技術選型

以下為建議，供第 13 章決議；未定案前不應開始實作。

| 層級 | 建議 | 理由 |
|------|------|------|
| 後端語言 | Go | 單一靜態二進位檔易部署；標準庫 `crypto` 完整，`golang.org/x/crypto/ssh` 可直接處理 OpenSSH 格式；記憶體控制較易做 zeroize。 |
| 資料庫 | SQLite（第一階段）→ PostgreSQL（多節點） | 以 SQL 抽象層隔離，schema 遷移工具化。 |
| 加密 | AES-256-GCM、Argon2id、ed25519 / RSA / ECDSA 皆用標準庫 | 避免自行實作密碼學。 |
| Web UI | TypeScript + 輕量前端框架，建置後內嵌於後端 | 單一部署產物。 |
| CLI | 與後端同語言、同 repo，共用 API client | 減少重複與行為差異。 |
| 封裝 | 多平台二進位檔 + Docker 映像 | |

替代方案：若團隊以 TypeScript / Node.js 為主，可改用 Node.js 後端，但需評估 OpenSSH 金鑰格式處理與記憶體 zeroize 的成熟度。

---

## 12. 開發階段規劃

| 階段 | 目標 | 主要交付 | 對應需求 |
|------|------|----------|----------|
| **Phase 0：確認** | 規格審閱、技術選型定案、repo 骨架 | 本文件 1.0、CI 流程、專案結構 | 第 13 章全部 |
| **Phase 1：MVP** | 可自架、可用 CLI 完成所有核心操作 | 後端服務、CLI、SQLite、passphrase / file Unseal、本地帳號、Vault RBAC、帳密與通用 Secret、金鑰產生/匯入/取用/輪替/撤銷、Host 綁定、稽核、備份還原 | 所有 P0 |
| **Phase 2：日常可用** | Web UI、安全強化、提醒 | Web UI、TOTP MFA、到期與輪替提醒、群組授權、稽核查詢匯出與 syslog 輸出、KEK 輪替、Prometheus、密碼產生器、指紋反查 | 所有 P1 |
| **Phase 3：擴充** | 整合與規模化 | OIDC SSO、KMS 模式、PostgreSQL 多節點、Host 掃描與公鑰推送、附件型 Secret、PuTTY 匯出、稽核雜湊鏈 | P2 |

每階段結束條件：對應需求的自動化測試全部通過、安全檢查清單（第 5 章）逐項確認、文件更新。

---

## 13. 待決議事項

| 編號 | 問題 | 建議 | 影響 |
|------|------|------|------|
| Q-01 | 後端語言與框架 | Go（見第 11 章） | 全部 |
| Q-02 | CLI 執行檔名稱：`hsm` 易與硬體安全模組混淆 | 改為 `holisec` 或 `hsecret` | 第 9 章 |
| Q-03 | 第一版是否需要 Web UI，或僅 CLI | Web UI 排 Phase 2 | Phase 1 範圍 |
| Q-04 | 預設 KEK 模式 | 單機用 `passphrase`，容器用 `file` | FR-SYS-01、5.2 |
| Q-05 | 稽核紀錄保留期限與是否可清除 | 永久保留，只提供匯出歸檔 | FR-AUDIT-03 |
| Q-06 | 是否需要 Secret 層級權限（比 Vault 更細） | Phase 2 再評估 | 3.2 |
| Q-07 | 是否要在第一版就做 Host 的公鑰推送 | 否，只提供 `authorized-keys` 輸出 | FR-KEY-35 |
| Q-08 | 公鑰是否允許匿名（未登入）讀取，供伺服器自動抓取 | 否，一律需 Token | FR-KEY-11 |
| Q-09 | 版本保留數量與軟刪除保留期 | 10 版、30 天 | FR-CRED-02、FR-KEY-25 |
| Q-10 | 部署目標環境（實體機、VM、K8s） | 需確認以決定 Phase 1 封裝方式 | 6.3 |

---

## 附錄 A：假設清單

本文件在下列假設下撰寫，任一假設不成立時需修訂對應章節：

1. 使用者為公司內部團隊，規模在 100 人以內；不需要多租戶（multi-tenant）。
2. 系統部署於公司可控的環境，可自行管理 TLS 憑證與網路存取。
3. 「伺服器 private + public key」主要指 SSH 金鑰與 TLS 私鑰，不含 PGP 金鑰與程式碼簽章金鑰（後兩者可用 `purpose=signing` 保存，但不提供專屬功能）。
4. 系統不負責簽發憑證，只保管。
5. 第一版不需要與既有身分系統整合，使用本地帳號。
6. 文件與介面以繁體中文為主、英文為輔。

## 附錄 B：錯誤碼

| HTTP | code | 說明 |
|------|------|------|
| 400 | `INVALID_REQUEST` | 參數格式錯誤 |
| 400 | `INVALID_KEY_MATERIAL` | 私鑰 / 憑證格式無法解析或不匹配 |
| 400 | `REASON_REQUIRED` | 取用私鑰未提供原因 |
| 401 | `UNAUTHENTICATED` | Token 無效或過期 |
| 401 | `MFA_REQUIRED` | 需完成 MFA |
| 403 | `FORBIDDEN` | 角色不足或 Token 範圍不含該 Vault |
| 403 | `KEY_REVOKED` | 金鑰已撤銷，不可取用 |
| 404 | `NOT_FOUND` | 資源不存在或無權限得知其存在 |
| 409 | `CONFLICT` | 名稱重複、Vault 非空等 |
| 409 | `LAST_OWNER` | 不可移除最後一位 owner |
| 423 | `ACCOUNT_LOCKED` | 登入失敗次數過多 |
| 429 | `RATE_LIMITED` | 速率限制 |
| 500 | `AUDIT_WRITE_FAILED` | 稽核寫入失敗，操作已中止 |
| 503 | `SEALED` | 服務尚未 Unseal |
| 503 | `NOT_INITIALIZED` | 尚未完成初始化 |
