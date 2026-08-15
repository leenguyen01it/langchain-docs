# Tool calling

> Hiển thị tool call của agent bằng UI card phong phú, type-safe

Agent có thể gọi các tool bên ngoài như weather API, calculator, tìm kiếm web, truy vấn database, và nhiều hơn nữa. Kết quả trả về ở dạng JSON thô. Pattern này chỉ cho bạn cách render UI card có cấu trúc, type-safe cho mỗi tool call mà agent của bạn thực hiện, kèm trạng thái loading và xử lý lỗi đầy đủ.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/tool-calling

## Tool calling hoạt động như thế nào

Khi một agent LangGraph quyết định nó cần dữ liệu bên ngoài, nó phát ra một hoặc nhiều **tool call** như một phần của AI message. Mỗi tool call gồm:

* **name**: tool được gọi (ví dụ `"get_weather"`, `"calculator"`)
* **args**: các tham số có cấu trúc được truyền cho tool
* **id**: một định danh duy nhất liên kết lần gọi với kết quả của nó

Agent runtime thực thi tool, và kết quả trả về dưới dạng một `ToolMessage`. Hook [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hợp nhất tất cả điều này thành một mảng `toolCalls` duy nhất mà bạn có thể render trực tiếp.

## Thiết lập `useStream`

Bước đầu tiên là kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với backend agent của bạn. Hook trả về state reactive bao gồm một mảng `toolCalls` cập nhật theo thời gian thực khi agent streaming.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";

    const AGENT_URL = "http://localhost:2024";

    export function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "tool_calling",
      });

      return (
        <div>
          {stream.messages.map((msg) => (
            <Message key={msg.id} message={msg} toolCalls={stream.toolCalls} />
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
      assistantId: "tool_calling",
    });
    </script>

    <template>
      <div>
        <Message
          v-for="msg in stream.messages.value"
          :key="msg.id"
          :message="msg"
          :tool-calls="stream.toolCalls.value"
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
        assistantId: "tool_calling",
      });
    </script>

    <div>
      {#each stream.messages as msg (msg.id)}
        <Message message={msg} toolCalls={stream.toolCalls} />
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
          <app-message [message]="msg" [toolCalls]="stream.toolCalls()" />
        }
      `,
    })
    export class ChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "tool_calling",
      });
    }
    ```

## Kiểu AssembledToolCall

Mỗi entry trong mảng `toolCalls` là một object `AssembledToolCall`:

```ts
interface AssembledToolCall<
  TName extends string = string,
  TInput = unknown,
  TOutput = unknown,
> {
  name: TName;
  callId: string;
  id: string;
  namespace: string[];
  input: TInput;
  args: TInput;
  output: TOutput | null;
  status: "running" | "finished" | "error";
  error: string | undefined;
}
```

| Thuộc tính | Mô tả |
| --- | --- |
| `name` | Tên của tool (ví dụ `"get_weather"`) |
| `callId` | ID duy nhất khớp với entry `tool_calls` của AI message |
| `id` | Alias cho `callId`, khớp với tool call ở cấp message |
| `namespace` | Namespace nơi tool call được phát ra |
| `input` | Tham số có cấu trúc mà agent truyền cho tool |
| `args` | Alias cho `input`, khớp với tool call ở cấp message |
| `output` | Output của tool sau khi gọi thành công, hoặc `null` khi đang chạy hoặc sau lỗi |
| `status` | Trạng thái vòng đời: `"running"`, `"finished"`, hoặc `"error"` |
| `error` | Chi tiết lỗi khi tool call thất bại |

## Lọc tool call theo từng message

Một AI message có thể kích hoạt nhiều tool call, và chat của bạn có thể chứa nhiều AI message. Để render đúng tool card dưới mỗi message, lọc bằng cách khớp `callId` với mảng `tool_calls` của message:

```tsx
function Message({
  message,
  toolCalls,
}: {
  message: AIMessage;
  toolCalls: AssembledToolCall[];
}) {
  const messageToolCalls = toolCalls.filter((tc) =>
    message.tool_calls?.find((t) => t.id === tc.callId)
  );

  return (
    <div>
      <p>{message.text}</p>
      {messageToolCalls.map((tc) => (
        <ToolCard key={tc.callId} toolCall={tc} />
      ))}
    </div>
  );
}
```

## Xây dựng tool card chuyên biệt

Thay vì hiển thị JSON thô, xây dựng các component UI riêng cho từng tool. Dùng `name` để chọn đúng card:

```tsx
function ToolCard({ toolCall }: { toolCall: AssembledToolCall }) {
  if (toolCall.status === "running") {
    return <LoadingCard name={toolCall.name} />;
  }

  if (toolCall.status === "error") {
    return <ErrorCard name={toolCall.name} error={toolCall.error} />;
  }

  switch (toolCall.name) {
    case "get_weather":
      return <WeatherCard input={toolCall.input} output={toolCall.output} />;
    case "calculator":
      return (
        <CalculatorCard input={toolCall.input} output={toolCall.output} />
      );
    case "web_search":
      return <SearchCard input={toolCall.input} output={toolCall.output} />;
    default:
      return <GenericToolCard toolCall={toolCall} />;
  }
}
```

### Ví dụ weather card

```tsx
function WeatherCard({
  input,
  output,
}: {
  input: { location: string };
  output: { temperature: number; condition: string };
}) {
  return (
    <div className="rounded-lg border p-4">
      <div className="flex items-center gap-2">
        <CloudIcon />
        <h3 className="font-semibold">{input.location}</h3>
      </div>
      <div className="mt-2 text-3xl font-bold">{output.temperature}°F</div>
      <p className="text-muted-foreground">{output.condition}</p>
    </div>
  );
}
```

### Trạng thái loading và lỗi

Luôn xử lý trạng thái đang chờ và lỗi để đem lại phản hồi rõ ràng cho người dùng:

```tsx
function LoadingCard({ name }: { name: string }) {
  return (
    <div className="flex items-center gap-2 rounded-lg border p-4 animate-pulse">
      <Spinner />
      <span>Running {name}...</span>
    </div>
  );
}

function ErrorCard({ name, error }: { name: string; error?: unknown }) {
  return (
    <div className="rounded-lg border border-red-300 bg-red-50 p-4">
      <h3 className="font-semibold text-red-700">Error in {name}</h3>
      <p className="text-sm text-red-600">
        {String(error ?? "Tool execution failed")}
      </p>
    </div>
  );
}
```

## Tool argument type-safe

Nếu tool của bạn được định nghĩa với schema có cấu trúc, bạn có thể dùng kiểu tiện ích `ToolCallFromTool` để có `args` được gõ kiểu đầy đủ:

```ts
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(async ({ location }) => { /* ... */ }, {
  name: "get_weather",
  description: "Get the current weather for a location",
  schema: z.object({
    location: z.string().describe("City name"),
  }),
});

type WeatherToolCall = ToolCallFromTool<typeof getWeather>;
// WeatherToolCall.input và WeatherToolCall.args giờ có kiểu { location: string }
```

!!! tip "Mẹo"
    Dùng `ToolCallFromTool` cho bạn độ an toàn ngay lúc biên dịch (compile-time). Nếu tool schema thay đổi, các component UI của bạn sẽ báo lỗi kiểu ngay lập tức.

## Render tool call inline cùng text streaming

Tool call thường đến xen kẽ với text đang streaming. Hook [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) giữ `toolCalls` đồng bộ với luồng, nên các card đang chờ xuất hiện ngay khi agent phát ra lệnh gọi, trước khi tool thực thi xong.

Điều này nghĩa là người dùng thấy:

1. Text của AI khi nó streaming vào
2. Một loading card ngay khi một tool call được phát ra
3. Card cập nhật để hiển thị kết quả khi tool hoàn tất

!!! note "Ghi chú"
    Tool call cập nhật tại chỗ. Cùng một `callId` chuyển từ `"running"` sang `"finished"` (hoặc `"error"`), nên UI của bạn render lại cùng một component với state mới.

## Xử lý nhiều tool call đồng thời

Agent có thể gọi nhiều tool song song. Mảng `toolCalls` sẽ chứa nhiều entry với `status: "running"` cùng lúc. Mỗi entry hoàn tất độc lập, nên UI của bạn nên xử lý việc hoàn thành một phần một cách mượt mà:

```tsx
function ToolCallList({ toolCalls }: { toolCalls: AssembledToolCall[] }) {
  const pending = toolCalls.filter((tc) => tc.status === "running");
  const completed = toolCalls.filter((tc) => tc.status === "finished");

  return (
    <div className="space-y-2">
      {completed.map((tc) => (
        <ToolCard key={tc.callId} toolCall={tc} />
      ))}
      {pending.map((tc) => (
        <LoadingCard key={tc.callId} name={tc.name} />
      ))}
    </div>
  );
}
```

## Thực hành tốt nhất

Tuân theo các nguyên tắc sau khi xây dựng UI tool call:

* **Luôn xử lý cả ba trạng thái**: `running`, `finished`, và `error`. Người dùng không bao giờ nên thấy một card trống.
* **Validate kết quả một cách an toàn**. Tool output có kiểu `unknown` cho đến khi bạn thu hẹp kiểu cho một card cụ thể.
* **Cung cấp fallback chung**. Không phải tool nào cũng cần một card riêng biệt. Render một view JSON có thể thu gọn cho các tool name chưa biết.
* **Hiển thị tên tool và args trong lúc loading**. Người dùng muốn biết agent đang làm *gì*, ngay cả trước khi có kết quả.
* **Giữ card gọn gàng**. Tool card nằm inline cùng message chat. Tránh làm cuộc hội thoại bị quá tải bởi widget quá khổ.
