# Time travel

> Kiểm tra, điều hướng, và tiếp tục từ bất kỳ checkpoint nào trong lịch sử hội thoại

Mỗi lần thay đổi state trong một agent LangGraph tạo ra một **checkpoint**, một snapshot đầy đủ về state của agent tại thời điểm đó. Time travel cho phép bạn kiểm tra bất kỳ checkpoint nào, xem đúng state mà agent đã giữ, và **tiếp tục thực thi từ điểm đó** để khám phá các nhánh khác. Nó vừa là một debugger, vừa là nút undo, vừa là một audit log, tất cả trong một.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/time-travel

!!! note "Ghi chú"
    Tính năng này yêu cầu [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server). Chạy agent của bạn cục bộ bằng `langgraph dev` hoặc [deploy lên LangSmith](https://docs.langchain.com/langsmith/deployment) để dùng pattern này.

## Checkpoint hoạt động như thế nào

LangGraph lưu (persist) state của agent sau mỗi lần một node thực thi. Mỗi state đã lưu là một object [ThreadState](https://reference.langchain.com/javascript/langchain-langgraph-sdk/index/ThreadState) ghi lại:

* **checkpoint**: metadata định danh snapshot cụ thể này (ID, timestamp)
* **values**: toàn bộ state của agent tại thời điểm này (message, các key tuỳ chỉnh)
* **tasks**: các graph node đã được lên lịch chạy tiếp theo
* **next**: tên các node sắp tới trong kế hoạch thực thi

Điều này tạo ra một dòng thời gian tuyến tính của mọi quyết định agent đã đưa ra, mọi tool nó đã gọi, và mọi phản hồi nó đã tạo ra. UI của bạn có thể render dòng thời gian này và cho phép người dùng nhảy tới bất kỳ điểm nào.

## Thiết lập `useStream`

Tạo stream cho agent của bạn, sau đó fetch lịch sử checkpoint một cách tường minh từ LangGraph client cho thread đang hoạt động. Tiếp tục từ một checkpoint dùng `forkFrom: { checkpointId }`.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";
    import { useEffect, useState } from "react";

    const AGENT_URL = "http://localhost:2024";

    export function TimeTravelChat() {
      const [threadId, setThreadId] = useState<string | null>(null);
      const [history, setHistory] = useState<ThreadState[]>([]);
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "time_travel",
        threadId,
        onThreadId: setThreadId,
      });

      useEffect(() => {
        if (!threadId || stream.isLoading) return;
        stream.client.threads.getHistory(threadId).then(setHistory);
      }, [stream.client, threadId, stream.isLoading]);

      function resumeFrom(cp: ThreadState) {
        stream.submit({}, {
          forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
        });
      }

      return (
        <div className="flex h-screen">
          <ChatPanel messages={stream.messages} />
          <TimelineSidebar history={history} onSelect={resumeFrom} />
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";
    import { ref, watch } from "vue";

    const AGENT_URL = "http://localhost:2024";
    const threadId = ref<string | null>(null);
    const history = ref<ThreadState[]>([]);

    const stream = useStream<typeof myAgent>({
      apiUrl: AGENT_URL,
      assistantId: "time_travel",
      threadId,
      onThreadId: (id) => (threadId.value = id),
    });

    watch(
      [threadId, stream.isLoading],
      async ([id, isLoading]) => {
        if (isLoading) return;
        history.value = id
          ? ((await stream.client.threads.getHistory(id)) as ThreadState[])
          : [];
      },
      { immediate: true },
    );

    function resumeFrom(cp: ThreadState) {
      stream.submit({}, {
        forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
      });
    }
    </script>

    <template>
      <div class="flex h-screen">
        <ChatPanel :messages="stream.messages.value" />
        <TimelineSidebar :history="history" @select="resumeFrom" />
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";

      const AGENT_URL = "http://localhost:2024";
      let threadId = $state<string | null>(null);
      let history = $state<ThreadState[]>([]);

      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "time_travel",
        threadId: () => threadId,
        onThreadId: (id) => (threadId = id),
      });

      $effect(() => {
        if (!threadId) {
          history = [];
          return;
        }
        if (stream.isLoading) return;
        stream.client.threads.getHistory(threadId).then((states) => {
          history = states as ThreadState[];
        });
      });

      function resumeFrom(cp: ThreadState) {
        stream.submit({}, {
          forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
        });
      }
    </script>

    <div class="flex h-screen">
      <ChatPanel messages={stream.messages} />
      <TimelineSidebar {history} onSelect={resumeFrom} />
    </div>
    ```

=== "Angular"
    ```ts
    import { Component, effect, signal } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-time-travel-chat",
      template: `
        <div class="flex h-screen">
          <app-chat-panel [messages]="stream.messages()" />
          <app-timeline-sidebar
            [history]="history()"
            (select)="resumeFrom($event)"
          />
        </div>
      `,
    })
    export class TimeTravelChatComponent {
      threadId = signal<string | null>(null);
      history = signal<ThreadState[]>([]);

      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "time_travel",
        threadId: this.threadId,
        onThreadId: (id) => this.threadId.set(id),
      });

      constructor() {
        effect(() => {
          if (this.stream.isLoading()) return;
          void this.refreshHistory(this.threadId());
        });
      }

      async refreshHistory(id: string | null) {
        this.history.set(id
          ? ((await this.stream.client.threads.getHistory(id)) as ThreadState[])
          : []);
      }

      resumeFrom(cp: ThreadState) {
        this.stream.submit({}, {
          forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
        });
      }
    }
    ```

## Xây dựng dòng thời gian checkpoint

Sidebar dòng thời gian hiển thị mỗi checkpoint như một mục có thể click. Mỗi mục hiển thị node đã chạy và số message đã tồn tại tại thời điểm đó:

```tsx
function TimelineSidebar({
  history,
  onSelect,
}: {
  history: ThreadState[];
  onSelect: (cp: ThreadState) => void;
}) {
  return (
    <aside className="w-80 overflow-y-auto border-l bg-gray-50 p-4">
      <h2 className="mb-4 text-sm font-semibold uppercase text-gray-500">
        Checkpoint Timeline
      </h2>
      <div className="space-y-2">
        {history.map((cp, i) => {
          const taskName = cp.tasks?.[0]?.name ?? "unknown";
          const msgCount = (cp.values?.messages as unknown[])?.length ?? 0;

          return (
            <button
              key={cp.checkpoint.checkpoint_id}
              onClick={() => onSelect(cp)}
              className="w-full rounded-lg border bg-white p-3 text-left
                         hover:border-blue-400 hover:shadow-sm transition-all"
            >
              <div className="flex items-center justify-between">
                <span className="text-xs text-gray-400">#{i + 1}</span>
                <NodeBadge name={taskName} />
              </div>
              <p className="mt-1 text-sm font-medium">{taskName}</p>
              <p className="text-xs text-gray-500">
                {msgCount} message{msgCount !== 1 ? "s" : ""}
              </p>
            </button>
          );
        })}
      </div>
    </aside>
  );
}
```

## Kiểm tra state của checkpoint

Click vào một checkpoint nên hiển thị toàn bộ state tại thời điểm đó. Một JSON viewer cho developer khả năng quan sát đầy đủ về những gì agent đã biết và đã quyết định:

```tsx
function CheckpointInspector({ checkpoint }: { checkpoint: ThreadState }) {
  const [expanded, setExpanded] = useState(false);

  return (
    <div className="rounded-lg border bg-white p-4">
      <div className="flex items-center justify-between">
        <h3 className="font-semibold">
          Checkpoint {checkpoint.checkpoint.checkpoint_id.slice(0, 8)}...
        </h3>
        <button
          onClick={() => setExpanded(!expanded)}
          className="text-sm text-blue-600 hover:underline"
        >
          {expanded ? "Collapse" : "Expand"} state
        </button>
      </div>

      <div className="mt-2 space-y-1 text-sm">
        <p>
          <strong>Node:</strong>{" "}
          {checkpoint.tasks?.[0]?.name ?? "—"}
        </p>
        <p>
          <strong>Next:</strong>{" "}
          {checkpoint.next?.join(", ") || "—"}
        </p>
        <p>
          <strong>Messages:</strong>{" "}
          {(checkpoint.values?.messages as unknown[])?.length ?? 0}
        </p>
      </div>

      {expanded && (
        <div className="mt-3 max-h-96 overflow-auto rounded bg-gray-900 p-3">
          <pre className="text-xs text-gray-200">
            {JSON.stringify(checkpoint.values, null, 2)}
          </pre>
        </div>
      )}
    </div>
  );
}
```

!!! tip "Mẹo"
    Với UI production, cân nhắc dùng một component JSON viewer đúng nghĩa có node thu gọn được thay vì `JSON.stringify` thô. Các thư viện như `react-json-view` hoặc `react-json-tree` cho người dùng trải nghiệm khám phá tốt hơn nhiều.

## Tiếp tục từ một checkpoint

Cốt lõi của time travel là khả năng **tiếp tục thực thi từ bất kỳ checkpoint nào trước đó**. Khi người dùng chọn một checkpoint, gọi `submit` với input `null` và truyền checkpoint ID:

```ts
stream.submit({}, {
  forkFrom: { checkpointId: selectedCheckpoint.checkpoint.checkpoint_id },
});
```

Điều này báo cho LangGraph:

1. Rollback về state của checkpoint đã chọn
2. Thực thi lại graph từ điểm đó trở đi
3. Streaming kết quả mới tới client

Các message hiện có sau checkpoint đã chọn sẽ bị thay thế bởi đường thực thi mới. Điều này thực chất tạo ra một **nhánh (branch)** trong dòng thời gian hội thoại.

!!! note "Ghi chú"
    Tiếp tục từ một checkpoint không xoá dòng thời gian gốc. Các checkpoint trước đó vẫn có sẵn trong lịch sử. Nghĩa là người dùng luôn có thể quay lại và thử một nhánh khác mà không mất công việc trước đó.

## Layout SplitView

Time travel hoạt động tốt nhất với layout chia đôi (split), với chat chính ở bên trái và dòng thời gian ở bên phải:

```tsx
function TimeTravelLayout() {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [history, setHistory] = useState<ThreadState[]>([]);
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "time_travel",
    threadId,
    onThreadId: setThreadId,
  });

  const [selectedCheckpoint, setSelectedCheckpoint] =
    useState<ThreadState | null>(null);

  useEffect(() => {
    if (!threadId || stream.isLoading) return;
    stream.client.threads.getHistory(threadId).then(setHistory);
  }, [stream.client, threadId, stream.isLoading]);

  return (
    <div className="flex h-screen">
      {/* Khu vực chat chính */}
      <main className="flex-1 overflow-y-auto p-6">
        <div className="mx-auto max-w-2xl space-y-4">
          {stream.messages.map((msg) => (
            <Message key={msg.id} message={msg} />
          ))}
        </div>
        <ChatInput
          onSubmit={(text) =>
            stream.submit({ messages: [{ type: "human", content: text }] })
          }
          isLoading={stream.isLoading}
        />
      </main>

      {/* Sidebar dòng thời gian */}
      <aside className="w-96 overflow-y-auto border-l bg-gray-50">
        <TimelineSidebar
          history={history}
          selected={selectedCheckpoint}
          onSelect={setSelectedCheckpoint}
          onResume={(cp) =>
            stream.submit({}, {
              forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
            })
          }
        />
        {selectedCheckpoint && (
          <CheckpointInspector checkpoint={selectedCheckpoint} />
        )}
      </aside>
    </div>
  );
}
```

## Trích xuất metadata checkpoint

Chuyển dữ liệu checkpoint thô thành các mục hiển thị thân thiện cho dòng thời gian của bạn:

```ts
function formatCheckpoints(history: ThreadState[]) {
  return history.map((cp, index) => ({
    index,
    id: cp.checkpoint?.checkpoint_id,
    taskName: cp.tasks?.[0]?.name ?? "unknown",
    messageCount: (cp.values?.messages as unknown[])?.length ?? 0,
    hasInterrupts: cp.tasks?.some((t) => t.interrupts?.length) ?? false,
    nextNodes: cp.next ?? [],
  }));
}
```

Điều này giúp việc render các mục trong dòng thời gian dễ dàng hơn với nhãn có ý nghĩa thay vì ID thô.

## Trường hợp sử dụng

Time travel rất có giá trị trong nhiều tình huống:

* **Debug hành vi agent**: đi từng bước qua các quyết định của agent để hiểu vì sao nó chọn một đường cụ thể
* **Undo hành động**: nếu agent đi sai hướng, tiếp tục từ một checkpoint trước đó và thử lại
* **Khám phá các phương án**: fork từ một checkpoint giữa cuộc hội thoại để xem các input khác nhau thay đổi kết quả ra sao
* **Audit**: xem lại toàn bộ lịch sử hành động của agent để tuân thủ, đảm bảo chất lượng, hoặc phân tích sau sự cố
* **Giảng dạy**: đi từng bước qua quá trình thực thi của agent để giải thích cách reasoning nhiều bước hoạt động

!!! info "Thông tin"
    Time travel đặc biệt mạnh khi kết hợp với các pattern [human-in-the-loop](human-in-the-loop.md). Nếu một người review từ chối một hành động của agent tại một interrupt, họ có thể tiếp tục từ checkpoint trước khi hành động đó được thực hiện và cung cấp input điều chỉnh.

## Xử lý interrupt trong dòng thời gian

Các checkpoint chứa interrupt (điểm dừng human-in-the-loop) xứng đáng được xử lý hình ảnh đặc biệt. Chúng đại diện cho những khoảnh khắc agent dừng lại và chờ input từ con người:

```tsx
function TimelineEntry({
  checkpoint,
  index,
}: {
  checkpoint: ThreadState;
  index: number;
}) {
  const hasInterrupt = checkpoint.tasks?.some(
    (t) => t.interrupts && t.interrupts.length > 0
  );

  return (
    <div
      className={`rounded-lg border p-3 ${
        hasInterrupt
          ? "border-amber-300 bg-amber-50"
          : "border-gray-200 bg-white"
      }`}
    >
      <div className="flex items-center gap-2">
        <span className="text-xs text-gray-400">#{index + 1}</span>
        {hasInterrupt && (
          <span className="rounded bg-amber-200 px-1.5 py-0.5 text-xs font-medium text-amber-800">
            Interrupt
          </span>
        )}
      </div>
      <p className="mt-1 text-sm font-medium">
        {checkpoint.tasks?.[0]?.name ?? "—"}
      </p>
    </div>
  );
}
```

## Thực hành tốt nhất

* **Tải lịch sử một cách lười biếng (lazy)**: với các thread có hàng trăm checkpoint, phân trang hoặc chỉ tải N mục gần nhất để giữ UI phản hồi nhanh.
* **Hiển thị nhãn có ý nghĩa**: hiển thị tên node và số lượng message thay vì checkpoint ID thô. Người dùng cần ngữ cảnh, không phải UUID.
* **Xác nhận trước khi tiếp tục (resume)**: tiếp tục từ một checkpoint cũ sẽ thay thế đường thực thi hiện tại. Hiển thị hộp thoại xác nhận để người dùng không vô tình mất state hội thoại hiện tại.
* **Làm nổi bật checkpoint hiện tại**: làm rõ về mặt hình ảnh checkpoint nào tương ứng với state hiện tại của cuộc hội thoại.
* **Hỗ trợ điều hướng bằng bàn phím**: power user sẽ muốn đi từng bước qua các checkpoint bằng phím mũi tên. Thêm keyboard handler cho dòng thời gian để có trải nghiệm debug mượt mà.
* **Diff state giữa các checkpoint**: với người dùng nâng cao, hiển thị những gì đã thay đổi giữa hai checkpoint liên tiếp có thể tiết lộ chính xác cách state của agent đã tiến triển ở mỗi bước.
