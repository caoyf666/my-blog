---
title: Jev：把 Agent 的“判断题”从大模型里拆出来
top: false
cover: false
toc: true
mathjax: true
abbrlink: 3646782985
date: 2026-10-01 13:36:04
updated: 2026-10-01 17:20:00
password:
summary: Jev 给我的启发不是“再造一个大模型”，而是把 Agent 里高频、封闭、可回退的判断题拆成单独的工程层：生成负责表达，判断负责分流，代码负责执行。
tags:
  - AI
  - Agent
  - Jev
categories:
  - 学习笔记
---

我现在越来越觉得，Agent 工程里最容易被低估的不是“生成能力”，而是“判断能力”。

过去我们很习惯把所有事情都丢给一个通用大模型：让它理解需求、写方案、选工具、判断风险、输出 JSON，甚至再解释一遍为什么这么做。这种方式在 Demo 里很顺手，但一旦放进真实业务，就会遇到几个很现实的问题：调用慢、成本高、格式偶尔失控、每一步的置信度也很难直接写进业务代码。

Jev 这类模型给我的启发是：很多 Agent 调用其实并不是为了生成一段内容，而是为了回答一个很窄的判断题。

比如：

| 场景 | 真正需要的问题 | 输出应该是什么 |
| --- | --- | --- |
| 客服工单 | 这条消息该给哪个部门？ | `billing / technical / sales / other` |
| 工具路由 | 下一步该调用哪个工具？ | `search / read_file / run_test / ask_user` |
| 上下文压缩 | 这段工具结果还要不要保留？ | `true / false` |
| 风险控制 | 这个动作危险吗？ | `0 ~ 1` 或风险等级 |
| 内容质检 | 这段回答有没有偏题？ | `pass / review / reject` |

这些问题的共同点是：需要语义理解，但输出空间很小。它们不像写文章，也不像复杂推理，更像是“智能 if”。

Jev 的 [官方介绍](https://docs.typesafe.ai/introduction) 把它称为面向决策的 System One 模型。我的理解更工程化一点：它不是 Agent 的大脑，而是 Agent 的判断层。

{% asset_img jev-agent-architecture.svg Jev 在 Agent 系统中的位置 %}

## 先把问题拆清楚

我不想把 Jev 理解成“更便宜的大模型”。这个说法太粗了，也容易把系统做歪。

在我看来，Agent 至少应该拆成四层：

| 层次 | 职责 | 更适合谁来做 |
| --- | --- | --- |
| 生成层 | 规划、解释、写代码、写文本 | GPT / Claude / Gemini 等生成模型 |
| 判断层 | 分类、评分、路由、门禁、是否保留 | Jev / Laya / 自训练判别模型 |
| 执行层 | 权限、状态机、数据库写入、回滚 | 确定性代码 |
| 兜底层 | 高风险、低置信度、样本外问题 | 人工或更强模型 |

这套分工的核心不是“谁更聪明”，而是“谁更适合承担这个责任”。

生成模型适合处理开放式问题，但它不应该承担所有分支判断。确定性代码适合处理规则明确的问题，但很多语义判断又写不成普通 if。Jev-like 判断层正好站在中间：规则写不动，通用大模型又太重。

## 一个最小调用长什么样

如果我把它放到客服工单分流里，最小接口大概会长这样。这里重点不是某个 SDK 的语法，而是这种输入输出方式：状态是结构化的，问题是提前定义的，输出能直接进代码分支。

```js
const state = {
  from: "user@example.com",
  subject: "Duplicate charge on invoice #4411",
  body: "We were billed twice. Please refund the duplicate today.",
  plan: "pro",
};

const questions = {
  department: {
    type: "choice",
    instructions: "Which team should handle this ticket?",
    criteria: {
      billing: "Invoices, payments, refunds, duplicate charges",
      technical: "Bugs, outages, integration failures",
      sales: "Pricing, upgrade, contract discussion",
      other: "Everything else",
    },
  },
  isUrgent: {
    type: "noul",
    instructions: "Does this ticket require same-day handling?",
  },
};

const result = await client.systemOne({
  model: "jev-latest",
  state,
  questions,
});

const department = result.answers.department.choice;
const confidence = result.answers.department.confidence;
```

这段代码背后的思想很重要：我不是让模型“解释一下这张工单应该怎么办”，而是让它在固定候选里做判断。解释可以有，但不能成为系统继续运行的唯一依据。

## 我真正想要的是概率门禁

判断层有价值，不只是因为它快，还因为它能把“不确定”变成系统设计的一部分。

我会把返回结果分成三档：

```js
function routeByConfidence(answer, riskLevel) {
  const confidence = answer.confidence ?? answer.probability ?? 0;

  if (riskLevel === "high") {
    return confidence >= 0.95 ? "strong_model_review" : "human_review";
  }

  if (confidence >= 0.9) return "auto_execute";
  if (confidence >= 0.65) return "strong_model_review";
  return "human_review";
}
```

这段逻辑体现了我对 Jev-like 模型的基本态度：概率不是为了炫技，而是为了分流。

低风险、高置信度的动作可以自动化；中间地带交给强模型复核；高风险或低置信度直接保守处理。这样系统不会因为模型“看起来很自信”就把不可逆动作交出去。

{% asset_img jev-routing-matrix.svg Jev-like 判断层的风险分流矩阵 %}

## 本地部署为什么值得关注

如果只是玩 Demo，云端 API 最省事。但如果要接企业内部系统，本地部署会变得很有吸引力。

原因很简单：

| 维度 | 云端 API | 本地部署 |
| --- | --- | --- |
| 上手速度 | 快 | 需要环境配置 |
| 数据隐私 | 取决于供应商 | 更可控 |
| 延迟 | 受网络影响 | 局域网/本机更稳定 |
| 成本 | 按量计费 | 前期部署成本更高，稳定流量更可控 |
| 定制 | 受接口限制 | 更方便接入自己的数据和评估 |

像 Laya 这类开源 Jev-like 项目，给了我一个更贴近工程实践的方向：先把本地最小链路跑通，再拿自己的中文业务样本测试，而不是直接相信通用权重。

一个最小的本地实验大概可以这样写：

```python
from laya import Router

router = Router(
    device="cuda",
    preload=True,
    max_loaded=1,
)

state = {
    "title": "登录后支付页空白",
    "body": "用户反馈 Safari 可以登录，但进入支付页后一直空白，Chrome 正常。",
    "plan": "business",
}

questions = {
    "department": {
        "type": "choice",
        "instructions": "这条工单应该分配给哪个团队？",
        "criteria": {
            "billing": "支付、发票、扣款、退款",
            "frontend": "浏览器兼容、页面渲染、交互异常",
            "backend": "接口错误、服务不可用、数据库异常",
            "other": "无法判断或不属于以上类别",
        },
    }
}

result = router.predict(state, questions)
print(result["answers"]["department"])
```

这类代码看起来很短，但它背后需要补的工程工作不少：样本怎么来、标签怎么定、错分怎么统计、置信度阈值怎么选、线上怎么回退。这些才是判断层能不能落地的关键。

## 自己训练时，我会先关心数据而不是模型

如果通用权重直接接业务不准，我不会第一时间怪模型。我会先检查三个问题：

1. 我的候选标签是不是互斥？
2. 每个标签的判断标准是不是足够具体？
3. 训练集、验证集、校准集有没有数据泄漏？

一个训练配置可以先保持朴素，不必一开始就追求复杂：

```json
{
  "max_steps": 600,
  "microbatch_questions": 4,
  "gradient_accumulation": 4,
  "backbone_lr": 0.00001,
  "head_lr": 0.0001,
  "warmup_steps": 20,
  "loss": "cross_entropy + brier_calibration"
}
```

我更关心的是切分方式。比如同一个工单案例可能衍生出多个问题，如果把这些问题随机打散，训练集和验证集就可能共享同一个状态，最终评估会虚高。

所以我会按案例 ID 做确定性切分：

```js
import crypto from "node:crypto";

function splitByCaseId(cases) {
  return cases
    .map((item) => {
      const hash = crypto
        .createHash("sha256")
        .update(`jev-blog:${item.caseId}`)
        .digest("hex");

      return { ...item, hash };
    })
    .sort((a, b) => a.hash.localeCompare(b.hash))
    .map((item, index, arr) => {
      const ratio = index / arr.length;
      if (ratio < 0.7) return { ...item, split: "train" };
      if (ratio < 0.85) return { ...item, split: "dev" };
      return { ...item, split: "calibration" };
    });
}
```

这个细节很不起眼，但它决定了评估是不是可信。模型训练本身可以套框架，业务评估不能糊弄。

## 我会怎么把它接进 Agent

我认为最适合做第一个实验的，不是“让模型控制整个 Agent”，而是上下文压缩。

Agent 在修 bug 或跑任务时，会不断产生工具调用结果：搜索结果、文件内容、命令输出、测试日志。真正撑爆上下文的往往不是用户需求，而是这些中间结果。

传统摘要的问题是：它会改写事实。改写一旦丢掉关键路径、错误码、行号，后面的推理就可能偏掉。

我的做法会更保守：不让判断层总结内容，只让它判断是否保留。

```js
const questions = {
  keepFullResult: {
    type: "noul",
    instructions:
      "This tool result contains details that may still be needed to complete the user's task.",
  },
  keepShortNote: {
    type: "noul",
    instructions:
      "A short note is enough; the full raw result can be removed from context.",
  },
};

function compactToolResult(call, answer) {
  if (answer.keepFullResult.noul >= 0.8) {
    return call.fullResult;
  }

  if (answer.keepShortNote.noul >= 0.7) {
    return `${call.tool}: ${call.status}, ${call.resultChars} chars`;
  }

  return null;
}
```

这个设计里，Jev-like 模型不负责写代码，也不负责全局规划。它只负责一个很窄的问题：这段历史还值不值得继续占上下文。

{% asset_img jev-business-loop.svg 判断型模型的业务闭环 %}

## 从业务角度看，它值钱在哪里

Jev-like 判断层的商业价值，我觉得不在“便宜”两个字上，而在“把语义判断标准化”。

很多公司内部都有大量半结构化流程：

| 业务线 | 高频判断 | 自动化收益 |
| --- | --- | --- |
| 客服 | 分流、紧急度、是否转人工 | 降低人工初筛成本 |
| 风控 | 异常、欺诈、敏感动作 | 把低风险流量快速放行 |
| 研发工具 | 工具路由、日志保留、代码审查初筛 | 提升 Agent 成功率 |
| 内容平台 | 违规、重复、低质、偏题 | 减少审核压力 |
| 销售运营 | 线索质量、跟进优先级 | 提高人效 |

这些场景的共同特点是：规则写不全，人工做太慢，大模型做太重。

如果判断层能做到低延迟、概率可校准、接口稳定，它就不是一个“模型功能”，而会变成业务系统里的基础设施。就像搜索、推荐、风控评分一样，最后它会沉到流程里，成为每次动作前的一道语义门禁。

## 我的保留意见

我对这个方向乐观，但不会无脑乐观。

第一，校准不等于聪明。模型能诚实表达不确定，不代表它已经理解业务。

第二，中文场景要单独验证。中文客服语气、行业术语、隐含威胁和委婉表达，都可能让通用权重失真。

第三，高风险动作不能只靠模型。扣款、删除、权限变更、合同承诺这类动作，必须有确定性规则和人工兜底。

第四，接入成本也是真成本。多一个模型，就多一套监控、日志、回退、评估和数据闭环。如果业务量不大，简单规则可能更划算。

所以我会用一句话约束自己：只把 Jev-like 判断层用在高频、封闭、可回退、可评估的判断题上。

## 结论

Jev 给我的启发不是“又来了一个新模型”，而是一个更朴素的工程问题：

> Agent 里哪些地方真的需要生成？哪些地方其实只是判断？

如果把所有判断都塞给大模型，系统会越来越慢、越来越贵，也越来越难解释。如果把判断层拆出来，生成模型、判断模型和确定性代码就能各做各擅长的事。

我现在更认可这样一种 Agent 架构：

生成负责表达，判断负责分流，代码负责执行，人工负责最终责任。

这个分工一旦成立，Jev-like 模型的意义就不只是“快一点、便宜一点”，而是让 Agent 从“一个模型想办法包办所有事”，变成真正可工程化、可监控、可回退的系统。
