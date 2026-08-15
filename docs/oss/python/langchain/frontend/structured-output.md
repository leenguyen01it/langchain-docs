# Structured output

> Render phản hồi agent có cấu trúc bằng component UI tuỳ chỉnh thay vì plain text

Structured output cho phép agent trả về dữ liệu có kiểu (typed), máy đọc được, thay vì plain text. Thay vì render một chuỗi text đơn, bạn nhận được một object có cấu trúc mà bạn có thể map vào bất kỳ UI nào: card, bảng, biểu đồ, phân tách từng bước, hoặc renderer chuyên biệt theo domain.

!!! info "Demo trực tiếp"
    Bản gốc có một demo tương tác cho pattern này. Xem trực tiếp tại: https://docs.langchain.com/oss/python/langchain/frontend/structured-output

## Structured output là gì?

Thay vì trả về một phản hồi text tự do, agent dùng một tool call để trả về một object có cấu trúc tuân theo schema định trước. Điều này mang lại cho bạn:

* **Dữ liệu type-safe**: parse phản hồi thành một kiểu TypeScript đã biết
* **Kiểm soát render chính xác**: render mỗi field với cách xử lý UI riêng
* **Định dạng nhất quán**: mọi phản hồi đều tuân theo cùng một cấu trúc bất kể model bên dưới là gì

Agent thực hiện điều này bằng cách gọi một tool "structured output" mà các tham số của nó chứa dữ liệu phản hồi. Bản thân tool này không thực thi logic nào cả và thuần tuý chỉ là phương tiện để trả về dữ liệu có kiểu.

## Trường hợp sử dụng

* **So sánh sản phẩm**: bảng tính năng, danh sách ưu/nhược điểm, đánh giá
* **Phân tích dữ liệu**: tóm tắt với các metric, phân tách, và điểm nổi bật
* **Hướng dẫn từng bước**: chỉ dẫn theo thứ tự kèm mô tả và đoạn code
* **Công thức nấu ăn**: nguyên liệu, các bước, thời gian, và thông tin dinh dưỡng
* **Toán học và khoa học**: công thức render bằng LaTeX, suy luận từng bước
* **Lập kế hoạch du lịch**: lịch trình kèm ngày tháng, địa điểm, và ước tính chi phí

## Định nghĩa một schema

Định nghĩa một kiểu TypeScript cho dữ liệu có cấu trúc mà agent trả về. Hình dạng của schema này quyết định cách bạn render UI.

Dưới đây là schema math-solution được dùng trong demo nhúng:

```ts
interface MathSolution {
  problem: string; // Đề bài toán gốc
  steps: {
    explanation: string;
    latex: string; // Công thức toán hiển thị (tuỳ chọn) cho bước này
  }[]; // Suy luận từng bước
  finalAnswer: string; // Đáp án cuối cùng dạng plain text
  finalAnswerLatex: string; // Biểu diễn LaTeX của đáp án cuối cùng
}
```

Schema của bạn có thể là bất cứ hình dạng nào. Pattern này hoạt động như nhau bất kể hình dạng schema.

## Trích xuất structured output từ message

Structured output nằm trong mảng `tool_calls` của `AIMessage` cuối cùng. Trích xuất nó bằng cách tìm AI message và truy cập vào arguments của tool call đầu tiên:

```ts
import { AIMessage } from "langchain";

function extractStructuredOutput<T>(messages: any[]): T | null {
  const aiMessage = messages.find(AIMessage.isInstance);
  const toolCall = aiMessage?.tool_calls?.[0];
  if (!toolCall) return null;

  return toolCall.args as T;
}
```

!!! note "Ghi chú"
    Tool call structured output có thể chưa có `args` được điền cho đến khi agent streaming xong. Trong lúc streaming, `args` có thể chỉ được điền một phần hoặc undefined. Luôn kiểm tra tính đầy đủ trước khi render.

## Thiết lập `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với agent structured-output của bạn, sau đó đọc `stream.messages` và trích xuất payload có kiểu từ tool call của [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) mới nhất. Render UI tuỳ chỉnh của bạn khi `args` đã đầy đủ, hiển thị trạng thái loading khi `stream.isLoading` là true (tool arguments có thể streaming dần dần), và dùng `stream.submit()` để gửi prompt tiếp theo.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem Type inference cho backend [Python](overview.md#type-inference) hoặc JavaScript.

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";
    import { AIMessage } from "langchain";

    function MathSolutionChat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "structured_output_latex",
      });

      const solution = extractStructuredOutput<MathSolution>(stream.messages);

      return (
        <div>
          {!solution && !stream.isLoading && (
            <PromptInput onSubmit={(text) =>
              stream.submit({ messages: [{ type: "human", content: text }] })
            } />
          )}
          {stream.isLoading && <LoadingIndicator />}
          {solution && <SolutionCard solution={solution} />}
        </div>
      );
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
      assistantId: "structured_output_latex",
    });

    const solution = computed(() =>
      extractStructuredOutput<MathSolution>(stream.messages.value)
    );

    function handleSubmit(text: string) {
      stream.submit({ messages: [{ type: "human", content: text }] });
    }
    </script>

    <template>
      <div>
        <PromptInput v-if="!solution && !stream.isLoading" @submit="handleSubmit" />
        <LoadingIndicator v-if="stream.isLoading" />
        <SolutionCard v-if="solution" :solution="solution" />
      </div>
    </template>
    ```

=== "Svelte"
    ```svelte
    <script lang="ts">
      import { useStream } from "@langchain/svelte";
      import { AIMessage } from "langchain";

      const stream = useStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "structured_output_latex",
      });

      const solution = $derived(extractStructuredOutput<MathSolution>(stream.messages));

      function handleSubmit(text: string) {
        stream.submit({ messages: [{ type: "human", content: text }] });
      }
    </script>

    <div>
      {#if !solution && !stream.isLoading}
        <PromptInput on:submit={(e) => handleSubmit(e.detail)} />
      {/if}
      {#if stream.isLoading}
        <LoadingIndicator />
      {/if}
      {#if solution}
        <SolutionCard {solution} />
      {/if}
    </div>
    ```

=== "Angular"
    ```ts
    import { Component, computed } from "@angular/core";
    import { injectStream } from "@langchain/angular";

    @Component({
      selector: "app-math-solution-chat",
      template: `
        @if (!solution() && !stream.isLoading()) {
          <prompt-input (onSubmit)="handleSubmit($event)" />
        }
        @if (stream.isLoading()) {
          <loading-indicator />
        }
        @if (solution()) {
          <solution-card [solution]="solution()" />
        }
      `,
    })
    export class MathSolutionChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: "http://localhost:2024",
        assistantId: "structured_output_latex",
      });

      solution = computed(() =>
        extractStructuredOutput<MathSolution>(this.stream.messages())
      );

      handleSubmit(text: string) {
        this.stream.submit({
          messages: [{ type: "human", content: text }],
        });
      }
    }
    ```

## Render dữ liệu có cấu trúc

Khi đã có một object có kiểu, xây dựng một component map mỗi field vào phần tử UI phù hợp. Đây là phần cốt lõi của pattern: biến dữ liệu có cấu trúc thành một giao diện chuyên biệt.

```tsx
function LatexBlock({ latex }: { latex: string }) {
  return <div className="latex-block">{latex}</div>; // Render bằng KaTeX hoặc MathJax.
}

function SolutionCard({ solution }: { solution: MathSolution }) {
  return (
    <div className="solution-card">
      <h3>{solution.problem}</h3>
      <ol>
        {solution.steps.map((step, i) => (
          <li key={i}>
            <span>{step.explanation}</span>
            {step.latex && <LatexBlock latex={step.latex} />}
          </li>
        ))}
      </ol>
      <strong>{solution.finalAnswer}</strong>
      {solution.finalAnswerLatex && <LatexBlock latex={solution.finalAnswerLatex} />}
    </div>
  );
}
```

## Xử lý dữ liệu streaming chưa hoàn chỉnh

Trong lúc streaming, các tool call argument có thể là JSON chưa hoàn chỉnh. Bảo vệ trước tình huống này trong logic trích xuất của bạn:

```ts
function extractStructuredOutput<T>(
  messages: any[],
  requiredFields: string[] = [],
): T | null {
  const aiMessages = messages.filter(AIMessage.isInstance);
  if (aiMessages.length === 0) return null;

  const lastAI = aiMessages[aiMessages.length - 1];
  const toolCall = lastAI.tool_calls?.[0];
  if (!toolCall?.args) return null;

  const args = toolCall.args as Record<string, unknown>;
  const hasRequired = requiredFields.every(
    (field) => args[field] !== undefined
  );

  if (requiredFields.length > 0 && !hasRequired) return null;
  return args as T;
}
```

Dùng tham số `requiredFields` để chờ đến khi các field quan trọng được điền trước khi render:

```ts
const solution = extractStructuredOutput<MathSolution>(stream.messages, [
  "problem",
  "steps",
  "finalAnswer",
]);
```

## Render tăng dần trong khi streaming

Thay vì chờ structured output hoàn chỉnh, render các field ngay khi chúng đến. Điều này mang lại phản hồi tức thì cho người dùng trong khi agent vẫn đang sinh dữ liệu:

```tsx
function ProgressiveSolutionCard({ messages }: { messages: any[] }) {
  const partial = extractStructuredOutput<Partial<MathSolution>>(messages);
  if (!partial) return null;

  return (
    <div className="solution-card">
      {partial.problem && <h3>{partial.problem}</h3>}

      {partial.steps && partial.steps.length > 0 && (
        <div className="solution-steps">
          <h4>Steps</h4>
          {partial.steps.map((step, i) => (
            <div key={i} className="step">
              <div className="step-number">Step {i + 1}</div>
              <p>{step.explanation}</p>
              {step.latex && <LatexBlock latex={step.latex} />}
            </div>
          ))}
        </div>
      )}

      {partial.finalAnswer && <strong>{partial.finalAnswer}</strong>}
    </div>
  );
}
```

!!! tip "Mẹo"
    Render tăng dần hoạt động tốt khi schema có thứ tự tự nhiên từ trên xuống: đề bài, rồi các bước suy luận, rồi đáp án cuối cùng. Agent thường sinh các field theo đúng thứ tự schema, nên UI tự nhiên được điền dần.

## Thực hành tốt nhất

* **Validate trước khi render**: luôn kiểm tra các field bắt buộc đã tồn tại trước khi render, vì streaming có thể trả về dữ liệu chưa hoàn chỉnh
* **Dùng một hàm trích xuất chung**: tham số hoá logic trích xuất của bạn bằng một kiểu và danh sách field bắt buộc để nó hoạt động được với nhiều schema khác nhau
* **Render tăng dần**: hiển thị field ngay khi chúng đến thay vì chờ toàn bộ object, để người dùng thấy phản hồi tức thì
* **Cung cấp biểu diễn dự phòng (fallback)**: nếu một field hỗ trợ render phong phú (LaTeX, Markdown, biểu đồ), hãy đưa thêm một phiên bản plain-text tương đương vào schema làm fallback
* **Giữ schema phẳng khi có thể**: schema lồng sâu khó render tăng dần hơn và dễ bị lỗi hơn khi streaming chưa hoàn chỉnh
* **Khớp UI với dữ liệu**: chọn chiến lược render phù hợp nhất cho từng loại field (bảng cho mảng, card cho object lồng nhau, badge cho field trạng thái)
