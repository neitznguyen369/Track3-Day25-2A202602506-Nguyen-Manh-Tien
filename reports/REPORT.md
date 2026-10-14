# Lab 25B — Report

**Họ tên / nhóm:** Nguyen Manh Tien — MSV 2A202602506  **Commit:** <điền hash sau khi push>  **Điểm tự động (`make grade`):** 84 / 88 (4đ "dữ liệu ẩn" của Part E chỉ chấm trên máy giảng viên)

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.719 | 7.16 | 7.74 | 6.74 | 2.16 | 4 | ✗ |
| G1 | yours | 1.000 | 4.78 | 1.06 | 0.001 | 2.10 | 0 | ✓ |
| G2 | naive | 0.456 | 15.03 | 9.90 | 8.90 | 4.63 | 155 | ✗ |
| G2 | yours | 1.000 | 10.36 | 1.04 | 0.005 | 2.11 | 0 | ✓ |
| G3 | naive | 1.000 | 4.77 | 1.16 | 0.164 | 2.33 | 0 | ✗ |
| G3 | yours | 1.000 | 4.67 | 1.06 | 0.030 | 2.14 | 0 | ✓ |
| G4 | naive | 1.000 | 13.76 | 1.00 | 0.004 | 2.01 | 0 | ✗ |
| G4 | yours | 1.000 | 5.63 | 1.01 | 0.002 | 2.07 | 0 | ✓ |
| G5 | naive | 0.370 | 5.36 | 17.03 | 16.03 | 2.12 | – | ✗ |
| G5 | yours | 0.818 | 5.48 | 1.00 | 0.002 | 2.50 | – | ✓ |
| G6 | naive | 0.517 | 4.84 | 12.34 | 11.34 | 2.01 | 0 | ✗ |
| G6 | yours | 1.000 | 6.03 | 1.06 | 0.019 | 2.19 | 0 | ✓ |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao |
|---|---:|---|
| attempt_timeout_s | 8.0 | G6: alpha-eu "treo"; 8 s đủ hơn p95 bình thường (~5 s) nhưng vẫn còn thời gian fail over trong deadline 30 s. Thử 30 s: G6 availability 1.00 → 0.949 (mục 3). |
| max_attempts_per_provider | 2 | 1 retry rồi fail over: G3 retry_ratio 0.03 (≤ 0.10), amplification ≤ 1.06 ở mọi băng. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter phân tán retry; cap 4 s để delay không ăn hết deadline. |
| retry_budget_ratio | 0.1 | Chặn retry ≤ ~10% traffic; tắt budget khi breaker hỏng: G3 retry_ratio 0.068 → 0.158 (mục 3). |
| breaker.window_size / min_calls | 20 / 10 | Đủ mẫu để tránh trip vì 2–3 lỗi lẻ, vẫn trip trong vài giây khi G2/G6. |
| breaker.failure_rate_threshold | 0.5 | G3 (30% lỗi 500) không làm mở breaker thừa → availability 1.0; G2/G6 (≥ 75% lỗi) trip ngay. |
| breaker.slow_call_s | 5.0 | G4: alpha chậm +5 s, không có lỗi; slow-call trip đưa p95 13.76 → 5.63 s. |
| breaker.open_duration_s | 10 | Đủ ngắn để quay lại alpha ngay khi hồi phục (G2 recovery 0 s, G4 quay lại alpha). |
| breaker.half_open_max_probes | 2 | Chỉ 2 request đang bay khi thăm dò → không dội thundering herd lên provider đang hồi phục. |

Quyết định của `on_error`:

```
loại lỗi          ghi breaker?   retry cùng provider?              failover?   cooldown?
CLIENT_ERROR      release        không                             không       không  -> GiveUp("client_error")
QUOTA_EXHAUSTED   release        không                             có (ngay)   quota_cooldown_s = 300
RATE_LIMITED      release        không                             có (ngay)   đến now + retry_after
OVERLOADED/5xx/   on_failure     có nếu tries<2 và còn budget      có          không (breaker lo)
TIMEOUT                          và vừa deadline (full jitter)
Hết ladder: chờ cooldown rate-limit ngắn (<=5 s) nếu vừa deadline, ngược lại GiveUp.
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối** (breaker bị vô hiệu `min_calls: 1e9`, `max_attempts_per_provider: 5`).
   G2: có budget → amplification 1.38, retry_ratio 0.057, availability 0.9987; tắt budget → amplification 1.58, retry_ratio 0.378, recovery 24 s, availability 0.975, 58 lượt hết deadline (so với 3). G3: retry_ratio 0.068 (có) vs 0.158 (tắt). Budget giới hạn retry theo *lượng traffic* bất kể nguyên nhân; khi breaker đúng thì nó ít kích hoạt, nhưng breaker có thể cấu hình sai/chậm phát hiện nên cần cả hai (breaker: ngừng gửi vào provider chết; budget: trần khuếch đại tải).
2. **`attempt_timeout_s: 30`.** G6: availability 1.000 → 0.949; give_up: 109 `deadline` + 13 `no_provider_available`; p95 *giảm* (6.03 → 5.31 s) vì chỉ tính lượt thành công (survivorship bias). Mỗi attempt vào alpha-eu "treo" chiếm hết 30 s deadline nên không còn thời gian fail over. G4 gần như không đổi (1.000, p95 5.63 → 5.61) vì alpha vẫn trả lời (chậm 5 s), không bao giờ chạm timeout. Timeout nên dựa trên p99 bình thường + đủ chừa chỗ cho ít nhất 1–2 lần fail over trong deadline của user.
3. **Bỏ lọc tool khỏi `ladder()`.** G6: `incompatible_routes` 0 → 253, availability 1.000 → 0.894 (253/2385 lượt thất bại với `incompatible_model_output`). Nói với PM: ~1/10 khách bị trả lời hỏng vì ta gửi yêu cầu cần công cụ (tra đơn, hoàn tiền) sang model không có khả năng đó — rẻ hơn nhưng không làm được việc, còn tệ hơn báo lỗi.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | dup refunds | dup emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1 496 700 ¢ | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Chỉ idempotency key chưa đủ: sau crash, worker mới không biết step đã xong nên chạy lại cả workflow (classify/lookup/summary gọi LLM lại → tốn tiền, `wasted_llm_per_crash` cao) và key vẫn phải được tính ổn định mới an toàn. Chỉ checkpoint chưa đủ: nếu payment đã chạy nhưng response mất (`ServiceTimeout`) hoặc crash trước khi ghi checkpoint, lần resume sẽ gọi lại payment → hoàn tiền trùng; key ổn định khiến provider trả lại kết quả cũ. Ghi checkpoint *trước* khi gọi payment thì crash giữa chừng làm resume tưởng đã hoàn tiền → **mất hoàn tiền** (lost refund).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter (ngưỡng 5 phút) | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (1 phút) | page | 27 phút |
| provider_outage | page | page (0 phút) | page | 4 phút |
| deploy_blip | none | false page | không page | – |
| slow_leak | ticket | bỏ sót (không phát hiện) | ticket (không page) | 1337 phút |
| degraded_evening | page | page (214 phút) | page | 291 phút |
| night_spike | none | false page | không page | – |
| small_bump | none | false page | không page | – |

Naive: 7 page, 0 ticket, **3 false page** (deploy_blip, night_spike, small_bump), bỏ sót slow_leak. Burn-rate: 3 page, 1 ticket, **0 false page** — đổi lại phát hiện chậm hơn ở incident chậm (27 vs 1 phút cho checkout_bug) nhưng không đánh thức oan.

Error budget 30 ngày: đã tiêu **42,95%** (55 618 bad / 129 508,8 cho phép; availability 99,7853%), còn **57,05%** → **không** đóng băng release. `small_bump` (20 phút ở 12% lỗi) chỉ đốt ~0,4% budget tháng; không đáng đánh thức ai lúc 11 giờ trưa — xử lý ở giờ làm việc.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt:** Lúc t=120 s alpha mất 75% GPU (529 + hàng đợi dài). Client v1 với 3 tầng retry (tối đa 36 attempt/call, delay cố định) nhân tải lên ~9,9×. Provider hồi phục lúc t=300 nhưng hệ thống chỉ hồi phục ở ~t=455.

**Ảnh hưởng:** availability 45,6% (54,4% lượt user thất bại), kéo dài ~340 s (t≈120–460); amplification 9,90; p95 15,0 s. Nếu đây là sự cố duy nhất trong tháng: với SLO 99,5%, 54,4% lỗi trong ~340 s là một phần rất lớn của budget — vượt xa mức cho phép (≥ 1 sự cố như vậy đã đốt đáng kể budget 30 ngày; xem `make slo` cho cách tính theo phút).

**Timeline** (CSV `naive_G2_overload_storm_timeseries.csv`):

| t (s) | Sự kiện | Bằng chứng |
|---:|---|---|
| 100 | Bình thường | calls_ok 4 / calls_done 4 / attempts_alpha 10 |
| 120 | alpha mất 75% năng lực | calls_ok 7, attempts 5 (ngay trước sụp) |
| 140 | Retry storm bắt đầu | attempts_alpha 216, calls_ok 0 |
| 160–300 | Thành công = 0 | attempts_alpha 105–182 / 20 s, calls_ok 0 mọi mốc |
| 300 | alpha hồi phục hoàn toàn | attempts vẫn 115 / 20 s, calls_ok 0 |
| 320–440 | Vẫn chưa hồi phục — hàng đợi toàn request mà client đã bỏ + retry mới | attempts 19–107 / 20 s, calls_ok 0 |
| 460 | Hồi phục | calls_ok 9 / calls_done 12, attempts 14 |

**Trigger vs root cause:** Trigger = alpha mất 75% GPU. Root cause = retry 3 tầng chồng nhau, delay cố định (không jitter), không retry budget, không breaker, không fallback → khuếch đại tải ~10× đúng lúc provider yếu. Vòng lặp metastable: hàng đợi đầy → request chờ > timeout → client bỏ nhưng server vẫn xử lý (công sức phí) → client retry → hàng đợi càng đầy. Sau t=300 năng lực đã đủ cho traffic thật, nhưng tải retry vẫn > năng lực và hàng đợi còn đầy request "chết", nên goodput vẫn 0 đến khi retry cạn (~t=455).

**Điều gì giúp / làm tệ hơn:** Giúp: simulator ghi được time-series, có seed lặp lại. Tệ hơn: retry không jitter đồng bộ thành sóng; không có kill-switch/giảm tải; không alert theo amplification; không fallback sang alpha-eu/beta.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Chỉ retry ở 1 tầng, full jitter + retry budget 10% | prevent | Backend/Platform | G2: amplification ≤ 1,2 trong CI (hiện 1,04) |
| 2 | Circuit breaker sliding-window + fallback ladder theo khả năng | mitigate | Platform | G2/G6 availability ≥ 95%, recovery ≤ 30 s (hiện 100%, 0 s) |
| 3 | Alert theo `amplification` / retry_ratio và burn-rate đa cửa sổ | detect | SRE on-call | Diễn tập: page ≤ 5 phút kể từ khi lỗi bắt đầu (provider_outage: 4 phút) |

## 7. Một điểm yếu còn lại

G2 p95 vẫn 10,4 s (sát giới hạn) và khi beta (cost 1,4) gánh tải chi phí tăng; nếu cả 4 provider cùng chậm, ladder chỉ còn chờ rồi `GiveUp`. Cách sửa: thêm load shedding/ưu tiên theo loại lượt, hedged request có giới hạn, và adaptive timeout theo p99 đo được thay vì hằng số 8 s.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.837 | 0.842 | 0.000 | 0 | 0 | 0 | 1.386 | ✗ |
| cached (của bạn) | 0.912 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | ✓ |

| Tham số `cache:` | Giá trị | Vì sao |
|---|---:|---|
| similarity_threshold | 0.94 | Starter 0,80 → `wrong_answers`=181, tất cả `other_question` (cặp "gần mà khác", centroid cosine ≈ 0,86, ví dụ phí ship nội thành vs ra tỉnh). 0,94 cho 0 sai, hit 0,652; 0,97 hit tụt 0,579, cost cao hơn. |
| ttl_s | 60 | Thay đổi bảng phí ship không có webhook → chỉ TTL giới hạn được: ttl 30/60 → outdated 2/3; ttl 300 → 95 (> 15, FAIL). 60 cho hit cao hơn 30 (0,652 vs 0,592). |
| max_stale_s | 300 | Sự cố toàn bộ provider kéo dài 120 s; `max_stale_s: 0` → faq_availability 0,9645 → 0,873 (FAIL). 300 s phủ trọn sự cố. |

Thí nghiệm:

1. **Ngưỡng** 0,80 / 0,94 / 0,97: wrong 175 / 0 / 0, hit 0,659 / 0,652 / 0,579, cost/turn 0,978 / 0,915 / 0,965. Ở 0,80 câu hỏi gần nghĩa nhưng khác ý bị trả nhầm (`other_question`).
2. **TTL** 30 / 60 / 300: outdated 2 / 3 / 95. Webhook chỉ báo cho tài liệu *có* báo; thay đổi không báo trước thì không có tín hiệu nào để invalidate, nên chỉ TTL (tuổi tối đa của câu trả lời) giới hạn được độ cũ.
3. **Stale-if-error** (`max_stale_s: 0`): faq_availability 0,9645 → 0,873, availability 0,912 → 0,857, stale_served 157 → 0.

- Câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache kể cả theo từng user: câu trả lời phụ thuộc trạng thái đơn thay đổi theo thời gian (cache cũ = sai/ lộ trạng thái cũ), embedding gần giống giữa user khác nhau dễ trả nhầm dữ liệu người khác (`other_user`), và giá trị tái sử dụng thấp.
- Khi cache che sự cố, alert dựa trên `stale_served` / tỷ lệ stale, tỷ lệ lỗi provider (inner client), breaker mở và `cache_hit_ratio` bất thường — không dùng availability phía user vì nó vẫn xanh.