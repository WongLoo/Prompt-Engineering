# Prompt Engineering Application Experience

在2022年11月，ChatGPT的问世引发了一场AI技术革命。作为公司的人工智能部门，我们迅速拥抱了这一革命性工具。从最初的摸索到如今的熟练应用，我们团队在提示工程和RAG技术方面经历了多次实验和改进。
提示工程所有人都可以接触，在ChatGPT平台的输入框中输入任意词语发送，这便是输入给ChatGPT的提示词。但是和模型对话很简单，但让其听从指令却并非易事。如何让提示词发挥更大的作用，使得大语言模型能够更好地完成任务呢？这里我将分享在这一年多的实践中，我所运用和观察得到的提示工程技巧。

# 明确的指示

- 带有明确的“翻译”指示，成功执行翻译任务
![image](https://github.com/user-attachments/assets/04b7fa10-1d19-4e64-abda-b07ae2b738ca)
- 未明确指示，导致任务理解出错
![image](https://github.com/user-attachments/assets/8e589255-10bb-4882-8aa6-266ddecef0b5)

```
① 当设计一个提示时，我们需要从LLM的角度出发，思考：模型会如何理解我的任务的最终目标？如果我们自己都不清楚任务是什么，LLM又怎能理解呢？ 
② LLM擅长从1到100的任务，但在从0到1的创新性任务上则相对薄弱。
```
# 如何令提示词变得更加明确？
## 分隔符的使用：

分隔符的使用可以避免用户输入的文本中可能存在的误导性信息干扰原有指令。

分隔符是特殊的符号，但选择哪种特殊符号并不重要。关键的是这些字符需要足够独特，使得LLM能将其识别为分隔符，帮助LLM辨识提示中哪些部分应当被视为一个完整的意义单元。
分隔符可以是任何不常见组合的特殊字符序列，```, """, < >, <tag> </tag>等。

例如：

```
Summarize the text delimited by triple backticks into a single sentence.
```{text}```
```

## 结构化的输出：

为了便于后续开发，我们可以要求模型以JSON格式输出。然而，在多次运行后，可能会发现输出的格式并不稳定，例如缺少双引号或括号。为了解决这一问题，可以尝试以下方法，以提高输出格式的稳定性：
- Function Calling：利用函数调用的方式，确保输出符合预期格式。
- Json Mode (GPT-4.0)：参考[OpenAI文档](https://platform.openai.com/docs/api-reference/chat/create#chat-create-response_format)，使用JSON模式来生成结构化的输出。
- 提示词约束：通过在提示词中明确要求以JSON格式输出，并提供示例以说明输出的JSON应遵循的结构（即包含哪些键）。
  
例如：

```
The output should be a well-formatted JSON instance that conforms to the JSON schema below. For instance, the schema is as follows:

{
    "title": "string",
    "description": "a list of strings",
    "type": "array",
    "items": {"type": "string"}
}
```
LinkedIn 团队的经验是： 
1). 使用 YAML 格式而不是 JSON，相对来说容错率更高 
2). 用日志记录常见的 YAML 错误，优化自己的 YAML 解析器，可以解析 LLM 返回的不规范的 YAML 3). 如果还是无法解析则将错误信息交给 LLM 修复，并且不断优化提示词，提升 LLM 修复的成功率
3). 如果还是无法解析则将错误信息交给 LLM 修复，并且不断优化提示词，提升 LLM 修复的成功率

# 少样本提示&步骤拆解：

通过提供给模型一个或多个样本提示，模型可以更清楚地理解预期的输出格式和内容。少样本提示结合步骤拆解的技巧在处理复杂推理任务或需要较强业务背景理解的场景中尤为有效。
例如复杂数学推理：

```
The odd numbers in this group add up to an even number: 4, 8, 9, 15, 12, 2, 1.
Odd Number List: 9, 15, 1
Odd Number Sum: 9 + 15 + 1 = 25
25 is not an even number.
A: The answer is False.

The odd numbers in this group add up to an even number: 17,  10, 19, 4, 8, 12, 24.
Odd Number List: 17, 19
Odd Number Sum: 17 + 19 = 36
36 is an even number.
A: The answer is True.

The odd numbers in this group add up to an even number: 15, 32, 5, 13, 82, 7, 1. 
A: 
```
# 给模型时间“思考”
## 将任务拆分为更简单的子任务

CoT( Chain of Thought) 链式思考，将中间步骤拆解为小任务，以降低任务复杂度，促使LLM更好完成任务。
![image](https://github.com/user-attachments/assets/367dd299-1739-4adb-83ac-31236342aacb)

## 输出中间计算结果

同样的加入了CoT（Chain of Though）的Prompt，如果让GPT将中间计算结果也输出来，效果会更佳。但是如果不让GPT输出中间思路（这样做可以省点token，并且更容易解析），那么GPT就会偷懒，完全没有按照要求的步骤来做。

```
Based on the following known information, provide a professional and concise answer to the user's question.

## Role:
You will play the role of a chart query assistant, with professional chart analysis and business metrics understanding skills. Your task is to provide relevant chart information based on the user's question.

## Chart Information:
The information contained within <context> and </context> is retrieved from the database and is the most relevant chart information to the user's question (though some chart information may be unrelated). For ease of understanding, the information is presented in JSON format, containing five key fields: "chart_name", "chart_id", "chart_desc", "chart_subject", and "chart_metrics".

<context>
[
  {
    "chart_name": "Monthly New Devices on Korean Server",
    "chart_id": "sdgafdgdhryjnynmtujy",
    "chart_desc": "This chart shows the number of new devices added each month on the Korean server from January 2018 to the end of last month, based on the monthly reports of devices on various servers."
  },
  {
    "chart_name": "Daily New Devices on Korean Server",
    "chart_id": "sdferbarttyjnunyn",
    "chart_desc": "This bar chart shows the number of new devices added daily on the Korean server from February 28, 2018, to the day before yesterday, based on the daily reports of devices and users on various servers."
  },
  {
    "chart_name": "Daily New Devices on Korean Server",
    "chart_id": "srdzgfdvfdgthmumu",
    "chart_desc": "This chart is from the daily reports of devices on various servers and shows the number of new devices added daily on the Korean server from February 28, 2018, to the day before yesterday."
  },
  {
    "chart_name": "Monthly New Users on Korean Server",
    "chart_id": "fdhtsehrtgdfvdfhnhny",
    "chart_desc": "This chart shows the number of new users added each month on the Korean server from January 2018 to the end of last month, based on the monthly reports of devices and users on various servers."
  }, 
  {
    "chart_name": "Total Device Trend on Korean Server",
    "chart_id": "thdtfvdxfhxfynhgmgmy",
    "chart_desc": "This chart shows the trend of the total number of devices on the Korean server from January 2018 to the end of last month, based on the monthly reports of devices and users on various servers."
  },
  {
    "chart_name": "Daily Cumulative Device Trend on Korean Server",
    "chart_id": "jdthrrrgthfdxvttjmny",
    "chart_desc": "This chart shows the daily cumulative number of devices on the Korean server from February 28, 2022, to the day before yesterday, based on the daily reports of devices and users on various servers."
  },
  {
    "chart_name": "Korean Server Data Dashboard",
    "chart_id": "yhdfgtrgxdrggdrththhhh",
    "chart_desc": "This chart is a monthly report on the devices and users of the Korean server up to the end of last month, including total devices, total users, monthly new devices, and monthly new users."
  },
  {
    "chart_name": "Daily New Users on Korean Server",
    "chart_id": "sfdcdfvrtgerggvfvv",
    "chart_desc": "This chart shows the number of new users added daily on the Korean server from February 28, 2018, to the day before yesterday, based on the daily reports of users on various servers."
  },
  {
    "chart_name": "Monthly New Devices Comparison Trend of Various Servers",
    "chart_id": "zfsdfsfrgfdvfgftdvsvsevv",
    "chart_desc": "This line chart shows the comparison trend of the number of new devices added each month on various servers worldwide from January 2021 to the end of last month, based on the monthly reports of devices and users on various servers."
  }
]
</context>

Please use the above information and follow the guidelines below to answer the user's question.

## Task Flow:
1. Analyze the user's question: Understand the theme and metrics the user is asking about.
2. Retrieve chart information:
   - Based on the user's question, determine if the chart information contains relevant themes and metrics;
   - If relevant information exists, use that information to answer the user;
   - If no relevant information exists, return an empty chart name and ID.

## Output Requirements:
Please respond in the following JSON format, ensuring the returned JSON can be parsed by Python's json.loads method:
```
{
  "asking_object_metrics": "string", // The theme and metrics the user is asking about
  "thoughts": "string", // Compare if the chart information's theme and metrics match
  "exist_or_not": "Boolean", // Determine if relevant chart information exists
  "chart_names": "list", // Names of the matching charts, up to two or empty
  "chart_ids": "list" // IDs of the matching charts, up to two or empty
}
```

## Input:
Human: New Devices Added on the Korean Server in the Past Year

```
- JSON 解析准确率：输出中间结果的 JSON 解析率为 100%，而不输出中间结果的解析率为 84.5%。
- 结果准确性：相比仅输出最终结果，输出中间解析结果的正确率提升了 15.68%。
- 耗时增加：然而，输出中间解析结果会增加耗时。在 150 条测试数据中，由于输出中间结果的 token 数量约为不输出中间结果的 4 倍，导致 LLM 的处理时间显著延长，每条数据的处理时间平均增加了 80%（由1.52s增加至2.82s）

# 使用框架令提示词更清晰

这里我将介绍我常用的提示词框架；注意这些框架的内容都不是一定要有的，需要根据实际情况进行增改。

```
## Role : [请填写你想定义的角色名称]

## Background : [请描述角色的背景信息，例如其历史、来源或特定的知识背景]

## Preferences : [请描述角色的偏好或特定风格，例如对某种设计或文化的偏好]

## Profile :

- author: Arthur
- Jike ID: Emacser
- version: 0.2
- language: 中文
- description: [请简短描述该角色的主要功能，50 字以内]

## Goals :
[请列出该角色的主要目标 1]
[请列出该角色的主要目标 2]
...

## Constrains :
[请列出该角色在互动中必须遵循的限制条件 1]
[请列出该角色在互动中必须遵循的限制条件 2]
...

## Skills :

[为了在限制条件下实现目标，该角色需要拥有的技能 1]
[为了在限制条件下实现目标，该角色需要拥有的技能 2]
...

## Examples :

[提供一个输出示例 1，展示角色的可能回答或行为]
[提供一个输出示例 2]
...

## OutputFormat :

[请描述该角色的工作流程的第一步]
[请描述该角色的工作流程的第二步]
...

## Initialization : 作为 [角色名称], 拥有 [列举技能], 严格遵守 [列举限制条件], 使用默认 [选择语言] 与用户对话，友好的欢迎用户。然后介绍自己，并提示用户输入.
```

CO-STAR框架
```
# CONTEXT（上下文） #
我想推广公司的新产品。我的公司名为 Alpha，新产品名为 Beta，是一款新型超快速吹风机。

# OBJECTIVE（目标） #
帮我创建一条 Facebook 帖子，目的是吸引人们点击产品链接进行购买。

# STYLE（风格） #
参照 Dyson 等成功公司的宣传风格，它们在推广类似产品时的文案风格。

# TONE（语调） #
说服性

# AUDIENCE（受众） #
我们公司在 Facebook 上的主要受众是老年人。请针对这一群体在选择护发产品时的典型关注点来定制帖子。

# RESPONSE（响应） #
保持 Facebook 帖子简洁而深具影响力。
```
