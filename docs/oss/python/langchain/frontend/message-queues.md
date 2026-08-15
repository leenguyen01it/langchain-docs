# Message queues

> Xếp hàng đợi (queue) nhiều message và quản lý chúng trong khi agent xử lý tuần tự

Message queuing cho phép người dùng gửi nhiều message liên tiếp mà không cần chờ agent xử lý xong message hiện tại. Mỗi message được chấp nhận ngay lập tức, xếp vào hàng đợi của thread đang hoạt động, và được xử lý tuần tự, giúp bạn có toàn quyền quan sát và kiểm soát công việc đang chờ.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/message-queues

!!! note "Ghi chú"
    Tính năng này yêu cầu [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server). Chạy agent của bạn cục bộ bằng `langgraph dev` hoặc [deploy lên LangSmith](https://docs.langchain.com/langsmith/deployment) để dùng pattern này.

## Vì sao cần message queue?

Trong một giao diện chat thông thường, người dùng phải chờ agent phản hồi xong mới gửi được message tiếp theo. Điều này gây khó chịu trong một số tình huống:

* **Câu hỏi hàng loạt**: người dùng muốn hỏi năm câu hỏi liên quan cùng lúc thay vì chờ từng câu trả lời
* **Chuỗi follow-up**: gửi thêm phần làm rõ hoặc bổ sung ngữ cảnh trong khi agent vẫn đang xử lý
* **Chuỗi test tự động**: gửi tự động một loạt prompt để kiểm tra hành vi agent
* **Quy trình nhập liệu**: đưa các input có cấu trúc lần lượt vào để xử lý

Message queuing giải quyết vấn đề này bằng cách chấp nhận mọi lượt gửi ngay lập tức và xử lý chúng theo thứ tự.

Đây là một primitive về UX của agent chứ không chỉ là một tính năng thẩm mỹ của chat. SDK theo dõi hàng đợi như một phần của stream controller, nên UI của bạn có thể hiển thị công việc đang chờ, huỷ các request đã cũ, và giữ ô soạn thảo hoạt động trong khi run hiện tại vẫn tiếp diễn.

## Cách hoạt động

Truyền `multitaskStrategy: "enqueue"` khi bạn muốn một lượt gửi chờ phía sau request đang chạy. Trong khi agent đang xử lý, các lượt gửi được xếp hàng sẽ được thêm vào hàng đợi của thread đang hoạt động. Khi run hiện tại hoàn tất, message tiếp theo trong hàng đợi sẽ tự động được gửi đi.

Đọc trạng thái hàng đợi bằng helper hàng đợi tương ứng cho framework của bạn:

| Thuộc tính | Kiểu | Mô tả |
| --- | --- | --- |
| `queue.entries` | `SubmissionQueueEntry[]` | Mảng tất cả các entry đang chờ trong hàng đợi |
| `queue.size` | `number` | Số lượng entry hiện có trong hàng đợi |
| `queue.cancel(id)` | `(id: string) => Promise<void>` | Huỷ một entry cụ thể trong hàng đợi theo ID |
| `queue.clear()` | `() => Promise<void>` | Huỷ toàn bộ entry trong hàng đợi |

Mỗi đối tượng [SubmissionQueueEntry](https://reference.langchain.com/javascript/langchain-react/SubmissionQueueEntry) chứa:

| Trường | Kiểu | Mô tả |
| --- | --- | --- |
| `id` | `string` | Định danh duy nhất cho entry này trong hàng đợi |
| `values` | `object` | Các giá trị input (bao gồm message) đã được gửi |
| `options` | `object` | Bất kỳ tuỳ chọn bổ sung nào được truyền kèm lượt gửi |
| `createdAt` | `string` | Timestamp ISO ghi lại thời điểm entry được tạo |

## Thiết lập `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với agent của bạn, sau đó ghép nó với helper submission queue cho framework của bạn. Gọi `stream.submit()` để gửi message trong khi một run đang chạy; truyền `multitaskStrategy: "enqueue"` cho các lượt gửi cần chờ phía sau request đang hoạt động. Đọc `queue.entries` và `queue.size` để render công việc đang chờ, và dùng `queue.cancel()` hoặc `queue.clear()` để loại bỏ mục trước khi chúng bắt đầu xử lý.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream, useSubmissionQueue } from "@langchain/react";

    function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "simple_agent",
      });
      const queue = useSubmissionQueue(stream);

      const handleSubmit = (text: string) => {
        stream.submit({
          messages: [{ type: "human", content: text }],
        });
      };

      const pendingCount = queue.size;
      const entries = queue.entries;

      return (
        <div>
          <MessageList messages={stream.messages} />
          {pendingCount > 0 && <QueueList entries={entries} queue={queue} />}
          <ChatInput onSubmit={handleSubmit} />
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream, useSubmissionQueue } from "@langchain/vue";
    import { computed } from "vue";

    const stream = useStream<typeof myAgent>({
      apiUrl: "http://localhost:2024",
      assistantId: "simple_agent",
    });
    const queue = useSubmissionQueue(stream);

    function handleSubmit(text: string) {
      stream.submit({
        messages: [{ type: "human", content: text }],
      });
    }

    const pendingCount = computed(() => queue.size.value);
    const entries = computed(() => queue.entries.value);
    </script>

    <template>
      <div>
        <MessageList :messages="stream.messages" />
        <QueueList v-if="pendingCount > 0" :entries="entries" :queue="queue" />
        <ChatInput @submit="handleSubmit" />
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream, useSubmissionQueue } from "@langchain/svelte";

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "simple_agent",
      });
      const queue = useSubmissionQueue(stream);

      function handleSubmit(text: string) {
        stream.submit({
          messages: [{ type: "human", content: text }],
        });
      }
    </script>

    <div>
      <MessageList messages={stream.messages} />
      {#if queue.size > 0}
        <QueueList entries={queue.entries} {queue} />
      {/if}
      <ChatInput on:submit={(e) => handleSubmit(e.detail)} />
    </div>
    ```

=== "Angular"
    ```ts
    import { Component } from "@angular/core";
    import { injectStream, injectSubmissionQueue } from "@langchain/angular";

    @Component({
      selector: "app-chat",
      template: `
        <message-list [messages]="stream.messages()" />
        @if (queue.size() > 0) {
          <queue-list [entries]="queue.entries()" [queue]="queue" />
        }
        <chat-input (onSubmit)="handleSubmit($event)" />
      `,
    })
    export class ChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "simple_agent",
      });
      queue = injectSubmissionQueue(this.stream);

      handleSubmit(text: string) {
        this.stream.submit({
          messages: [{ type: "human", content: text }],
        });
      }
    }
    ```

## Hiển thị hàng đợi

Xây dựng một component `QueueList` hiển thị mỗi message đang chờ kèm nút huỷ. Điều này giúp người dùng thấy được những gì đang chờ và có thể loại bỏ những mục họ không còn cần nữa.

```tsx
function QueueList({ entries, queue }) {
  return (
    <div className="queue-panel">
      <div className="queue-header">
        <span>Queued messages ({entries.length})</span>
        <button onClick={() => queue.clear()}>Clear all</button>
      </div>
      <ul className="queue-entries">
        {entries.map((entry) => {
          const text = entry.values?.messages?.at(-1)?.content ?? "Pending...";
          return (
            <li key={entry.id} className="queue-entry">
              <span className="queue-text">{text}</span>
              <span className="queue-time">
                {new Date(entry.createdAt).toLocaleTimeString()}
              </span>
              <button
                className="queue-cancel"
                onClick={() => queue.cancel(entry.id)}
              >
                Cancel
              </button>
            </li>
          );
        })}
      </ul>
    </div>
  );
}
```

!!! tip "Mẹo"
    Hiển thị vài ký tự đầu của mỗi message đang chờ như một bản xem trước để người dùng nhanh chóng nhận ra mục nào cần huỷ mà không phải đọc toàn bộ message.

## Huỷ message đang chờ

Bạn có hai mức huỷ:

### Huỷ một entry đơn lẻ

Loại bỏ một message cụ thể khỏi hàng đợi theo ID. Agent sẽ bỏ qua nó và chuyển sang entry tiếp theo.

```ts
await queue.cancel(entryId);
```

### Xoá toàn bộ hàng đợi

Loại bỏ tất cả message đang chờ cùng lúc. Hữu ích khi người dùng đổi ngữ cảnh hoặc muốn bắt đầu lại.

```ts
await queue.clear();
```

!!! note "Ghi chú"
    Huỷ một entry trong hàng đợi chỉ ảnh hưởng đến các message **chưa bắt đầu xử lý**. Nếu agent đã đang xử lý một message, việc huỷ nó khỏi hàng đợi sẽ không có tác dụng. Dùng `stream.stop()` để ngắt run hiện tại.

## Nối chuỗi các lượt gửi follow-up bằng `onCreated`

Callback `onCreated` được gọi khi một run mới được tạo, cho bạn một hook để gửi message follow-up bằng code. Điều này hữu ích khi xây dựng các workflow nhiều bước, nơi câu hỏi tiếp theo phụ thuộc vào việc lượt gửi trước đó đã được chấp nhận.

```ts
stream.submit(
  { messages: [{ type: "human", content: "What is quantum computing?" }] },
  {
    onCreated(run) {
      console.log("Run created:", run.runId);
      // Nối thêm một follow-up
      stream.submit({
        messages: [{ type: "human", content: "Give me a simple analogy." }],
      });
    },
  }
);
```

Pattern này tự nhiên làm đầy hàng đợi. Message đầu tiên bắt đầu xử lý ngay lập tức, và follow-up được xếp hàng ngay sau nó.

## Bắt đầu một thread mới

Khi người dùng muốn bắt đầu một cuộc trò chuyện mới, cập nhật giá trị `threadId` reactive mà bạn truyền vào luồng. Truyền `null` sẽ xoá liên kết thread hiện tại; lượt gửi tiếp theo sẽ tạo một thread mới.

=== "React"
    ```tsx
    function NewThreadButton() {
      const [threadId, setThreadId] = useState<string | null>(null);
      const stream = useStream<typeof myAgent>({ threadId, onThreadId: setThreadId });

      return (
        <button onClick={() => setThreadId(null)}>
          New conversation
        </button>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    const threadId = ref<string | null>(null);
    const stream = useStream<typeof myAgent>({
      threadId,
      onThreadId: (id) => (threadId.value = id),
    });
    </script>

    <template>
      <button @click="threadId = null">New conversation</button>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      let threadId = $state<string | null>(null);
      const stream = useStream<typeof myAgent>({
        threadId: () => threadId,
        onThreadId: (id) => (threadId = id),
      });
    </script>

    <button onclick={() => (threadId = null)}>New conversation</button>
    ```

=== "Angular"
    ```ts
    threadId = signal<string | null>(null);
    stream = injectStream<typeof myAgent>({
      threadId: this.threadId,
      onThreadId: (id) => this.threadId.set(id),
    });

    // Trong template:
    // <button (click)="threadId.set(null)">New conversation</button>
    ```

## Thực hành tốt nhất

* **Giới hạn kích thước hàng đợi**: dù không có giới hạn cứng ở phía client cho kích thước hàng đợi, hãy lưu ý rằng hàng đợi quá lớn có thể làm giảm trải nghiệm người dùng. Cân nhắc hiển thị cảnh báo khi hàng đợi vượt một ngưỡng hợp lý (ví dụ 10 mục).
* **Hiển thị vị trí trong hàng đợi**: đánh số mỗi mục đang chờ để người dùng biết thứ tự xử lý.
* **Giữ focus ô input**: giữ ô input ở trạng thái focus sau khi gửi để người dùng có thể gõ message tiếp theo ngay lập tức.
* **Hoạt hoạ (animate) khi chuyển trạng thái**: di chuyển mượt các mục từ panel hàng đợi sang danh sách message khi chúng bắt đầu được xử lý.
* **Xử lý lỗi một cách nhẹ nhàng**: nếu một message trong hàng đợi bị lỗi, hiển thị lỗi mà không chặn các entry tiếp theo trong hàng đợi.
* **Debounce các lượt gửi liên tiếp nhanh**: với các lượt gửi tự động hoặc bằng code, thêm một khoảng trễ nhỏ giữa các message để tránh làm quá tải server.
