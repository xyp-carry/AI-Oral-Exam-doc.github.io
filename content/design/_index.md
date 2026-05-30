---
title: 系统设计
weight: 10
bookToc: true
---

# 系统设计

AI-oral-exam 的系统设计围绕“实时口试”展开：用户层提供登录注册、角色区分、学生考试操作和教师监控入口；应用层承载口试管理系统与实时会话系统；基础层提供工具、智能体和监控能力。整体目标是让一次口试能够被创建、执行、记录、评分和复核。

## 总体架构

系统可拆分为三层：

- 用户层：提供多角色分类、登录/注册、学生考试等操作和教师监控入口。
- 应用层：由口试管理系统和会话系统组成。口试管理系统负责出题、候选人状态维护、面试推进和判卷评分；会话系统负责全双工对话管线、STT 和 TTS。
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
    subgraph OralManagement["口试管理系统"]
      ExamSetterAgent["ExamSetterAgent 出题者"]
      JudgerAgent["JudgerAgent 判卷者"]
      CandidateStatus["Candidate Status"]
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

  UserLayer -->|"配置"| OralManagement
  UserLayer --> Session
  OralManagement --> Agent
  OralManagement --> Tool
  Session --> STT
  Session --> TTS
  STT --> Tool
  TTS --> Tool
  Tool --> Agent
  Agent --> Monitor
{{< /mermaid >}}

## 子页面

- [实时语音链路 Pipecat](pipecat/)：说明会话系统如何通过可插拔 pipeline 组织 STT、LLM、TTS 与 WebRTC 实时交互。
- [多智能体分工](agents/)：说明 ExamSetterAgent、JudgerAgent、Interviewer 等智能体在口试中的职责边界。
- [口试管理系统](oral-management/)：说明口试管理系统中的业务角色、候选人状态、工具能力和运行监控。
