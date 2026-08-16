# Custom stream channels

> Stream dữ liệu tùy chỉnh phía server tới frontend và đọc bằng useExtension và useChannel

Agent LangGraph stream nhiều hơn là chỉ message và tool call. Một **stream transformer** phía server có thể kiểm tra hoặc viết lại protocol khi nó chảy tới client, và publish dữ liệu có cấu trúc riêng lên một **custom channel** được đặt tên. Frontend đọc channel đó bằng hai selector: [`useExtension`](https://reference.langchain.com/javascript/langchain-react/useExtension) cho payload mới nhất, và [`useChannel`](https://reference.langchain.com/javascript/langchain-react/useChannel) như một lối thoát (escape hatch) để đọc raw event.

Ví dụ dưới đây là một agent customer-support có transformer redact (che) PII (email, số điện thoại, SSN, số thẻ, IP) khỏi mọi event trước khi nó đến trình duyệt, và publish số liệu đếm redaction đang chạy lên channel `redaction-stats`. Side panel render các số liệu đếm này theo thời gian thực.

!!! info "Demo trực tiếp"
    Trang gốc có một demo tương tác trực tiếp (pattern `custom-stream-channel`). Xem demo tại [trang gốc trên docs.langchain.com](https://docs.langchain.com/oss/python/langgraph/frontend/custom-stream-channels).

## Custom channel hoạt động như thế nào

Một custom channel có hai đầu. Ở phía server, một [`StreamTransformer`](https://reference.langchain.com/python/langgraph/stream/_types/StreamTransformer) mở một [`StreamChannel`](https://reference.langchain.com/python/langgraph/stream/stream_channel/StreamChannel) được đặt tên và đẩy payload vào đó. Ở phía client, một selector subscribe vào channel `custom:<name>` tương ứng và expose các payload đó dưới dạng reactive state.

Method `process` của transformer chạy cho mọi protocol event. Nó có thể sửa event tại chỗ (ở đây là xóa PII khỏi dữ liệu `messages`, `tools`, và `values`) và đẩy cập nhật side-channel bất cứ khi nào có gì để báo cáo.

Các selector phía client (`useExtension`, `useChannel`) đi kèm trong các package frontend SDK v1 (`@langchain/react`, `@langchain/vue`, `@langchain/svelte`, `@langchain/angular`).

!!! note
    Stream transformer và `StreamChannel` yêu cầu `langgraph>=1.2`.

```python
import time

from langgraph.stream import ProtocolEvent, StreamChannel, StreamTransformer


class RedactionStatsTransformer(StreamTransformer):
    def __init__(self, scope: tuple[str, ...] = ()) -> None:
        super().__init__(scope)
        # Mở một channel tên "redaction-stats".
        self.redaction_stats = StreamChannel("redaction-stats")
        self.counts = empty_counts()

    def init(self) -> dict[str, StreamChannel]:
        return {"redactionStats": self.redaction_stats}

    def process(self, event: ProtocolEvent) -> bool:
        # Redact event["params"]["data"] tại chỗ và cộng dồn những gì tìm thấy.
        delta = redact_in_place(event, self.counts)
        if delta:
            # Publish một payload lên channel.
            self.redaction_stats.push(
                {
                    "kind": "update",
                    "at": int(time.time() * 1000),
                    "delta": delta,
                    "counts": dict(self.counts),
                    "total": sum(self.counts.values()),
                }
            )
        return True  # Giữ event (đã được redact) trong stream.


def create_redaction_stats_transformer() -> RedactionStatsTransformer:
    return RedactionStatsTransformer()
```

Gắn transformer khi bạn xây agent:

```python
from langchain.agents import create_agent

agent = create_agent(
    model="anthropic:claude-haiku-4-5",
    tools=[...],
    transformers=[create_redaction_stats_transformer],
)
```

Kiểu payload là bất kỳ thứ gì transformer đẩy vào. Các ví dụ client dưới đây đọc theo dạng sau:

```ts
type PiiType = "email" | "phone" | "ssn" | "credit_card" | "ip_address";

type RedactionStatsEvent = {
  kind: "update";
  at: number;
  delta: Partial<Record<PiiType, number>>;
  counts: Record<PiiType, number>;
  total: number;
};
```

## Thiết lập `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) như bình thường. Các selector custom-channel dùng chung `stream` handle được trả về ở đây.

!!! info
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](../../langchain/frontend/overview.md#type-inference) hoặc [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference).

=== "React"

    ```tsx
    import { useStream } from "@langchain/react";

    const AGENT_URL = "http://localhost:2024";

    export function RedactionChat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "custom_stream_channel",
      });

      return <RedactionStatsPanel stream={stream} />;
    }
    ```

=== "Vue"

    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";

    const AGENT_URL = "http://localhost:2024";

    const stream = useStream<typeof myAgent>({
      apiUrl: AGENT_URL,
      assistantId: "custom_stream_channel",
    });
    </script>

    <template>
      <RedactionStatsPanel :stream="stream" />
    </template>
    ```

=== "Svelte"

    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";

      const AGENT_URL = "http://localhost:2024";

      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "custom_stream_channel",
      });
    </script>

    <RedactionStatsPanel {stream} />
    ```

=== "Angular"

    ```ts
    import { Component } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-redaction-chat",
      template: `<app-redaction-stats-panel [stream]="stream" />`,
    })
    export class RedactionChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "custom_stream_channel",
      });
    }
    ```

## Đọc payload mới nhất bằng `useExtension`

`useExtension` subscribe vào một channel `custom:<name>` và trả về payload gần nhất mà transformer đã đẩy, đã được unwrap và gõ kiểu (typed) sẵn. Đây là lựa chọn tiện lợi khi UI chỉ cần giá trị hiện tại, chẳng hạn một bộ đếm trực tiếp, phần trăm tiến độ, hay status badge.

Truyền tên channel trần (`"redaction-stats"`), không kèm prefix `custom:`:

=== "React"

    ```tsx
    import { useExtension } from "@langchain/react";

    const latest = useExtension<RedactionStatsEvent>(stream, "redaction-stats");
    // latest?.total, latest?.counts.email, latest?.delta
    ```

=== "Vue"

    ```ts
    import { useExtension } from "@langchain/vue";

    const latest = useExtension<RedactionStatsEvent>(stream, "redaction-stats");
    // latest.value?.total
    ```

=== "Svelte"

    ```ts
    import { useExtension } from "@langchain/svelte";

    const latest = useExtension<RedactionStatsEvent>(stream, "redaction-stats");
    // latest?.total
    ```

=== "Angular"

    ```ts
    import { injectExtension } from "@langchain/angular";

    const latest = injectExtension<RedactionStatsEvent>(stream, "redaction-stats");
    // latest()?.total
    ```

Giá trị trả về theo đúng mô hình reactivity của từng framework: một giá trị thuần trong React và Svelte, một `Ref` trong Vue (`latest.value`), và một signal trong Angular (`latest()`). Giá trị là `undefined` cho tới khi payload đầu tiên đến.

Một tham số `target` thứ ba, tùy chọn, giới hạn subscription vào một namespace, tương tự cách `useMessages(stream, node)` giới hạn message vào một graph node được phát hiện. Xem [Graph execution](graph-execution.md) để biết cách nhắm namespace.

## Buffer raw event bằng `useChannel`

`useChannel` là lối thoát để đọc raw event. Nó subscribe vào một hoặc nhiều channel và trả về một buffer có giới hạn kích thước gồm các protocol event gốc, thay vì một giá trị unwrap duy nhất. Dùng nó khi bạn cần lịch sử thay vì giá trị mới nhất, chẳng hạn event log hay audit trail, hoặc khi bạn cần một channel mà không selector cấp cao nào bao phủ.

Truyền đầy đủ channel id (`"custom:redaction-stats"`):

=== "React"

    ```tsx
    import { useChannel } from "@langchain/react";

    const rawEvents = useChannel(stream, ["custom:redaction-stats"]);
    ```

=== "Vue"

    ```ts
    import { useChannel } from "@langchain/vue";

    const rawEvents = useChannel(stream, ["custom:redaction-stats"]);
    // rawEvents.value
    ```

=== "Svelte"

    ```ts
    import { useChannel } from "@langchain/svelte";

    const rawEvents = useChannel(stream, ["custom:redaction-stats"]);
    ```

=== "Angular"

    ```ts
    import { injectChannel } from "@langchain/angular";

    const rawEvents = injectChannel(stream, ["custom:redaction-stats"]);
    // rawEvents()
    ```

Mỗi phần tử là một raw protocol event, nên payload nằm dưới `event.params.data`. Tự bạn unwrap nó:

```ts
function parseRedactionStatsEvents(rawEvents: Event[]): RedactionStatsEvent[] {
  const out: RedactionStatsEvent[] = [];
  for (const event of rawEvents) {
    const data = event.params?.data;
    const payload = data?.payload ?? data;
    if (payload?.kind === "update") out.push(payload);
  }
  return out;
}
```

Kiểm soát buffer bằng tham số options:

```ts
const rawEvents = useChannel(
  stream,
  ["custom:redaction-stats"],
  undefined, // namespace đích
  { bufferSize: 200, replay: true },
);
```

| Tùy chọn     | Mặc định    | Tác dụng                                                                                         |
| ------------ | ----------- | ------------------------------------------------------------------------------------------------- |
| `bufferSize` | `"default"` | Số lượng event buffer tối đa. Event cũ bị loại khi đạt giới hạn.                                  |
| `replay`     | `true`      | Replay lại các event đã thấy trên channel khi selector mount, thay vì chỉ nhận event mới (live).  |

!!! note
    Ưu tiên các selector cấp cao hơn (`useExtension`, `useMessages`, `useToolCalls`, `useValues`) cho các trường hợp thông thường. Chúng trả về giá trị đã gõ kiểu, đã unwrap, và chỉ theo dõi những gì bạn render. Dùng `useChannel` khi bạn thực sự cần raw event stream.

## Chọn giữa `useExtension` và `useChannel`

Cả hai đều đọc cùng một custom channel nhưng khác nhau ở giá trị trả về:

|                  | `useExtension`                     | `useChannel`                                             |
| ---------------- | ----------------------------------- | ---------------------------------------------------------- |
| **Trả về**        | Payload mới nhất (`T \| undefined`) | Buffer có giới hạn của raw event (`Event[]`)               |
| **Hình dạng**     | Payload đã unwrap, đã gõ kiểu       | Raw protocol event; tự unwrap `event.params.data`          |
| **Subscribe theo**| Tên channel (`"redaction-stats"`)  | Đầy đủ channel id (`["custom:redaction-stats"]`)            |
| **Dùng khi**      | Bạn cần giá trị hiện tại            | Bạn cần lịch sử, log, hoặc nhiều channel                    |
| **Tùy chọn**      | (không có)                          | `bufferSize`, `replay`                                     |

Một pattern thường gặp là dùng cả hai trên cùng một channel: `useExtension` dẫn động một bảng tóm tắt trực tiếp (tổng số hiện tại), trong khi `useChannel` cấp dữ liệu cho một event log cuộn được, ghi lại mọi cập nhật trong suốt thread.

## Trường hợp sử dụng

Custom channel phù hợp với mọi tín hiệu phía server không map gọn vào message, tool call, hay graph state:

* **Số liệu tuân thủ và redaction**: số lượng PII đã xóa, nội dung bị chặn, hoặc vi phạm chính sách, như ví dụ trên.
* **Báo cáo tiến độ**: phần trăm hoàn thành hoặc nhãn bước do một tool chạy dài phát ra.
* **Số liệu trực tiếp**: mức dùng token, độ trễ, hoặc chi phí tích lũy trong một lần chạy.
* **Nguồn và trích dẫn**: tài liệu đã retrieve được đẩy tới side panel khi agent grounding câu trả lời của nó.
* **Domain event**: bất kỳ cập nhật có cấu trúc nào backend của bạn muốn hiển thị mà không thay đổi transcript message.

## Liên quan

* [Overview](overview.md): stream API và kiến trúc frontend của LangGraph.
* [Graph execution](graph-execution.md): các selector giới hạn theo namespace cho pipeline nhiều node.
