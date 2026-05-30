---
title: 系统设计
weight: 10
bookToc: true
---

# 系统设计

AI-oral-exam 的系统设计围绕“实时口试”展开：用户层提供登录注册、角色区分、学生考试操作和教师监控入口；应用层承载智能判题打分与实时会话；基础层提供工具、智能体和监控能力。整体目标是让一次口试能够被创建、执行、记录、评分和复核。

## 总体架构

系统可拆分为三层：

- 用户层：提供多角色分类、登录/注册、学生考试等操作和教师监控入口。
- 应用层：由智能判题打分系统和会话系统组成，前者负责考试状态、出题、面试与评分，后者负责全双工对话管线、STT 和 TTS。
- 基础层：提供 Tool、Agent、Monitor 等底层能力，为上层流程提供工具调用、智能体执行和运行监控支撑。

{{< mermaid >}}
flowchart TB
  subgraph UserLayer["用户层"]
    Role["多角色分类"]
    Auth["登录/注册"]
    StudentOps["学生考试等操作"]
    TeacherMonitor["教师监控"]
  end

  subgraph AppLayer["应用层"]
    direction LR
    subgraph Scoring["智能判题打分系统"]
      Judger["Judger"]
      CandidateStatus["Candidate Status"]
      ExamSetter["examSetter"]
      Interviewer["Interviewer"]
    end

    subgraph Session["会话系统"]
      PipelineA["全双工对话可插拔 pipeline"]
      PipelineB["全双工对话可插拔 pipeline"]
      More["..."]
    end

    STT["STT"]
    TTS["TTS"]
  end

  subgraph BaseLayer["基础层"]
    Tool["Tool"]
    Agent["Agent"]
    Monitor["Monitor"]
  end

  UserLayer -->|"配置"| Scoring
  UserLayer --> Session
  Scoring --> Tool
  Scoring --> Agent
  Session --> STT
  Session --> TTS
  STT --> Tool
  TTS --> Tool
  Tool --> Agent
  Agent --> Monitor
{{< /mermaid >}}

## 实时语音链路

项目计划采用可插拔的 Pipecat 构建面试功能模块，并通过 WebRTC 建立实时连接。语音链路包含三个关键步骤：

1. STT/ASR 将学生语音转换为文本。
2. LLM 根据考试上下文、学生材料和历史回答生成提问或追问。
3. TTS 将 AI 输出播报给学生，形成连续口试体验。

这种链路适合后续替换不同模型或服务商，也方便在性能、成本和可用性之间做工程权衡。

## 多智能体分工

系统不是让单个模型同时承担所有职责，而是把口试拆成多个角色：

- Interviewer：负责主持口试，提出主问题和追问，控制对话节奏。
- Judge：负责根据评分规则评估回答质量，并给出分数与理由。
- FileParser：负责解析学生上传的文档、图片、报告或代码材料。
- Coder/Tool：负责辅助代码分析、运行验证和静态检查。

这种分工可以让每个智能体的提示词、工具权限和输出结构更清晰，也方便后续引入更多评委角色。

## 材料与规则

口试质量很大程度取决于上下文。系统需要同时处理两类输入：

- 考试规则：由教师提供，定义考试边界、评分维度、追问策略和通过标准。
- 学生材料：由学生上传，包括作业、代码、报告、图片等，用于生成个性化问题。

系统应尽量把材料解析结果结构化，避免智能体只拿到长文本后盲目追问。对于代码类作业，可结合代码分析工具和运行工具进行验证。

## 管理与监控

为了支持多人考试，系统还需要提供身份认证、角色权限、会话状态监控和资源管理。Client 或管理端负责查看考试进度、会话状态、系统资源消耗和最终评分结果，使教师可以从“逐个面试”转向“批量管理与重点复核”。
