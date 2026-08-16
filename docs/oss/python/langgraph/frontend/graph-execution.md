# Graph execution

> Trực quan hoá pipeline graph nhiều bước với trạng thái từng node và nội dung streaming

!!! info "Demo trực tiếp"
    Trang gốc có nhúng một demo tương tác (pattern `graph-execution-cards`). Xem trực tiếp tại [docs.langchain.com/oss/python/langgraph/frontend/graph-execution](https://docs.langchain.com/oss/python/langgraph/frontend/graph-execution).

Agent LangGraph không phải hộp đen. Mỗi graph được cấu thành từ các **node có tên**
chạy tuần tự hoặc song song: classify, research, analyze,
synthesize. Graph execution card giúp hiển thị rõ pipeline này bằng cách render một card
cho mỗi node, hiển thị trạng thái của nó, stream nội dung theo thời gian thực, và
theo dõi tiến độ hoàn thành xuyên suốt toàn bộ workflow. Người dùng thấy chính xác agent
đang làm gì, đang ở bước nào, và mỗi bước tạo ra kết quả gì.

Pattern này đặc biệt hữu ích cho agent production vì nó biến cấu trúc graph
thành product UX. Thay vì coi run là một phản hồi assistant đơn lẻ, bạn có thể phơi bày cùng
các checkpoint, tên node, state key, và stream metadata mà LangGraph dùng nội bộ.

## Cách node của graph ánh xạ sang card trên UI

Một graph LangGraph định nghĩa một chuỗi node, mỗi node phụ trách một
nhiệm vụ cụ thể. Ví dụ, một pipeline research có thể có:

1. **Classify**: phân loại truy vấn của user
2. **Research**: thu thập thông tin liên quan
3. **Analyze**: rút ra kết luận từ phần research
4. **Synthesize**: tạo ra một phản hồi cuối cùng, hoàn chỉnh

Mỗi node ghi output của nó vào một key cụ thể trong state của graph. Ở phía
frontend, bạn không cần hardcode ánh xạ đó vì [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) tự khám phá
từng node khi nó chạy thông qua `stream.subgraphs` và phơi bày một
[`SubgraphDiscoverySnapshot`](https://reference.langchain.com/javascript/langchain-react/SubgraphDiscoverySnapshot) cho mỗi bước quan sát được:

```ts
// Node được tự động khám phá, không cần danh sách hardcode
const graphNodes = [...stream.subgraphs.values()];

// Mỗi snapshot mang tên node và trạng thái hiện tại
graphNodes.forEach((node) => {
  console.log(node.nodeName, node.status); // "classify", "running"
});
```

Dùng `node.nodeName` cho label trong progress bar và header của card. Truyền mỗi
snapshot vào `useMessages(stream, node)` để render nội dung streaming theo phạm vi (scope) của node
mà không cần gắn UI với tên state key của graph.

Ánh xạ này trở thành hợp đồng (contract) giữa graph và UI của bạn. Người viết backend
có thể thêm, đổi tên, hoặc sắp xếp lại node một cách có chủ đích, trong khi người viết frontend
quyết định mỗi state key nên được trực quan hoá như thế nào: một status badge, panel markdown,
bảng, biểu đồ, view trace, hoặc card yêu cầu phê duyệt.

## Thiết lập `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) như bình thường. Các thuộc tính chính bạn sẽ dùng là `messages`
(cho hội thoại) và `subgraphs` (cho các node graph được khám phá trong
run hiện tại). Truyền mỗi snapshot subgraph được khám phá vào một selector để đọc các
message trong phạm vi của node đó.

!!! info
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](../../langchain/frontend/overview.md#type-inference) hoặc [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference).

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";

    const AGENT_URL = "http://localhost:2024";

    export function PipelineChat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "graph_execution_cards",
      });
      const graphNodes = [...stream.subgraphs.values()];

      return (
        <div>
          <PipelineProgress nodes={graphNodes} isLoading={stream.isLoading} />
          <NodeCardList nodes={graphNodes} stream={stream} isLoading={stream.isLoading} />
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
      assistantId: "graph_execution_cards",
    });
    </script>

    <template>
      <div>
        <PipelineProgress
          :nodes="[...stream.subgraphs.value.values()]"
          :is-loading="stream.isLoading.value"
        />
        <NodeCardList
          :nodes="[...stream.subgraphs.value.values()]"
          :stream="stream"
          :is-loading="stream.isLoading.value"
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
        assistantId: "graph_execution_cards",
      });
    </script>

    <div>
      <PipelineProgress nodes={[...stream.subgraphs.values()]} isLoading={stream.isLoading} />
      <NodeCardList
        nodes={[...stream.subgraphs.values()]}
        {stream}
        isLoading={stream.isLoading}
      />
    </div>
    ```

=== "Angular"
    ```ts
    import { Component, computed } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-pipeline-chat",
      template: `
        <div>
          <app-pipeline-progress
            [nodes]="graphNodes()"
            [isLoading]="stream.isLoading()"
          />
          <app-node-card-list
            [nodes]="graphNodes()"
            [stream]="stream"
            [isLoading]="stream.isLoading()"
          />
        </div>
      `,
    })
    export class PipelineChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "graph_execution_cards",
      });

      graphNodes = computed(() => [...this.stream.subgraphs().values()]);
    }
    ```

## Định tuyến streaming token đến các node

Khi graph stream, mỗi snapshot subgraph được khám phá xác định node mà nó
thuộc về. Truyền snapshot đó vào một selector hook hoặc composable để đọc các
message trong phạm vi của node đó:

```tsx
import { AIMessage } from "langchain";
import { useMessages, type AnyStream, type SubgraphDiscoverySnapshot } from "@langchain/react";

function NodeCard({
  node,
  stream,
}: {
  node: SubgraphDiscoverySnapshot;
  stream: AnyStream;
}) {
  const messages = useMessages(stream, node);
  const lastAIMessage = messages.find(AIMessage.isInstance);
  const streamingContent = lastAIMessage?.text ?? "";

  return <NodeCardBody node={node} content={streamingContent} />;
}
```

Selector được mount đầu tiên sẽ mở một subscription theo phạm vi cho namespace của node đó.
Khi node card unmount, subscription sẽ được giải phóng tự động.

## Xác định trạng thái node

Mỗi node được khám phá mang trạng thái hiện tại của nó. Dùng `node.status` trực tiếp;
discovery snapshot báo cáo `"pending"`, `"running"`, `"complete"`, hoặc
`"error"`:

```ts
type NodeStatus = SubgraphDiscoverySnapshot["status"];

const status: NodeStatus = node.status;
```

## Xây dựng thanh tiến độ pipeline (progress bar)

Một progress bar ngang ở trên cùng cho user cái nhìn tổng quan (bird's-eye view) về
toàn bộ pipeline. Mỗi bước là một đoạn có nhãn (segment) được lấp đầy khi node hoàn thành:

```tsx
function PipelineProgress({
  nodes,
  isLoading,
}: {
  nodes: SubgraphDiscoverySnapshot[];
  isLoading: boolean;
}) {
  const firstIncompleteIdx = nodes.findIndex((node) => node.status !== "complete");

  return (
    <div className="flex items-center gap-1">
      {nodes.map((node, i) => {
        const isRunning =
          isLoading && node.status !== "complete" && firstIncompleteIdx === i;
        const colors = {
          pending: "bg-gray-200 text-gray-500",
          running: "bg-blue-400 text-white animate-pulse",
          complete: "bg-green-500 text-white",
          error: "bg-red-500 text-white",
        };
        const status = isRunning ? "running" : node.status;

        return (
          <div key={node.id} className="flex items-center">
            <div
              className={`rounded-full px-3 py-1 text-xs font-medium ${colors[status]}`}
            >
              {node.nodeName}
            </div>
            {i < nodes.length - 1 && (
              <div
                className={`mx-1 h-0.5 w-6 ${
                  status === "complete" ? "bg-green-500" : "bg-gray-200"
                }`}
              />
            )}
          </div>
        );
      })}
    </div>
  );
}
```

## Xây dựng component NodeCard có thể thu gọn (collapsible)

Mỗi node có card riêng hiển thị status badge, nội dung (đang stream hoặc
đã hoàn tất), và một phần thân có thể thu gọn cho các output dài:

```tsx
function NodeCard({
  node,
  stream,
}: {
  node: SubgraphDiscoverySnapshot;
  stream: AnyStream;
}) {
  const [open, setOpen] = useState(node.status === "running");
  const messages = useMessages(stream, node);
  const lastAIMessage = messages.find(AIMessage.isInstance);

  useEffect(() => {
    if (node.status === "running") setOpen(true);
    if (node.status === "complete") setOpen(false);
  }, [node.status]);

  return (
    <div className="rounded-lg border bg-white shadow-sm">
      <button
        onClick={() => setOpen(!open)}
        className="flex w-full items-center justify-between p-4"
      >
        <div className="flex items-center gap-3">
          <h3 className="font-semibold">{node.nodeName}</h3>
          <StatusBadge status={node.status} />
        </div>
        <span className={open ? "rotate-90" : ""}>▶</span>
      </button>

      {open && (
        <div className="border-t px-4 py-3">
          <div className="prose prose-sm max-w-none">
            {lastAIMessage?.text?.trim()
              ? <Markdown>{lastAIMessage.text}</Markdown>
              : <p className="italic text-gray-500">Processing...</p>}
          </div>
        </div>
      )}
    </div>
  );
}
```

## Nội dung đang stream so với nội dung đã hoàn tất

Node card đọc message theo phạm vi cho cả nội dung đang stream lẫn nội dung cuối cùng. Cách này
tránh giả định rằng tên node của graph trùng với state key mà nó ghi vào (ví dụ,
`do_research` ghi vào `research` trong graph mẫu của playground):

| Nguồn                        | Khi nào dùng                                                                            |
| ----------------------------- | --------------------------------------------------------------------------------------- |
| `useMessages(stream, node)`  | Render message đang stream và message cuối cùng theo phạm vi của node                   |
| `stream.values`               | Đọc state của toàn graph, ví dụ field `synthesis` cuối cùng, dùng đúng state key thực tế |

Nguyên tắc là: hiển thị AI message theo phạm vi gần nhất trong node card, và
chỉ dùng `stream.values` khi bạn thực sự cần một field state của graph.

Vì message theo phạm vi được gắn với node tạo ra nó, UI có thể hỗ trợ
các nhánh graph song song mà không cần đoán dựa trên thứ tự message. Mỗi card cập nhật từ
các stream event thuộc về node của nó, và các giá trị đã hoàn tất vẫn còn khả dụng
thông qua `stream.values`.

```ts
function NodeContent({ stream, node }: { stream: AnyStream; node: SubgraphDiscoverySnapshot }) {
  const messages = useMessages(stream, node);
  const content = messages.find(AIMessage.isInstance)?.text ?? "";

  return <Markdown>{content}</Markdown>;
}
```

!!! tip
    Nội dung đang stream có thể chứa token một phần hoặc markdown chưa được hình thành
    đầy đủ. Nếu bạn render markdown, hãy đảm bảo renderer của bạn xử lý cú pháp
    chưa hoàn chỉnh một cách hợp lý (ví dụ, một dấu in đậm chưa đóng `**`).

## Kết hợp tất cả lại

Đây là danh sách card đầy đủ, kết hợp định tuyến, phát hiện trạng thái, và render
card:

```tsx
function NodeCardList({
  nodes,
  stream,
  isLoading,
}: {
  nodes: SubgraphDiscoverySnapshot[];
  stream: AnyStream;
  isLoading: boolean;
}) {
  const firstIncompleteIdx = nodes.findIndex((node) => node.status !== "complete");

  return (
    <div className="space-y-3">
      {nodes.map((node, i) => {
        const isComplete = node.status === "complete";
        const isRunning = isLoading && !isComplete && firstIncompleteIdx === i;
        if (!isComplete && !isRunning) return null;

        return <NodeCard key={node.id} node={node} stream={stream} />;
      })}
    </div>
  );
}
```

## Trường hợp sử dụng

Graph execution card phù hợp cho mọi pipeline nhiều bước cần khả năng quan sát:

* **Pipeline research**: classify → thu thập nguồn → analyze → synthesize thành
  báo cáo
* **Sinh nội dung**: outline → draft → fact-check → edit → publish
* **Xử lý dữ liệu**: ingest → validate → transform → aggregate → export
* **Sinh code**: hiểu requirement → lên kế hoạch kiến trúc → viết
  code → review → test
* **Workflow ra quyết định**: thu thập context → đánh giá phương án → chấm điểm
  các lựa chọn → đề xuất

## Xử lý pipeline động

Không phải mọi graph đều có một tập node cố định. Một số pipeline thêm hoặc bỏ qua node
tuỳ theo input. Discovery map chỉ chứa các node được quan sát cho
thread hiện tại:

```ts
const activeNodes = [...stream.subgraphs.values()];
```

Điều này đảm bảo UI của bạn chỉ hiển thị card cho các node liên quan đến
lần thực thi hiện tại, tránh các card placeholder trống.

!!! info
    Nếu graph của bạn có nhánh rẽ có điều kiện (ví dụ, bỏ qua "Research" cho các
    truy vấn factual đơn giản), các node bị bỏ qua sẽ không xuất hiện trong `stream.subgraphs`. Progress bar
    pipeline của bạn có thể chỉ render các node được khám phá, hoặc làm mờ các node kỳ vọng
    mà không có snapshot tương ứng.

## Thực hành tốt nhất

* **Khám phá node từ stream**. Render card từ `stream.subgraphs`
  thay vì hardcode các node kỳ vọng; các bước có điều kiện hoặc bị bỏ qua sẽ không
  xuất hiện cho đến khi chúng chạy.
* **Coi state key là hợp đồng UI**. Quyết định output nào của graph nên đủ
  ổn định để frontend render, và giữ tài liệu về các key đó cạnh định nghĩa graph.
* **Dùng message theo phạm vi cho node card**. Chúng hoạt động cả khi node đang stream
  và sau khi hoàn tất, mà không gắn card UI với tên state key.
* **Tự động thu gọn node đã hoàn thành**. Trong các pipeline dài, tự động thu gọn các
  card đã hoàn thành để user tập trung vào bước đang hoạt động.
* **Hiển thị ước tính thời gian**. Nếu bạn có dữ liệu lịch sử về thời gian mỗi node
  tốn, hiển thị ước tính thời gian để đặt kỳ vọng cho user.
* **Thêm chỉ báo tiến độ toàn cục**. Bổ sung cho các card theo từng node bằng một
  progress bar tổng thể (ví dụ, "Step 2 of 4") ở đầu view pipeline.
* **Xử lý lỗi theo từng node**. Nếu một node lỗi, hiển thị lỗi trong card của nó
  mà không thu gọn toàn bộ pipeline. Các node khác vẫn có thể hoàn thành
  thành công.
