# Join & rejoin streams

> Ngắt kết nối khỏi và kết nối lại vào luồng agent đang chạy

Join & rejoin cho phép bạn ngắt kết nối khỏi một luồng agent đang chạy mà không dừng agent lại, rồi kết nối lại với nó sau đó. Agent tiếp tục thực thi ở phía server trong khi client vắng mặt, và bạn tiếp tục theo dõi luồng đúng ngay chỗ đã dừng lại.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/join-rejoin

!!! note "Ghi chú"
    Tính năng này yêu cầu [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server). Chạy agent của bạn cục bộ bằng `langgraph dev` hoặc [deploy lên LangSmith](https://docs.langchain.com/langsmith/deployment) để dùng pattern này.

## Vì sao cần join & rejoin?

Các API streaming truyền thống gắn chặt client và server: nếu client ngắt kết nối, luồng sẽ mất. Join & rejoin phá vỡ sự ràng buộc này, cho phép một số pattern quan trọng:

* **Gián đoạn mạng**: người dùng di động di chuyển giữa các trạm phát sóng hoặc mạng Wi-Fi có thể tiếp tục liền mạch
* **Điều hướng trang**: người dùng rời khỏi trang chat và quay lại sau mà không mất tiến trình
* **Ứng dụng chạy nền trên mobile**: các app bị hệ điều hành tạm dừng có thể join lại luồng khi được đưa lên foreground
* **Tác vụ chạy lâu**: agent thực hiện các thao tác kéo dài nhiều phút (nghiên cứu, sinh code, phân tích dữ liệu) mà người dùng không cần giữ trang mở
* **Chuyển đổi đa thiết bị**: bắt đầu cuộc trò chuyện trên điện thoại, join lại trên desktop

## Khái niệm cốt lõi

Pattern join/rejoin liên quan đến ba cơ chế chính:

| Phương thức / Tuỳ chọn | Mục đích |
| --- | --- |
| `threadId` | Gắn luồng với thread LangGraph bạn muốn theo dõi |
| `onThreadId` | Lưu lại thread ID mới tạo để một lần remount có thể kết nối lại |
| `stream.disconnect()` | Rời khỏi luồng ở phía client trong khi agent vẫn tiếp tục chạy ở phía server |
| Remount với cùng `threadId` | Gắn lại vào công việc đang chạy dở cho thread đó |

!!! note "Ghi chú"
    **Join/rejoin dùng `stream.disconnect()`, không dùng `stream.stop()`.** Theo mặc định, `stream.stop()` **huỷ run đang hoạt động**: nó ngắt kết nối client *và* huỷ run ở phía server. Với join/rejoin, gọi `stream.disconnect()` (alias của `stop({ cancel: false })`) để agent tiếp tục xử lý trong khi bạn vắng mặt.

    Để huỷ thực thi một cách tường minh từ code của app, dùng `stream.stop()` hoặc [`client.runs.cancel`](https://reference.langchain.com/javascript/langchain-langgraph-sdk/client/RunsClient/cancel).

## Thiết lập `useStream`

Bước thiết lập quan trọng là lưu lại `threadId`. Khi component remount với cùng thread ID, luồng sẽ gắn vào state hiện tại của thread và bất kỳ run nào đang chạy dở.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";
    import { useCallback, useState } from "react";

    function Chat() {
      const [connected, setConnected] = useState(true);
      const [mountKey, setMountKey] = useState(0);
      const [threadId, setThreadId] = useState<string | null>(
        () => sessionStorage.getItem("activeThreadId"),
      );

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "join_rejoin",
        threadId,
        onThreadId(id) {
          setThreadId(id);
          if (id) sessionStorage.setItem("activeThreadId", id);
        },
      });

      const disconnect = useCallback(() => {
        void stream.disconnect();
        setConnected(false);
      }, [stream]);

      const rejoin = useCallback(() => {
        setMountKey((key) => key + 1);
        setConnected(true);
      }, []);

      return (
        <div key={mountKey}>
          <ConnectionStatus connected={connected} />
          <MessageList messages={stream.messages} />
          <ChatControls
            stream={stream}
            threadId={threadId}
            connected={connected}
            onDisconnect={disconnect}
            onRejoin={rejoin}
          />
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";
    import { ref } from "vue";

    const connected = ref(true);
    const mountKey = ref(0);
    const threadId = ref<string | null>(sessionStorage.getItem("activeThreadId"));

    const stream = useStream<typeof myAgent>({
      apiUrl: "http://localhost:2024",
      assistantId: "join_rejoin",
      threadId,
      onThreadId(id) {
        threadId.value = id;
        if (id) sessionStorage.setItem("activeThreadId", id);
      },
    });

    function disconnect() {
      void stream.disconnect();
      connected.value = false;
    }

    function rejoin() {
      mountKey.value += 1;
      connected.value = true;
    }
    </script>

    <template>
      <div :key="mountKey">
        <ConnectionStatus :connected="connected" />
        <MessageList :messages="stream.messages" />
        <ChatControls
          :stream="stream"
          :threadId="threadId"
          :connected="connected"
          @disconnect="disconnect"
          @rejoin="rejoin"
        />
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";

      let connected = $state(true);
      let mountKey = $state(0);
      let threadId = $state<string | null>(sessionStorage.getItem("activeThreadId"));

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "join_rejoin",
        threadId: () => threadId,
        onThreadId(id) {
          threadId = id;
          if (id) sessionStorage.setItem("activeThreadId", id);
        },
      });

      function disconnect() {
        void stream.disconnect();
        connected = false;
      }

      function rejoin() {
        mountKey += 1;
        connected = true;
      }
    </script>

    <div key={mountKey}>
      <ConnectionStatus {connected} />
      <MessageList messages={stream.messages} />
      <ChatControls
        {threadId}
        {connected}
        onDisconnect={disconnect}
        onRejoin={rejoin}
      />
    </div>
    ```

=== "Angular"
    ```ts
    import { Component, signal } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    @Component({
      selector: "app-chat",
      template: `
        <connection-status [connected]="connected()" />
        <message-list [messages]="stream.messages()" />
        <chat-controls
          [stream]="stream"
          [threadId]="threadId()"
          [connected]="connected()"
          (disconnect)="disconnect()"
          (rejoin)="rejoin()"
        />
      `,
    })
    export class ChatComponent {
      threadId = signal<string | null>(sessionStorage.getItem("activeThreadId"));
      connected = signal(true);
      mountKey = signal(0);

      stream = injectStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "join_rejoin",
        threadId: this.threadId,
        onThreadId: (id) => {
          this.threadId.set(id);
          if (id) sessionStorage.setItem("activeThreadId", id);
        },
      });

      disconnect() {
        void this.stream.disconnect();
        this.connected.set(false);
      }

      rejoin() {
        this.mountKey.update((key) => key + 1);
        this.connected.set(true);
      }
    }
    ```

## Gửi (submit) message

Gửi message như bình thường. Việc gắn thread ID là thứ cho phép một lần remount sau đó kết nối lại đúng cuộc hội thoại:

```ts
stream.submit({ messages: [{ type: "human", content: text }] });
```

## Ngắt kết nối khỏi một luồng

Gọi `stream.disconnect()` để rời khỏi luồng mà không huỷ run. Agent tiếp tục xử lý ở phía server.

```ts
await stream.disconnect();
// tương đương với: await stream.stop({ cancel: false })
```

**Không** dùng `stream.stop()` ở đây, vì mặc định nó sẽ huỷ run trên server.

Sau khi gọi `disconnect()`:

* `stream.isLoading` trở thành `false`
* Cờ `connected` của riêng bạn cũng nên trở thành `false`
* Danh sách message giữ nguyên tất cả message đã nhận được đến thời điểm ngắt kết nối
* Agent vẫn tiếp tục chạy trên server
* Không có message mới nào được nhận cho đến khi bạn rejoin

## Kết nối lại (rejoin) một luồng

Remount lại consumer của luồng với thread ID đã lưu để kết nối lại. Trong React, demo tăng giá trị `mountKey`; ở các framework khác, dùng pattern remount hoặc conditional-render tương đương:

```ts
setMountKey((key) => key + 1);
setConnected(true);
```

Sau khi rejoin:

* `connected` trở thành `true`
* Mọi message được sinh ra trong lúc ngắt kết nối sẽ được gửi tới
* Message streaming mới tiếp tục theo thời gian thực
* Nếu agent vẫn đang chạy, `stream.isLoading` trở thành `true`; nếu đã hoàn tất, bạn nhận được state cuối cùng ngay lập tức

## Thực hành tốt nhất

* **Dùng `disconnect()` cho join/rejoin, `stop()` để huỷ**: việc rời trang hoặc đưa app xuống nền nên gọi `stream.disconnect()`. Một nút "Stop" hoặc "Cancel" hiển thị cho người dùng nên gọi `stream.stop()` (hoặc [`client.runs.cancel`](https://reference.langchain.com/javascript/langchain-langgraph-sdk/client/RunsClient/cancel)).
* **Luôn lưu thread ID**: nếu không có nó, việc rejoin là không thể. Dùng cả component state lẫn bộ nhớ lưu trữ bền vững (persistent storage) để đảm bảo độ tin cậy.
* **Hiển thị rõ trạng thái kết nối**: người dùng phải luôn biết họ đang nhận cập nhật trực tiếp hay đang xem một snapshot.
* **Tự động rejoin khi thay đổi visibility**: dùng Page Visibility API để tự động rejoin khi người dùng quay lại tab.
* **Đặt timeout hợp lý**: nếu một lần thử rejoin mất quá nhiều thời gian, hãy fallback về việc fetch lịch sử thread.
* **Dọn dẹp thread cũ**: xoá thread ID đã lưu khi người dùng bắt đầu lại từ đầu hoặc khi backend báo rằng thread không còn khả dụng.
