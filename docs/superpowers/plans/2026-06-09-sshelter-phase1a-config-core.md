# SSHelter — Phase 1a: Config CST Core 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development. Steps use checkbox (`- [ ]`) tracking. TDD throughout.

**Goal:** 一個經完整測試、能**無損 round-trip** 讀寫 `~/.ssh/config`（含多檔 `Include`）的 Rust 核心，提供 `config_*` Tauri 命令；未編輯的檔案 `parse→serialize` byte-identical，單一欄位編輯只改到該行。

**Architecture:** 自寫 **Concrete Syntax Tree (CST)**——每個實體行保留 `raw` 原文與結構化欄位（keyword 大小寫、分隔符、縮排、行內註解、enabled 旗標）。序列化時：未 `dirty` 的節點吐 `raw`、`dirty` 的依結構重繪。多檔 Include 各自一份 CST，編輯路由到 host 實體所在檔。`ssh2-config` 僅作唯讀 effective-config 預覽（不寫回）。

**Tech Stack:** Rust（手寫 line scanner，**不用 nom/pest**）、`shellexpand`（Include `~`/glob 展開）、`ssh2-config`（唯讀預覽）、`ts-rs`（DTO 型別橋接）、Tauri 2 commands。

> 設計來源：`docs/superpowers/specs/2026-06-08-ssh-config-manager-design.md` §7–§9。對應 spec Phase 1 的後端部分；UI 在 Phase 1b。

---

## 無損不變量（最高優先，所有 task 共同的硬性測試）

1. **Round-trip byte-identical**：對 golden 語料庫每個 fixture，`serialize(parse(text)) == text`（連同空行、縮排、`=` vs 空格、引號、大小寫、未知關鍵字、結尾換行與否）。
2. **最小變動**：對任一 host 編輯單一欄位後序列化，**只有該欄位那一行**（或被新增/刪除的行）改變；其餘 byte-identical。
3. **未知關鍵字永不丟棄**：無法辨識的指令以一般 `Directive` 直通保存。

每個改動程式碼的 task 都必須讓 `cargo test` 綠燈，且不得破壞既有 golden 測試。

---

## ssh_config 語意規則（parser 必守）

- 大小寫不敏感的關鍵字，比對用小寫；輸出未動行保留原始大小寫。
- **first-value-wins**（重排/重複指令的語意警示留待 UI；parser 只保留順序）。
- 無 line continuation：每個實體行就是一個指令。
- 引號內的 `#` **不是**註解；拆行尾註解要 quote-aware。
- `Key value` 與 `Key=value` 皆合法，逐行保留分隔符風格與其周邊空白。
- 縮排純裝飾但要保留；往區塊新增行時沿用該區塊縮排。
- `Include` glob 以 `~/.ssh` 為相對基準、lexical order 展開。

---

## File Structure（本階段建立）

| 檔案 | 職責 |
|---|---|
| `src-tauri/src/config/mod.rs` | 模組匯出 + 對外 API（`load_doc`、`save_host` 等的 re-export） |
| `src-tauri/src/config/model.rs` | CST 型別：`Separator`、`Directive`、`Item`、`HostBlock`、`MatchBlock`、`ConfigFile`、`SshConfigDoc` |
| `src-tauri/src/config/lexer.rs` | 單行分類與 directive 行解析（quote-aware、分隔符、縮排、行內註解、`raw`） |
| `src-tauri/src/config/parser.rs` | lines → `Vec<Item>` + 區塊分組（Host/Match/global）；`parse_file(text) -> Vec<Item>` |
| `src-tauri/src/config/serialize.rs` | `Vec<Item>`/`ConfigFile` → `String`（未 dirty 吐 raw、dirty 重繪、結尾換行保真） |
| `src-tauri/src/config/edit.rs` | 編輯操作：set_field、add/remove directive、toggle line/block、add/remove/reorder host、group/tags sentinel |
| `src-tauri/src/config/include.rs` | Include glob 解析、多檔 `SshConfigDoc` 載入、編輯路由到來源檔 |
| `src-tauri/src/config/dto.rs` | CST ↔ 前端 DTO（`HostSummary`、`HostDetail`{first-class 欄位 + `extra_options`}），含 ts-rs 匯出 |
| `src-tauri/src/config/commands.rs` | `config_*` `#[tauri::command]`，注入 `State<AppState>` |
| `src-tauri/src/state.rs` | `AppState`（持有已載入的 `SshConfigDoc` + 路徑），`Mutex` 包裝 |
| `src-tauri/tests/fixtures/*.sshconfig` | golden 語料庫 |
| `src-tauri/src/config/tests.rs` 或各檔 `#[cfg(test)]` | 單元 + golden 測試 |

`lib.rs` 加 `mod config; mod state;` 並註冊 `config_*` 命令。

---

## CST 型別契約（model.rs，實作須符合）

```rust
use std::path::PathBuf;
use serde::{Deserialize, Serialize};
use crate::fsutil::Fingerprint;

/// keyword 與 value 之間的分隔（含周邊空白，原樣保存）。
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Separator {
    Space(String),   // 例如 " ", "\t", "  "
    Equals(String),  // 例如 "=", " = ", "= "
}

/// 一行 key/value 指令（可被停用＝註解掉）。
#[derive(Debug, Clone)]
pub struct Directive {
    pub keyword: String,                  // 原始大小寫，如 "HostName"
    pub key: String,                      // 小寫，比對用，如 "hostname"
    pub value: String,                    // 值（不含行內註解；引號原樣）
    pub separator: Separator,
    pub indent: String,                   // 行首空白
    pub inline_comment: Option<String>,   // 行尾註解原文（含前導空白與 '#'）
    pub enabled: bool,                    // false ⇒ 輸出時整行以 '#' 註解
    pub raw: String,                      // 原始整行（不含換行）；未 dirty 時序列化直接吐這個
    pub dirty: bool,                      // 任一結構化欄位被改 ⇒ true ⇒ 序列化改用重繪
}

/// 檔案內依序排列的元素。
#[derive(Debug, Clone)]
pub enum Item {
    Blank(String),         // 原始空行（可能含空白）
    Comment(String),       // 整行註解原文（含縮排與 '#'）
    Directive(Directive),
    Host(HostBlock),
    Match(MatchBlock),
}

#[derive(Debug, Clone)]
pub struct HostBlock {
    pub header: Directive,     // `Host ...` 那一行
    pub patterns: Vec<String>, // 解析出的樣式（比對/UI 用）
    pub body: Vec<Item>,       // 到下一個 Host/Match 前的所有行
}

#[derive(Debug, Clone)]
pub struct MatchBlock {
    pub header: Directive,     // `Match ...`
    pub criteria: String,
    pub body: Vec<Item>,
}

#[derive(Debug, Clone)]
pub struct ConfigFile {
    pub path: PathBuf,
    pub items: Vec<Item>,           // 頂層依序：global 指令/註解/空行 + Host/Match 區塊
    pub trailing_newline: bool,     // 原檔是否以換行結尾（序列化保真用）
    pub fingerprint: Fingerprint,   // 載入當下的指紋（漂移偵測）
}

#[derive(Debug, Clone)]
pub struct SshConfigDoc {
    pub files: Vec<ConfigFile>,     // 主檔 + 解析到的 Include 目標
}
```

**序列化規則**：
- `ConfigFile` → 把每個 `Item` 的 render 以 `"\n"` 串接；若 `trailing_newline` 為真，結尾補 `"\n"`。
- `Item::Blank/Comment` → 吐其儲存的原文。
- `Item::Directive(d)` → 若 `!d.dirty` 吐 `d.raw`；否則重繪：`enabled ? "" : "#"` + `indent` + `keyword` + `separator` + `value` + `inline_comment.unwrap_or("")`。（停用行的具體註解風格：重繪時於 `indent` 後加 `# `；解析停用行時把 `# ` 後內容當 directive 解析、`enabled=false`、保留 raw。）
- `Host/Match` → 先 header（同 Directive 規則），再依序 body。

> 解析 by line：以 `"\n"` 切分並記住是否有結尾換行；每行原文（可能殘留 `\r`）存入對應 `raw`，保證 CRLF 也 byte-identical（重繪 dirty 行時才會把該行正規化為 LF——可接受的邊界）。

---

## Golden 語料庫（`src-tauri/tests/fixtures/`，至少涵蓋）

建立這些 fixture（內容要刻意包含難點），round-trip 測試逐一比對 byte-identical：

1. `simple.sshconfig`：兩個一般 Host、`HostName`/`User`/`Port`/`IdentityFile`。
2. `comments_blanks.sshconfig`：開頭註解、區塊間多個空行、行內註解（含**引號內的 `#`**，如 `ProxyCommand "sh -c '# not a comment'"`）。
3. `equals_indent.sshconfig`：混用 `Key=value`、`Key = value`、tab 與空白縮排、大小寫變體（`HostName`/`hostname`）。
4. `match_include.sshconfig`：`Match host ...` 區塊、`Include config.d/*`、`Host *` 結尾預設區塊、wildcard/negation 樣式（`Host prod-* !prod-old`）。
5. `unknown_dup.sshconfig`：未知/未來關鍵字、同一 Host 內重複關鍵字、`SetEnv`/`Ciphers` 等進階。
6. `disabled.sshconfig`：被 `#` 註解掉的指令與整個被註解的 Host 區塊（供 enabled 切換測試）。

---

## Tasks（subagent-driven；每個 task = implementer + 規格 review + 品質 review）

### Task A — `model.rs` + `lexer.rs`（CST 型別 + 單行解析，TDD）
- 依上面契約建 `model.rs`。
- `lexer.rs`：`classify(line: &str) -> RawKind`（Blank/Comment/Directive/HostHeader/MatchHeader，含「被註解掉的 directive」偵測）；`parse_directive(line) -> Directive`（quote-aware 拆行內註解、分隔符偵測、indent、keyword/key、value）。
- **驗收**：lexer 單元測試涵蓋 `=`/空格分隔、引號內 `#`、tab 縮排、大小寫、停用行；全綠。

### Task B — `parser.rs` + `serialize.rs`（解析 + 序列化 + golden round-trip，TDD）
- `parse_file(text) -> (Vec<Item>, trailing_newline)`：逐行分類、區塊分組（Host/Match 收攏其後行直到下一個 Host/Match）。
- `serialize_items(&[Item], trailing_newline) -> String` 依序列化規則。
- 建立全部 golden fixtures；**round-trip byte-identical property test** 對每個 fixture。
- **驗收**：所有 golden fixtures round-trip byte-identical；區塊分組單元測試綠。

### Task C — `edit.rs`（編輯操作，TDD）
- `set_host_field(doc, alias, key, value)`、`add_directive`/`remove_directive`、`set_line_enabled`/`set_host_enabled`（註解切換）、`add_host`/`remove_host`/`reorder_hosts`、`set_group`/`set_tags`（sentinel 註解 `#group:`/`#tags:`）。
- 編輯時設對應 `Directive.dirty=true`；新增行沿用區塊縮排。
- **驗收**：「編輯單一欄位後，只有該行改變、其餘 byte-identical」property test；各操作單元測試綠。

### Task D — `include.rs` + `dto.rs`（多檔 Include + DTO，TDD）
- `load_doc(main_path) -> SshConfigDoc`：解析主檔、依 `Include` glob（`shellexpand` + lexical order，相對 `~/.ssh`）載入目標檔各成 `ConfigFile`。
- 編輯路由：給定 alias 找到其實體所在 `ConfigFile`。
- `dto.rs`：`HostSummary{alias, source_file, group, tags}`、`HostDetail{ first-class 欄位（hostname/user/port/identity_file…）+ extra_options: Vec<(String,String)> }`；CST→DTO 與 DTO→edit 映射；ts-rs 匯出到 `src/bindings/`。
- **驗收**：多檔 fixture 載入、來源檔標示、跨檔編輯寫對檔的測試綠；DTO round-trips 已知欄位 + 未知進 extra_options。

### Task E — `state.rs` + `config/commands.rs` + wire（Tauri 命令，TDD on 底層 fn）
- `AppState { doc: Mutex<Option<SshConfigDoc>>, ... }`，`tauri::Builder.manage(AppState::default())`。
- `config_*` 命令（薄包裝呼叫 config 模組）：`config_load`、`config_list_files`、`config_get_host`、`config_save_host`、`config_add_host`、`config_remove_host`、`config_reorder_hosts`、`config_set_line_enabled`、`config_set_host_enabled`、`config_set_group`、`config_set_tags`、`config_check_drift`、`config_effective`（`ssh2-config` 唯讀）。
- 存檔流程：`fsutil::backup`（session 首次）→ `fsutil::atomic_write`(0o600) 寫對應來源檔 → 更新 fingerprint。
- 在 `lib.rs` 註冊命令；DTO 型別 ts-rs 匯出。
- **驗收**：底層 fn 單元測試綠；`cargo build` 含命令註冊通過；DTO `.ts` 產生。

---

## Self-Review（撰寫後）
- Spec §7–§9 覆蓋：CST 無損（A/B）、編輯最小變動（C）、多檔 Include（D）、`config_*` 命令（E）、effective 預覽（E）、group/tags sentinel（C）、drift（E）✅
- 無 placeholder：型別契約、不變量、golden 語料、驗收皆具體。
- 型別一致性：`Directive`/`Item`/`HostBlock`/`ConfigFile`/`SshConfigDoc` 跨 task 一致；`AppError`（含 Phase 1 將新增的 `Parse` 變體）為回傳型別。

## 與 Phase 0 的銜接（readiness）
- `AppError` 需新增結構化 `Parse { msg, line }` 變體（Task A/B 用），而非塞進 `Other`。
- `tauriInvoke` 的 `cmd` 後續收斂為產生的 command union（隨命令面變大）。
- 每個 `#[derive(TS)]` DTO 都配一個 `#[ts(export)]` 測試，`cargo test` 為單一新鮮度閘門。
