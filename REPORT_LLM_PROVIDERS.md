# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Nguyễn Việt Hoàng
- **Student ID**: 2A202600455
- **Date**: 2026-04-06

---

## I. Technical Contribution (15 Points)

- **Modules Implemented**:
  - `src/core/llm_provider.py` (interface chung)
  - `src/core/openai_provider.py`
  - `src/core/gemini_provider.py`
  - `src/core/local_provider.py`
  - `src/core/mock_provider.py`
- **Code Highlights**:
  - Thiết kế provider theo một contract thống nhất: mọi provider đều có `generate()` và `stream()`.
  - Chuẩn hóa output trả về từ provider theo dạng:
    - `content`
    - `usage` (`prompt_tokens`, `completion_tokens`, `total_tokens`)
    - `latency_ms`
    - `provider`
  - `OpenAIProvider` dùng `chat.completions.create(...)` với system/user messages.
  - `GeminiProvider` dùng `model.generate_content(...)`, kèm logic prepend system prompt.
  - `LocalProvider` dùng `llama-cpp-python` để chạy GGUF offline và tự format prompt kiểu chat template.
  - `MockProvider` tạo response giả lập để test parser/ReAct loop không cần API key.
- **Documentation**:
  - Trong `src/agent/agent.py`, agent chỉ phụ thuộc vào `LLMProvider` nên có thể đổi provider mà không sửa vòng ReAct.
  - Cách tách này giúp phần reasoning và phần gọi model tách biệt rõ ràng, dễ debug và mở rộng.

---

## II. Debugging Case Study (10 Points)

- **Problem Description**:
  - Trong quá trình chạy đa provider, các lỗi lặp lại nhiều nhất là lỗi xác thực, lỗi model endpoint, và lỗi local model path.
- **Log Source**:
  - Log runtime từ terminal khi gọi provider.
  - Telemetry event `AGENT_ERROR` trong `logs/` khi exception bị đẩy ra tầng agent.
- **Diagnosis**:
  - **Authentication lỗi**:
    - OpenAI: thiếu/sai `OPENAI_API_KEY` làm request bị từ chối.
    - Gemini: thiếu/sai `GEMINI_API_KEY` làm `genai.configure(...)` không xác thực được.
  - **Model endpoint lỗi**:
    - Tên model không hợp lệ gây lỗi kiểu `model not found`/request reject.
    - Mỗi SDK có API gọi khác nhau (OpenAI chat completion vs Gemini generate_content), nếu map sai sẽ fail.
  - **Local runtime lỗi**:
    - `LOCAL_MODEL_PATH` sai -> `FileNotFoundError` tại `LocalProvider.__init__`.
    - Cấu hình tài nguyên không phù hợp (context/thread) -> chậm hoặc timeout.
- **Solution**:
  - Validate cấu hình provider trước khi chạy (key/model/path).
  - Chuẩn hóa dữ liệu trả về ở từng provider để parser phía agent không phụ thuộc SDK.
  - Dùng `MockProvider` để cô lập lỗi parser/tool, tránh nhầm với lỗi API/network.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**:  
   ReAct có lợi thế khi cần gọi tool và xử lý đa bước; phần `Thought/Action/Observation` giúp suy luận tuần tự rõ hơn chatbot trả lời trực tiếp.
2. **Reliability**:  
   Agent có thể kém hơn chatbot nếu output model không đúng format, parser lỗi, hoặc provider trả response khác schema kỳ vọng.
3. **Observation**:  
   Observation từ tool giúp agent “neo” vào dữ liệu thật, giảm trả lời cảm tính; chất lượng observation càng rõ thì bước tiếp theo càng ổn định.

### LLM Providers Used

- **OpenAIProvider** (`src/core/openai_provider.py`):
  - Vai trò: provider cloud chính cho chất lượng phản hồi ổn định.
  - Điểm mạnh: output đều, usage rõ, tích hợp dễ với ReAct loop.
- **GeminiProvider** (`src/core/gemini_provider.py`):
  - Vai trò: provider cloud thay thế để tăng khả năng chuyển đổi vendor.
  - Điểm mạnh: tốc độ tốt; điểm cần chú ý: cách tổ chức prompt/system khác OpenAI.
- **LocalProvider** (`src/core/local_provider.py`):
  - Vai trò: chạy offline bằng GGUF (không phụ thuộc API bên ngoài).
  - Điểm mạnh: chủ động dữ liệu; hạn chế: yêu cầu tài nguyên máy và tuning.
- **MockProvider** (`src/core/mock_provider.py`):
  - Vai trò: test deterministic cho pipeline parser/tools.
  - Điểm mạnh: không tốn chi phí API, dễ tái lập lỗi.

### Phân tích lỗi provider/authentication/model endpoint

- **Provider/Auth**:
  - Sai key hoặc thiếu key là nguyên nhân phổ biến nhất khi gọi cloud provider.
- **Model endpoint**:
  - Sai model name gây fail ngay ở request tầng SDK.
  - Khác biệt endpoint contract giữa OpenAI/Gemini đòi hỏi adapter chính xác.
- **Response schema**:
  - Nếu output provider không được normalize thống nhất (`content/usage/...`) thì parser ở agent dễ gãy.

---

## IV. Future Improvements (5 Points)

- **Scalability**:
  - Thêm cơ chế provider fallback tự động (OpenAI -> Gemini -> Local/Mock).
  - Tách tầng inference thành async worker để giảm block request.
- **Safety**:
  - Bổ sung guardrail kiểm tra output trước khi trả user.
  - Ẩn/mask key và thông tin nhạy cảm trong logs.
- **Performance**:
  - Thêm timeout + retry + backoff theo từng provider.
  - Tối ưu local model (`n_ctx`, `n_threads`) theo cấu hình máy.

### Hạn chế và rủi ro tích hợp API

- Phụ thuộc nhà cung cấp bên ngoài (rate limit, downtime, thay đổi model/policy).
- Drift SDK theo phiên bản gây rủi ro tương thích.
- Không đồng nhất chất lượng/format giữa provider làm kết quả thiếu nhất quán.
- Local model có rủi ro hiệu năng nếu tài nguyên không đủ.

---

> [!NOTE]
> Đổi tên file thành `REPORT_[YOUR_NAME].md` trước khi nộp chính thức nếu giảng viên yêu cầu đúng convention.
