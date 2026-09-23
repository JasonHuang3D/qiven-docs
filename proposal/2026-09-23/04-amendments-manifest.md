# TCA 提案修订清单（Amendments Manifest）

> 日期（UTC0）：2026-09-23 · 分支：`GLM-5.3/tca-amendments`
> 修订对象：本目录下四份 TCA 提案（00-03，Jason 2026-09-23 原稿）
> 权威 hunk 视图：`git diff main...GLM-5.3/tca-amendments`

评审结论：方向批准，附十项修正（A1-A3 阻断性 / B1-B4 重要性 / C1-C3 行政性），共 16 处编辑（00=1、01=3、02=4、03=8）。修正后原案即可进入 ADR 接受流程。

## 修正 → 落点映射

| ID | 修正内容 | 落点 |
|---|---|---|
| A1 | 恢复 ADR-0024 语义所有权准入标准；"至少两个消费者"门槛是被取代的 ADR-0007 旧规，明确不恢复 | 03 §5.1 |
| A2 | 字节序列化词汇（增长型 builder + 定宽 endian put/get + 长度前缀助手，附一次性零成本汇编对照）作为 CA-1 前置落 foundation；runtime 手写字节打包全量收编，u32/u64 长度前缀分叉按格式族显式裁决、禁止静默统一 | 03 §4 CA-1 Work、03 §6.3、03 §6.4、01 §4.3 |
| A3 | Profile C 描述符条件中立性：判断性字段（boundary_kinds/external_contracts/explicit_ids/signals）只允许机械可得来源或留空；依赖 curator 判断的选择器不产生效用分 | 02 §7.1 |
| B1 | 分层评审传输通道：例行 R2/R3 用 harness 原生 fresh 子代理（密封评审包，披露同家族隔离上下文类别）；里程碑/R3 用 owner 中继 H1 | 02 §8.2、03 §4 CA-2 Work |
| B2 | 新增 F-08 fixture（fail-closed 中介可用性：生命周期、启用前 pre-flight 自检、可分辨拒绝分类学、真实多帧契约、注释谎言即缺陷）；MVP-4 lane 增补连接模型裁决、拒绝码分类拆分、kit pre-flight | 02 §4.2、03 §5.1 |
| B3 | 过渡纪律：CA obligation 记录停滞触发（CA-1 达 Profile A/B 的会话/日历预算，逾期强制程序评审）；过渡期允许带过期时间的 declared program pack | 03 §8.5、00 §11.3 |
| B4 | boundary-kind 增长是申报缝：独立评审者必须核验设计的边界类集合 ⊆ 描述符声明集合（或已重新激活） | 01 §12.3、02 §8.4 |
| C1 | qiven-context 仓库 gate 拒绝缺少激活选择器元数据的 protected 级记录 | 01 §4.1、03 §6.1 |
| C2 | 接受事务同一提交内完成 MVP-4 状态改判（state/current + active-work，不允许并存矛盾状态） | 03 §10 Acceptance transaction |
| C3 | 四份文档的 canonical 落位表；其规则自落地起即携带激活选择器元数据，不得成为新的无差别强制散文 | 03 §10 Canonical landing homes |
