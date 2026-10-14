#局限性，应用面太窄
1、通用办公
2、垂直办公（设计、视频、教育...）
3、客服助手（嵌入网页、app）


**Agent原理**

大模型（大脑cpu）+ Harness（一系列影响Agent最终输出的工具）  
[https://dg-ai-notes.pages.dev/](https://dg-ai-notes.pages.dev/)

  
**Agent Loop**  
loop1  
模型(提示词、是否调用工具tool_call对象)  
工具执行(工具schema：name、description、parameters)

loop2  
模型(提示词、+上轮工具执行结果回传、是否继续调用工具)  
工具执行  
.  
.  
loopn  
模型(不用再调用工具)  
组织回复

  

**Loop关键节点留钩子**  
工具（组织schema进system prompt、工具返回格式处理、工具报错处理）  
日志/trace  
循环上线  
危险检测（执行权限、敏感信息）  
内容截断(工具返回内容过多，只传关键信息给大模型 ，由大模型自己决定是否读取完整内容)  
上下文记忆（最近/历史）  
上下文压缩（时机与内容/goal目标/约束条件/工作流程/关键决策/下一步计划）

  

**Agent的常见的几种模式**

1. ReAct 思考与行动（reasoning and acting）【大模型决定是否继续】  
       - 上方的loop就是最常见的ReAct

- → 思考(大模型本身开启思考/强行让大模型输出思考标签)
- → 行动(解析指令/参数验证/执行工具)
- → 观察(分析和格式化结果，再喂给大模型进行下一个turn)

          
  2.  Plan and Execute 计划模式

       先想好完整计划（列出所有步骤），再按计划执行  
      
  3. **混合模式**：先做 Plan（列出步1、步2、步3），然后在每一步里用 ReAct（思考→行动→观察）去执行。  

  4. 循环模式 【程序决定是否继续】

       - 一般在模型外层决定，判断代码是否通过验证，或者是否计算出来某个确定值  
  
  5. Reflection 反思【在React外多套一层】

      - 把finalanser 与query 进行分析(Reasoning) 判断是否要重新执行