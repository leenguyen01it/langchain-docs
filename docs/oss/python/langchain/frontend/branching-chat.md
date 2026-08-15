# Chat phân nhánh (Branching chat)

> Chỉnh sửa tin nhắn và tạo lại phản hồi bằng cách fork (phân nhánh) từ các checkpoint

Các cuộc hội thoại với agent AI hiếm khi diễn ra tuyến tính. Bạn có thể muốn diễn đạt lại một câu hỏi, tạo lại một phản hồi mà bạn không thích, hoặc khám phá một nhánh hội thoại khác mà không mất lịch sử checkpoint. Branching chat sử dụng các checkpoint của LangGraph làm điểm fork: mỗi lần chỉnh sửa hoặc tạo lại sẽ gửi một run mới bắt đầu từ checkpoint cha của tin nhắn được chọn.

!!! note "Ghi chú"
    Tính năng này yêu cầu [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server). Chạy agent của bạn cục bộ bằng `langgraph dev` hoặc [triển khai nó lên LangSmith](https://docs.langchain.com/langsmith/deployment) để sử dụng pattern này.

## Branching chat là gì?

Branching chat coi một cuộc hội thoại như một dòng thời gian có checkpoint thay vì một danh sách phẳng. Mỗi tin nhắn có metadata trỏ đến checkpoint trước khi tin nhắn đó được tạo ra. Việc chỉnh sửa một tin nhắn hoặc tạo lại một phản hồi sẽ gửi một run mới bắt đầu từ checkpoint đó.

Các khả năng chính:

* **Chỉnh sửa bất kỳ tin nhắn người dùng nào:** viết lại một prompt trước đó và chạy lại agent từ điểm đó
* **Tạo lại bất kỳ phản hồi AI nào:** yêu cầu agent tạo ra một câu trả lời khác cho cùng một đầu vào
* **Kiểm tra lịch sử:** dùng LangGraph client để tải các checkpoint khi bạn cần dòng thời gian của một nhánh

## Thiết lập metadata cho stream

Dùng root stream cho các tin nhắn, sau đó đọc metadata checkpoint theo từng tin nhắn trong component render tin nhắn đó. Metadata bao gồm ID checkpoint cha để fork từ đó.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để suy luận kiểu (type-safe) cho trạng thái stream. Xem Type inference cho backend [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) hoặc [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference).

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";

    const AGENT_URL = "http://localhost:2024";

    export function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "simple_agent",
      });

      return (
        <div>
          {stream.messages.map((msg) => (
            <MessageWithForkControls key={msg.id} stream={stream} message={msg} />
          ))}
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";

    const AGENT_URL = "http://localhost:2024";

    const stream = useStream<typeof myAgent>({
      apiUrl: AGENT_URL,
      assistantId: "simple_agent",
    });
    </script>

    <template>
      <div>
        <MessageWithForkControls
          v-for="msg in stream.messages.value"
          :key="msg.id"
          :stream="stream"
          :message="msg"
        />
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";

      const AGENT_URL = "http://localhost:2024";

      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "simple_agent",
      });
    </script>

    <div>
      {#each stream.messages as msg (msg.id)}
        <Message
          message={msg}
          {stream}
        />
      {/each}
    </div>
    ```

=== "Angular"
    ```ts
    import { Component } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-chat",
      template: `
        @for (msg of stream.messages(); track msg.id) {
          <app-message
            [message]="msg"
            [stream]="stream"
          />
        }
      `,
    })
    export class ChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "simple_agent",
      });
    }
    ```

## Hiểu về metadata của tin nhắn

Helper `useMessageMetadata(stream, messageId)` trả về [MessageMetadata](https://reference.langchain.com/javascript/langchain-react/MessageMetadata)
cho một tin nhắn. Dùng nó trong component render mỗi tin nhắn để metadata luôn gắn đúng với ID tin nhắn đó:

```tsx
import type { BaseMessage } from "langchain";
import { useState } from "react";
import { useMessageMetadata, useStream } from "@langchain/react";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "simple_agent",
  });

  return stream.messages.map((message) => (
    <MessageWithForkControls
      key={message.id}
      stream={stream}
      message={message}
    />
  ));
}

function MessageWithForkControls({
  stream,
  message,
}: {
  stream: ReturnType<typeof useStream>;
  message: BaseMessage;
}) {
  const metadata = useMessageMetadata(stream, message.id);
  const checkpointId = metadata?.parentCheckpointId;
  const [editedText, setEditedText] = useState(message.text);

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault();
        if (!checkpointId) return;

        stream.submit(
          { messages: [{ type: "human", content: editedText }] },
          { forkFrom: { checkpointId } }
        );
      }}
    >
      <textarea
        value={editedText}
        onChange={(event) => setEditedText(event.target.value)}
      />
      <button disabled={!checkpointId || editedText === message.text}>
        Submit edited branch
      </button>
    </form>
  );
}
```

`parentCheckpointId` là checkpoint ngay trước tin nhắn đó. Dùng nó làm điểm fork cho việc chỉnh sửa và tạo lại.

## Chỉnh sửa một tin nhắn

Để chỉnh sửa một tin nhắn người dùng và fork cuộc hội thoại:

1. Lấy `parentCheckpointId` từ metadata của tin nhắn
2. Gửi tin nhắn đã chỉnh sửa với `forkFrom: { checkpointId }`
3. Agent chạy lại từ điểm đó

```ts
function handleEdit(
  stream: ReturnType<typeof useStream>,
  originalMsg: HumanMessage,
  metadata: MessageMetadata | undefined,
  newText: string
) {
  if (!metadata?.parentCheckpointId) return;

  stream.submit(
    {
      messages: [{ type: "human", content: newText }],
    },
    { forkFrom: { checkpointId: metadata.parentCheckpointId } }
  );
}
```

Sau khi chỉnh sửa:

* Agent chạy lại từ điểm fork với tin nhắn đã cập nhật
* Nhánh gốc vẫn còn khả dụng trong lịch sử thread

## Tạo lại một phản hồi

Để tạo lại một phản hồi AI mà không thay đổi đầu vào:

1. Lấy `parent_checkpoint` từ metadata của tin nhắn AI
2. Gửi với đầu vào rỗng và `forkFrom: { checkpointId }`
3. Agent tạo ra một phản hồi mới từ điểm đó

```ts
function handleRegenerate(
  stream: ReturnType<typeof useStream>,
  metadata: MessageMetadata | undefined
) {
  if (!metadata?.parentCheckpointId) return;

  stream.submit(undefined, {
    forkFrom: { checkpointId: metadata.parentCheckpointId },
  });
}
```

Mỗi lần tạo lại sẽ tạo ra một nhánh mới cho tin nhắn AI tại vị trí đó.

!!! tip "Mẹo"
    Tạo lại phản hồi rất hữu ích cho các agent không xác định (non-deterministic). Vì đầu ra của LLM thay đổi theo temperature, việc tạo lại cùng một prompt thường cho ra các phản hồi khác biệt đáng kể.

## Cách hoạt động của branching phía dưới nền

LangGraph lưu lại mọi chuyển đổi trạng thái dưới dạng một **checkpoint**. Khi bạn gửi với `forkFrom`, backend sẽ bắt đầu một đường thực thi mới từ điểm đó thay vì nối tiếp vào cuộc hội thoại hiện tại. Kết quả là một cấu trúc cây:

```
User: "What is React?"
  └─ AI: "React is a JavaScript library..." (branch A)
  └─ AI: "React is a UI framework..." (branch B, regenerated)

User: "Tell me about hooks" (branch A)
  └─ AI: "Hooks are functions..."

User: "Tell me about JSX" (edited from branch A)
  └─ AI: "JSX is a syntax extension..."
```

Mỗi nhánh được lưu lại trong bộ lưu trữ checkpoint. Dùng `stream.client.threads.getHistory(threadId)` khi bạn cần dựng một view dòng thời gian riêng biệt trên các checkpoint.

## Thực hành tốt nhất

* **Đọc metadata gần tin nhắn**: gọi `useMessageMetadata` trong component render các control fork của tin nhắn.
* **Hiển thị control fork khi hover**: các nút edit và regenerate nên xuất hiện khi hover để giữ giao diện gọn gàng.
* **Làm mới lịch sử theo yêu cầu**: chỉ gọi `client.threads.getHistory()` khi render dòng thời gian hoặc sau khi một lần fork đã ổn định.
* **Vô hiệu hóa control khi đang streaming**: không cho phép chỉnh sửa hoặc tạo lại trong khi agent đang stream phản hồi. Kiểm tra `stream.isLoading` trước khi bật các hành động này.
* **Giữ lại nội dung chỉnh sửa khi hủy**: nếu người dùng bắt đầu chỉnh sửa rồi hủy, đặt lại textarea về nội dung tin nhắn gốc.
* **Kiểm thử với cây checkpoint sâu**: người dùng chỉnh sửa và tạo lại thường xuyên có thể tạo ra nhiều nhánh. Đảm bảo việc render dòng thời gian vẫn đạt hiệu năng tốt.
