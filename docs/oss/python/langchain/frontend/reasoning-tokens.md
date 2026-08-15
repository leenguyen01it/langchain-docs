# Reasoning tokens

> Hiển thị quá trình suy nghĩ và lập luận (reasoning) của model trong các khối có thể thu gọn

Reasoning tokens phơi bày quá trình suy nghĩ nội bộ của các model tiên tiến như GPT-5 của OpenAI và Claude với extended thinking của Anthropic. Các model này sinh ra các khối nội dung có cấu trúc, tách phần reasoning khỏi câu trả lời cuối cùng, cho phép bạn xây dựng UI hiển thị *cách* model đi đến kết quả đó.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/reasoning-tokens

## Reasoning token là gì?

Khi các model có khả năng reasoning xử lý một prompt, chúng sinh ra hai loại nội dung riêng biệt:

1. **Khối reasoning**: chuỗi suy nghĩ nội bộ của model (chain-of-thought), phân rã vấn đề, và phân tích từng bước
2. **Khối text**: câu trả lời cuối cùng, đã hoàn chỉnh, trình bày cho người dùng

Chúng được truyền dưới dạng các khối nội dung có kiểu (typed) bên trong một `AIMessage`, truy cập được qua thuộc tính `contentBlocks`:

```ts
// Khối reasoning
{ type: "reasoning", reasoning: "Let me think about this step by step..." }

// Khối text
{ type: "text", text: "The answer is 42." }
```

!!! note "Ghi chú"
    Không phải model nào cũng sinh ra reasoning token. Pattern này áp dụng riêng cho các model hỗ trợ extended thinking hoặc đầu ra chain-of-thought. Chat model tiêu chuẩn chỉ trả về khối text.

## Trường hợp sử dụng

* **Minh bạch**: cho người dùng thấy quá trình lập luận của model để tăng độ tin cậy vào câu trả lời
* **Debug**: kiểm tra quá trình suy nghĩ của model để xác định chỗ nó đi sai
* **Công cụ giáo dục**: dạy học sinh cách giải quyết vấn đề bằng cách phơi bày cách một AI tiếp cận câu hỏi
* **Hỗ trợ ra quyết định**: cho phép chuyên gia trong lĩnh vực xác thực lập luận đằng sau các đề xuất
* **Đảm bảo chất lượng**: audit các chuỗi reasoning để tuân thủ trong các ngành có quy định chặt

## Trích xuất khối reasoning và text

Mảng `contentBlocks` trên một `AIMessage` chứa tất cả các khối theo đúng thứ tự chúng được sinh ra. Lọc chúng theo `type` để tách reasoning khỏi text:

```ts
import { AIMessage } from "langchain";

function extractBlocks(msg: AIMessage) {
  const reasoningBlocks = msg.contentBlocks
    .filter((b) => b.type === "reasoning")
    .map((b) => b.reasoning);

  const textBlocks = msg.contentBlocks
    .filter((b) => b.type === "text")
    .map((b) => b.text);

  return {
    reasoning: reasoningBlocks.join(""),
    text: textBlocks.join(""),
  };
}
```

Một message đơn lẻ có thể chứa nhiều khối reasoning (ví dụ nếu model tạm dừng reasoning, sinh ra text một phần, rồi reasoning tiếp). Nối chúng lại cho bạn toàn bộ quá trình suy nghĩ.

## Truy cập message từ `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với agent có khả năng reasoning của bạn và lặp qua `stream.messages` trong UI chat. Rẽ nhánh theo `HumanMessage.isInstance` và `AIMessage.isInstance`, sau đó truyền mỗi message của assistant vào một component đọc `contentBlocks` và tách reasoning khỏi text. Đặt `isStreaming` trên message cuối cùng khi `stream.isLoading` là true để khối thinking cập nhật theo khi token đến.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";
    import { AIMessage, HumanMessage } from "langchain";

    function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "reasoning",
      });

      return (
        <div className="messages">
          {stream.messages.map((msg, i) => {
            if (HumanMessage.isInstance(msg)) {
              return <HumanBubble key={i} text={msg.text} />;
            }
            if (AIMessage.isInstance(msg)) {
              return (
                <AIResponse
                  key={i}
                  message={msg}
                  isStreaming={stream.isLoading && i === stream.messages.length - 1}
                />
              );
            }
            return null;
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

    const stream = useStream<typeof myAgent>({
      apiUrl: "http://localhost:2024",
      assistantId: "reasoning",
    });
    </script>

    <template>
      <div class="messages">
        <template v-for="(msg, i) in stream.messages.value" :key="i">
          <HumanBubble v-if="HumanMessage.isInstance(msg)" :text="msg.text" />
          <AIResponse
            v-else-if="AIMessage.isInstance(msg)"
            :message="msg"
            :isStreaming="stream.isLoading.value && i === stream.messages.value.length - 1"
          />
        </template>
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";
      import { AIMessage, HumanMessage } from "langchain";

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "reasoning",
      });
    </script>

    <div class="messages">
      {#each stream.messages as msg, i}
        {#if HumanMessage.isInstance(msg)}
          <HumanBubble text={msg.text} />
        {:else if AIMessage.isInstance(msg)}
          <AIResponse
            message={msg}
            isStreaming={stream.isLoading && i === stream.messages.length - 1}
          />
        {/if}
      {/each}
    </div>
    ```

=== "Angular"
    ```ts
    import { Component } from "@angular/core";
    import { injectStream } from "@langchain/angular";
    import { AIMessage, HumanMessage } from "langchain";

    @Component({
      selector: "app-chat",
      template: `
        <div class="messages">
          @for (msg of stream.messages(); track $index) {
            @if (isHuman(msg)) {
              <human-bubble [text]="msg.text" />
            } @else if (isAI(msg)) {
              <ai-response
                [message]="msg"
                [isStreaming]="stream.isLoading() && $index === stream.messages().length - 1"
              />
            }
          }
        </div>
      `,
    })
    export class ChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "reasoning",
      });

      isHuman = HumanMessage.isInstance;
      isAI = AIMessage.isInstance;
    }
    ```

## Xây dựng component ThinkingBubble

`ThinkingBubble` trình bày reasoning token trong một container có thể thu gọn, khác biệt về mặt hình ảnh. Người dùng có thể mở rộng để xem toàn bộ quá trình suy nghĩ hoặc thu gọn để tập trung vào câu trả lời cuối cùng.

```tsx
import { useState } from "react";

function ThinkingBubble({
  reasoning,
  isStreaming,
}: {
  reasoning: string;
  isStreaming: boolean;
}) {
  const [isExpanded, setIsExpanded] = useState(false);

  const charCount = reasoning.length;
  const previewLength = 120;
  const preview =
    reasoning.length > previewLength
      ? reasoning.slice(0, previewLength) + "..."
      : reasoning;

  return (
    <div className="thinking-bubble">
      <button
        className="thinking-header"
        onClick={() => setIsExpanded(!isExpanded)}
      >
        <span className="thinking-icon">
          {isStreaming ? (
            <span className="thinking-spinner" />
          ) : (
            "💭"
          )}
        </span>
        <span className="thinking-label">
          {isStreaming ? "Thinking..." : `Thought process (${charCount} chars)`}
        </span>
        <span className={`chevron ${isExpanded ? "expanded" : ""}`}>▶</span>
      </button>

      {isExpanded && (
        <div className="thinking-content">
          <pre>{reasoning}</pre>
        </div>
      )}

      {!isExpanded && !isStreaming && (
        <div className="thinking-preview">{preview}</div>
      )}
    </div>
  );
}
```

## Render phản hồi AI hoàn chỉnh

Kết hợp `ThinkingBubble` và một bong bóng text tiêu chuẩn thành một component `AIResponse` duy nhất:

```tsx
function AIResponse({
  message,
  isStreaming,
}: {
  message: AIMessage;
  isStreaming: boolean;
}) {
  const reasoningBlocks = message.contentBlocks
    .filter((b) => b.type === "reasoning")
    .map((b) => b.reasoning)
    .join("");

  const textBlocks = message.contentBlocks
    .filter((b) => b.type === "text")
    .map((b) => b.text)
    .join("");

  const hasReasoning = reasoningBlocks.length > 0;
  const hasText = textBlocks.length > 0;

  const isReasoningPhase = isStreaming && !hasText;
  const isTextPhase = isStreaming && hasText;

  return (
    <div className="ai-response">
      {hasReasoning && (
        <ThinkingBubble
          reasoning={reasoningBlocks}
          isStreaming={isReasoningPhase}
        />
      )}
      {hasText && (
        <div className="ai-text-bubble">
          <p>{textBlocks}</p>
          {isTextPhase && <span className="cursor-blink">▊</span>}
        </div>
      )}
    </div>
  );
}
```

## Xử lý các trường hợp biên

### Message không có reasoning

Không phải mọi AI message đều chứa khối reasoning. Khi `contentBlocks` chỉ có khối text, render một bong bóng message tiêu chuẩn không có ThinkingBubble.

### Khối reasoning rỗng

Một số model sinh ra khối reasoning rỗng như placeholder. Lọc bỏ chúng:

```ts
const meaningfulReasoning = message.contentBlocks
  .filter((b) => b.type === "reasoning" && b.reasoning.trim().length > 0);
```

### Nhiều chu kỳ reasoning-text xen kẽ

Một message đơn lẻ có thể xen kẽ giữa khối reasoning và khối text. Nếu bạn cần giữ đúng thứ tự xen kẽ này, hãy lặp qua `contentBlocks` theo đúng thứ tự thay vì gộp nhóm theo type:

```ts
message.contentBlocks.forEach((block) => {
  if (block.type === "reasoning") {
    // Render ThinkingBubble
  } else if (block.type === "text") {
    // Render đoạn text
  }
});
```

## Thực hành tốt nhất

* **Mặc định thu gọn**: chỉ hiển thị reasoning khi có yêu cầu, không mặc định mở
* **Hiển thị số ký tự**: giúp người dùng nhanh chóng nắm được lượng suy nghĩ đã bỏ ra cho phản hồi
* **Phân biệt rõ về mặt hình ảnh**: dùng màu sắc, đường viền, hoặc nền khác biệt để reasoning không bao giờ bị nhầm với câu trả lời thực sự
* **Hoạt hoạ chuyển trạng thái**: animation expand/collapse mượt mà giúp cải thiện cảm nhận chất lượng
* **Cân nhắc khả năng tiếp cận (accessibility)**: dùng đúng thuộc tính ARIA (`aria-expanded`, `aria-controls`) trên nút toggle
* **Cắt ngắn trong preview**: hiển thị một đoạn preview ngắn của reasoning khi thu gọn để người dùng quyết định có mở rộng hay không
