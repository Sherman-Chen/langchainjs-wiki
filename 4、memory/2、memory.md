# memory
在langchain中，chatHistory用于如何存放历史消息，比如说保存在内存中，则是用buffer，保存在redis中，则是用redis。你可以把chatHistory理解成store。

而memory则是定义了如何返回对话记录，毕竟大模型的输入长度是有限制的，从数据库拿到对话记录之后，我们还需要确定如何返回这里历史记录。langchain内部封装了几种返回的策略。

* BufferMemory：这个最简单了，就是将chatHistory原封不动地返回，通常用于测试
* ConversationBufferWindowMemory：返回倒数K条，这个比较常用
* Entity memory：会调用模型将历史记录按照实体进行总结
* Combined memory：可以组合其他的memory策略
* summary memory：会调用模型将历史记录总结




# 历史消息合并的时机
之前在model章节，已经得知大模型本身是没有记忆的，需要我们自己的prompt中控制。

那么在langchain中，历史消息一定会在发送大模型请求之前合并到prompt中的。

具体的，是在BaseChain的invoke中被合并到prompt中的。

```typescript
async invoke(input: RunInput, config?: RunnableConfig): Promise<RunOutput> {
    // 在这个里面，会将memory合并到prompt
    const fullValues = await this._formatValues(input);
    ...
    this._call(fullValues as RunInput, runManager, config));
    
    // 这里是保存大模型返回的结果到历史记录中
    if (!(this.memory == null)) {
      await this.memory.saveContext(
        this._selectMemoryInputs(input),
        outputValues
      );
    }
    ...
  }
```

再来看下_formatValues：

```typescript
protected async _formatValues(
    values: ChainValues & { signal?: AbortSignal; timeout?: number }
  ) {
    const fullValues = { ...values } as typeof values;
    ...
    if (!(this.memory == null)) {
      // 在这里读取内存
      const newValues = await this.memory.loadMemoryVariables(
        this._selectMemoryInputs(values)
      );
      // 这里合并到prompt中
      for (const [key, value] of Object.entries(newValues)) {
        fullValues[key] = value;
      }
    }
    return fullValues;
  }
```

看下buffer的loadMemoryVariables：

```typescript
async loadMemoryVariables(_values: InputValues): Promise<MemoryVariables> {
    // 获取历史记录
    const messages = await this.chatHistory.getMessages();
    // 如果是messages类型的，则直接返回{memoryKey:历史消息}
    if (this.returnMessages) {
      const result = {
        [this.memoryKey]: messages,
      };
      return result;
    }
    const result = {
      [this.memoryKey]: getBufferString(
        messages,
        this.humanPrefix,
        this.aiPrefix
      ),
    };
    return result;
  }
```

这样就能够读取到历史记录的值了。通常在使用历史记录的时候，还需要用到一个messagePlaceholder。然后将占位符替换为该值即可。

替换的过程是在prompt中进行的，比如说ChatPromptTemplate：

链条是：
```typescript
LLMChain._call
prompt.formatPromptValue(valuesForPrompt)

// BaseChatPromptTemplate
async formatPromptValue(
    values: TypedPromptInputValues<RunInput>
  ): Promise<ChatPromptValueInterface> {
    // formatMessages是一个抽象方法
    const resultMessages = await this.formatMessages(values);
    return new ChatPromptValue(resultMessages);
  }

// 比如说ChatPromptTemplate
async formatMessages(
    values: TypedPromptInputValues<RunInput>
  ): Promise<BaseMessage[]> {
    const allValues = await this.mergePartialAndUserVariables(values);
    let resultMessages: BaseMessage[] = [];
    // 这里遍历所有的prompt定义
    for (const promptMessage of this.promptMessages) {
      // 是baseMessage的话，则直接添加
      if (promptMessage instanceof BaseMessage) {
        resultMessages.push(
          await this._parseImagePrompts(promptMessage, allValues)
        );
      } else {
        const inputValues = promptMessage.inputVariables.reduce(
          (acc, inputVariable) => {
            if (
              !(inputVariable in allValues) &&
              !(isMessagesPlaceholder(promptMessage) && promptMessage.optional)
            ) {
              throw new Error(
                `Missing value for input variable \`${inputVariable.toString()}\``
              );
            }
            acc[inputVariable] = allValues[inputVariable];
            return acc;
          },
          {} as InputValues
        );
        // 这里调用各个prompt的formatMessages
        // MessagePlaceHolder就是在这里被调用添加到最终的prompt中
        const message = await promptMessage.formatMessages(inputValues);
        resultMessages = resultMessages.concat(message);
      }
    }
    return resultMessages;
  }
```


有了结果之后，还需要保存：

```typescript
// 以BufferMemory为例
// 这个是在libs\langchain-community\src\memory\chat_memory.ts
async saveContext(
    inputValues: InputValues,
    outputValues: OutputValues
  ): Promise<void> {
    // this is purposefully done in sequence so they're saved in order
    await this.chatHistory.addUserMessage(
      getInputValue(inputValues, this.inputKey)
    );
    await this.chatHistory.addAIChatMessage(
      getOutputValue(outputValues, this.outputKey)
    );
  }
```
