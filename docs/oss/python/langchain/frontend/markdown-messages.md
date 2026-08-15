# Markdown messages

> Render phản hồi của LLM thành markdown định dạng phong phú, có hỗ trợ streaming đúng cách

LLM tự nhiên sinh ra text theo định dạng markdown, gồm heading, list, code block, bảng, và định dạng inline. Render nội dung này thành plain text sẽ lãng phí cấu trúc mà model đã cung cấp. Trang này chỉ cho bạn cách parse và render markdown theo thời gian thực khi nó streaming từ agent, trên tất cả các framework frontend chính.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/markdown-messages

## Cách render markdown hoạt động

Pipeline render có ba bước:

1. **Nhận (Receive):** [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) tích luỹ text được streaming vào `msg.text` trên mỗi AI message, cập nhật một cách reactive khi token mới đến.
2. **Parse:** một markdown parser chuyển text thô thành HTML (hoặc cây phần tử React). Việc này chạy trên mỗi lần cập nhật nhưng đủ nhanh cho nội dung dài cỡ chat (dưới 5ms cho một message 5 KB).
3. **Render:** kết quả đã parse được render vào DOM. React dùng virtual DOM diffing; Vue và Svelte dùng `v-html` / `{@html}` với HTML đã sanitize.

## Thiết lập `useStream`

Pattern markdown dùng một chat agent đơn giản, không cần cấu hình đặc biệt. Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với URL agent và assistant ID của bạn.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";
    import { AIMessage, HumanMessage } from "langchain";

    const AGENT_URL = "http://localhost:2024";

    export function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "simple_agent",
      });

      return (
        <div>
          {stream.messages.map((msg) => {
            if (AIMessage.isInstance(msg)) {
              return <Markdown key={msg.id}>{msg.text}</Markdown>;
            }
            if (HumanMessage.isInstance(msg)) {
              return <p key={msg.id}>{msg.text}</p>;
            }
          })}
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { useStream } from "@langchain/vue";
    import { AIMessage, HumanMessage } from "langchain";

    const AGENT_URL = "http://localhost:2024";

    const stream = useStream<typeof myAgent>({
      apiUrl: AGENT_URL,
      assistantId: "simple_agent",
    });
    </script>

    <template>
      <div>
        <template v-for="msg in stream.messages.value" :key="msg.id">
          <Markdown v-if="AIMessage.isInstance(msg)">{{ msg.text }}</Markdown>
          <p v-else-if="HumanMessage.isInstance(msg)">{{ msg.text }}</p>
        </template>
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";
      import { AIMessage, HumanMessage } from "langchain";

      const AGENT_URL = "http://localhost:2024";

      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "simple_agent",
      });
    </script>

    <div>
      {#each stream.messages as msg (msg.id)}
        {#if AIMessage.isInstance(msg)}
          <Markdown content={msg.text} />
        {:else if HumanMessage.isInstance(msg)}
          <p>{msg.text}</p>
        {/if}
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
          <app-markdown [content]="msg.text" />
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

## Chọn thư viện markdown

Mỗi framework có một lựa chọn tự nhiên để render markdown:

| Framework | Thư viện | Đầu ra | Vì sao |
| --- | --- | --- | --- |
| React | `react-markdown` + `remark-gfm` | Phần tử React | Component-based, virtual DOM diffing, không cần `dangerouslySetInnerHTML` |
| Vue | `marked` + `dompurify` | HTML đã sanitize qua `v-html` | Nhẹ, nhanh, hỗ trợ GFM sẵn |
| Svelte | `marked` + `dompurify` | HTML đã sanitize qua `{@html}` | Giống Vue, API nhất quán |
| Angular | `marked` + `dompurify` | HTML đã sanitize qua `[innerHTML]` | Giống Vue/Svelte |

!!! tip "Mẹo"
    `react-markdown` của React chuyển markdown trực tiếp thành phần tử React, nên không cần sanitize HTML. Không có `dangerouslySetInnerHTML` nào liên quan cả. Với Vue, Svelte, và Angular, luôn sanitize HTML đã parse bằng `dompurify` trước khi render.

## Xây dựng component Markdown

=== "React"
    ```tsx
    import ReactMarkdown from "react-markdown";
    import remarkGfm from "remark-gfm";

    export function Markdown({ children }: { children: string }) {
      return (
        <div className="markdown-content">
          <ReactMarkdown remarkPlugins={[remarkGfm]}>
            {children}
          </ReactMarkdown>
        </div>
      );
    }
    ```

=== "Vue"
    ```vue
    <script setup lang="ts">
    import { computed, useSlots } from "vue";
    import { marked } from "marked";
    import DOMPurify from "dompurify";

    marked.setOptions({ gfm: true, breaks: true });

    const slots = useSlots();

    const html = computed(() => {
      const slot = slots.default?.();
      const text = slot
        ?.map((vnode) =>
          typeof vnode.children === "string" ? vnode.children : ""
        )
        .join("") ?? "";
      if (!text) return "";
      return DOMPurify.sanitize(marked.parse(text) as string);
    });
    </script>

    <template>
      <div class="markdown-content" v-html="html" />
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { marked } from "marked";
      import DOMPurify from "dompurify";

      let { content }: { content: string } = $props();

      marked.setOptions({ gfm: true, breaks: true });

      let html = $derived.by(() => {
        if (!content) return "";
        return DOMPurify.sanitize(marked.parse(content) as string);
      });
    </script>

    <div class="markdown-content">
      {@html html}
    </div>
    ```

=== "Angular"
    ```ts
    import { Component, Input, computed, signal } from "@angular/core";
    import { marked } from "marked";
    import DOMPurify from "dompurify";

    marked.setOptions({ gfm: true, breaks: true });

    @Component({
      selector: "app-markdown",
      template: `<div class="markdown-content" [innerHTML]="html()"></div>`,
    })
    export class MarkdownComponent {
      @Input() set content(value: string) {
        this._content.set(value);
      }

      private _content = signal("");

      html = computed(() => {
        const text = this._content();
        if (!text) return "";
        return DOMPurify.sanitize(marked.parse(text) as string);
      });
    }
    ```

## Sanitize đầu ra HTML

Khi render markdown đã parse thành HTML thô (`v-html`, `{@html}`, `[innerHTML]`), bạn phải sanitize đầu ra để ngăn cross-site scripting (XSS). Phản hồi của LLM có thể chứa text bất kỳ, kể cả markup mà một markdown parser có thể biến thành HTML thực thi được.

Dùng `dompurify` để loại bỏ các phần tử nguy hiểm:

```ts
import DOMPurify from "dompurify";

const safeHtml = DOMPurify.sanitize(rawHtml);
```

DOMPurify loại bỏ thẻ `<script>`, thuộc tính `onclick`, URL `javascript:`, và các vector XSS khác trong khi vẫn giữ nguyên đầu ra markdown an toàn như heading, list, code block, bảng, và link.

!!! note "Ghi chú"
    `react-markdown` của React không cần `dompurify` vì nó tạo ra phần tử React trực tiếp, không có việc chèn HTML thô nào liên quan.

## Cân nhắc về streaming

`useStream` cập nhật `msg.text` một cách reactive khi mỗi token đến. Component markdown parse lại trên mỗi lần cập nhật. Với các message chat thông thường, điều này có hiệu năng tốt:

* `marked` parse ở tốc độ khoảng 1 MB/giây. Một message 5 KB mất dưới 5ms
* Pipeline `react-markdown` + remark cũng nhanh tương tự với nội dung dài cỡ chat
* Layout engine của trình duyệt xử lý cập nhật DOM hiệu quả

Với các phản hồi rất dài (trên 50 KB), cân nhắc các tối ưu sau:

* **Throttle lần render:** dùng `requestAnimationFrame` để gộp cập nhật ở 60fps thay vì render lại trên mỗi token
* **Parse tăng dần (incremental):** chỉ parse nội dung mới và nối vào buffer đã render (nâng cao, thường không cần thiết cho UI chat)

!!! info "Thông tin"
    Với hầu hết ứng dụng chat, cách tiếp cận đơn giản là parse lại toàn bộ message trên mỗi token là đủ. Chỉ tối ưu khi bạn quan sát thấy giật hình hoặc rớt frame với message rất dài.

## Thực hành tốt nhất

* **Luôn sanitize:** khi dùng `v-html`, `{@html}`, hoặc `[innerHTML]`, luôn chạy đầu ra đã parse qua `dompurify`. Không bao giờ tin tưởng HTML thô từ một markdown parser được nạp bằng đầu ra của LLM.
* **Bật GFM:** GitHub Flavored Markdown thêm bảng, strikethrough, task list, và autolink. Đây là các tính năng LLM thường dùng.
* **Xử lý nội dung rỗng:** kiểm tra chuỗi rỗng trước khi parse để tránh render các container rỗng.
* **Dùng `breaks: true`:** bật chuyển đổi xuống dòng để các newline đơn trong đầu ra LLM render thành `<br>` thay vì bị bỏ qua. LLM thường dùng newline đơn để phân tách trực quan.
* **Style phù hợp ngữ cảnh chat:** dùng margin và kích thước gọn phù hợp với bong bóng chat, không phải layout bài viết full-width.
* **Test với nội dung phong phú:** kiểm tra render với heading, list lồng nhau, code block dòng dài, bảng rộng, và blockquote để phát hiện lỗi tràn hoặc layout.
