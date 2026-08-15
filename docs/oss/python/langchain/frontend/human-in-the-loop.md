# Human-in-the-Loop

> Thêm luồng phê duyệt (approval workflow) với cơ chế review con người dựa trên interrupt

Không phải mọi hành động của agent đều nên chạy mà không có giám sát. Khi một agent sắp gửi email, xoá một record, thực hiện một giao dịch tài chính, hoặc thực hiện bất kỳ thao tác không thể hoàn tác (irreversible) nào, bạn cần một con người review và phê duyệt hành động đó trước. Pattern Human-in-the-Loop (HITL) cho phép agent của bạn tạm dừng thực thi, hiển thị hành động đang chờ cho user, và chỉ resume sau khi được phê duyệt rõ ràng (explicit approval).

Vì HITL được xây dựng trên nền interrupt và checkpoint của LangGraph, việc tạm dừng này có tính bền vững (durable). User có thể refresh trang, một reviewer có thể trả lời từ một component khác, và agent vẫn resume đúng tại điểm mà việc thực thi đã dừng lại, thay vì phải chạy lại (replay) toàn bộ lượt chạy (run).

!!! tip "Xem demo"
    Trang gốc có một demo tương tác trực tiếp cho pattern này. Xem tại: [https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop](https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop)

## Interrupt hoạt động như thế nào

Agent LangGraph hỗ trợ **interrupt**, các điểm tạm dừng rõ ràng (explicit pause point) nơi agent nhường lại quyền kiểm soát cho client. Khi agent chạm tới một interrupt:

1. Agent dừng thực thi và phát ra một interrupt payload
2. Hook [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hiển thị interrupt đó qua `stream.interrupt`
3. UI của bạn render một card review với các lựa chọn phê duyệt/từ chối/sửa
4. User đưa ra quyết định
5. Code của bạn gọi `stream.submit()` với một resume command
6. Agent tiếp tục từ đúng điểm nó đã dừng

Frontend SDK giữ interrupt cùng với phần state còn lại của thread, nhờ vậy UI của bạn có thể render nó ở bất cứ đâu hợp lý: ngay trong transcript, trong một hàng đợi review, trong một dashboard quản trị, hoặc trong một modal chặn hành động tiếp theo của user cho tới khi có quyết định.

## Thiết lập `useStream`

Kết nối [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) với agent human-in-the-loop của bạn. Khi graph chạm tới một interrupt, hook sẽ expose payload đang chờ tại `stream.interrupt`. Render một card phê duyệt trong khi giá trị đó được set, sau đó resume lượt chạy bằng `stream.submit(null, { command: { resume: response } })` sau khi user phê duyệt, từ chối, hoặc sửa hành động đó.

!!! info "Thông tin"
    Các ví dụ code dùng `useStream<typeof myAgent>` để có type-safe stream state. Xem phần Type inference cho backend [Python](overview.md#type-inference) hoặc [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference).

=== "React"
    ```tsx
    import { useStream } from "@langchain/react";

    const AGENT_URL = "http://localhost:2024";

    export function Chat() {
      const stream = useStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "human_in_the_loop",
      });

      const interrupt = stream.interrupt;

      return (
        <div>
          {stream.messages.map((msg) => (
            <Message key={msg.id} message={msg} />
          ))}
          {interrupt && (
            <ApprovalCard
              interrupt={interrupt}
              onRespond={(response) =>
                stream.submit(null, { command: { resume: response } })
              }
            />
          )}
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
      assistantId: "human_in_the_loop",
    });

    function handleRespond(response: HITLResponse) {
      stream.submit(null, { command: { resume: response } });
    }
    </script>

    <template>
      <div>
        <Message
          v-for="msg in stream.messages.value"
          :key="msg.id"
          :message="msg"
        />
        <ApprovalCard
          v-if="stream.interrupt.value"
          :interrupt="stream.interrupt.value"
          @respond="handleRespond"
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
        assistantId: "human_in_the_loop",
      });

      function handleRespond(response: HITLResponse) {
        stream.submit(null, { command: { resume: response } });
      }
    </script>

    <div>
      {#each stream.messages as msg (msg.id)}
        <Message message={msg} />
      {/each}

      {#if stream.interrupt}
        <ApprovalCard interrupt={stream.interrupt} onRespond={handleRespond} />
      {/if}
    </div>
    ```

=== "Angular"
    ```ts
    import { Component } from "@angular/core";
    import { injectStream } from "@langchain/angular";
    import type { HITLResponse } from "langchain";

    const AGENT_URL = "http://localhost:2024";

    @Component({
      selector: "app-chat",
      template: `
        @for (msg of stream.messages(); track msg.id) {
          <app-message [message]="msg" />
        }
        @if (stream.interrupt()) {
          <app-approval-card
            [interrupt]="stream.interrupt()"
            (respond)="handleRespond($event)"
          />
        }
      `,
    })
    export class ChatComponent {
      stream = injectStream<typeof myAgent>({
        apiUrl: AGENT_URL,
        assistantId: "human_in_the_loop",
      });

      handleRespond(response: HITLResponse) {
        this.stream.submit(null, { command: { resume: response } });
      }
    }
    ```

## Interrupt payload

Khi agent tạm dừng, `stream.interrupt` chứa một [HITLRequest](https://reference.langchain.com/javascript/langchain/index/HITLRequest) với cấu trúc sau:

```ts
interface HITLRequest {
  actionRequests: ActionRequest[];
  reviewConfigs: ReviewConfig[];
}

interface ActionRequest {
  name: string;
  args: Record<string, unknown>;
  description?: string;
}

interface ReviewConfig {
  allowedDecisions: ("approve" | "reject" | "edit" | "respond")[];
}
```

| Thuộc tính                          | Mô tả                                                                  |
| ------------------------------------- | ----------------------------------------------------------------------- |
| `actionRequests`                      | Mảng các hành động đang chờ mà agent muốn thực hiện                     |
| `actionRequests[].name`               | Tên hành động (ví dụ `"send_email"`, `"delete_record"`)                 |
| `actionRequests[].args`               | Tham số có cấu trúc cho hành động                                       |
| `actionRequests[].description`        | Mô tả (tuỳ chọn) dễ đọc, giải thích hành động đó làm gì                 |
| `reviewConfigs`                       | Cấu hình theo từng hành động, kiểm soát quyết định nào được cho phép     |
| `reviewConfigs[].allowedDecisions`    | Những nút nào sẽ hiện: `"approve"`, `"reject"`, `"edit"`, `"respond"`   |

## Các loại quyết định

Pattern HITL hỗ trợ bốn loại quyết định:

### Approve (Phê duyệt)

User xác nhận hành động nên được thực hiện đúng như đề xuất:

```ts
const response: HITLResponse = {
  decisions: [{ type: "approve" }],
};

stream.submit(null, { command: { resume: response } });
```

### Reject (Từ chối)

User từ chối hành động, kèm lý do tuỳ chọn. Tool không được thực thi:

```ts
const response: HITLResponse = {
  decisions: [
    {
      type: "reject",
      message: "The email tone is too aggressive. Do not send it.",
    },
  ],
};

stream.submit(null, { command: { resume: response } });
```

!!! note "Ghi chú"
    Khi một hành động bị từ chối, agent nhận được lý do từ chối và có thể quyết định cách tiếp tục. Nếu bạn bỏ qua `message`, backend sẽ dùng một message mặc định báo cho model biết tool đã không được thực thi và không nên thử lại đúng tool call đó trừ khi user yêu cầu. Với các tool có side effect, hãy truyền một message rõ ràng cho biết agent nên bỏ hành động, hỏi lại thêm, hay thử một phương án an toàn hơn.

### Edit (Sửa)

User sửa tham số của hành động trước khi phê duyệt:

```ts
const response: HITLResponse = {
  decisions: [
    {
      type: "edit",
      editedAction: {
        name: actionRequest.name,
        args: {
          ...actionRequest.args,
          subject: "Updated subject line",
          body: "Revised email body with softer language.",
        },
      },
    },
  ],
};

stream.submit(null, { command: { resume: response } });
```

### Respond (Trả lời trực tiếp)

User cung cấp một câu trả lời trực tiếp cho các tool kiểu "hỏi user" (ask user). `message` trở thành kết quả của tool và bản thân tool không được thực thi:

```ts
const response: HITLResponse = {
  decisions: [{ type: "respond", message: "Blue." }],
};

stream.submit(null, { command: { resume: response } });
```

!!! note "Ghi chú"
    Dùng `respond` khi tool đó chủ đích là một placeholder chờ input từ con người, ví dụ một tool `ask_user` yêu cầu agent thu thập thông tin từ user. Đừng dùng `respond` để từ chối một hành động được đề xuất, vì nó sẽ được trả về cho model như một kết quả tool thành công.

## Xây dựng ApprovalCard

Đây là cách nối quyết định (decision wiring) được dùng bởi các approval card. UI có thể tách mỗi hành động thành một card riêng, nhưng resume payload là một `HITLResponse` duy nhất, với một quyết định cho mỗi hành động đang chờ:

```tsx
async function approveAll() {
  const resume: HITLResponse = {
    decisions: actionRequests.map(() => ({ type: "approve" })),
  };
  await stream.submit(null, { command: { resume } });
}

async function rejectOne(index: number, message: string) {
  const resume: HITLResponse = {
    decisions: actionRequests.map((_, i) =>
      i === index
        ? { type: "reject", message }
        : { type: "reject", message: "Rejected along with other actions" },
    ),
  };
  await stream.submit(null, { command: { resume } });
}

async function editOne(index: number, editedArgs: Record<string, unknown>) {
  const originalAction = actionRequests[index];
  const resume: HITLResponse = {
    decisions: actionRequests.map((_, i) =>
      i === index
        ? {
            type: "edit",
            editedAction: { name: originalAction.name, args: editedArgs },
          }
        : { type: "approve" },
    ),
  };
  await stream.submit(null, { command: { resume } });
}
```

## Luồng resume

Sau khi user đưa ra quyết định, toàn bộ chu trình diễn ra như sau:

1. Gọi `stream.submit(null, { command: { resume: hitlResponse } })`
2. Hook [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) gửi resume command tới backend LangGraph
3. Agent nhận `HITLResponse` và tiếp tục thực thi. Mỗi entry trong `decisions` có thể là:
   * `{ type: "approve" }`: agent tiếp tục thực thi hành động
   * `{ type: "reject", message }`: tool không được thực thi, và agent nhận được message từ chối trước khi quyết định bước tiếp theo
   * `{ type: "edit", editedAction }`: agent chạy tool với tham số đã sửa
   * `{ type: "respond", message }`: message của con người được trả về trực tiếp làm kết quả tool, mà không thực thi tool
4. Thuộc tính `interrupt` reset về `null` khi agent tiếp tục stream

!!! tip "Mẹo"
    Bạn có thể nối nhiều HITL checkpoint trong một lượt chạy agent duy nhất. Ví dụ, agent có thể xin phê duyệt để search, sau đó xin phê duyệt lần nữa trước khi gửi email chứa kết quả. Mỗi interrupt được xử lý độc lập.

## Xử lý nhiều hành động đang chờ

Một interrupt có thể chứa nhiều `actionRequests` khi agent muốn thực hiện nhiều hành động cùng lúc. Render một card cho mỗi hành động và thu thập toàn bộ quyết định trước khi resume:

```tsx
function MultiActionReview({
  interrupt,
  onRespond,
}: {
  interrupt: { value: HITLRequest };
  onRespond: (response: HITLResponse) => void;
}) {
  const [decisions, setDecisions] = useState<Record<number, HITLResponse["decisions"][number]>>({});
  const request = interrupt.value;

  const allDecided =
    Object.keys(decisions).length === request.actionRequests.length;

  return (
    <div className="space-y-4">
      {request.actionRequests.map((action, i) => (
        <SingleActionCard
          key={i}
          action={action}
          config={request.reviewConfigs[i]}
          onDecide={(response) =>
            setDecisions((prev) => ({ ...prev, [i]: response }))
          }
        />
      ))}
      {allDecided && (
        <button
          className="rounded bg-green-600 px-4 py-2 text-white"
          onClick={() =>
            onRespond({
              decisions: request.actionRequests.map((_, i) => decisions[i]),
            })
          }
        >
          Submit All Decisions
        </button>
      )}
    </div>
  );
}
```

## Form interrupt tuỳ chỉnh

[Luồng resume](#luong-resume) dùng `humanInTheLoopMiddleware`, thứ bọc một tool bằng một card phê duyệt/từ chối/sửa/trả lời chung. Đôi khi một bộ nút chung là chưa đủ: đặt vé máy bay, phê duyệt hoàn tiền, và review một bài đăng mạng xã hội đều cần một form *khác nhau*, với field, validation, và câu chữ riêng. Với trường hợp đó, hãy raise `interrupt()` **từ bên trong tool** và để payload mô tả chính xác form mà UI nên render. Mỗi tool có thể hiện một giao diện hoàn toàn khác nhau.

!!! tip "Xem demo"
    Trang gốc có một demo tương tác trực tiếp cho pattern này. Xem tại: [https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop](https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop)

### Mô tả form trong interrupt payload

`interrupt()` chấp nhận bất kỳ giá trị JSON-serializable nào, cho phép bạn cung cấp một "card" mà frontend biết cách render, ví dụ như một loại form, một tiêu đề, ngữ cảnh mà con người đang review, và các field cần thu thập. `interrupt()` là generic trên kiểu input và return (`interrupt<I, R>(value: I): R`), nên bạn có thể gán kiểu cho cả card bạn gửi (`InterruptCard`) lẫn giá trị mà user resolve (`ReviewDecision`). Export các type đó để frontend có thể import và luôn đồng bộ:

```ts
import { createAgent, tool } from "langchain";
import { interrupt } from "@langchain/langgraph";
import { z } from "zod";

export interface FormField {
  name: string;
  label: string;
  type: "select" | "checkbox" | "textarea" | "currency";
  options?: string[];
  default?: unknown;
}

/** Giá trị user dùng để resolve interrupt. */
export interface ReviewDecision {
  approved: boolean;
  /** Giá trị form đã sửa/thu thập mà tool sẽ dùng để hành động. */
  values?: Record<string, unknown>;
}

/** Spec form ("card") mà một interrupt trao cho frontend. */
export interface InterruptCard {
  formType: "flight-booking" | "refund-approval" | "content-review";
  tool: string;
  title: string;
  context: Record<string, unknown>;
  fields: FormField[];
  /** Được frontend điền khi nó commit card đã resolve vào state. */
  resolved?: boolean;
  decision?: ReviewDecision;
}

const bookFlight = tool(
  async ({ origin, destination, date, passengers }) => {
    // Tạm dừng tool và trao cho frontend một form spec có kiểu; giá trị trả
    // về có kiểu là bất cứ thứ gì UI dùng để resolve interrupt này.
    const decision = interrupt<InterruptCard, ReviewDecision>({
      formType: "flight-booking",
      tool: "book_flight",
      title: "Confirm flight booking",
      context: { origin, destination, date, passengers },
      fields: [
        {
          name: "seatClass",
          label: "Seat class",
          type: "select",
          options: ["Economy", "Premium Economy", "Business"],
          default: "Economy",
        },
        { name: "insurance", label: "Add trip insurance", type: "checkbox", default: false },
      ],
    });

    if (!decision.approved) {
      return `Booking cancelled. No flight from ${origin} to ${destination} was reserved.`;
    }

    // Chạy công việc thật (có thể chậm) với giá trị con người đã xác nhận.
    const seatClass = String(decision.values?.seatClass ?? "Economy");
    return `Flight booked from ${origin} to ${destination} in ${seatClass}.`;
  },
  {
    name: "book_flight",
    description: "Book a flight. Requires human confirmation of trip details.",
    schema: z.object({
      origin: z.string(),
      destination: z.string(),
      date: z.string(),
      passengers: z.number().int().min(1),
    }),
  },
);
```

Đặt cho mỗi tool một `formType` riêng biệt (ví dụ `"refund-approval"`, `"content-review"`), nhờ vậy frontend có thể switch theo giá trị đó và render đúng form.

### Render một form khác nhau cho mỗi tool

Ở phía client, card đến dưới dạng `stream.interrupt.value`. Import các type `InterruptCard` và `ReviewDecision` từ module agent của bạn để form và payload luôn đồng bộ, switch theo `formType` để chọn đúng form, và đưa `fields` vào input của bạn:

```tsx
import { useStream } from "@langchain/react";
import type { InterruptCard, ReviewDecision } from "./agent";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "hitl_interrupt_forms",
  });

  const card = stream.interrupt?.value as InterruptCard | undefined;

  return (
    <div>
      {stream.messages.map((msg) => (
        <Message key={msg.id} message={msg} />
      ))}
      {card && <InterruptForm card={card} onResolve={handleResolve} />}
    </div>
  );
}

// `InterruptForm` render một card flight / refund / content dựa trên
// `card.formType`, thu thập `card.fields`, và gọi `onResolve` với quyết
// định của user cùng các giá trị đã sửa.
```

### Giữ card trên màn hình với `respond(decision, { update })`

Khi bạn resolve một interrupt thông thường, card sẽ biến mất ngay khi interrupt được clear, và chỉ có kết quả tool trả về. Điều đó có nghĩa một card review phong phú sẽ biến mất giữa chừng lượt chạy. Để giữ nó trên màn hình, hãy resolve interrupt **và** commit một message mang theo card vào state trong *cùng* một superstep, dùng `respond` của [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream):

```tsx
import { AIMessage } from "langchain";

function handleResolve(decision: ReviewDecision) {
  // Chụp lại (snapshot) card với quyết định đã gắn vào, để nó render dạng chỉ đọc.
  const resolvedCard = { ...card, resolved: true, decision };
  const cardMessage = new AIMessage({
    content: `Review ${decision.approved ? "approved" : "declined"}.`,
    response_metadata: { cards: resolvedCard },
  });

  // Resume interrupt VÀ push card vào state một cách nguyên tử (atomic). Tương
  // đương `Command(resume, update)` của LangGraph: một checkpoint, không ghi
  // thêm state thừa.
  stream.respond(decision, { update: { messages: [cardMessage] } });
}
```

`respond(response, { update })` áp dụng `update` theo kiểu **lạc quan (optimistic)**: card được vẽ ngay lập tức và được đối chiếu (reconcile) theo ID khi lượt chạy đã resume phản hồi lại đúng message đó. Backend không bao giờ phát lại card, nên nó luôn được render mà không nháy (flicker) trong lúc tool (có thể chậm) đang chạy. Render card đã resolve bằng cách đọc lại nó từ message:

```tsx
{stream.messages.map((msg) => {
  const card = (msg.response_metadata as { cards?: InterruptCard })?.cards;
  if (card) return <InterruptForm key={msg.id} card={card} readOnly />;
  return <Message key={msg.id} message={msg} />;
})}
```

!!! tip "Mẹo"
    Vì card đã resolve nằm trong lịch sử message, nó sẽ tồn tại qua các lần refresh và hiển thị cho mọi component đọc thread, và quyết định của con người trở thành một phần của transcript bền vững, chứ không chỉ là UI state tạm thời.

## Thực hành tốt nhất

Ghi nhớ các nguyên tắc sau khi triển khai luồng HITL:

* **Hiển thị ngữ cảnh rõ ràng**. Luôn hiển thị *cái gì* agent muốn làm và *tại sao*. Bao gồm mô tả hành động và toàn bộ tham số.
* **Làm cho việc phê duyệt là con đường dễ nhất**. Nếu hành động trông đúng, việc phê duyệt nên chỉ mất một cú click. Dành các luồng nhiều bước cho reject/edit.
* **Validate tham số đã sửa**. Khi user sửa tham số hành động, validate cấu trúc JSON trước khi gửi. Hiện lỗi inline cho input sai định dạng.
* **Giữ bền vững trạng thái interrupt**. Nếu user refresh trang, interrupt vẫn nên hiển thị. [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) xử lý việc này thông qua checkpoint của thread.
* **Ghi log mọi quyết định**. Để phục vụ audit trail, ghi log mỗi quyết định approve/reject/edit kèm timestamp và user đã đưa ra quyết định.
* **Đặt timeout hợp lý**. Các agent chạy lâu không nên chặn vô thời hạn để chờ review của con người. Cân nhắc hiển thị agent đã chờ bao lâu.
