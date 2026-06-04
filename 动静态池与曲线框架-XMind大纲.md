# 动静态池与曲线 · XMind 大纲（优化版）

> 导入：`动静态池与曲线框架.opml`  
> 布局建议：**逻辑图向右**；一级分支按序号着色（见文末）

---

## 纯文本大纲（Tab 缩进，复制到 XMind）

```
动静态池与曲线
	① 链路分层（公共）
		L0 离线数仓
			app_ml_qd_static_pool_inc_m
			app_ml_qd_dynamic_pool_inc_m
		L1 QD对接
			abs_to_qd_static_pool_data
			abs_to_qd_dynamic_pool_data
		L2 池模型
			dbo_Entities / dbo_EntityAttributes
			Static：RawData → ProcessedData
			Dynamic：RawData（Processed 表存在，主链路未写）
		L3 曲线
			dbo_Curves
			Analysis_CurveGroup
			Analysis_CurveGroupCurves
		业务线区分
			池名：{creditor}_微贷 | {creditor}_月付
			asset_type：1微贷 | 2月付
			准入池名单：analysis_eligible_creditor_code
	② 静态池链路
		公共路径
			离线 → abs_to_qd_static_pool_data
			→ dbo_StaticPoolRawData
			→ dbo_StaticPoolProcessedData
			→ dbo_Curves → CurveGroup
		微贷分叉
			标识：category=微贷 | MicroLoan | 池名_微贷
			脚本：LoadStaticPoolFromDataInterfaceWithCutOff.py
			规则：分母 PerformingLoanBalance（可配）
			分组：小额/中额 × 生意贷/生活费 × A/B/C/D × 期限
			group_id：4段
		月付分叉
			标识：category=月付 | Maiton | 池名_月付
			脚本：LoadStaticPoolFromDataInterfaceWithCutOff_YF.py
			规则：LoanIssueStartDate=2023-09-01
			规则：2_2_*→PerformingLoanBalance，其余→LoanAmount
			分组：未分期/分期 × 消费/循环/分期 × [A1,A3]…NULL × 期限
			group_id：4段
	③ 动态池链路
		公共路径
			离线 → abs_to_qd_dynamic_pool_data
			→ dbo_DynamicPoolRawData
			脚本共用：LoadDynamicPoolFromDataInterface.py
			不产曲线
		微贷分叉
			过滤：池名含_微贷 | asset_type=1
			group_id：5段（末尾补位 1）
		月付分叉
			上游：静态池聚合产出
			过滤：池名含_月付 | asset_type=2
	④ 任务调度
		前置依赖
			abs_to_qd_static_pool_data ✓
			abs_to_qd_dynamic_pool_data ✓
		调度入口
			agent_static_pool_task
			monthly_static_dynamic_pool.py
		微贷一轮
			POOL_LIST[微贷] → WithCutOff 逐池 → 动态全量加载
		月付一轮
			POOL_LIST[月付] → WithCutOff_YF 逐池 → 动态全量加载
		频率：monthly 全量 | 单池失败跳过
	⑤ 应用层次
		查看层
			收益分析 → 静态池
			收益分析 → 动态池
		生产层
			自动：静态池跑批（微贷/月付分脚本）
			手工：静态池合并
			补充：MOB回收率 v8
		管理层
			曲线组
			曲线
		消费层
			压力测试 → CurveGroup
			测算任务 → 消费曲线
	附录｜微贷 vs 月付（差异对照）
		静态脚本
			微贷 WithCutOff.py
			月付 WithCutOff_YF.py
		静态独有参数
			月付 LoanIssueStartDate=2023-09-01
		违约率分母
			微贷 PerformingLoanBalance
			月付 按 group_id 分支
		动态 group_id
			微贷 5段
			月付 同脚本，过滤 _月付
		跑批
			各跑一轮 agent_static_pool_task
```

---

## 一级分支配色

| 分支 | 色 | 内容 |
|------|----|------|
| ① 链路分层 | 灰 | 公共表结构 |
| ② 静态池 | 蓝 | 公共路径 + 微贷/月付分叉 |
| ③ 动态池 | 青 | 公共路径 + 微贷/月付分叉 |
| ④ 任务调度 | 黄 | 依赖与两轮跑批 |
| ⑤ 应用层次 | 紫 | 查看→生产→管理→消费 |
| 附录 | 橙 | 差异速查 |

---

## XMind 操作提示

1. 「微贷分叉 / 月付分叉」可设为**子主题样式**（虚线框），与「公共路径」区分  
2. 「附录」可折叠，日常讲解只展开 ①~⑤  
3. 节点上的 `→` 链可改为 XMind **联系线**（从静态 Processed 连到 Curves）
