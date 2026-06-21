---
title: API 接口草稿
weight: 30
bookToc: true
---

# API 接口草稿

本文档根据下面四个入口文件整理，先作为前后端联调和接口梳理的粗版说明：

- `/root/Vibe-coding-test/Authentication/main.py`
- `/root/Vibe-coding-test/AIOralExamSystem/url.py`
- `/root/Vibe-coding-test/LLM/url.py`
- `/root/Vibe-coding-test/main.py`

当前服务使用 FastAPI。`main.py` 将认证路由、考试/课程路由和 LLM 模型配置路由注册到同一个 `app` 上。大部分业务接口通过 `get_current_user` 读取当前登录用户，认证方式主要依赖登录后写入的 Cookie 会话。

## 通用约定

| 项 | 说明 |
| --- | --- |
| 认证 | 除注册、登录、根路径、健康检查、模型供应商列表等少量接口外，默认需要已登录用户。 |
| 成功响应 | 多数业务接口返回 `{ "success": true, ... }`。 |
| 业务错误 | 课程、考试、模型配置等接口会把部分 `ValueError` 映射为结构化错误，如 `{ "code": "...", "message": "..." }`。 |
| 权限错误 | 无权限通常返回 `403`。未登录或用户身份无效通常返回 `401`。 |
| 文件上传 | 文件相关接口使用 `multipart/form-data`。 |

## 认证与用户接口

来源：`Authentication/main.py`

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| GET | `/` | 否 | 无 | 系统根路径，返回系统信息和主要端点提示。 |
| GET | `/health` | 否 | 无 | 健康检查，返回 `status` 和当前时间。 |
| POST | `/register` | 否 | `RegisterRequest` | 注册新用户，账号默认需要激活后才能登录。 |
| POST | `/login` | 否 | `LoginRequest` | 登录并写入会话 Cookie。 |
| POST | `/logout` | 可选 | Cookie 会话 | 注销当前设备会话并清除 Cookie。 |
| GET | `/me` | 是 | 无 | 获取当前登录用户信息。 |
| GET | `/users` | 是 | 无 | 获取用户列表和用户总数。 |
| PUT | `/profile` | 是 | `UpdateProfileRequest` | 更新当前用户邮箱或昵称。 |
| PUT | `/password` | 是 | `ChangePasswordRequest` | 修改密码，并注销所有设备。 |
| GET | `/sessions` | 是 | 无 | 查看当前用户所有活跃会话。 |
| DELETE | `/sessions` | 是 | 无 | 注销其他设备，会重新保留当前设备会话。 |
| DELETE | `/account` | 是 | 无 | 删除当前用户账号及会话。 |
| GET | `/stats` | 是 | 无 | 返回系统用户和会话统计。 |
| POST | `/gateway` | 是 | `ProxyRequest` | 认证后代理请求到目标服务。 |

### 认证请求体

| 模型 | 字段 |
| --- | --- |
| `RegisterRequest` | `username`, `password`, `email`, `role`, `nickname?`；`role` 仅支持 `teacher` 或 `student`。 |
| `LoginRequest` | `username`, `password`, `remember_me=false`。 |
| `UpdateProfileRequest` | `email?`, `nickname?`。 |
| `ChangePasswordRequest` | `old_password`, `new_password`。 |
| `ProxyRequest` | `target_url`, `method=GET`, `headers?`, `body?`, `params?`。 |

## LLM 模型配置接口

来源：`LLM/url.py`

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| GET | `/model_providers` | 否 | 无 | 获取支持的模型供应商、模型 key、默认 `base_url` 和参数 schema。 |
| POST | `/models/test` | 是 | `ModelTestRequest` | 临时测试一个模型配置，会发起一次 `Reply only OK.` 测试请求。 |
| POST | `/models` | 是 | `ModelCreateRequest` | 创建当前用户的模型配置，创建前会先测试模型可用性。 |
| GET | `/models` | 是 | 无 | 获取当前用户的模型配置列表，不返回 API Key。 |
| GET | `/models/{model_id}` | 是 | `model_id` | 获取当前用户某个模型配置，不返回 API Key。 |
| DELETE | `/models/{model_id}` | 是 | `model_id` | 删除当前用户某个模型配置。 |

### LLM 请求体

| 模型 | 字段 |
| --- | --- |
| `ModelTestRequest` | `provider`, `provider_model_key`, `model_api_key`, `params={}`。 |
| `ModelCreateRequest` | `provider`, `provider_model_key`, `model_api_key`, `display_name?`, `params={}`。 |

当前内置供应商包括 `kimi`、`glm`、`deepseek`。`params` 会按供应商模型的 `params_schema` 校验，例如 `temperature`、`top_p`、`max_tokens`、`thinking`、`reasoning_effort` 等。

## 课程与考试管理接口

来源：`AIOralExamSystem/url.py`

### Git 仓库与考试记录

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/git/repository` | 是 | JSON 或 `multipart/form-data` | 为某次考试上传代码仓库。支持传 `git_url`，或上传 `.zip` 文件，二者只能选一个。 |
| GET | `/exam_sessions/repository_status` | 是 | Query: `course_id`, `exam_id` | 查询考试记录是否已有仓库地址。 |
| GET | `/exam_history/{course_id}` | 是 | Path: `course_id`; Query: `exam_item_id?` | 查询课程下考试历史，可按考试项过滤。 |
| GET | `/exam_items/{exam_item_id}/questions` | 是 | Path: `exam_item_id` | 查询某考试项下的问题记录。 |
| GET | `/exam_record` | 是 | Query: `exam_id` | 查询某次考试记录详情。 |

`/git/repository` 的 JSON 请求体为 `GitRepositoryUploadRequest`：`course_id`, `exam_id`, `git_url?`, `git_branch=main`, `reload=false`。使用表单上传时字段相同，并可额外传 `file`，文件必须是 `.zip`。

### 课程管理

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/courses` | 是 | `CourseCreateRequest` | 创建课程。 |
| GET | `/courses` | 是 | 无 | 查询当前用户可见课程。 |
| GET | `/courses/by_invite_code/{invite_code}` | 是 | Path: `invite_code` | 根据邀请码查询课程。 |
| PUT | `/courses/{course_id}/invite_code` | 是 | `CourseInviteCodeRequest` | 重置课程邀请码有效期并生成新邀请码。 |
| POST | `/courses/{course_id}/join_requests` | 是 | Path: `course_id` | 创建加入课程申请。 |
| GET | `/courses/{course_id}/join_requests` | 是 | Path: `course_id` | 查询课程加入申请。 |
| POST | `/courses/join_requests/approve` | 是 | `CourseJoinApprovalRequest` | 审批加入课程申请。 |
| PUT | `/courses/{course_id}` | 是 | `CourseUpdateRequest` | 更新课程名称或描述。 |
| DELETE | `/courses/{course_id}` | 是 | Path: `course_id` | 删除课程。 |

### 考试项管理

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/courses/{course_id}/exam_items` | 是 | `ExamItemCreateRequest` | 在课程下创建考试项，同时可保存判定模型配置。 |
| GET | `/courses/{course_id}/exam_items` | 是 | Path: `course_id` | 查询课程下考试项列表。 |
| PUT | `/courses/{course_id}/exam_items/{exam_item_id}` | 是 | `ExamItemUpdateRequest` | 更新考试项配置，同时可更新判定模型配置。 |
| DELETE | `/courses/{course_id}/exam_items/{exam_item_id}` | 是 | Path 参数 | 删除考试项。 |
| PUT | `/exam_items/{exam_item_id}/availability` | 是 | `ExamItemAvailabilityRequest` | 重置考试项可开启时长。 |
| GET | `/courses/{course_id}/exam_sessions` | 是 | Path: `course_id` | 查询课程下考试记录。 |

### 课程资料

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/courses/{course_id}/exam_items/{exam_item_id}/course_documents` | 是 | `multipart/form-data`: `document_name`, `files[]` | 上传课程资料并写入检索库。 |
| GET | `/courses/{course_id}/exam_items/{exam_item_id}/course_documents` | 是 | Path 参数 | 查询考试项绑定的课程资料来源列表。 |
| DELETE | `/courses/{course_id}/exam_items/{exam_item_id}/course_documents/{document_name}` | 是 | Path 参数 | 删除课程资料来源，并清理检索库中的相关文档。 |

### 报告评分与预设题

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/courses/{course_id}/exam_items/{exam_item_id}/report_score` | 是 | `ReportScoreRequest` | 对报告进行评分，并可选择是否准备后续口试题。 |
| PUT | `/courses/{course_id}/exam_sessions/{exam_id}/preset_questions_usage` | 是 | `ExamSessionPresetUsageRequest` | 更新某次考试是否使用预设题。 |
| POST | `/courses/{course_id}/exam_items/{exam_item_id}/preset_questions` | 是 | `PresetQuestionCreateRequest` | 创建预设题。 |
| GET | `/courses/{course_id}/exam_items/{exam_item_id}/preset_questions` | 是 | Path 参数 | 查询考试项预设题列表。 |
| PUT | `/courses/{course_id}/exam_items/{exam_item_id}/preset_questions/{preset_question_id}` | 是 | `PresetQuestionUpdateRequest` | 更新预设题。 |
| DELETE | `/courses/{course_id}/exam_items/{exam_item_id}/preset_questions/{preset_question_id}` | 是 | Path 参数 | 删除预设题。 |

### 考试系统请求体

| 模型 | 字段 |
| --- | --- |
| `CourseCreateRequest` | `course_name`, `description?`, `invite_code_valid_times=2592000`。 |
| `CourseUpdateRequest` | `course_name?`, `description?`。 |
| `CourseJoinApprovalRequest` | `course_id`, `user_id`。 |
| `CourseInviteCodeRequest` | `invite_code_valid_times`。 |
| `ExamItemDimension` | `name`, `score`。 |
| `ExamItemCreateRequest` | `exam_item_name`, `dimensions`, `exam_available_valid_times`, `description?`, `item_type?`, `need_code_repository=false`, `use_preset_questions=false`, `enable_report_analysis=false`, `report_total_score?`, `report_judge_rule?`, `judge_model_ids?`, `setter_model_id?`, `main_judger_model_id?`。 |
| `ExamItemUpdateRequest` | 与创建字段基本一致，但全部为可选字段。 |
| `ExamItemAvailabilityRequest` | `exam_available_valid_times`。 |
| `PresetQuestionCreateRequest` | `question_dimension`, `question_content`, `standard_answer?`, `score=1.0`, `sort_order?`。 |
| `PresetQuestionUpdateRequest` | `question_dimension?`, `question_content?`, `standard_answer?`, `score?`, `sort_order?`。 |
| `ExamSessionPresetUsageRequest` | `use_preset_questions`。 |
| `ReportScoreRequest` | `user_id?`, `exam_id?`, `prepare_questions=true`。 |

## 文件与口试会话控制接口

来源：`main.py`

| 方法 | 路径 | 认证 | 请求体/参数 | 用途 |
| --- | --- | --- | --- | --- |
| POST | `/file/get_chunks` | 是 | `multipart/form-data`: `course_id`, `exam_id`, `files[]` | 上传资料文件，保存到 `updateFile/{user_uuid}/{course_id}/{exam_id}`，并调用 `InsertTool` 写入检索库。 |
| POST | `/close` | 是 | 无 | 取消并清理当前用户正在运行的口试任务。 |

## 待补充

- 目前文档只根据路由文件和请求模型做了粗粒度整理，未运行服务导出完整 OpenAPI schema。
- 具体响应 `data` 字段结构主要由 `QAserver`、repository 和数据库返回值决定，后续需要结合这些实现继续补充。
- 部分权限规则依赖当前用户角色和课程归属，建议后续按教师、学生、管理员分角色补一版调用矩阵。
