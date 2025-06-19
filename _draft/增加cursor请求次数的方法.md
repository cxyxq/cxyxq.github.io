增加cursor请求次数的方法

## 1. 配置MCP: interactive-feedback-mcp

1. 下载代码，参考 [interactive-feedback-mcp](https://github.com/noopstudios/interactive-feedback-mcp) 中的[Installation (Cursor)]

2. 增加cursor规则配置,我增加的是：Project Rules，也可以把它加到user rules
   复制下面的提示词到：
   
   ```text
   1. 当你需要向用户提问时，必须调用 MCP 的 interactive_feedback 收集反馈
   2. 在响应用户请求即将结束前，必须再次调用 interactive_feedback，以确认用户是否满意或有补充需求，只有用户明确表示“没有进一步反馈”时，才可终止对话流程
   3. 你不可以自行决定开启新的对话流程，必须等待用户明确表示需要开启新对话
   ```

3. 手动配置mcp server，把里面的路径**换成你自己的**
   
   ```json
   {
   "mcpServers": {
    "interactive-feedback-mcp": {
      "command": "D:\\software\\miniconda3\\Scripts\\uv.exe",
      "args": [
        "--directory",
        "E:\\my_code\\interactive-feedback-mcp",
        "run",
        "server.py"
      ],
      "timeout": 600,
      "autoApprove": [
        "interactive_feedback"
      ]
    }
   }
   }
   ```

## 2. 增加feedback提示词方式

1. 增加user rules:

```
interactive_feedback_cmd = `python -c "i = input('输入您的反馈:'); print(i)"`
1. 当你需要向用户提问时，必须执行 interactive_feedback_cmd 收集用户输入的反馈
2. 在响应用户请求即将结束前，必须再次调用 interactive_feedback_cmd， 以确认用户是否满意或有补充需求，只有用户明确表示“没有进一步反馈”时，才可终止对话流程
```

2. 效果：
   在每次cursor结束前，它会：

```shell
现在让我执行交互反馈命令，收集您的意见：
> python -c "i = input('输入您的反馈:'); print(i)"
```

此时你就可以输入新的需求了，cursor会基于你输入的内容继续回答问题，此时仍在一个会话内！！！