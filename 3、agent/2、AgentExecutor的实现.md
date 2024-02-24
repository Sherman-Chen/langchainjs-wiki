# invoke

agentExecutor是代码执行的起点。就是说我们定义了agent、tools这些之后，然后就可以使用invoke让agent进行执行了。

```typescript
const executor = AgentExecutor.fromAgentAndTools({
   agent: async () => loadAgentFromLangchainHub(),
   tools: [new SerpAPI(), new Calculator()],
   returnIntermediateSteps: true,
 });

 const result = await executor.invoke({
   input: `Who is Olivia Wilde's boyfriend? What is his current age raised to the 0.23 power?`,
 });
```

看下AgentExecutor的定义：

```typescript
import { BaseChain } from "../chains/base.js";
export class AgentExecutor extends BaseChain<ChainValues, AgentExecutorOutput> {}
```

可以看到AgentExecutor本质还是一个chain。之前章节中我们知道BaseChain是有一个invoke的方法的。这里再看一下invoke的定义：

```typescript
export abstract class BaseChain<
    RunInput extends ChainValues = ChainValues,
    RunOutput extends ChainValues = ChainValues
  >
  extends BaseLangChain<RunInput, RunOutput>
  implements ChainInputs
{
      async invoke(input: RunInput, config?: RunnableConfig): Promise<RunOutput> {
        let outputValues: RunOutput;
        outputValues = this._call(fullValues as RunInput, runManager, config));
        return outputValues;
      }

      abstract _call(values: RunInput,runManager?: CallbackManagerForChainRun,config?: RunnableConfig): Promise<RunOutput>;
}
```

可以看到BaseChain实际上是调用了_call方法，而_call方法是一个abstact，因此实际上的执行逻辑还是落在了AgentExecutor._call中。

# _call

_call的逻辑也很简单：
1. 循环调用模型(this.agent.plan)获得结果
2. 如果得到答案则直接返回，如果模型告诉要执行tools，则调用tool.call获得tool结果(observation)，放入到下次调用模型的input中
3. 继续1

```typescript
async _call(inputs, runManager) {
        const toolsByName = Object.fromEntries(this.tools.map((t) => [t.name.toLowerCase(), t]));
        const steps = [];
        let iterations = 0;
        // 从step中获取response，并且返回
        const getOutput = async (finishStep) => {
            const { returnValues } = finishStep;
            ...
            let response;
            // 在返回时，是否把中间步骤也返回。
            // 这个一般推荐返回，可以记录到日志中，方便调试
            if (this.returnIntermediateSteps) {
                response = { ...returnValues, intermediateSteps: steps, ...additional };
            }
            else {
                response = { ...returnValues, ...additional };
            }
            // 如果还配置了返回input，则将input也返回
            // 通常作为调试用
            if (!this.returnOnlyOutputs) {
                response = { ...inputs, ...response };
            }
            return response;
        };
        // 循环执行，直到得出结果
        while (this.shouldContinue(iterations)) {
            let output;
            try {
                // 调用模型获得结果
                output = await this.agent.plan(steps, inputs, runManager?.getChild());
            }
            catch (e) {
                ...
            }
            // 如果有returnValues，则直接返回
            if ("returnValues" in output) {
                return getOutput(output);
            }
            let actions;
            if (Array.isArray(output)) {
                actions = output;
            }
            else {
                actions = [output];
            }
            const newSteps = await Promise.all(actions.map(async (action) => {
                ...
                const tool = action.tool === "_Exception"
                    ? new ExceptionTool()
                    : toolsByName[action.tool?.toLowerCase()];
                let observation;
                try {
                    // 调用tool获得tool结果
                    observation = tool
                        ? await tool.call(action.toolInput, runManager?.getChild())
                        : `${action.tool} is not a valid tool, try another one.`;
                }
                catch (e) {
                    ...
                }
                return { action, observation: observation ?? "" };
            }));
            steps.push(...newSteps);
            const lastStep = steps[steps.length - 1];
            const lastTool = toolsByName[lastStep.action.tool?.toLowerCase()];
            if (lastTool?.returnDirect) {
                return getOutput({
                    returnValues: { [this.agent.returnValues[0]]: lastStep.observation },
                    log: "",
                });
            }
            iterations += 1;
        }
        const finish = await this.agent.returnStoppedResponse(this.earlyStoppingMethod, steps, inputs);
        return getOutput(finish);
    }
```

### runnable直接当agent
从上面看到，agent主要是调用了plan方法。因此agent还可以直接就是一个runable，只要我们将invoke封装成plan即可。

```typescript
// 在agent的构造函数中，有这个一个判断
if (Runnable.isRunnable(input.agent)) {
      agent = new RunnableAgent({ runnable: input.agent })
}

// RunnableAgent的定义
export declare class RunnableAgent extends BaseMultiActionAgent {
    protected lc_runnable: boolean;
    lc_namespace: string[];
    runnable: Runnable<ChainValues & {
        steps: AgentStep[];
    }, AgentAction[] | AgentAction | AgentFinish>;
    stop?: string[];
    get inputKeys(): string[];
    constructor(fields: RunnableAgentInput);
    plan(steps: AgentStep[], inputs: ChainValues, callbackManager?: CallbackManager): Promise<AgentAction[] | AgentFinish>;
}

// plan的实现
async plan(steps, inputs, callbackManager) {
        const invokeInput = { ...inputs, steps };
        const output = await this.runnable.invoke(invokeInput, {
            callbacks: callbackManager,
            runName: "RunnableAgent",
        });
        if (isAgentAction(output)) {
            return [output];
        }
        return output;
    }
```

### shouldContinue

在上面的代码中，可以看到每次循环都调用这个是否继续。这个实际上是用来判断有没有超过最大的迭代次数的。

```typescript
while (this.shouldContinue(iterations)) {
    ...
}

// 判断有没有超过最大的迭代次数
private shouldContinue(iterations: number): boolean {
    return this.maxIterations === undefined || iterations < this.maxIterations;
  }
```

在实际上使用的过程中，这个我们一般会设置一个值。对于简单任务，这个可以设置小一点，比如3。对于复杂的任务，则需要设置大一些。

最好都设置一下，因为有可能模型返回的结果一直有问题，这个会导致模型陷入死循环中，这样会浪费非常大的token。

# 总结

了解了agentExecutor的执行，实际上整个agent的执行逻辑就已经清晰了。

无非就是循环调用模型，然后通过模型返回的结果，判断是调用tool还是直接返回给用户。
