# AI Elements

> Các component composable dựa trên shadcn/ui cho giao diện chat AI, dùng cùng useStream

[AI Elements](https://elements.ai-sdk.dev/) là một thư viện component composable, dựa trên shadcn/ui, được xây dựng chuyên biệt cho giao diện chat AI. Các component như `Conversation`, `Message`, `Tool`, `Reasoning`, và `PromptInput` được thiết kế để nhúng thẳng vào bất kỳ dự án React nào và nối trực tiếp với `stream.messages` với rất ít code kết nối (glue code).

!!! tip "Xem demo"
    Trang gốc có một demo tương tác trực tiếp cho pattern này. Xem tại: [https://docs.langchain.com/oss/python/langchain/frontend/integrations/ai-elements](https://docs.langchain.com/oss/python/langchain/frontend/integrations/ai-elements)

!!! tip "Mẹo"
    Clone và chạy [ví dụ AI Elements đầy đủ](https://github.com/langchain-ai/langgraphjs/tree/main/examples/ai-elements) để xem cách render tool call, hiển thị reasoning, streaming message, và nhiều hơn nữa trong một dự án hoạt động thực tế.

## Cách hoạt động

1. **Cài component dưới dạng source file:** AI Elements phân phối qua một CLI, thêm component trực tiếp vào dự án của bạn (theo kiểu registry của shadcn/ui)
2. **Ánh xạ message sang component:** iterate qua `stream.messages`, render các instance `HumanMessage` thành bong bóng chat của user và các instance `AIMessage` thành response của assistant
3. **Soạn các UI phong phú hơn:** bọc tool call trong `<Tool>`, reasoning trong `<Reasoning>`, và mọi thứ trong `<Conversation>` để quản lý việc cuộn (scroll)

## Cài đặt

Cài các component AI Elements qua CLI. Chúng được thêm vào dưới dạng source file có thể chỉnh sửa trong dự án của bạn:

```bash
npm install @langchain/react
npx ai-elements@latest add conversation message prompt-input tool reasoning suggestion
```

## Nối với useStream

Render các component AI Elements trực tiếp từ `stream.messages`. Mỗi `BaseMessage` của LangChain ánh xạ sang một component:

```tsx
import { useStream } from "@langchain/react";
import { HumanMessage, AIMessage } from "langchain";

import {
  Conversation,
  ConversationContent,
  ConversationScrollButton,
} from "@/components/ai-elements/conversation";
import {
  Message,
  MessageContent,
  MessageResponse,
} from "@/components/ai-elements/message";
import {
  Tool,
  ToolHeader,
  ToolContent,
  ToolInput,
  ToolOutput,
} from "@/components/ai-elements/tool";
import {
  Reasoning,
  ReasoningTrigger,
  ReasoningContent,
} from "@/components/ai-elements/reasoning";
import {
  PromptInput,
  PromptInputBody,
  PromptInputTextarea,
  PromptInputFooter,
  PromptInputSubmit,
} from "@/components/ai-elements/prompt-input";

function getReasoningText(msg: AIMessage) {
  return msg.contentBlocks.find((block) => block.type === "reasoning")?.reasoning ?? "";
}

function getTextContent(msg: AIMessage) {
  return msg.text;
}

function getToolCalls(msg: AIMessage) {
  return (msg.tool_calls ?? []).map((tc) => ({
    id: tc.id,
    name: tc.name,
    args: tc.args,
    state: "input-available" as const,
  }));
}

export function Chat() {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "ai_elements",
  });

  return (
    <div className="flex flex-col h-dvh">
      <Conversation className="flex-1">
        <ConversationContent>
          {stream.messages.map((msg, i) => {
            if (HumanMessage.isInstance(msg)) {
              return (
                <Message key={i} from="user">
                  <MessageContent>{msg.text}</MessageContent>
                </Message>
              );
            }
            if (AIMessage.isInstance(msg)) {
              return (
                <div key={i}>
                  {/* Khối reasoning (hiện khi model phát ra thinking token) */}
                  <Reasoning>
                    <ReasoningTrigger />
                    <ReasoningContent>{getReasoningText(msg)}</ReasoningContent>
                  </Reasoning>

                  {/* Tool call hiển thị inline kèm input/output */}
                  {getToolCalls(msg).map((tc) => (
                    <Tool key={tc.id} defaultOpen>
                      <ToolHeader type={`tool-${tc.name}`} state={tc.state} />
                      <ToolContent>
                        <ToolInput input={tc.args} />
                        {tc.output && (
                          <ToolOutput output={tc.output} errorText={undefined} />
                        )}
                      </ToolContent>
                    </Tool>
                  ))}

                  {/* Response text đang stream */}
                  <Message from="assistant">
                    <MessageContent>
                      <MessageResponse>{getTextContent(msg)}</MessageResponse>
                    </MessageContent>
                  </Message>
                </div>
              );
            }
          })}
        </ConversationContent>
        <ConversationScrollButton />
      </Conversation>

      <PromptInput
        onSubmit={({ text }) =>
          stream.submit({ messages: [{ type: "human", content: text }] })
        }
      >
        <PromptInputBody>
          <PromptInputTextarea placeholder="Ask me something..." />
        </PromptInputBody>
        <PromptInputFooter>
          <PromptInputSubmit
            status={stream.isLoading ? "streaming" : "ready"}
          />
        </PromptInputFooter>
      </PromptInput>
    </div>
  );
}
```

## Thực hành tốt nhất

* **Tự do sửa source file:** component nằm ngay trong dự án của bạn, không phải một package phụ thuộc bên ngoài, nên bạn có thể thay đổi bất cứ thứ gì mà không cần fork.
* **Dùng `MessageResponse` cho streaming:** nó xử lý đúng các token từng phần đang stream; tránh render trực tiếp nội dung message thô trong lúc streaming.
* **Bọc trong `Conversation`:** component `Conversation` quản lý hành vi scroll để message mới tự động cuộn vào tầm nhìn.
* **Gate bằng `isInstance`:** dùng `HumanMessage.isInstance(msg)` và `AIMessage.isInstance(msg)` thay vì kiểm tra `msg.getType()`, để TypeScript narrow kiểu đúng cách.
