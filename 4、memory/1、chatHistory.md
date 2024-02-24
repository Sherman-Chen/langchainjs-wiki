# 前言
模型本身并没有参数提供记忆能力的，因此需要我们自己维护用户与模型的对话记录。

对于LLM来说，对话记录会添加到prompt中。对于chat model来说，对话记录会放到messages中。

# chat history
在langchain中，提供了一个类ChatMessageHistory，该类是其他memory类的基础类，用于存放历史的消息。

这个类比较简单，就是定义了一个messages的属性，添加到添加多一个元素，获取就直接返回。

```typescript
export class ChatMessageHistory extends BaseListChatMessageHistory {
  lc_namespace = ["langchain", "stores", "message", "in_memory"];

  private messages: BaseMessage[] = [];

  constructor(messages?: BaseMessage[]) {
    super(...arguments);
    this.messages = messages ?? [];
  }

  async getMessages(): Promise<BaseMessage[]> {
    return this.messages;
  }

  async addMessage(message: BaseMessage) {
    this.messages.push(message);
  }

  async clear() {
    this.messages = [];
  }
}
```

