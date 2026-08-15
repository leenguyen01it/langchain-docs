# Công cụ headless (Headless tools)

> Chạy các API trình duyệt và thiết bị trên client bằng cách triển khai công cụ (tool) dạng headless

Công cụ headless (headless tools) cho phép agent gọi các tool mà quá trình thực thi thật sự phải diễn ra trong ứng dụng của người dùng thay vì trên server. Agent vẫn nhìn thấy một tool schema bình thường, nhưng phần triển khai lại nằm ở frontend, nơi nó có thể truy cập các API trình duyệt như IndexedDB, geolocation, clipboard, canvas, hoặc trình chọn tệp (file picker).

Pattern này đặc biệt hữu ích khi dữ liệu cần được giữ cục bộ trên thiết bị. Ví dụ playground trên trang này dùng một bộ công cụ bộ nhớ trình duyệt (browser-memory) nhỏ được lưu trữ bởi IndexedDB, cùng với một tool geolocation chạy hoàn toàn trên client.

## Cách công cụ headless hoạt động

Ở mức tổng quan, công cụ headless tách tool schema ra khỏi phần triển khai chỉ chạy trên trình duyệt.

1. Đăng ký một tool trên agent, tool này gọi `interrupt()` ngay lập tức để chuyển việc thực thi sang frontend.
2. Phản chiếu (mirror) cùng tên tool và các trường tham số trong các định nghĩa ở frontend.
3. Triển khai các tool tương ứng ở frontend bằng `.implement(...)` và truyền chúng
   vào `useStream({ tools: [...] })`.
4. Khi agent gọi một tool khớp, client sẽ xử lý hành động đó và
   tiếp tục (resume) lượt chạy đã bị interrupt với kết quả của tool.

## Đăng ký tool trên agent

Playground định nghĩa một tập nhỏ các tool phía client theo cùng
một pattern: agent phơi bày (expose) một tool schema, còn frontend xử lý việc
thực thi thật sự.

Định nghĩa các tool bình thường trên server, các tool này gọi `interrupt()` ngay lập tức, sau đó
phản chiếu cùng tên tool và các trường tham số trong một
file `tools.ts` ở frontend.

```python agent.py
from typing import Any

from langchain import create_agent
from langchain.tools import ToolRuntime, tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt
from pydantic import BaseModel


class MemoryPutInput(BaseModel):
    key: str
    value: Any


class MemoryGetInput(BaseModel):
    key: str


class GeolocationGetInput(BaseModel):
    save: bool = True


def _interrupt_for_client(
    tool_name: str,
    args: dict[str, Any],
    runtime: ToolRuntime,
) -> Any:
    return interrupt({
        "type": "tool",
        "tool_call": {
            "id": runtime.tool_call_id,
            "name": tool_name,
            "args": args,
        },
    })


@tool(
    "memory_put",
    description="Store a memory in the user's browser.",
    args_schema=MemoryPutInput,
)
def memory_put(key: str, value: Any, runtime: ToolRuntime) -> Any:
    return _interrupt_for_client(
        "memory_put",
        {"key": key, "value": value},
        runtime,
    )


@tool(
    "memory_get",
    description="Look up a memory stored in the user's browser.",
    args_schema=MemoryGetInput,
)
def memory_get(key: str, runtime: ToolRuntime) -> Any:
    return _interrupt_for_client("memory_get", {"key": key}, runtime)


@tool(
    "geolocation_get",
    description="Get the user's current location from the browser.",
    args_schema=GeolocationGetInput,
)
def geolocation_get(runtime: ToolRuntime, save: bool = True) -> Any:
    return _interrupt_for_client(
        "geolocation_get",
        {"save": save},
        runtime,
    )

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[memory_put, memory_get, geolocation_get],
    checkpointer=MemorySaver(),
)
```

Mỗi tool sẽ interrupt kèm một payload có cấu trúc mà frontend có thể xử lý, sau đó
trả về giá trị được cung cấp khi lượt chạy được resume. Hãy phản chiếu cùng tên tool và
schema ở phía client để frontend có thể gắn các phần triển khai vào.

```ts tools.ts
import * as z from "zod";
import { tool } from "langchain";

// Mirror the Python tool names and schemas on the client.
export const memoryPut = tool({
  name: "memory_put",
  description: "Store a memory in the user's browser.",
  schema: z.object({
    key: z.string(),
    value: z.unknown(),
  }),
});

export const memoryGet = tool({
  name: "memory_get",
  description: "Look up a memory stored in the user's browser.",
  schema: z.object({
    key: z.string(),
  }),
});

export const geolocationGet = tool({
  name: "geolocation_get",
  description: "Get the user's current location from the browser.",
  schema: z.object({
    save: z.boolean().optional(),
  }),
});
```

## Triển khai hành vi trình duyệt (browser behavior)

Đặt phần hành vi chỉ dành riêng cho client trong một module tách biệt và gắn nó bằng
`.implement(...)`. Playground thật đi kèm một bộ lưu trữ IndexedDB đầy đủ hơn với
tìm kiếm, liệt kê, hết hạn (expiration), và các thao tác xóa. Ví dụ dưới đây minh họa
cùng hình dạng đó ở mức cao hơn:

```ts impl.ts
import {
  geolocationGet as geolocationGetDefinition,
  memoryGet as memoryGetDefinition,
  memoryPut as memoryPutDefinition,
} from "./tools";

async function saveMemory(key: string, value: unknown) {
  localStorage.setItem(`agent-memory:${key}`, JSON.stringify(value));
}

async function getMemory(key: string) {
  const value = localStorage.getItem(`agent-memory:${key}`);
  return value ? JSON.parse(value) : null;
}

export const memoryPut = memoryPutDefinition.implement(async ({ key, value }) => {
  await saveMemory(key, value);
  return { success: true, key };
});

export const memoryGet = memoryGetDefinition.implement(async ({ key }) => {
  const value = await getMemory(key);
  return value === null ? { found: false, key } : { found: true, key, value };
});

export const geolocationGet = geolocationGetDefinition.implement(
  async ({ save = true }) => {
    const position = await new Promise<GeolocationPosition>((resolve, reject) =>
      navigator.geolocation.getCurrentPosition(resolve, reject),
    );

    const location = {
      latitude: position.coords.latitude,
      longitude: position.coords.longitude,
      accuracy: position.coords.accuracy,
    };

    if (save) {
      await saveMemory("user_location", location);
    }

    return location;
  },
);
```

## Nối các phần triển khai vào `useStream`

Truyền các tool đã triển khai vào `useStream`. Khi agent phát ra một lệnh gọi tool khớp,
hook sẽ chạy phần triển khai phía client và resume lượt chạy giúp bạn.

Định nghĩa một interface TypeScript khớp với state schema của agent và truyền nó như
một type parameter cho `useStream` để truy cập an toàn về kiểu (type-safe) vào các giá trị state:

```ts types.ts
export interface AgentState {
  messages: BaseMessage[];
}
```

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";

    import { geolocationGet, memoryGet, memoryPut } from "./impl";
    import type { AgentState } from "./types";

    const AGENT_URL = "http://localhost:2024";

    export function Chat() {
      const stream = useStream<AgentState>({
        apiUrl: AGENT_URL,
        assistantId: "headless_tools",
        tools: [memoryPut, memoryGet, geolocationGet],
      });

      return <ChatView messages={stream.messages} toolCalls={stream.toolCalls} />;
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";

    import { geolocationGet, memoryGet, memoryPut } from "./impl";
    import type { AgentState } from "./types";

    const AGENT_URL = "http://localhost:2024";

    const stream = useStream<AgentState>({
      apiUrl: AGENT_URL,
      assistantId: "headless_tools",
      tools: [memoryPut, memoryGet, geolocationGet],
    });
    </script>

    <template>
      <ChatView
        :messages="stream.messages.value"
        :tool-calls="stream.toolCalls.value"
      />
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";

      import { geolocationGet, memoryGet, memoryPut } from "./impl";
      import type { AgentState } from "./types";

      const AGENT_URL = "http://localhost:2024";

      const { messages, toolCalls } = useStream<AgentState>({
        apiUrl: AGENT_URL,
        assistantId: "headless_tools",
        tools: [memoryPut, memoryGet, geolocationGet],
      });
    </script>

    <ChatView messages={$messages} toolCalls={$toolCalls} />
    ```

=== "Angular"
    ```ts
    import { Component } from "@angular/core";
    import { useStream } from "@langchain/angular";

    import { geolocationGet, memoryGet, memoryPut } from "./impl";
    import type { AgentState } from "./types";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-chat",
      template: `
        <app-chat-view
          [messages]="stream.messages()"
          [toolCalls]="stream.toolCalls()"
        />
      `,
    })
    export class ChatComponent {
      stream = useStream<AgentState>({
        apiUrl: AGENT_URL,
        assistantId: "headless_tools",
        tools: [memoryPut, memoryGet, geolocationGet],
      });
    }
    ```

## Render hoạt động của tool ngay trong luồng hội thoại (inline)

Playground render mỗi thao tác bộ nhớ hoặc geolocation thành một thẻ (card) riêng và
giữ một panel thống kê bộ nhớ nhỏ gần ô nhập liệu. Bước quan trọng là khớp mỗi
mục trong `stream.toolCalls` trở lại với AI message đã kích hoạt nó:

```tsx
import type { ToolCallWithResult, DefaultToolCall } from "@langchain/react";

function Message({ message, toolCalls }: {
  message: AIMessage,
  toolCalls: ToolCallWithResult[]
}) {
  const messageToolCalls = toolCalls.filter((tc) =>
    message.tool_calls?.some((call) => call.id === tc.call.id),
  );

  return (
    <div>
      {message.text && <p>{message.text}</p>}
      {messageToolCalls.map((tc) => (
        <HeadlessToolCard key={tc.call.id} toolCall={tc} />
      ))}
    </div>
  );
}
```

Cách này đặc biệt hiệu quả khi kết hợp với các pattern UI phong phú hơn từ
[Tool calling](https://docs.langchain.com/oss/python/langchain/frontend/tool-calling), nơi mỗi kết quả tool có thể
render thành một thẻ chuyên biệt thay vì JSON thô.

## Trường hợp sử dụng

Hãy dùng công cụ headless khi công việc phụ thuộc vào các API hoặc dữ liệu chỉ tồn tại ở
client:

* Bộ nhớ cục bộ trong IndexedDB hoặc `localStorage`
* Các API thiết bị như geolocation, clipboard, camera, hoặc trình chọn tệp
* Canvas, audio, hoặc các primitive render khác chỉ có ở trình duyệt
* Dữ liệu nhạy cảm về quyền riêng tư cần được giữ trên thiết bị của người dùng
* Các hành động UI cần truy cập trực tiếp vào state trong bộ nhớ (in-memory) của frontend

## Thực hành tốt nhất (Best practices)

* Giữ các tool nhỏ và có kiểu (typed) rõ ràng. Ưu tiên nhiều tool hẹp thay vì
  một tool tổng quát kiểu "chạy mã trình duyệt bất kỳ".
* Trả về kết quả có thể tuần tự hóa JSON (JSON-serializable). Đừng cố trả về DOM node, file
  handle, hay các đối tượng trình duyệt không thể tuần tự hóa khác.
* Chia sẻ định nghĩa, tách riêng phần triển khai. Agent và client nên thống nhất
  về tên tool và schema, nhưng chỉ client mới nên nạp các API trình duyệt.
* Hiển thị trạng thái tool trên UI. Dùng `stream.toolCalls` và `onTool` để hiển thị
  các trạng thái đang chờ (pending), thành công, và lỗi.
* Thêm bước review khi cần thiết. Với các hành động phía client nhạy cảm, hãy kết hợp pattern này
  với [Human-in-the-loop](human-in-the-loop.md).
