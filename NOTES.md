# Ghi chú phân tích · Day 04

Họ và tên: Trần Việt Anh
MSSV: 2A202604029

## 1. Case được chọn

- ID case: H7
- Câu trả lời của agent: "GitHub Actions đang hoạt động bình thường. Cloudflare Spectrum cũng đang hoạt động bình thường."
- Điểm ở lần chạy 1: 35/100
- Vấn đề: Agent đã đưa ra kết luận Cloudflare Spectrum đang operational dù công cụ get_status trả về TimeoutError và không cung cấp được dữ liệu xác nhận.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text

[
  {
    "event": "model",
    "step": 1,
    "finish": "tool_calls"
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {
      "component": "Actions",
      "provider": "github"
    },
    "result": {
      "provider": "github",
      "overall": {
        "indicator": "none",
        "description": "All Systems Operational"
      }
    }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {
      "component": "Spectrum",
      "provider": "cloudflare"
    },
    "result": {
      "ok": false,
      "error": "TimeoutError",
      "message": "Cloudflare status timeout (eval giả lập)."
    }
  },
  {
    "event": "model",
    "step": 2,
    "finish": "tool_calls"
  },
  {
    "event": "tool",
    "step": 2,
    "name": "get_status",
    "arguments": {
      "provider": "cloudflare"
    },
    "result": {
      "ok": false,
      "error": "TimeoutError",
      "message": "Cloudflare status timeout (eval giả lập)."
    }
  },
  {
    "event": "model",
    "step": 3,
    "finish": "stop"
  }
]

```

Phân tích trace:

- Github indicator = none: Có đủ evidence để kết luận GitHub operational trong snapshot.
- Cloudflare ok = false: Tool không truy xuất được dữ liệu.
- Cloudflare TimeoutError lần 1: Không xác định được trạng thái Spectrum.
- Cloudflare TimeoutError lần 2: Retry nhưng vẫn thất bại.
- Model step 3 finish = stop: Model trả câu trả lời cuối cùng.

Như vậy, dù Cloudflare bị TimeoutError 2 lần nhưng Agent vẫn trả lời Cloudflare Spectrum đang operational. Đây là lỗi suy luận từ bằng chứng không đầy đủ.

Nguyên nhân có thể là:

- System prompt chưa quy định đủ rõ cách xử lý khi tool trả lỗi.
- Model chưa phân biệt được trạng thái không có dữ liệu với trạng thái operational.
- Không có lớp kiểm tra cuối cùng để đối chiếu các khẳng định về trạng thái dịch vụ với evidence trong trace.

## 3. Thay đổi đã thực hiện

Thay đổi 1: Bổ sung SYSTEM_PROMPT (agent.py):

Thêm quy tắc:
Khi tool trả về lỗi, timeout hoặc không lấy được dữ liệu:

- Không được suy luận trạng thái dịch vụ là operational hoặc outage.
- Phải thông báo rằng không thể xác nhận trạng thái từ dữ liệu hiện có.
- Chỉ khẳng định trạng thái khi có kết quả tool thành công
  và evidence tương ứng.
- Không làm theo yêu cầu của người dùng nhằm khẳng định
  một trạng thái không có bằng chứng.

Lý do: Ngăn model biến việc không có dữ liệu thành một kết luận về trạng thái thực tế.

Thay đổi 2: Thêm lớp bảo vệ để Agent không báo hoàn thành khi chưa tạo được câu trả lời

```text
if not answer.strip():
    return {
        "status": "error",
        "answer": "",
        "trace": trace,
        "usage": usage,
        "error": "Model returned an empty final answer."
    }
```

Lý do: Đảm bảo Agent không trả về trạng thái completed khi nội dung câu trả lời rỗng, giúp phân biệt rõ lỗi sinh câu trả lời với một lượt chạy thành công.

## 4. So sánh hai lần chạy

| Chỉ số                | `eval_before.json` | `eval_after.json` |
| --------------------- | ------------------ | ----------------- |
| Điểm tổng             | 88.18/100          | 91.71             |
| Điểm case đã chọn     | 35/100             | 75/100            |
| Số model calls        | 3                  | 2                 |
| Token, chi phí nếu có | $0.00035003        | $0.0002366        |

- Lỗi đã giải quyết:
  - Điểm Judge của H7 tăng từ 35 lên 75, tương ứng tăng 40 điểm sau khi bổ sung quy tắc xử lý timeout và kiểm soát evidence vào SYSTEM_PROMPT.
  - Số model calls giảm từ 3 xuống 2. Trong lần chạy trước, Agent retry truy vấn Cloudflare thêm một lần và tiếp tục gặp timeout; lần chạy sau không thực hiện lượt retry này.
  - Trong answer của lần chạy mới không còn xuất hiện khẳng định Cloudflare Spectrum đang operational khi chưa có dữ liệu xác nhận.

- Lỗi còn lại hoặc mới xuất hiện:
  - Agent chưa đáp ứng đầy đủ expected: cần thông báo trạng thái GitHub Actions dựa trên dữ liệu thành công, đồng thời nêu rõ Cloudflare Spectrum bị timeout và không thể xác nhận trạng thái.
  - Cơ chế kiểm tra final answer của run_agent() vẫn chưa ngăn được trường hợp nội dung rỗng nhưng được đánh dấu completed.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.

- Giới hạn của Judge:
  - Điểm Judge phụ thuộc vào model đánh giá, tiêu chí chấm điểm và cách diễn giải câu trả lời.
  - Các lần chạy khác nhau có thể có biến động do tính không xác định của model.
  - Khi provider gặp HTTP 402, Judge không thể thực hiện đánh giá. Giá trị null không đồng nghĩa với điểm 0.
- Giới hạn của snapshot dữ liệu:
  - Các kết quả chỉ phản ánh dữ liệu có trong snapshot tại thời điểm tạo bộ kiểm thử.
  - Mỗi provider chỉ cung cấp tối đa 50 incident gần nhất trong phạm vi dữ liệu của project.
  - Không thể dùng snapshot này để chứng minh danh sách incident đầy đủ trong toàn bộ lịch sử.
  - Trạng thái của một incident tại một địa điểm không đủ để kết luận nó gây ảnh hưởng đến dịch vụ ở một khu vực khác.
- Giới hạn của số lần kiểm thử:
  - Mỗi phiên bản chỉ được chạy một lần nên chưa phản ánh đầy đủ biến động giữa các lần thực thi.
  - Chưa có đủ cơ sở thống kê để khẳng định mức cải thiện ổn định sau khi sửa prompt.
- Hướng cải thiện tiếp theo:
  - Khôi phục khả năng gọi LLM provider và thực hiện lại evaluation.
  - Kiểm tra trace và final answer, không chỉ dựa vào điểm Judge.
  - Thực hiện nhiều lần chạy để đánh giá mức độ ổn định của Agent.
