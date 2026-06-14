---
title: Tool 集合
weight: 20
bookToc: true
---

# Tool 集合

Tool 是考试系统中的基础能力集合，负责把外部材料、代码仓库、结构化文档和知识库检索能力提供给上层的出题、问答和判定流程。当前先介绍工具职责，具体调用方式和接口细节后续再补充。

## code_analysis

`code_analysis` 用于读取和理解学生提交的代码内容。代码分析结果可以被出题者用于查看代码、生成问题，也可以为后续判定提供代码层面的依据。

`code_analysis` 由 `ast_tool`、`lsp_tool` 和 `code_reader` 构成：

- `ast_tool`：负责读取单个文件的语法树，以便快速获取文件中的函数、类或其他关键结构。
- `lsp_tool`：负责获取整个项目的 `project_map`，也可以获取某个函数的符号引用，用于快速预览项目结构或获取跨文件引用信息。
- `code_reader`：负责读取文件的某几行内容，用于在定位到目标代码后进一步查看代码细节。

## git_tool

`git_tool` 支持获取 GitHub 上的仓库代码，也支持解析用户上传的本地压缩包。为了让系统正确识别材料结构，仓库或压缩包必须符合 `/code` 和 `/doc` 的目录格式，系统才能将代码与文档解析到对应的本地文件夹中。

## file_tool

`file_tool` 负责文件解析。对于 `doc`、`pdf` 等非结构化文档，系统利用 MinerU 进行解析；对于 `json` 等结构化文档，则可以直接通过规则进行解析。

## insert_tool

`insert_tool` 负责知识入库。它可以将 `list[str]` 或 `str` 按课程、考试、来源的形式存储到 Meilisearch 中，为后续检索和 RAG 提供数据基础。

## search_tool

`search_tool` 负责知识检索。它根据 query 以及课程、考试、来源等相关条件，在数据库中找出最匹配的 N 个知识块，并将检索结果提供给出题、问答或判定流程使用。
