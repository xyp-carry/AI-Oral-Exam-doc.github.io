---
title: C 模式
weight: 10
bookToc: true
---

# C 模式

C 模式的取题与回答由用户端、STT、Interview、问题端和 TTS 协同完成。每道主问题最多追问两次。下面的时序图描述一次口试从开始到结束的问答过程。

## 取题与回答时序

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant U as 用户端
    participant STT as STT
    participant I as Interview
    participant Q as 问题端
    participant TTS as TTS

    U->>I: 开始口试
    I->>Q: 获取第一道主问题
    Q-->>I: 返回主问题
    I-->>U: 返回问题文本
    I->>TTS: 发送问题播报内容
    TTS-->>U: 返回问题语音

    loop 逐轮作答，直到口试结束
        alt 语音回答
            U->>STT: 提交语音
            STT-->>I: 返回回答文字
        else 文字回答
            U->>I: 提交回答文字
        end
        I->>I: 整理本轮回答
        I->>Q: 提交回答
        Q->>Q: 处理回答并确定下一步

        alt 继续追问，最多两次
            Q-->>I: 返回追问
            I-->>U: 返回问题文本
            I->>TTS: 发送追问播报内容
            TTS-->>U: 返回问题语音
        else 当前主问题完成
            alt 还有下一道主问题
                Q-->>I: 返回下一道主问题
                I-->>U: 返回问题文本
                I->>TTS: 发送问题播报内容
                TTS-->>U: 返回问题语音
            else 所有主问题已完成
                Q-->>I: 通知口试结束
                I-->>U: 结束口试
            end
        end
    end
{{< /mermaid >}}

用户端开始口试后，Interview 从问题端取得第一道主问题，将文本返回用户端，并交给 TTS 播报。用户可以用语音或文字回答；语音先由 STT 转写，再由 Interview 整理本轮回答并提交问题端。

问题端根据回答决定继续追问或结束当前主问题。每道主问题最多追问两次；每次返回的新问题都同时提供文本和 TTS 语音。所有主问题完成后，问题端通知 Interview，由 Interview 结束口试。问题生成、阶段判断及候选追问的等待与失败分支如下。

### 回答过程中的问题生成与判断时序

流程从预设主问题输入开始，重点区分两个时点：当前问题就绪时立即异步预生成候选追问；新回答片段到达后，才请求考官节点做阶段判断。回答结束后，问题节点依据已有判断选择候选；候选尚未完成时，由 Interview 查询生成结果。

{{< mermaid >}}
sequenceDiagram
    autonumber
    participant IN as 预设问题输入
    participant I as Interview
    participant Q as 问题节点
    participant E as 考官节点

    IN->>I: 输入预设主问题队列
    I->>Q: 启动题目流程
    Q->>Q: 取第一道主问题

    loop 每个当前问题，直到全部主问题结束
        Q-->>I: 输出当前问题
        opt 当前主问题已发出的追问少于两次
            Q->>Q: 问题就绪时立即异步预生成两种候选追问
        end

        loop 回答尚未结束且有新片段
            I->>Q: 提交新回答片段
            Q->>E: 收到片段后开始判断
            E-->>Q: 返回阶段判断
            opt 需要额外候选追问
                Q->>Q: 作答期间追加启动候选生成
            end
        end

        I->>Q: 通知本轮回答结束
        opt 结束消息还带有新回答内容
            Q->>E: 按需判断最后一段内容
            E-->>Q: 返回判断
        end
        alt 已完成两次追问
            Q->>Q: 完成当前主问题的题链
        else 已发出的追问少于两次
            Q->>Q: 根据已有判断选定候选追问
            alt 选中的候选已经就绪
                Q->>Q: 切换当前问题，追问次数加一
                Q-->>I: 回答结束后立即发出追问
            else 选中的候选仍在生成
                Q-->>I: 通知等待追问
                loop 候选尚未完成
                    I->>Q: 查询候选状态
                end
                alt 生成成功
                    Q->>Q: 切换当前问题，追问次数加一
                    Q-->>I: 生成完成后发出追问
                else 生成失败
                    Q->>Q: 完成当前主问题的题链
                end
            else 选中的候选已失败或不可用
                Q-->>I: 报告追问生成失败
                Note over I,Q: 当前实现此处没有同步推进到下一主问题
            end
        end

        opt 当前题链已结束
            alt 还有下一道主问题
                Q->>Q: 取下一道预设主问题
            else 全部主问题完成
                Q-->>I: 通知口试结束
            end
        end
    end
{{< /mermaid >}}

候选追问已就绪时，问题节点在回答结束后直接发出追问；仍在生成时，Interview 查询到生成结果后再继续。生成过程中失败会结束当前主问题的题链。若选中的候选已失败或不可用，当前实现只报告追问生成失败，没有同步推进到下一道主问题。

考试准备、通用评分规则和文件管理见[实施口试链路](../../../oral-exam/)；模式选择和共同职责见[口试核心服务](../../)。
