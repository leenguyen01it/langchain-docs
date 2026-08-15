# Generative UI khai báo (Declarative generative UI)

> Kết hợp các giao diện do agent tạo ra từ một catalog component đã đăng ký sẵn, sử dụng json-render và A2UI

## Tổng quan

Generative UI khai báo nằm ở giữa
[phổ generative UI](generative-ui-overview.md). Agent phát ra một
đặc tả (specification) có cấu trúc, và phía frontend sẽ ghép giao diện lại từ một
catalog các component mà bạn đã đăng ký trước. Thay vì render các câu trả lời dạng
văn bản trong bong bóng chat, đầu ra của agent **chính là** UI: form, card, dashboard,
và nhiều hơn nữa. Bạn định nghĩa những component nào khả dụng (gọi là "catalog"), và
agent sẽ ghép chúng thành một cây UI hợp lệ.

Catalog chính là rào chắn (guardrail) giúp cách tiếp cận này an toàn: agent có thể sắp
xếp và kết hợp các component của bạn một cách tự do, nhưng không thể vượt ra ngoài tập
mà bạn đã cho phép. Điều này cân bằng giữa tính sáng tạo và tính dự đoán được. Đây là
nơi "phần đuôi dài" (long tail) tồn tại, đánh đổi độ hoàn hảo từng pixel để lấy độ
rộng bao phủ, phù hợp với các tương tác thứ cấp, công cụ nội bộ, và dashboard, nơi mà
việc hiển thị được thứ gì đó hữu ích quan trọng hơn là kiểm soát chính xác tuyệt đối.
Trang này trình bày generative UI khai báo với
[json-render](https://json-render.dev), một framework generative UI
định nghĩa các catalog component, tạo ra đặc tả bằng AI, và render chúng một cách an
toàn trên React, Vue, Svelte, và Angular. Với đặc tả A2UI của Google (tích hợp qua
CopilotKit), xem [A2UI](#a2ui-mot-dac-ta-khai-bao-thay-the) bên dưới.

## Khi nào nên dùng cách tiếp cận này

Dùng generative UI khai báo cho "phần đuôi dài" của sản phẩm, nơi
agent có thể ghép các layout mà bạn không lường trước hết trong khi vẫn ở trong một
tập component mà bạn đã phê duyệt: các tương tác thứ cấp, công cụ nội bộ, và
dashboard. Khi một bề mặt (surface) có lưu lượng cao hoặc quan trọng với thương hiệu và
cần phải chính xác tuyệt đối, hãy chuyển sang
[controlled generative UI](controlled-generative-ui.md). Khi
bạn muốn tạo giao diện bên ngoài ứng dụng của mình, hãy chuyển sang
[open-ended generative UI](https://docs.langchain.com/oss/python/langchain/frontend/open-ended-generative-ui).

## Cách hoạt động

1. **Định nghĩa một catalog**: khai báo agent AI được phép dùng những component nào, kèm theo props có kiểu dữ liệu
2. **Prompt cho AI**: mô tả UI bạn muốn bằng ngôn ngữ tự nhiên
3. **AI tạo ra một đặc tả**: một tài liệu JSON mô tả cây component
4. **Render an toàn**: `Renderer` của json-render render đặc tả đó bằng các component của bạn

Catalog đóng vai trò như một rào chắn: AI chỉ có thể dùng những component bạn đã
định nghĩa, với các props khớp schema của bạn. Đầu ra luôn dự đoán được và an toàn.

## Định nghĩa một catalog component

Catalog mô tả mọi component mà AI được phép dùng. Mỗi component có một
schema Zod cho props của nó và một mô tả mà AI đọc để hiểu khi nào nên
dùng nó:

```ts
import { defineCatalog } from "@json-render/core";
import { schema } from "@json-render/react/schema";
import { z } from "zod";

const catalog = defineCatalog(schema, {
  components: {
    Card: {
      description: "A card container with optional title and padding",
      props: z.object({
        title: z.string().optional(),
        padding: z.enum(["sm", "md", "lg"]).optional(),
      }),
    },
    Stack: {
      description: "Layout children vertically or horizontally with consistent spacing",
      props: z.object({
        direction: z.enum(["vertical", "horizontal"]).optional(),
        gap: z.enum(["sm", "md", "lg"]).optional(),
      }),
    },
    TextInput: {
      description: "A text input field with optional label and placeholder",
      props: z.object({
        label: z.string().optional(),
        placeholder: z.string().optional(),
        type: z.enum(["text", "email", "password", "number", "textarea"]).optional(),
      }),
    },
    Button: {
      description: "A clickable button with label and style variants",
      props: z.object({
        label: z.string(),
        variant: z.enum(["primary", "secondary", "ghost", "link"]).optional(),
        fullWidth: z.boolean().optional(),
      }),
    },
  },
  actions: {},
});
```

!!! tip "Mẹo"
    Giữ catalog tập trung. Chỉ đưa vào những component mà AI cần cho use case đó.
    Một catalog nhỏ gọn cho kết quả tốt hơn so với cách tiếp cận "cho tất cả vào một chỗ".

## Xây dựng một registry component

Registry ánh xạ mỗi component trong catalog tới phần triển khai render thực tế của
nó. Dùng `defineRegistry` để có được các liên kết an toàn về kiểu (type-safe) giữa
props trong catalog và các hàm component của bạn:

=== "React"

    ```tsx
    import { defineRegistry, Renderer, JSONUIProvider } from "@json-render/react";

    const { registry } = defineRegistry(catalog, {
      components: {
        Card: ({ props, children }) => (
          <div className="card">
            {props.title && <h2>{props.title}</h2>}
            {children}
          </div>
        ),
        Stack: ({ props, children }) => (
          <div className={`stack stack-${props.direction ?? "vertical"} gap-${props.gap ?? "md"}`}>
            {children}
          </div>
        ),
        TextInput: ({ props }) => (
          <div>
            {props.label && <label>{props.label}</label>}
            <input type={props.type ?? "text"} placeholder={props.placeholder} />
          </div>
        ),
        Button: ({ props }) => (
          <button className={props.variant ?? "primary"}>
            {props.label}
          </button>
        ),
      },
    });
    ```

=== "Vue"

    ```vue
    <script setup lang="ts">
    import { h } from "vue";
    import { defineRegistry, Renderer, JSONUIProvider } from "@json-render/vue";

    const { registry } = defineRegistry(catalog, {
      components: {
        Card: ({ props, children }) =>
          h("div", { class: "card" }, [
            props.title ? h("h2", null, props.title) : null,
            children,
          ]),
        Stack: ({ props, children }) =>
          h("div", { class: `stack stack-${props.direction ?? "vertical"} gap-${props.gap ?? "md"}` }, children),
        TextInput: ({ props }) =>
          h("div", null, [
            props.label ? h("label", null, props.label) : null,
            h("input", { type: props.type ?? "text", placeholder: props.placeholder }),
          ]),
        Button: ({ props }) =>
          h("button", { class: props.variant ?? "primary" }, props.label),
      },
    });
    </script>
    ```

## Kết nối với agent

Agent dùng structured output để trả về một đặc tả json-render. Thiết lập
`useStream` với assistant ID của agent, sau đó trích xuất đặc tả từ
`tool_calls` của AI message:

=== "React"

    ```tsx
    import { useStream } from "@langchain/react";
    import { AIMessage } from "langchain";

    function GenerativeUI() {
      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "generative_ui",
      });

      const aiMessage = stream.messages.find(AIMessage.isInstance);
      const rawSpec = aiMessage?.tool_calls?.[0]?.args;

      // ... filter and render (see streaming section below)
    }
    ```

=== "Vue"

    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";
    import { AIMessage } from "langchain";
    import { computed } from "vue";

    const stream = useStream<typeof myAgent>({
      apiUrl: "http://localhost:2024",
      assistantId: "generative_ui",
    });

    const aiMessage = computed(() => stream.messages.value.find(AIMessage.isInstance));
    const rawSpec = computed(() => aiMessage.value?.tool_calls?.[0]?.args);
    </script>
    ```

=== "Svelte"

    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";
      import { AIMessage } from "langchain";

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "generative_ui",
      });

      const aiMessage = $derived(stream.messages.find((m) => AIMessage.isInstance(m)));
      const rawSpec = $derived(aiMessage?.tool_calls?.[0]?.args);
    </script>
    ```

=== "Angular"

    ```ts
    import { Component } from "@angular/core";
    import { injectStream } from "@langchain/angular";
    import { AIMessage } from "langchain";

    @Component({
      selector: "app-generative-ui",
      template: `...`,
    })
    export class GenerativeUIComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "generative_ui",
      });

      get rawSpec() {
        const ai = this.stream.messages().find(AIMessage.isInstance);
        return ai?.tool_calls?.[0]?.args;
      }
    }
    ```

## Streaming và render tuần tự

Trong lúc streaming, đặc tả được xây dựng dần từng phần. Các element đến từng cái
một và ban đầu có thể thiếu `type` hoặc `props`. Hãy lọc để chỉ giữ lại những
element hoàn chỉnh và truyền `loading={true}` vào `Renderer`, việc này báo cho nó bỏ
qua một cách im lặng những children chưa đến. UI sẽ được xây dần từng component một:

```tsx
/*
 * Filter the streamed spec to only include elements with valid type/props,
 * enabling progressive rendering as the AI response builds up. Passing
 * loading={true} to the Renderer tells it to skip missing children silently.
 */
const spec = (() => {
  if (!rawSpec?.root || !rawSpec?.elements) return null;
  const rootEl = rawSpec.elements[rawSpec.root];
  if (!rootEl?.type || rootEl?.props == null) return null;

  const safeElements = {};
  for (const [key, el] of Object.entries(rawSpec.elements)) {
    if (el?.type && el?.props != null) {
      safeElements[key] = el;
    }
  }
  return { root: rawSpec.root, elements: safeElements };
})();

return (
  <>
    {spec && (
      <JSONUIProvider registry={registry}>
        <Renderer spec={spec} registry={registry} loading={stream.isLoading} />
      </JSONUIProvider>
    )}
  </>
);
```

!!! note "Ghi chú"
    `JSONUIProvider` là bắt buộc để thiết lập các context provider nội bộ của
    json-render (state, visibility, validation, actions). Component `Renderer`
    phải được render bên trong nó.

## Định dạng đặc tả

Agent AI tạo ra một đặc tả JSON dạng phẳng với một khóa `root` trỏ tới
element gốc và một map `elements` chứa tất cả các component:

```json
{
  "root": "login-card",
  "elements": {
    "login-card": {
      "type": "Card",
      "props": { "title": "Login" },
      "children": ["login-stack"]
    },
    "login-stack": {
      "type": "Stack",
      "props": { "direction": "vertical", "gap": "md" },
      "children": ["email-input", "password-input", "submit-btn"]
    },
    "email-input": {
      "type": "TextInput",
      "props": { "label": "Email", "placeholder": "Enter your email", "type": "email" },
      "children": []
    },
    "password-input": {
      "type": "TextInput",
      "props": { "label": "Password", "placeholder": "Enter your password", "type": "password" },
      "children": []
    },
    "submit-btn": {
      "type": "Button",
      "props": { "label": "Sign In", "variant": "primary", "fullWidth": true },
      "children": []
    }
  }
}
```

Mỗi element tham chiếu tới các con của nó bằng ID, và các element lá (leaf) như
`TextInput` và `Button` có mảng `children` rỗng.

## A2UI: một đặc tả khai báo thay thế

Một cách để mô tả một giao diện theo hướng khai báo là json-render. A2UI là một
cách khác: đặc tả generative UI theo hướng streaming-first, khai báo của Google,
được tích hợp qua CopilotKit. Giống json-render, nó ghép giao diện từ các component
mà bạn đăng ký, nên agent vẫn ở trong rào chắn mà bạn định nghĩa. A2UI có hai
biến thể:

* **Dynamic schema**: một model phụ tạo ra toàn bộ giao diện, bao gồm cả
  schema, dữ liệu, và layout, từ cuộc hội thoại, để đạt độ linh hoạt tối đa.
* **Fixed schema**: cây component được định nghĩa ở phía frontend và agent
  chỉ stream dữ liệu vào đó, để có tốc độ render nhanh nhất và dự đoán được nhất.

Để biết chi tiết, xem tài liệu của CopilotKit về [A2UI](https://docs.copilotkit.ai/generative-ui/a2ui),
[dynamic schema](https://docs.copilotkit.ai/generative-ui/a2ui/dynamic-schema), và
[fixed schema](https://docs.copilotkit.ai/generative-ui/a2ui/fixed-schema). Để
kết nối CopilotKit với một LangGraph deployment, xem [CopilotKit](integrations/copilotkit.md).

## Thực hành tốt nhất

* **Dùng mô tả component rõ ràng**: AI dùng những mô tả này để hiểu khi nào
  nên dùng mỗi component. Mô tả rõ ràng giúp tạo UI tốt hơn.
* **Xác thực trước khi render**: luôn kiểm tra rằng element có `type` hợp lệ và
  `props` khác null trước khi truyền vào Renderer, vì streaming trả về dữ liệu chưa đầy đủ theo từng phần.
* **Thiết kế cho streaming**: truyền `loading={true}` trong lúc streaming để
  Renderer xử lý mượt mà những children chưa đến. Người dùng thấy UI được xây dựng
  theo thời gian thực thay vì phải chờ toàn bộ phản hồi.
* **Style bằng design token**: dùng CSS custom property để các component được
  render tự động thích ứng với theme sáng và tối.
* **Bọc bằng JSONUIProvider**: `Renderer` phải nằm bên trong một `JSONUIProvider`
  để truy cập context nội bộ của json-render cho state, visibility, và actions.

## Xem thêm

* [Tổng quan Generative UI](generative-ui-overview.md)
* [Controlled generative UI](controlled-generative-ui.md)
* [Open-ended generative UI](https://docs.langchain.com/oss/python/langchain/frontend/open-ended-generative-ui)
