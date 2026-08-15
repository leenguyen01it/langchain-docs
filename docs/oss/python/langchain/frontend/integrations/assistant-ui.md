# assistant-ui

> Framework UI headless cho chat AI, tích hợp assistant-ui với useStream qua adapter useExternalStoreRuntime

[assistant-ui](https://www.assistant-ui.com/) là một framework UI React headless dành cho chat AI. Nó cung cấp một tầng runtime đầy đủ, gồm quản lý thread, phân nhánh message, xử lý attachment, kết nối với [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) qua adapter `useExternalStoreRuntime`.

!!! tip "Xem demo"
    Trang gốc có một demo tương tác trực tiếp cho pattern này. Xem tại: [https://docs.langchain.com/oss/python/langchain/frontend/integrations/assistant-ui](https://docs.langchain.com/oss/python/langchain/frontend/integrations/assistant-ui)

!!! tip "Mẹo"
    Clone và chạy [ví dụ assistant-ui đầy đủ](https://github.com/langchain-ai/langgraphjs/tree/main/examples/assistant-ui-claude) để xem một giao diện chat kiểu Claude được nối với agent LangChain bằng `useExternalStoreRuntime`.

## Cách hoạt động

1. **Stream với [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream)**: kết nối với agent của bạn và nhận message có tính phản ứng (reactive), loading state, và callback submit/cancel
2. **Adapt bằng `useExternalStoreRuntime`**: cầu nối `stream.messages` sang định dạng runtime của assistant-ui, bằng cách chuyển `BaseMessage[]` thành `ThreadMessageLike[]`
3. **Cung cấp runtime**: bọc UI của bạn trong `AssistantRuntimeProvider` và render bất kỳ component thread nào của assistant-ui

## Cài đặt

```bash
bun add @assistant-ui/react @assistant-ui/react-markdown
```

## Nối với `useStream`

Adapter `useExternalStoreRuntime` là cầu nối `stream.messages` vào runtime của assistant-ui. Truyền nó cho `AssistantRuntimeProvider` và render bất kỳ component thread nào:

```tsx
import { useCallback, useMemo } from "react";
import {
  AssistantRuntimeProvider,
  useExternalStoreRuntime,
  type AppendMessage,
  type ThreadMessageLike,
} from "@assistant-ui/react";
import { useStream } from "@langchain/react";
import { Thread } from "@assistant-ui/react";

export function Chat() {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "claude",
  });

  const onNew = useCallback(
    async (message: AppendMessage) => {
      const text = message.content
        .filter((c) => c.type === "text")
        .map((c) => c.text)
        .join("");
      await stream.submit({ messages: [{ type: "human", content: text }] });
    },
    [stream],
  );

  // Chuyển message của LangChain sang định dạng ThreadMessageLike của assistant-ui
  const messages = useMemo(
    () => toThreadMessages(stream.messages),
    [stream.messages],
  );

  const runtime = useExternalStoreRuntime<ThreadMessageLike>({
    messages,
    onNew,
    onCancel: () => stream.stop(),
    convertMessage: (m) => m,
  });

  return (
    <AssistantRuntimeProvider runtime={runtime}>
      <Thread />
    </AssistantRuntimeProvider>
  );
}
```

### Chuyển đổi message

`toThreadMessages` ánh xạ `BaseMessage[]` của LangChain sang định dạng `ThreadMessageLike[]` mà assistant-ui yêu cầu. Xử lý từng loại message (human, AI, và tool) và chuyển đổi content block, tool call, và reasoning token:

```tsx
import { AIMessage, HumanMessage, ToolMessage, type BaseMessage } from "langchain";
import type { ThreadMessageLike } from "@assistant-ui/react";

export function toThreadMessages(messages: BaseMessage[]): ThreadMessageLike[] {
  const result: ThreadMessageLike[] = [];

  for (const msg of messages) {
    if (HumanMessage.isInstance(msg)) {
      result.push({
        role: "user",
        content: [{ type: "text", text: msg.text }],
      });
    } else if (AIMessage.isInstance(msg)) {
      const parts: ThreadMessageLike["content"] = [];

      // Reasoning token
      const reasoning = msg.contentBlocks.find((block) => block.type === "reasoning")?.reasoning;
      if (reasoning) parts.push({ type: "reasoning", text: reasoning });

      // Tool call
      for (const tc of msg.tool_calls ?? []) {
        parts.push({
          type: "tool-call",
          toolCallId: tc.id ?? "",
          toolName: tc.name,
          args: tc.args,
        });
      }

      // Text response
      const text = msg.text;
      if (text) parts.push({ type: "text", text });

      result.push({ role: "assistant", content: parts });
    } else if (ToolMessage.isInstance(msg)) {
      // Gắn kết quả tool vào assistant message ngay trước đó
      const last = result[result.length - 1];
      if (last?.role === "assistant") {
        for (const part of last.content) {
          if (
            part.type === "tool-call" &&
            part.toolCallId === msg.tool_call_id
          ) {
            (part as { result?: string }).result = msg.text;
          }
        }
      }
    }
  }

  return result;
}
```

## Tuỳ chỉnh giao diện thread

`<Thread />` cung cấp sẵn một giao diện thread mặc định đầy đủ, gồm danh sách message, composer, và quản lý scroll. Tuỳ chỉnh từng phần bằng cách override các component slot:

```tsx
import { Thread, ThreadMessages, Composer } from "@assistant-ui/react";

function CustomThread() {
  return (
    <Thread.Root>
      <ThreadMessages
        components={{
          UserMessage: MyUserMessage,
          AssistantMessage: MyAssistantMessage,
          ToolFallback: MyToolCard,
        }}
      />
      <Composer />
    </Thread.Root>
  );
}
```

## Thực hành tốt nhất

* **Memo hoá việc chuyển đổi message:** bọc `toThreadMessages(stream.messages)` trong `useMemo` để tránh chạy lại việc chuyển đổi ở mỗi lần render.
* **Xử lý attachment:** dùng `CompositeAttachmentAdapter` với `SimpleImageAttachmentAdapter` cho việc upload ảnh; mở rộng bằng adapter tuỳ chỉnh cho file.
* **Dùng branching:** assistant-ui có sẵn hỗ trợ phân nhánh message qua `MessageBranch`; kết hợp việc sửa message với `useMessageMetadata` và `forkFrom` khi bạn cần fork checkpoint kiểu LangGraph.
* **Duy trì thread:** lưu `threadId` bằng `onThreadId` và truyền lại nó cho [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) khi tải trang, để assistant-ui kết nối lại đúng thread cũ.
