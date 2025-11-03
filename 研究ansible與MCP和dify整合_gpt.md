# 以 Ansible × MCP × Dify 實現「AI Agent 自動化 CVE 升級派送」—Markdown 研究報告

> **事實檢查聲明**：以下內容嚴格引用官方或一手技術文件與可驗證來源；凡屬設計建議、範例或推論，均已標註為「**設計建議**／**推論**」。
> 主要依據：MCP 規格（JSON-RPC、STDIO/HTTP 傳輸）、Dify 對 MCP 的原生支援、Ansible AWX/Controller API（`/api/v2/job_templates/{id}/launch/` 等）、EDA（webhook/Kafka 事件源），以及 NVD/CVSS/EPSS/CISA KEV 等風險資料源。([modelcontextprotocol.io][1]) ([docs.dify.ai][2]) ([docs.ansible.com][3]) ([ansible.readthedocs.io][4]) ([nvd.nist.gov][5])

---

## 一、執行摘要

**核心結論：可以做到。**
以 **Dify Agent（MCP client）→ Ansible MCP Server → Ansible AWX/Controller API** 作為主鏈路：當 Agent 依**CVSS/EPSS/CISA KEV** 等指標判定「需升級」時，由 MCP 工具呼叫 AWX/Controller 的 **Job Template Launch API** 派送升級任務，並回收 **Job/Events** 以呈現結果與稽核。MCP 傳輸採 **HTTP（含 SSE/Streamable HTTP）或 STDIO**；控制面採 **HTTPS REST** 與最小權限 Token/RBAC。若遇限制作業，可退回 **Webhook/EDA** 或純 **REST** 直呼。([modelcontextprotocol.io][6]) ([docs.ansible.com][3]) ([ansible.readthedocs.io][4])

**關鍵發現**

* **Dify 原生支援 MCP（雙向）**：可連外部 MCP servers，也能將 Dify App/Workflow 發佈為 MCP Server。([docs.dify.ai][2])
* **Ansible 可被遠端觸發與監看**：AWX/Controller API 支援 **launch job template / 查詢 job events/stdout**；亦可用 **EDA** 接 webhook/Kafka 等事件。([docs.ansible.com][3]) ([ansible.readthedocs.io][4])
* **CVE 決策需多因子**：CVSS v3.1/v4.0（嚴重度）、**CISA KEV**（已被利用）、**EPSS**（被利用機率）、與**業務關鍵度**。([first.org][7]) ([cisa.gov][8]) ([api.first.org][9])

---

## 二、技術可行性評估

### 1) MCP 核心與整合機制

* **訊息格式與傳輸**：MCP 使用 **JSON-RPC 2.0**；標準傳輸包含 **STDIO** 與 **Streamable HTTP**（HTTP/SSE）。並定義工具／資源／提示等核心原語。([modelcontextprotocol.io][1])
* **實作注意**：STDIO server **不可寫 stdout 日誌**（避免破壞 JSON-RPC），HTTP-based server 可正常記錄。([modelcontextprotocol.io][10])

### 2) Dify 對 MCP 的支援現況

* 官方文件與公告顯示：**在 Dify 後台可直接管理 MCP servers**，並可**將 Dify App 發佈為 MCP Server**；**v1.6.0** 公告強調「雙向 MCP 原生化」。([docs.dify.ai][2])

### 3) Ansible 是否有 MCP Server 實作

* 已見多個**社群 MCP Server for Ansible/AAP** 專案，可列舉作為 PoC 基礎或參考（**非官方**）：`mcp-server-aap`、`AAP-Enterprise-MCP-Server`、`mcp-ansible`、`mcp-sysoperator` 等。([GitHub][11])
* Red Hat 文件亦描述 Ansible Lightspeed 與 MCP 整合（說明 MCP 作為外部能力來源標準），顯示供應商生態正推進。([docs.redhat.com][12])

### 4) 三者之間的資料流與協定

* **Dify（MCP client） ↔ Ansible MCP Server**：**HTTP（SSE/流式）或 STDIO**；MCP 工具 schema 探測與呼叫（JSON）。([modelcontextprotocol.io][1])
* **Ansible MCP Server ↔ AWX/Controller**：**HTTPS REST API**（例如 **POST** `/api/v2/job_templates/{id}/launch/` 啟動任務；**GET** `/api/v2/jobs/{id}/job_events/` 取事件）。([docs.ansible.com][3])

---

## 三、CVE 判斷與決策機制

### 5) 判斷所需資料來源

* **NVD API v2**（CVE/CPE 查詢、分頁/過濾）。([nvd.nist.gov][5])
* **CVSS v3.1 / v4.0 規格**（Base/Threat/Environmental/Supplemental）。([first.org][7])
* **CISA KEV Catalog**（已被野外利用）。([cisa.gov][8])
* **EPSS API**（被利用機率，支援 time-series）。([api.first.org][9])

### 6) 重要判斷因子（**設計建議**）

* **是否列入 KEV**、**EPSS 閾值**（如 `≥0.5`）、**CVSS 基礎分/情境分**、**資產業務關鍵度（CMDB 標籤）**、**相依性/相容性（SBOM/套件管理器）**、**替代緩解措施**等。（來源見上一節）

### 7) 決策樹／規則引擎（**設計建議**）

```yaml
rules:
  - name: "強制立即修補"
    when: { kev_listed: true }      # CISA KEV
    action: "deploy_now"

  - name: "高風險優先"
    when:
      any:
        - epss_gte: 0.5              # EPSS
        - all:                       # CVSS + 業務權重
            - cvss_base_gte: 9.0
            - business_criticality_in: ["P0","P1"]
    action: "deploy_canary"

  - name: "觀察/排程"
    when:
      all:
        - epss_lt: 0.05
        - kev_listed: false
    action: "schedule_window"
```

### 8) 人工審核（Human-in-the-Loop）

* 生產環境建議採「**雙簽審批**（安全/系統）→ 才允許下發**」，並在 Dify 流程中加入「人為確認節點」與變更單關聯，以合規與審計對應（**設計建議**）。MCP 架構天然配合人機互動與授權節點。([modelcontextprotocol.io][13])

---

## 四、技術架構設計

### 9) 角色與責任

* **Dify Agent（MCP client）**：聚合 NVD/KEV/EPSS 資料、執行規則、提出升級建議並透過 MCP 呼叫工具。([docs.dify.ai][2])
* **Ansible MCP Server**：以 MCP tools 暴露「列出模板 / 啟動 Job Template / 查詢 Job/Events」等能力（可採用社群 server）。([GitHub][11])
* **AWX/Controller（或 AAP）/EDA**：承接工作執行、事件產生（可改走 EDA 由 webhook/Kafka 觸發）。([docs.ansible.com][3])

### 10) Dify 透過 MCP 呼叫 Ansible Playbook（**設計示意**）

**MCP Tool 定義（片段）**：

```json
{
  "name": "launch_job_template",
  "description": "Launch an Ansible Job Template with extra_vars",
  "input_schema": {
    "type": "object",
    "properties": {
      "template_id": {"type": "integer"},
      "extra_vars": {"type": "object"},
      "limit": {"type": "string"}
    },
    "required": ["template_id"]
  }
}
```

對應 Ansible 端：`POST /api/v2/job_templates/{id}/launch/`，回傳 `job`/`id` 後，以 `GET /api/v2/jobs/{id}/job_events/`/`stdout` 追蹤狀態與輸出。([docs.ansible.com][3])

### 11) 中介層 / API Gateway（**設計建議**）

* 以 **Gateway** 聚合與限流（IP 白名單 / Scope 限制 / 審計）；或以 **Message Queue → EDA** 落實事件驅動（降曝控制面）。([ansible.readthedocs.io][4])

### 12) 認證與授權

* **MCP 面**：遵循 MCP HTTP 授權與連線規範（在企業環境導入 OAuth/憑證管理，**設計建議**）。([modelcontextprotocol.io][6])
* **AWX 面**：採用 **Token（OAuth2 Personal Token）/RBAC**，API 一般以 Bearer 使用。([Debug This][14])

---

## 五、Ansible 自動化實施

### 13) Playbook 設計：多類型 CVE 升級

**RHEL/DNF：僅安裝安全性更新**（`security: true`）

```yaml
- hosts: linux_group
  become: true
  tasks:
    - name: 安裝安全性更新
      ansible.builtin.dnf:
        name: "*"
        state: latest
        security: true
        update_cache: true
```

([docs.ansible.com][15])

**Debian/Ubuntu（APT）**：APT 無原生 `security: true`；常以 **unattended-upgrades** 或分離 security sources／自訂篩選達成（**實務做法**）。([GitHub][16])

**配置變更＋服務重啟（handlers/tags）**

```yaml
- name: 更新設定並通知重啟
  ansible.builtin.template:
    src: app.conf.j2
    dest: /etc/app.conf
  notify: restart_app
  tags: [config]

handlers:
  - name: restart_app
    ansible.builtin.service:
      name: app
      state: restarted
```

### 14) 冪等與回滾

* **冪等**：優先用具冪等模組（package/file/service），搭配 **check mode / diff**。([docs.ansible.com][17])
* **回滾**：以 **block/rescue/always** 撰寫補償步驟（失敗即還原版本／設定），並於管控層落實映像/快照策略（**設計建議**）。([docs.ansible.com][18])

### 15) 多環境部署策略（**設計建議**）

* 透過 **inventory 分組＋vars** 區分 **DEV/UAT/PROD**；生產採 **serial** 分批、**canary** 先行、必要時 **維護視窗**。

### 16) 監控與記錄

* 以 **Jobs / Job Events API** 回收執行細節與審計；亦可用 **bulk job launch** 進行批次啟動。([ansible.readthedocs.io][19])

---

## 六、風險管理與最佳實踐

### 17) 風險與防護

* **誤判自動推送**：以 **KEV + EPSS + CVSS + 業務權重**多因子治理，並採「**生產必須人工覆核**」。([cisa.gov][8])
* **供應鏈／MCP 安全**：第三方 MCP 伺服器需**來源審核、簽章校驗、最小權限**；近期有惡意 MCP 伺服器事件曝險示例（供警惕）。([IT Pro][20])
* **控制面曝露**：以 **API Gateway**、私網/防火牆、Token 有效期、細粒度 RBAC；可用 **EDA** 事件通道減少直接暴露。([ansible.readthedocs.io][4])

### 18) 升級前後檢查（**設計建議**）

* **前**：健康度/容量（Disk/CPU/RAM）、快照/備份、相依性檢查、維護視窗公告。
* **後**：服務健康檢測、合規掃描、回報 Job/Events 與變更單關聯。

### 19) 失敗處理與告警（**設計建議**）

* `block/rescue` 邏輯、**重試/回滾**、告警（ChatOps/Email/ITSM），並以 API 抽取 stdout 與事件偵錯。([ansible.readthedocs.io][19])

### 20) 參考框架

* **NIST SP 800-40r4**（企業修補治理）、**OWASP VMG**、**CIS 指南**。([nvlpubs.nist.gov][21])

---

## 七、實施路徑與替代方案

### 21) 替代整合路徑

* **Webhook 直觸**：Dify/外部服務 → **AWX Webhook** 或 **EDA Webhook**（常見事件源之一）。([ansible.readthedocs.io][4])
* **純 REST**：略過 MCP，由 Agent 直接呼叫 AWX/Controller API。([docs.ansible.com][3])
* **Message Queue**：Agent → **Kafka/MQ** → **EDA** 觸發（降低控制面直接暴露）。([ansible.readthedocs.io][4])

### 22) 分階段實施計畫（**設計建議**）

1. **PoC（2–3 週）**

   * Dify 連接一個**示範用 Ansible MCP Server**（或自建簡版）。
   * AWX 建立 1 個 **Job Template**（安全更新）與 **canary 群組**。
   * 觸發條件僅用 **KEV** 或 **EPSS ≥ 0.5**，**必須人工確認**後派送至 canary。([cisa.gov][8])
2. **Pilot（4–6 週）**

   * 擴充規則（納入 CVSS/業務標籤），完善 **回滾腳本** 與 **serial** 分批。
3. **Production**

   * 導入 **EDA**（webhook/Kafka），完成審計報表（Jobs/Events 歸檔）與 ITSM 串接。

### 23) 技術能力與資源（**設計建議**）

* **平台**：Dify、AWX/Controller（或 AAP）、EDA（可選）。
* **開發**：MCP Server 調校、API Gateway、憑證/Token/RBAC、CMDB/標籤治理。
* **安全**：風險規則制定、變更管理、審計。

### 24) PoC 驗證項目（**檢核清單**）

* Dify 能列出並成功呼叫 MCP 工具（HTTP/STDIO 連線正常）。([docs.dify.ai][2])
* 成功 **POST** 啟動 Job Template、並以 **GET job_events/stdout** 回收執行證據。([docs.ansible.com][3])
* 規則依 **KEV/EPSS/CVSS** 產生差異化動作（拒絕／排程／canary／立即修補）。([cisa.gov][8])

---

## 八、參考架構圖（文字示意，**設計建議**）

```
Dify Agent (MCP client)
    └─(HTTP/SSE or STDIO, JSON-RPC via MCP)→  Ansible MCP Server
        └─(HTTPS REST)→  AWX/Controller → Targets
                           └─ Job/Events/Stdout → MCP Server → Dify Agent

Risk Data:
  NVD API v2 | CISA KEV | EPSS | CMDB(業務標籤)
```

---

## 九、實作片段

### A) 以 MCP 工具呼叫「啟動 Job Template」（**設計示例**）

```json
{
  "method": "tools/call",
  "params": {
    "name": "launch_job_template",
    "arguments": {
      "template_id": 42,
      "extra_vars": {"cve": "CVE-2025-12345", "mode": "canary"},
      "limit": "role:canary"
    }
  }
}
```

### B) 對應 AWX API（`launch`）

```bash
curl -s -X POST \
  -H "Authorization: Bearer $AWX_TOKEN" \
  -H "Content-Type: application/json" \
  "https://<controller>/api/v2/job_templates/42/launch/" \
  -d '{"extra_vars":{"cve":"CVE-2025-12345","mode":"canary"},"limit":"role:canary"}'
```

([docs.ansible.com][3])

### C) RHEL 僅安裝安全性更新（DNF）

```yaml
- hosts: rhel_like
  become: true
  tasks:
    - name: Security-only update
      ansible.builtin.dnf:
        name: "*"
        state: latest
        security: true
        update_cache: true
```

([docs.ansible.com][15])

### D) 失敗補償（block/rescue/always）

```yaml
- block:
    - name: 升級與設定變更
      import_tasks: tasks/upgrade.yml
  rescue:
    - name: 回滾到上一版本
      import_tasks: rollback/restore.yml
  always:
    - name: 收斂與回報
      import_tasks: tasks/report.yml
```

([docs.ansible.com][18])

---

## 十、結論與建議

* **結論**：以 **Dify（MCP）× Ansible（AWX/Controller/EDA）** 建構「AI Agent 自動化 CVE 升級派送」**技術可行**，且具可審計、可回滾、可差異化分環境釋出之條件。([docs.dify.ai][2])
* **建議**：

  1. 由 **PoC** 起步：採 **KEV/EPSS 觸發 + canary + 人工覆核**；
  2. Pilot 導入 **CVSS/業務權重**、**回滾腳本**、**serial 分批**；
  3. 正式期接入 **EDA** 與 **ITSM/審計報表**，落實組織級治理（NIST/OWASP/CIS 參照）。([cisa.gov][8])

---

## 參考資料（重點）

* **MCP**：Transports（STDIO/HTTP/SSE）、架構與原語、Server 開發注意。([modelcontextprotocol.io][1])
* **Dify × MCP**：使用 MCP Tools／將 App 發佈為 MCP Server／v1.6.0 雙向支援。([docs.dify.ai][2])
* **Ansible**：AWX/Controller API（launch job、job events、bulk job）、認證實務。([docs.ansible.com][3])
* **EDA**：事件源（webhook、Kafka…）與入門。([ansible.readthedocs.io][4])
* **風險資料源**：NVD API v2、CVSS v3.1/v4.0、CISA KEV、EPSS API。([nvd.nist.gov][5])
* **NIST/OWASP/CIS**：企業修補治理與漏洞管理最佳實踐。([nvlpubs.nist.gov][21])

---

> 需要我把上述內容拆解成 **PoC 待辦清單**、**Dify-MCP 連線參數表**、**AWX Job/Workflow 樣板** 與 **EDA 規則範例** 的專案檔（含目錄與 YAML/JSON 範例）嗎？我可以直接產出。

[1]: https://modelcontextprotocol.io/specification/2024-11-05/basic/transports?utm_source=chatgpt.com "Transports"
[2]: https://docs.dify.ai/en/guides/tools/mcp?utm_source=chatgpt.com "Using MCP Tools"
[3]: https://docs.ansible.com/ansible-tower/3.2.6/html/towerapi/launch_jobtemplate.html?utm_source=chatgpt.com "11. Launching a Job Template — Ansible Tower API Guide v3 ..."
[4]: https://ansible.readthedocs.io/projects/rulebook/en/v1.0.5/sources.html?utm_source=chatgpt.com "Event Source Plugins — Ansible Rulebook Documentation"
[5]: https://nvd.nist.gov/developers/vulnerabilities?utm_source=chatgpt.com "Vulnerability APIs - NVD"
[6]: https://modelcontextprotocol.io/specification/2025-06-18/basic/transports?utm_source=chatgpt.com "Transports"
[7]: https://www.first.org/cvss/v3-1/specification-document?utm_source=chatgpt.com "CVSS v3.1 Specification Document"
[8]: https://www.cisa.gov/known-exploited-vulnerabilities-catalog?utm_source=chatgpt.com "Known Exploited Vulnerabilities Catalog"
[9]: https://api.first.org/epss/?utm_source=chatgpt.com "GET /epss"
[10]: https://modelcontextprotocol.io/docs/develop/build-server?utm_source=chatgpt.com "Build an MCP server"
[11]: https://github.com/mancubus77/mcp-server-aap?utm_source=chatgpt.com "MCP Ansible Automation Platform Server"
[12]: https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/installing_on_openshift_container_platform/deploying-chatbot-operator?utm_source=chatgpt.com "Chapter 6. Deploying the Ansible Lightspeed intelligent ..."
[13]: https://modelcontextprotocol.io/docs/learn/architecture?utm_source=chatgpt.com "Architecture overview"
[14]: https://debugthis.dev/posts/2020/03/ansible-awx-using-python-to-launch-a-job-template/?utm_source=chatgpt.com "Ansible AWX – Using Python to launch a Job template"
[15]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html?utm_source=chatgpt.com "ansible.builtin.dnf module – Manages packages with the dnf ..."
[16]: https://github.com/jnv/ansible-role-unattended-upgrades?utm_source=chatgpt.com "jnv/ansible-role-unattended-upgrades"
[17]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_module.html?utm_source=chatgpt.com "ansible.builtin.package module – Generic OS package manager"
[18]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html?utm_source=chatgpt.com "Blocks — Ansible Community Documentation"
[19]: https://ansible.readthedocs.io/projects/awx/en/latest/rest_api/api_ref.html?utm_source=chatgpt.com "12. AWX API Reference Guide - Ansible documentation"
[20]: https://www.itpro.com/security/a-malicious-mcp-server-is-silently-stealing-user-emails?utm_source=chatgpt.com "A malicious MCP server is silently stealing user emails"
[21]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-40r4.pdf?utm_source=chatgpt.com "Guide to Enterprise Patch Management Planning"
