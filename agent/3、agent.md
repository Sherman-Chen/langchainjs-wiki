# 前言
在AgentExecutor章节中，我们了解到_call会循环调用agent.plan，这一章节就挑一个具体的agent，分析一下plan的逻辑。

# Agent类

BaseSingleAction是一个基，可以看到plan是一个抽象的方法。具体的实现是由具体的agent实现的。

```typescript
export abstract class BaseSingleActionAgent extends BaseAgent {
  abstract plan(
    steps: AgentStep[],
    inputs: ChainValues,
    callbackManager?: CallbackManager
  ): Promise<AgentAction | AgentFinish>;
}

// agent也是一个抽象类
export abstract class Agent extends BaseSingleActionAgent {
    // 内部实际上是调用_plan
    plan(
    steps: AgentStep[],
    inputs: ChainValues,
    callbackManager?: CallbackManager
  ): Promise<AgentAction | AgentFinish> {
    return this._plan(steps, inputs, undefined, callbackManager);
  }

    // 实际的逻辑
    private async _plan(
    steps: AgentStep[],
    inputs: ChainValues,
    suffix?: string,
    callbackManager?: CallbackManager
  ): Promise<AgentAction | AgentFinish> {
    const thoughts = await this.constructScratchPad(steps);
    const newInputs: ChainValues = {
      ...inputs,
      agent_scratchpad: suffix ? `${thoughts}${suffix}` : thoughts,
    };

    if (this._stop().length !== 0) {
      newInputs.stop = this._stop();
    }
    // 实际上就是调用llm.predict
    const output = await this.llmChain.predict(newInputs, callbackManager);
    if (!this.outputParser) {
      throw new Error("Output parser not set");
    }
    // 得到结果然后解析并返回
    return this.outputParser.parse(output, callbackManager);
  }
}
```

这里是调用llmChain.predict，这个llmChain就是由不同的agent创建的chain。

比如ChatAgent，有代码如下：

```typescript
import { LLMChain } from "../../chains/llm_chain.js";

export class ChatAgent extends Agent {
    static fromLLMAndTools(
    llm: BaseLanguageModelInterface,
    tools: ToolInterface[],
    args?: ChatCreatePromptArgs & AgentArgs
  ) {
    ChatAgent.validateTools(tools);
    const prompt = ChatAgent.createPrompt(tools, args);
    // 这里使用llm创建了一个LLMChain
    const chain = new LLMChain({
      prompt,
      llm,
      callbacks: args?.callbacks ?? args?.callbackManager,
    });
    const outputParser =
      args?.outputParser ?? ChatAgent.getDefaultOutputParser();

    return new ChatAgent({
      llmChain: chain,
      outputParser,
      allowedTools: tools.map((t) => t.name),
    });
  }
}
```

之前在chain章节已经介绍过，我们这里再简单看下LLMChain的定义：

```typescript
// 继承BaseChain
export class LLMChain<
    T extends string | object = string,
    Model extends LLMType = LLMType
  >
  extends BaseChain
  implements LLMChainInput<T>
{}

// 继承BaseLnagChain
export abstract class BaseChain<
    RunInput extends ChainValues = ChainValues,
    RunOutput extends ChainValues = ChainValues
  >
  extends BaseLangChain<RunInput, RunOutput>
  implements ChainInputs
{}
```

再对比上一章的agentExecutor的定义：
```typescript
export class AgentExecutor extends BaseChain<ChainValues, AgentExecutorOutput> {}
```

可以看到LLMChain和AgentExecutor都继承了BaseChain。不同的是LLMChain实现了大模型的逻辑，而AgentExecutor实现了agent的执行逻辑。

# agent的作用

AgentExecutor是描述了agent的执行链路，而agent则是定义agent的能力(tools)和模型执行链路。

同一个模型，要执行不同的agent，依靠的是prompt。因此可以看到langchain中各种预定义的agent，总体的逻辑都是一样的。

区别只是在prompt如何定义和如何传参的上面。

因此我们可以在langchain的各种不同的类上面看到方法`createPrompt`和prompt.ts文件用来定义prompt。











