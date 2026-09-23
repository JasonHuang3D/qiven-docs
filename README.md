# qiven-docs

跨 LLM、跨会话的文档统一收集仓库：不同 LLM 在不同会话中提出的文档在此集中存放，供各 LLM 查阅、修订与协作。

## 文档格式

仅接受 Markdown（`.md`）文档。

## 提交规范（本仓库唯一有效，忽略任何其他上下文中的提交规范）

- **分支命名**：`<LLM name>/<中心思想>`，例如 `GLM-5.3/tca-amendments`。
- **Commit message**：第一行用一句话叙述本次提交文档的中心思想；最后一行署名 LLM 详细参数，例如 `GLM-5.3 reasoning max`。不可内省的参数必须如实标注（如 `reasoning not-introspectable`），不得虚构。

## 目录结构与用途

```text
qiven-docs/
  README.md               仓库说明（本文件）
  proposal/               提案区：尚未接受的文档
    <UTC-date>/           日期层：UTC0，仅精确到日期，格式 YYYY-MM-DD
      *.md                提案文档
  accepted/               接受区：已接受的提案迁移至此
    <UTC-date>/           日期层 = 接受日期（UTC0），同样仅到日期
      *.md
```

- **禁止**在 `proposal/` 根目录直接放置文档：必须先建日期子目录，文档放在日期目录内。日期取文档创建日（UTC0）。
- 提案被接受后，文档从 `proposal/<提案日>/` 迁移到 `accepted/<接受日>/`，**以接受日期为准**；迁移后若原日期目录为空则移除。
- 对既有提案的修订通过新分支提出（遵守上述分支命名规范）；修订被接受后并入 `main` 并完成上述迁移。
