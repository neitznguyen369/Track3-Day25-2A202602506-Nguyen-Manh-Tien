# Rubric — Lab 25B Game Day (100 điểm)

## Tự động — 88 điểm (`make grade`)

### Unit test — 42 điểm

Điểm từng nhóm test được tính theo công thức `điểm tối đa × số test PASS / tổng số test`.

| Phần | Điểm tối đa | Bộ test |
|---|---:|---|
| A. Phân loại lỗi & retry | 12 | `tests/test_errors_retry.py` |
| B. Circuit breaker | 12 | `tests/test_breaker.py` |
| C. ResilientClient | 6 | `tests/test_client.py` |
| D. Workflow executor | 4 | `tests/test_executor.py` |
| E. Alerting | 4 | `tests/test_alerting.py` |
| G. Semantic cache | 4 | `tests/test_cache.py` |

### Tích hợp — 46 điểm

| Phần | Điểm tối đa | Điều kiện đạt đủ điểm |
|---|---:|---|
| C. Game Day | 24 | 6 băng × 4 điểm. Mỗi băng chỉ được điểm khi **mọi check đều PASS trên mọi seed**; giảng viên dùng băng ẩn và nhiều seed. |
| D. Workflow có crash | 8 | Trên 500 workflow: 0 hoàn tiền trùng, 0 email trùng, 0 hoàn tiền mất, completion ≥ 99%, `wasted_llm_per_crash ≤ 0,6`. |
| E. SLI public | 4 | `data/sli_30d.csv`: phát hiện đúng hạn các sự cố cần page/ticket và 0 false page. |
| E. SLI ẩn | 4 | Cùng tiêu chí trên dữ liệu riêng của giảng viên. |
| G. Băng cache C1 | 6 | Mọi check của băng C1 PASS trên mọi seed; giảng viên dùng băng ẩn và nhiều seed. |

Điểm mỗi hạng mục tích hợp là toàn bộ hoặc 0. Tổng điểm tự động tối đa **88**; `make grade`
ghi kết quả vào `reports/scoreboard.json`. Bốn điểm SLI ẩn chỉ được chấm trên dữ liệu của giảng viên.

## Chấm tay — 12 điểm (`reports/REPORT.md`)

Chấm theo bằng chứng trong báo cáo. Số liệu phải lấy từ kết quả chạy, CSV hoặc scoreboard; nhận định
không có số liệu đối chiếu không được tính là bằng chứng.

| Mục | Điểm | Tiêu chí |
|---|---:|---|
| Kết quả Game Day và cấu hình | 2 | Có bảng đủ 6 băng × 2 client; giải thích các tham số chính bằng số liệu của băng liên quan. |
| Ba thí nghiệm | 2 | Có ít nhất 3 thí nghiệm với số liệu trước/sau và giải thích đúng cơ chế thay đổi. |
| Workflow và alerting | 1 | Giải thích được vì sao cần cả idempotency lẫn checkpoint; tính đúng error budget và quyết định freeze release. |
| Cache (mục 8) | 2 | Có so sánh không cache/có cache; lý do định lượng cho **mọi** tham số `cache:`; trả lời đúng về câu hỏi cá nhân và alert khi phục vụ stale. |
| Postmortem G2 | 5 | Được chấm theo 5 tiêu chí bên dưới, mỗi tiêu chí tối đa 1 điểm. |

### Hướng dẫn chấm các mục báo cáo

#### Kết quả Game Day và cấu hình (2 điểm)

- **0 điểm:** Thiếu bảng kết quả hoặc không có lý do cấu hình.
- **1 điểm:** Có bảng nhưng thiếu một hay nhiều băng/client, hoặc lý do chung chung, không gắn với kết quả đo.
- **2 điểm:** Đủ 6 băng × 2 client; giải thích các tham số chính bằng số liệu cụ thể, phù hợp với băng được nêu.

#### Ba thí nghiệm (2 điểm)

- **0 điểm:** Thiếu thí nghiệm hoặc chỉ nêu kết luận, không có số liệu so sánh.
- **1 điểm:** Có ít nhất 3 thí nghiệm và số liệu trước/sau, nhưng giải thích cơ chế chưa chính xác hoặc chưa đầy đủ.
- **2 điểm:** Ít nhất 3 thí nghiệm có số liệu trước/sau và giải thích chính xác các cơ chế, bao gồm:
  - retry budget giới hạn khuếch đại tải khác với circuit breaker ngừng gọi provider lỗi;
  - timeout quá dài làm mất thời gian failover, còn timeout ngắn không đồng nghĩa với xử lý được sự cố treo;
  - model không hỗ trợ tool không thể thay thế model có tool calling.

#### Workflow và alerting (1 điểm)

- **0 điểm:** Thiếu một trong hai phần hoặc có kết luận/tính toán sai.
- **0,5 điểm:** Đề cập workflow và alerting nhưng giải thích vì sao cần cả idempotency lẫn checkpoint,
  hoặc tính error budget/quyết định freeze chưa chính xác.
- **1 điểm:** Nêu đúng vai trò bổ sung cho nhau của idempotency và checkpoint; tính đúng lượng/tỷ lệ
  error budget đã dùng và còn lại, đồng thời đưa ra quyết định freeze release phù hợp.

#### Cache — mục 8 (2 điểm)

- **0 điểm:** Thiếu so sánh cache hoặc bỏ qua rủi ro trả lời sai/rò rỉ dữ liệu cá nhân.
- **1 điểm:** Có bảng so sánh và giải thích một phần cấu hình; hoặc chưa trả lời đúng một trong hai câu hỏi
  về câu hỏi cá nhân và alert khi cache đang phục vụ stale.
- **2 điểm:** Có số liệu không cache/có cache; giải thích bằng kết quả đo cho từng tham số trong `cache:`
  (bao gồm `similarity_threshold`, `ttl_s`, `max_stale_s`, `max_entries`); dùng các chỉ số phù hợp như
  `wrong_answers_by_reason`, `outdated_answers`, `faq_availability`, `cache_hit_ratio` hoặc chi phí.
  Nêu rõ không cache câu trả lời cá nhân và đề xuất theo dõi stale/cache cùng tình trạng lỗi provider
  để alert không bị che bởi availability phía người dùng.

### Postmortem G2 — 5 điểm

Chấm mỗi tiêu chí tối đa 1 điểm:

| Tiêu chí | 0 điểm | 0,5 điểm | 1 điểm |
|---|---|---|---|
| Timeline | Không có timeline. | Có mốc sự kiện nhưng thiếu số liệu định lượng. | Ít nhất 5 mốc có số liệu từ CSV, chẳng hạn attempts/s và success ratio. |
| Trigger và root cause | Không phân biệt hoặc xác định sai. | Nêu được trigger nhưng chưa làm rõ root cause. | Trigger là mất GPU; root cause là retry ba tầng thiếu jitter/budget/breaker khuếch đại tải và duy trì vòng lặp metastable. |
| Không tự hồi phục sau `t=300` | Không giải thích. | Giải thích chung chung. | Nêu rõ hàng đợi vẫn chứa request mà client đã bỏ, trong khi retry tiếp tục giữ tải vượt năng lực provider. |
| Action items | Thiếu hoặc chỉ nêu khẩu hiệu. | Có hành động cụ thể nhưng thiếu cách đo, owner hoặc phân loại. | Ít nhất 3 mục phân loại prevent/detect/mitigate, mỗi mục có owner và cách kiểm chứng định lượng. |
| Blameless | Đổ lỗi cá nhân/nhóm. | Ngôn từ trung tính nhưng chưa tập trung vào nguyên nhân hệ thống. | Tập trung vào thiết kế, quy trình và biện pháp cải thiện hệ thống, không quy trách nhiệm cá nhân. |

## Trừ điểm

Các mức trừ dưới đây áp dụng ngoài điểm chấm theo rubric, không làm thay đổi ngưỡng/logic của grader tự động.

| Vi phạm | Mức trừ / xử lý |
|---|---:|
| Sửa file bị cấm (`sim/`, tests, scenarios, services, v.v.) | Mất toàn bộ điểm của phần bị ảnh hưởng. |
| Hard-code thời điểm hoặc tên băng trong client | Phần C = 0. |
| Hard-code thời điểm, tên băng hoặc tên tài liệu trong cache | Phần G = 0. |
| Báo cáo thiếu số liệu | Trừ tối đa 5 điểm. |
| Bài không chạy được trên máy sạch bằng `pip install -e ".[dev]" && make grade` | Trừ 10 điểm. |
