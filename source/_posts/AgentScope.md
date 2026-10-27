---
title: AgentScope2.0
tags:
  - agent
categories:
  - learning
date: 2026-10-27 12:23:26
---

测试agentscope2.0框架搭建agent

<!-- more -->
# Agent:AgentScope2.0
起因需要对比react框架下使用research skill和纯搜索+LLM生成的效果对比。  
但是由于可用的只有一个AgentScope框架，且不希望过于复杂，被平台限制，因此希望学习AgentScope框架，并搭建个人agent。  

## 构建一个基础的Agent
**model** ：使用deepseek 官方API接口的deepsekk-v4-flash模型。agentscope提供了DashScopeChatModel、OpenAIChatModel、MoonshotChatModel等几个公司的封装类，deepseek也有DeepSeekChatModel；  
**Agent** ：最基础的构建一个Agent也很简单，仅需一个模型即可;  
**Agent 是 AgentScope 的核心抽象：一个无状态的推理-行动循环引擎，将模型、工具、权限系统、人机交互、上下文管理、中间件、状态管理和事件系统整合到一个统一接口中。**  
其主要职责包括：    
接收输入消息或事件，调用工具完成任务  
生成符合用户给定 schema 的结构化输出  
管理上下文，包括上下文压缩与卸载  
通过运行时状态注入感知环境变化（时间、任务、上下文用量）  
在生命周期的关键节点运行中间件，执行自定义逻辑  
自动编排工具调用的并发与顺序执行  
处理用户中断，并从暂停状态继续运行  

**message** ：AgentScope 提供三个快捷方法构建Msg对象SystemMsg、UserMsg、AssistantMsg。

```python
import os
from agentscope.agent import Agent
from agentscope.model import DeepSeekChatModel
from agentscope.credential import DeepSeekCredential
from agentscope.message import UserMsg

#定义模型
dsmodel = DeepSeekChatModel(credential=DeepSeekCredential(api_key = os.environ.get('DeepSeek_API_KEY')),model="deepseek-v4-flash",client_kwargs={"base_url": "https://api.deepseek.com"},stream=False)

async def chat():
    #构建agent
    agent = Agent(name="myagent",system_prompt="""你是一个助手。""",model=dsmodel)
    msgs = [
            Msg(
                name="user",
                content=[TextBlock(text="Agent是什么")],
                role="user",
            )
        ]
    #调用agent
    resp = await agent.reply(inputs=msgs)
    #打印agent返回的内容；reply返回最终 Msg；reply_stream同 reply，但以流式方式逐一产出 AgentEvent 对象；
    print(resp.content)

await chat()
```
## Agent加入skill、tool
### **skill**：
Skill（技能）是 Markdown 格式的指令集，无需写代码即可拓展智能体能力。每个 skill 是一个目录，固定包含一个带 frontmatter 元数据与详细指令的 SKILL.md 文件。  
与工具不同，skill 不能被直接调用。智能体通过自动注册的 Skill 查看器读取 skill 指令，再用现有的工具按指令执行。  
#### **新建skill文件**：
- 1、在myskill文件夹下，构建SKILL.md，示例，构建desktop-file-type-identifier（桌面文件查找）技能，里面需要调用read_desktop_files.py获取本地桌面文件；

SKILL.md

```md
---
name: desktop-file-type-identifier
description: Identify and summarize file types present on the user's local Desktop by calling the desktop-file-reader tool. Use when the user asks what file types, extensions, or files are on their Desktop.
---

Desktop File Type Identifier
Purpose
Answer questions about the file types present on the user's local Desktop. Use the read_desktop_files tool to inspect the directory; do not guess or claim to have inspected files without calling the tool.
Workflow
1. Identify that the user is asking about files or file types on their Desktop.
2. Call read_desktop_files.
3. Read the tool result:
   - Summarize file extensions and counts.
   - Mention files without an extension as 无扩展名.
   - If the tool reports an error or the Desktop directory is unavailable, report that plainly.
4. Return a concise answer in the user's language.
Response requirements
- Report only information returned by the tool.
- Distinguish file count from file-type count.
- Do not claim to have read file contents. This tool only lists file names and extensions.
- If there are no files, say that no files were found.
- Do not expose unnecessary full local paths unless the user asks.
Example
User: “我的桌面上有哪些文件类型？”

After calling read_desktop_files, respond in a format such as:

“你的桌面上共发现 12 个文件，包含 4 种文件类型：
- PDF：3 个
- Word（.docx）：2 个
- 图片（.png）：5 个
- 无扩展名：2 个”
Use the actual tool result; the numbers above are illustrative.
```

read_desktop_files.py

```python
#!/usr/bin/env python3
"""List file names and extensions on the public Desktop.

This tool does not read file contents. It scans the Desktop directory and
returns a JSON summary suitable for an agent Skill.
"""

from collections import Counter
from pathlib import Path
import json
import os
import sys


def find_desktop() -> Path:
    """Resolve the public Desktop directory."""
    return Path(r"C:\Users\Public\Desktop")


def read_desktop_files() -> dict:
    desktop = find_desktop()

    if not desktop.is_dir():
        return {
            "ok": False,
            "desktop_path": str(desktop),
            "error": "Desktop directory was not found or is not accessible.",
            "file_count": 0,
            "type_count": 0,
            "types": [],
            "files": [],
        }

    files = []
    counts = Counter()

    try:
        for entry in desktop.iterdir():
            # Only include regular files, not folders.
            if not entry.is_file():
                continue

            suffix = entry.suffix.lower()
            file_type = suffix if suffix else "无扩展名"

            files.append({
                "name": entry.name,
                "extension": suffix if suffix else None,
                "type": file_type,
            })
            counts[file_type] += 1

    except OSError as exc:
        return {
            "ok": False,
            "desktop_path": str(desktop),
            "error": str(exc),
            "file_count": len(files),
            "type_count": len(counts),
            "types": [
                {"type": file_type, "count": count}
                for file_type, count in sorted(counts.items())
            ],
            "files": files,
        }

    return {
        "ok": True,
        "desktop_path": str(desktop),
        "file_count": len(files),
        "type_count": len(counts),
        "types": [
            {"type": file_type, "count": count}
            for file_type, count in sorted(counts.items())
        ],
        "files": sorted(files, key=lambda item: item["name"].lower()),
    }


if __name__ == "__main__":
    result = read_desktop_files()
    json.dump(result, sys.stdout, ensure_ascii=False, indent=2)
    sys.stdout.write(os.linesep)

```
#### agent中加入skill
agentscope中使用Toolkit 装配 skill、tool、mcp；

在agentscope中，Toolkit含有skill时，工作方式分注册和查看两阶段进行：  
初始化阶段：  
- Toolkit 扫描所有注册的 skill 来源，收集每个 skill 的名称、描述与目录。  
- 自动注册内置的 Skill 查看器。  
- 组装一段系统提示片段，列出可用 skill（仅名称与描述），并指示智能体通过 Skill 查看器读取完整内容。  
运行时阶段：  
- 智能体按名字选定一个 skill，调用 Skill 查看器。  
- 查看器读取对应 SKILL.md，返回完整 Markdown。  
- 智能体用已装备的工具按这些指令执行。  

**注册skill**:  

```python
from agentscope.tool import Toolkit
#简单的，非含子目录扫描，复杂建官方文档
toolkit = Toolkit(
    skills_or_loaders=["/path/to/skills"],
)
```
### Human-in-the-Loop（人机交互）
暂停以等待用户确认或外部执行，再通过结果事件恢复。  
Agent在tool调用时，可能会需要用户确认（如权限系统返回ASK），或标记为外部执行的tool。两种情况下，都需要通过向reply/reply_stream传回结果来恢复agent。   

**用户确认User Confirmation**  

当权限系统判断某个tool调用需要用户批准时，Agent会发出 RequireUserConfirmEvent 并暂停。  
1、接收 RequireUserConfirmEvent  
2、构建确认结果  
3、恢复Agent  

```python
from agentscope.event import RequireUserConfirmEvent
from agentscope.event import ConfirmResult, UserConfirmResultEvent
# 1、接收 RequireUserConfirmEvent = event==RequireUserConfirmEvent
async for event in agent.reply_stream(msg):
    if isinstance(event, RequireUserConfirmEvent): 
        for tc in event.tool_calls:
            print(f"工具: {tc.name}, 输入: {tc.input}")
            print(f"建议规则: {tc.suggested_rules}")
# 2、构建确认结果
confirm_results = []
for tc in event.tool_calls:
    confirm_results.append(ConfirmResult(
        confirmed=True,           # 或 False 表示拒绝  # 主要是修改这个地方，将原来的RequireUserConfirmEvent event回传回去
        tool_call=tc,             # 传回（可选择修改）
        rules=tc.suggested_rules, # 接受规则以便未来自动允许
    ))
3、恢复Agent 
confirm_event = UserConfirmResultEvent(
    reply_id=event.reply_id,
    confirm_results=confirm_results,
)
result = await agent.reply(confirm_event)  #重新调用智能体
```

**外部工具执行**  

同理类似于用户确认（User Confirmation），但是event不同，event是RequireExternalExecutionEvent。  

### 完整代码示例

```python
import os
from agentscope.agent import Agent
from agentscope.model import DeepSeekChatModel,OpenAIChatModel
from agentscope.credential import DeepSeekCredential
from agentscope.message import UserMsg,Msg,TextBlock,AssistantMsg
from agentscope.workspace import LocalWorkspace
from agentscope.tool import Toolkit,FunctionTool
from agentscope.skill import LocalSkillLoader
from agentscope.state import AgentState
from pathlib import Path
from my_skill_folder.read_desktop_files import read_desktop_files   # 加载函数，后续将其封装到tool 
from agentscope.event import TextBlockDeltaEvent,RequireUserConfirmEvent,RequireExternalExecutionEvent,ConfirmResult,UserConfirmResultEvent
# 定义model
dsmodel = DeepSeekChatModel(credential=DeepSeekCredential(api_key = os.environ.get('DeepSeek_API_KEY')),model="deepseek-v4-flash",client_kwargs={"base_url": "https://api.deepseek.com"},stream=False)
# 注册skill
skill_dir ="./my_skill_folder" # skill文件夹路径
loader = LocalSkillLoader(skill_dir)
toolkit = Toolkit(skills_or_loaders=[loader],
                  tools = [FunctionTool(read_desktop_files,name="read_desktop_files",description="列出当前用户桌面上的文件名和扩展名，并按类型统计。")]  # 将 read_desktop_files函数封为tool
                  )

# print(await toolkit.get_tool_schemas()) # 查看是否已装配tool

state_file = Path("agent_state.json") #agent state文件

print("-----")
async def chat(state_file):
    # 判断是否存在现有agent state文件，若无，新建AgentState()
    if state_file.exists():
        state = AgentState.model_validate_json(
            state_file.read_text(encoding="utf-8")
            )
    else:
        state = AgentState()
    # 定义agent
    agent = Agent(name="myagent",system_prompt="""你是一个助手。不能直接回答，而要先读取该 skill。""",model=dsmodel,toolkit=toolkit,state=state) 
    # print(await toolkit.get_skill_instructions())
    # print(await toolkit.get_tool_schemas())
    print("-----")
    # msgs =[]
    answer_parts=[]

    while True:
        user_input = input("你：") # 输入
        if user_input.strip().lower() in {"q","exit","quit"}: #loop退出条件
            print("_______________MSGS_______________________________________")
            #退出后将state写入文件
            state_file.write_text(
                agent.state.model_dump_json(indent=2),
                encoding="utf-8",
            )
            # print(msgs)
            break
        # msgs = [
        #         Msg(
        #             name="user",
        #             content=[TextBlock(text="我的桌面有哪些文件")],
        #             role="user",
        #         )
        #     ]
        user_msg = UserMsg(name="user",content=user_input) #将用户输入封成UserMsg Msg
        # msgs.append(UserMsg(name="user",content=user_input))

        pending_confirm = None

        async for event in agent.reply_stream(inputs=user_msg): #agent调用
            print(type(event).__name__, event)
            if isinstance(event, TextBlockDeltaEvent): #判断是否正常文本输出 ，因为reply_stream输出的是event，所以最后一个不是答案，需要看TextBlockDeltaEvent event
                answer_parts.append(event.delta)  # 加入answer列表
                print(event.delta, end="", flush=True) #输出agent未调用tool时的答案
            if isinstance(event, RequireUserConfirmEvent): # 判断是否需要用户确认
                pending_confirm = event #定义用户待确认的event
                # print(f"pending_confirm:{pending_confirm}")

        print("------------------------")
        print()  
        if pending_confirm is not None:  #若存在需要用户待确认的event
            confirm_event = UserConfirmResultEvent(
                reply_id=pending_confirm.reply_id,
                confirm_results=[
                    ConfirmResult(confirmed=True, tool_call=tool_call)
                    for tool_call in pending_confirm.tool_calls
                ],
            )# 修改confirmed=True，并装回原event
            # print(f"new pending_confirm:{pending_confirm}")
            resp = await agent.reply(inputs=confirm_event) #恢复agent，重新执行agent
            print("助手：", resp.get_text_content()) #输出答案

            # msgs.append(AssistantMsg(name="assistant",content=resp.content))

```

## Agent增加上下文压缩、记忆
memory  
compress_context  

## Agent调用agent
需要使用agent中的observe(msgs)【将消息添加到上下文，不触发推理】  




参考文献：  
https://docs.agentscope.io/zh/versions/2.0.11  

github(https://github.com/agentscope-ai/agentscope/tree/main)  