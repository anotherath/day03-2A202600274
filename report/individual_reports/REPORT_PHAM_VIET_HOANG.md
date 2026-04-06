# Báo Cáo Cá Nhân: Lab 3 - Chatbot vs ReAct Agent

- **Tên sinh viên**: Phạm Việt Hoàng
- **MSSV**: 20210001
- **Ngày**: 06/04/2026

---

## I. Đóng Góp Kỹ Thuật

### 1. Lên Ý Tưởng và Thiết Kế Hệ Thống

Trong quá trình phát triển dự án Travel Planning Agent, tôi đã tham gia tích cực vào việc đề xuất và thảo luận các ý tưởng chính:

- **Ý tưởng đề xuất**: Xây dựng một trợ lý lập kế hoạch du lịch sử dụng mô hình ReAct (Thought-Action-Observation) thay vì chatbot truyền thống, giúp agent có khả năng suy luận từng bước trước khi đưa ra quyết định.
- **Luồng hoạt động**: Đề xuất luồng hoạt động gồm 6 tools mock: suggest_destinations, check_weather, search_flights, search_hotels, get_attractions, và calculate_total_cost.
- **Kiến trúc hệ thống**: Đề xuất sử dụng pattern LLMProvider để dễ dàng chuyển đổi giữa các nhà cung cấp API (OpenAI, Gemini, Local models).

**File liên quan**: src/tools/mock_tools.py, src/agent/agent.py

---

### 2. Triển Khai Vòng Lặp ReAct

Tôi đã triển khai vòng lặp ReAct chính trong src/agent/agent.py, bao gồm các thành phần cốt lõi:

#### a) Vòng lặp chính - run()

Hàm run() là trung tâm của vòng lặp ReAct, thực hiện các bước:

1. Generate LLM response (streaming hoặc non-streaming)
2. Parse Final Answer - nếu có thì break
3. Parse Thought - hiển thị reasoning
4. Parse Action - trích xuất tool call
5. Execute tool - gọi mock tool
6. Append Observation vào prompt cho bước tiếp theo

#### b) Các hàm parsing quan trọng:

- **\_parse_thought()**: Trích xuất phần Thought từ response của LLM
- **\_parse_final_answer()**: Trích xuất Final Answer khi agent hoàn thành
- **\_parse_action()**: Parse action thành tool_name và arguments
- **\_execute_tool()**: Thực thi tool và trả về observation

#### c) Xử lý vòng lặp vô hạn

Để tránh agent bị lặp vô hạn, tôi đã implement:

- **max_steps parameter**: Giới hạn số bước tối đa (mặc định = 6)
- **Điều kiện dừng sớm**: Khi LLM trả về Final Answer
- **Fallback mechanism**: Nếu không parse được action, append raw response vào prompt để tránh stuck

---

### 3. Tích Hợp Kết Nối

#### a) Kết nối LLM Provider

Tôi đã tích hợp hệ thống provider linh hoạt qua interface LLMProvider:

- **OpenAI Provider**: Sử dụng OpenRouter API để gọi các model như Gemini, Claude
- **Local Provider**: Chạy model local (Phi-3) qua llama-cpp-python
- **Streaming support**: Hỗ trợ streaming response token-by-token

#### b) Kết nối Telemetry & Metrics

Tích hợp hệ thống theo dõi hiệu suất qua PerformanceTracker:

- **Token tracking**: Theo dõi prompt_tokens, completion_tokens
- **Cost estimation**: Tính toán chi phí dựa trên pricing thực tế
- **Latency tracking**: Đo thời gian phản hồi (p50, p90, p95, p99)
- **Session summary**: Tổng kết toàn bộ session

#### c) Kết nối Logging

Mọi hành động đều được log qua logger:

- AGENT_START: Bắt đầu session
- LLM_RESPONSE: Response từ LLM
- OBSERVATION: Kết quả tool execution
- FINAL_ANSWER: Câu trả lời cuối
- AGENT_END: Kết thúc session

---

## II. Test và Đánh Giá

### 1. Các Test Cases Đã Viết

#### a) Test streaming response - test_stream()

Kiểm tra khả năng streaming token-by-token từ LLM API.

#### b) Test ReAct Agent - test_agent()

Kiểm tra toàn bộ vòng lặp ReAct với mock tools.

#### c) Interactive Chat Test - test_chat.py

Test tương tác real-time với người dùng.

### 2. Kết Quả Test

| Test Case          | Trạng Thái | Ghi Chú                     |
| ------------------ | ---------- | --------------------------- |
| Streaming Response | Pass       | Streaming hoạt động ổn định |
| ReAct Agent Loop   | Pass       | Agent hoàn thành 4-6 steps  |
| Interactive Chat   | Pass       | Chat loop hoạt động tốt     |
| Mock Tools         | Pass       | Tất cả 6 tools trả về đúng  |

### 3. Metrics Thu Thập Được

- **Total LLM calls**: ~4-6 calls per session
- **Total tokens**: ~2000-4000 tokens per session
- **Estimated cost**: ~$0.00 (sử dụng free tier)
- **Avg latency**: ~1000-3000ms per call
- **Token efficiency**: ~0.3-0.5 ratio

### 4. Phân Tích Lỗi và Cách Sửa

| Lỗi                   | Nguyên Nhân                   | Giải Pháp                         |
| --------------------- | ----------------------------- | --------------------------------- |
| Agent lặp vô hạn      | LLM không trả về Final Answer | Thêm max_steps giới hạn           |
| Parse action thất bại | Format response không đúng    | Thêm fallback append raw response |
| Tool not found        | Tên tool không khớp           | Thêm validation và error message  |
| Encoding tiếng Việt   | Windows console encoding      | Thêm io.TextIOWrapper với UTF-8   |

---

## III. Personal Insights: Chatbot vs ReAct

### 1. Reasoning

Block Thought giúp agent suy luận từng bước trước khi hành động. Thay vì trả lời ngay như chatbot, agent:

- Phân tích yêu cầu người dùng
- Quyết định tool nào cần gọi
- Đánh giá kết quả trước khi tiếp tục

### 2. Reliability

Agent thực hiện tệ hơn chatbot khi:

- Câu hỏi đơn giản, không cần multi-step reasoning
- LLM bị hallucination và gọi tool sai
- Số bước bị giới hạn, agent không đủ thời gian thu thập thông tin

### 3. Observation

Feedback từ môi trường (observations) giúp agent:

- Điều chỉnh hướng đi dựa trên kết quả thực tế
- Tránh gọi lại tool đã thất bại
- Xây dựng câu trả lời cuối dựa trên dữ liệu thật

---

## IV. Future Improvements

### 1. Scalability

- Sử dụng async queue cho tool calls để xử lý song song
- Implement tool registry để dễ dàng thêm tools mới

### 2. Safety

- Thêm Supervisor LLM để kiểm tra actions trước khi thực thi
- Implement rate limiting cho API calls
- Validate input/output của tools

### 3. Performance

- Vector DB cho tool retrieval trong hệ thống nhiều tools
- Caching kết quả tool calls để giảm API calls
- Batch processing cho nhiều tool calls cùng lúc

---
