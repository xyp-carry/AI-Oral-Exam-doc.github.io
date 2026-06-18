---
title: Agents
weight: 40
bookToc: true
---

# Agents

Agents 是口试系统中的智能体集合，负责出题、阶段评分、评分汇总和报告评价。当前先介绍各 Agent 的职责和主要输入输出，具体提示词、接口和执行流程后续再补充。

## BaseAgent

`BaseAgent` 是所有其他 Agent 的基础，负责处理通用的任务和逻辑。它不直接与用户交互，而是作为其他 Agent 的父类，提供基础的功能和接口。
- 可以继承 get_tools 方法，返回工具组，用于子 Agent 调用外部工具。

```python
model_settings ={
  "model_name": "模型名称",
  "model_url": "模型 API 地址",
  "model_api_key": "模型 API 密钥",
}
thinking = False, # 是否开启思考模式，部分模型需要设置思考模式，比如glm-4.6、glm-4.7 等。
response_format = False, # 是否开启结构化输出，比如JSON格式。

## 设置规则，默认规则为：(但现在该部分内容没有使用)
async def rule(self, obj_id: str, active_nodes: dict) -> bool:
    if obj_id in active_nodes:
        logger.info(f"obj_id {obj_id} is in active_nodes")
        return False
    if len(active_nodes) > 2:
        logger.info(f"当前active_nodes数量 {active_nodes} 超过2个")
        return False
    return True

# 返回工具组
def get_tools(self):
    return []

# 返回响应格式
def get_response_format(self):
    return None
```

## ExamSetterAgent

`ExamSetterAgent` 负责出题。它会调用 `code_analysis` 工具组读取和分析学生代码，也会调用 `search_tool` 获取教师提供的参考文档或学生自己的报告内容，从而生成与考试维度、学生材料和参考资料相关的口试问题。

`ExamSetterAgent` 的出题结果采用结构化输出：

```json
{
  "difficulty_level": "{active_difficulty_level}",
  "dimension": "与维度规则中同序号维度完全对应",
  "Question": "简短中文问题摘要",
  "standard_answer": "标准答案或参考答案",
  "question_blocks": [
    {
      "type": "text",
      "content": "题面正文"
    }
  ],
  "code_fragments": [],
  "knowledge_point": "考察的核心知识点",
  "reason": "为什么这个问题适合作为口试题"
}
```

字段含义：

- `difficulty_level`：当前题目的难度等级。
- `dimension`：题目所属维度，必须与维度规则中同序号维度完全对应。
- `Question`：简短的中文问题摘要。
- `standard_answer`：标准答案或参考答案。
- `question_blocks`：题面内容块，当前先以文本题面为主。
- `code_fragments`：题目关联的代码片段，后续按代码题需要补充。
- `knowledge_point`：本题考察的核心知识点。
- `reason`：说明为什么这个问题适合作为口试题。

## StageJudgerAgent

`StageJudgerAgent` 负责阶段评分。它根据标准答案或参考答案，对学生当前回答进行打分，并说明判断依据。

该 Agent 的重点不是只给出分数，而是需要解释为什么当前回答符合或不符合标准答案，方便后续汇总和教师复核。

## StageJudgeAdjudicatorAgent

`StageJudgeAdjudicatorAgent` 负责整理多个 `StageJudgerAgent` 的评分结果。它会汇总不同阶段评分者的判断，处理评分差异，并形成更统一的阶段性评价结果。

## DocJudgerAgent

`DocJudgerAgent` 负责报告评价。它会调用 `search_tool` 获取学生报告内容，并在需要时检索教师提供的参考资料，然后根据教师设置的报告评判规则，对报告完成情况进行打分。

该 Agent 主要关注报告是否满足课程、实验或考试要求，包括内容完整性、关键知识点覆盖情况、与参考资料的一致性，以及报告表达是否能够支撑学生的口试表现。

## TextfixAgent

`TextfixAgent` 负责文本修复。它会根据提供的上下文语境对STT的音译结果进行修正，确保其符合要求。


