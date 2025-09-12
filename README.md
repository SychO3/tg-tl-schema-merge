# tg-tl-schema-merge

[![Workflow Status](https://github.com/SychO3/tg-tl-schema-merge/actions/workflows/tl-merge.yml/badge.svg)](https://github.com/SychO3/tg-tl-schema-merge/actions/workflows/tl-merge.yml)

> 自动从 **tdesktop** 与 **tdlib** 拉取 Telegram TL schema，进行**语义合并**，并在**内容变更**时：
> - 生成 `merged.tl`（仓库根目录与 `schemas/` 中各一份）
> - 生成 `merged.tl.sha256`（根目录）
> - 追加 `schemas/CHANGELOG.txt`（含统一 diff 预览）
> - 发布 GitHub **Release**（附带 `merged.tl` 与其 SHA256）

- 🔁 **频率**：默认 **每小时一次**（UTC，GitHub Actions `cron`）
- 🗂 **来源**：
  - `telegramdesktop/tdesktop` → `Telegram/SourceFiles/mtproto/scheme/api.tl`（`dev` 分支）
  - `tdlib/td` → `td/generate/scheme/telegram_api.tl`（`master` 分支）

---

## 特性

- **语义合并**（不是简单按行去重）
  - 解析 `---types---` / `---functions---`
  - 将定义解析为 `name[#id] params = result;`
  - **去重键**优先使用 `#id`；没有 `#id` 时使用 `name + 规范化参数 + 结果 + 所属段落`
  - 冲突策略：**以提交时间较新**的来源为准
  - 输出使用**规范分隔**：
    ```
    ---types---
    ... all types ...
    ---functions---
    ... all functions ...
    ```
- **头部注释与后处理**
  - 顶部加入本次合并元信息（来源 repo、path、commit、时间、大小等）
  - 在最终 `merged.tl` 中，用正则 `^\s*-+\s*types\s*-+\s*$`（不区分大小写）定位 `types` 分隔行，**将其上方所有内容注释化**（已有 `//` 的不重复加）
- **只在变更时提交与发版**：降低无意义提交与 Release 噪音
- **可复现**：下载使用 **以 commit SHA 固定**的原始文件，避免分支指针漂移

---

## 目录结构

```
.
├─ merged.tl                 # 根目录合并结果（便于直接引用）
├─ merged.tl.sha256          # 根目录 SHA256
├─ schemas/
│  ├─ merged.tl              # 同步保留一份
│  ├─ latest.tl              # 单源里“提交时间更新”的那个
│  ├─ CHANGELOG.txt          # 仅在变更时追加
│  ├─ metadata.json          # 本次运行元数据（来源、SHA、提交时间等）
│  ├─ td-<sha7>.tl           # tdlib 快照
│  └─ tdesktop-<sha7>.tl     # tdesktop 快照
└─ .github/workflows/tl-merge.yml
```

---

## GitHub Actions

- 工作流：`.github/workflows/tl-merge.yml`  
- 触发：
  - **定时**：每小时整点（`cron: "0 * * * *"`，UTC）
  - **手动**：在 Actions 页面用 “Run workflow”

> 只有当生成的 `merged.tl` 与上一次不同，才会：
> 1) `git commit` & `push`  
> 2) 计算 `merged.tl` 的 `SHA256`  
> 3) 以 `tl-YYYYMMDDHHMMSS-<sha1>-<sha2>` 创建 **Release**（上传 `merged.tl` 与 `merged.tl.sha256`）

---

## 本地运行

```bash
# 可选：创建虚拟环境
python -m venv .venv
source .venv/bin/activate

# 可选：设置令牌以提升 GitHub API 限额
export GITHUB_TOKEN=ghp_xxx

# 运行（默认输出到 ./schemas）
python fetch_and_merge_tl.py

# 或指定目录
python fetch_and_merge_tl.py --outdir ./schemas
```

---

## 合并细节

- **签名生成**：
  - 有 `#id`：`<name>#<id_hex>|<section>`
  - 无 `#id`：`<name>|<normalized_params>|<result>|<section>`
- **冲突处理**：若同一签名在两源的 **raw** 不同，保留**较新来源**的版本
- **注释化规则**：匹配 `^\s*-+\s*types\s*-+\s*$`（大小写不敏感），将其**上方**所有行加前缀 `// `（已有 `//` 的跳过）

> 如需把分隔标记行的“行尾注释”也纳入匹配（例如 `---types--- // note`），可把正则改为：  
> `^\s*-+\s*types\s*-+\s*(?://.*)?$`

---

## 常见问题（FAQ）

**Q1：为什么没发布新的 Release？**  
A：上游 TL **没变** 或本次合并结果与上次**完全一致**。查看 `schemas/metadata.json` 的 `entries[].sha/commit_date` 即可确认。

**Q2：如何调整频率？**  
A：在 `tl-merge.yml` 里改 `schedule.cron`，例如每 15 分钟：`*/15 * * * *`（UTC）。

**Q3：如何添加新的 TL 来源？**  
A：编辑 `fetch_and_merge_tl.py` 中的 `SOURCES` 列表，按 `{owner, repo, path, ref}` 增加即可。

---

## 注意与声明

- 本仓库仅聚合公开的 TL schema，不含任何私有或二次分发限制内容；来源版权归各自项目所有。  
- 请遵循上游项目（tdesktop / tdlib）的许可条款与贡献指南。

**致谢**  
- [telegramdesktop/tdesktop](https://github.com/telegramdesktop/tdesktop)  
- [tdlib/td](https://github.com/tdlib/td)

---

## License

本项目采用 MIT License。详见 [LICENSE](./LICENSE)。
