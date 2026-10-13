# Lab 25B — Report

**Họ tên / nhóm:** Dương Minh Hiếu (Mã SV: 2A202602488)  **Commit:** `eb5f21a`  **Điểm tự động (`make grade`):** 84.0 / 88 (Đạt điểm tuyệt đối 84/84 các hạng mục public)


> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G4 và G6: khi alpha chậm 5 s (latency ~6.2 s) hoặc treo, timeout 8.0 s đủ để bắt các call chậm/treo mà không làm cháy ngân sách 30 s của lượt user; ở G4 p95 đạt 5.562 s (thay vì 13.764 s ở naive). |
| max_attempts_per_provider | 2 | Dựa trên G3 và G1: cho phép retry đúng 1 lần trên cùng provider đối với lỗi transient (1 attempt đầu + 1 retry = 2 attempts); ở G3 retry_ratio giảm xuống 0.034 (so với 0.164 của naive), ngăn chặn retry bão. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Dựa trên G1: base 0.25 s giúp phản ứng nhanh khi có lỗi thoáng qua, cap 4.0 s đảm bảo retry không delay quá lâu vượt turn_deadline 30 s; kết hợp full jitter và tôn trọng `retry_after` giúp đạt 0 vi phạm `retry_after_violations` trên G1. |
| retry_budget_ratio | 0.1 | Dựa trên G3 (chặn retry storm): giới hạn số lượng retry không vượt quá 10% tổng số request gửi lần đầu; giữ retry_ratio ở G3 chỉ 0.034 (ngưỡng yêu cầu <= 0.10). |
| breaker.window_size / min_calls | 20 / 10 | Dựa trên G2, G3: cửa sổ 20 kết quả và tối thiểu 10 kết quả giúp breaker phản ứng nhanh khi lỗi dồn dập trong vài giây, đồng thời tránh việc trip giả tạo do 1-2 lỗi ngẫu nhiên trong điều kiện bình thường. |
| breaker.failure_rate_threshold | 0.5 | Dựa trên G2 và G3: ngắt mạch khi tỷ lệ lỗi >= 50% trong window; ở G2 khi alpha bị quá tải 75%, breaker trip ngay lập tức và chuyển hướng lưu lượng sang alpha-eu/beta, giữ availability ở 100%. |
| breaker.slow_call_s | 5.0 | Dựa trên G4: thời gian xử lý bình thường của alpha là median 1.2 s; khi chậm thêm 5 s (tổng ~6.2 s >= 5.0 s), breaker nhận diện là call xuống cấp và trip sớm, giảm p95 từ 13.764 s xuống 5.562 s. |
| breaker.open_duration_s | 10.0 | Dựa trên G2, G4: cho provider nghỉ 10 s trước khi cho probe ở HALF_OPEN, đủ để hàng đợi server giải phóng tải mà không bỏ quên provider quá lâu (G2 recovery_s = 0.0 s). |
| breaker.half_open_max_probes | 2 | Dựa trên G2: giới hạn tối đa 2 request probe đồng thời khi provider vừa bước vào HALF_OPEN, tránh hiện tượng thundering herd đè sập lại provider đang hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                    [ProviderError xảy ra]
                              |
                     Phân loại lỗi (classify)
                              |
        +---------------------+---------------------+
        |                     |                     |
  CLIENT_ERROR          QUOTA_EXHAUSTED       RATE_LIMITED        OVERLOADED / SERVER_ERROR / TIMEOUT
  (400/401/422)             (429)                 (429)                       (503/529/500/0)
        |                     |                     |                                |
  Breaker.release       Breaker.release       Breaker.release                 Breaker.on_failure
        |                     |                     |                                |
  GiveUp("client_error")  Cooldown 300s        Cooldown now+retry_after        tries < max_attempts AND
  (KHÔNG retry,          Fail over ngay        Fail over ngay                  budget cho phép AND
   KHÔNG failover)                                                             vừa deadline?
                                                                                /          \
                                                                            [Có]            [Không]
                                                                              |                |
                                                                        Retry cùng         Fail over sang
                                                                        provider với       provider kế tiếp
                                                                        jitter backoff     trong ladder
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi giả lập breaker bị vô hiệu hóa (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - (a) Giữ retry budget: Trên G2, amplification đạt 1.383, retry_ratio chỉ 0.058, recovery_s = 0.0 s, availability = 0.9987. Trên G3, amplification đạt 1.136, retry_ratio = 0.068.
   - (b) Tắt retry budget: Trên G2, amplification bùng nổ lên 1.672, retry_ratio tăng vọt gấp hơn 8 lần lên 0.491, recovery_s bị kéo dài lên 30.0 s, availability tụt xuống 0.9649. Trên G3, retry_ratio tăng lên 0.162.
   - Khi breaker hoạt động đúng, nó ngắt provider có vấn đề nên retry budget ít bị chạm ngưỡng. Tuy nhiên, vẫn bắt buộc phải có cả hai vì breaker chỉ bảo vệ khi tỷ lệ lỗi tập trung ở một provider, còn khi sự cố diện rộng, breaker trễ phản ứng hoặc cấu hình sai, retry budget là van an toàn toán học độc lập duy nhất khống chế trần tải retry của toàn hệ thống dưới 10%.

2. **`attempt_timeout_s: 30`.** Khi tăng timeout lên 30 s:
   - Trên G4 (slow region): p95 latency đạt 5.683 s, availability đạt 1.0000 (do breaker slow_call_s = 5.0 s vẫn kịp phát hiện các call chậm và trip mạch).
   - Trên G6 (provider outage): p95 latency là 5.306 s, nhưng availability giảm xuống 0.9488 (rớt khỏi ngưỡng pass 0.95) do request bị kẹt chờ tới 30 s ở provider chết trước khi failover, dẫn đến cạn kiệt deadline 30 s của toàn bộ user turn.
   - Timeout "đúng" nên dựa vào phân phối độ trễ SLA thực tế của provider (thường là p99 latency bình thường + một biên nhỏ, ví dụ 3–5× median latency) và ngân sách thời gian còn lại của user turn, tuyệt đối không đặt bằng tổng turn deadline.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Khi bỏ kiểm tra `supports_tools` trên G6:
   - Số lượng `incompatible_routes` tăng vọt từ 0 lên 258 lỗi, kéo availability giảm từ 1.0000 xuống 0.8918.
   - Dưới góc nhìn Product Manager: Có 258 khách hàng yêu cầu hoàn tiền nhưng hệ thống lại chuyển tiếp tác vụ thực thi tài chính sang một model giá rẻ không biết dùng công cụ (gamma-mini), khiến câu lệnh bị hiểu sai hoặc từ chối thực hiện, dẫn đến trải nghiệm khách hàng đổ vỡ và thất thoát niềm tin nghiêm trọng chỉ vì định tuyến kỹ thuật cẩu thả.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 đ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 đ | 0.303 |

**Trả lời:**
- Chỉ có Idempotency Key là **chưa đủ**, vì khi worker bị crash giữa chừng, nếu không có checkpoint lưu tiến trình, worker mới sẽ không biết workflow đã chạy tới bước nào và buộc phải tính toán lại từ đầu (gọi lại LLM, lãng phí tài nguyên và chi phí token).
- Chỉ có Checkpoint cũng **chưa đủ**, vì nếu một side effect (như trừ/hoàn tiền) đã gửi thành công tới cổng thanh toán nhưng worker bị crash ngay trước khi kịp ghi checkpoint vào DurableStore, worker mới chạy lại sẽ gọi lại lệnh hoàn tiền; nếu không có idempotency key, cổng thanh toán sẽ trừ/hoàn tiền lần thứ hai (duplicate refund).
- Nếu ghi checkpoint **trước** khi gọi payment: Trong trường hợp thanh toán gặp lỗi hoặc worker chết ngay thời điểm gọi cổng thanh toán, checkpoint đã ghi nhận là "đã thanh toán". Khi worker khởi động lại, nó đọc thấy checkpoint và bỏ qua bước thanh toán, dẫn đến khách hàng không nhận được tiền hoàn (**lost refund**).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (False) | none | — |
| slow_leak | ticket | none (Miss) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (False) | none | — |
| small_bump | none | page (False) | none | — |

**Trả lời:**
- Error budget 30 ngày: đã tiêu **42.95%** (55,618 lỗi trên 129,508.8 lỗi cho phép), còn **57.05%**.
- **Không đóng băng release** vì ngân sách lỗi còn lại lớn hơn 0 (freeze_releases: False, còn hơn 57%).
- Sự cố `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức on-call lúc 11 giờ trưa vì nó chỉ là một đợt tăng lỗi ngắn hạn tiêu tốn một phần rất nhỏ (< 0.5%) tổng ngân sách lỗi của cả tháng; hệ thống tự hồi phục nhanh và không đe dọa cam kết SLO 99.5% 30 ngày, do đó việc page đánh thức kỹ sư là một false alarm gây cảnh báo kiệt sức (alert fatigue).

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](/reports/gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):**
Vào t=120 s, cụm GPU của provider chính `alpha` bị suy giảm đột ngột 75% năng lực xử lý, gây tăng vọt thời gian chờ trong hàng đợi. Do client naive sử dụng cơ chế retry 3 tầng đồng loạt (tối đa 36 lần/call) với khoảng trễ cố định và không có circuit breaker hay retry budget, tải attempt đã bị khuếch đại gấp gần 10 lần (đạt đỉnh 254 req/s trên alpha). Hậu quả là ngay cả khi provider đã khôi phục 100% tài nguyên ở t=300 s, hệ thống vẫn chìm trong trạng thái tê liệt cận ổn định (metastable failure) thêm 155 giây trước khi có thể tự phục hồi.

**Ảnh hưởng:**
Có **1,285 lượt người dùng thất bại** (availability chỉ đạt 45.60% trên tổng số 2,362 lượt). Sự cố kéo dài liên tục **335 giây** (từ t=120 đến t=455). Sự cố này tiêu tốn hơn **988%** error budget cho phép của kịch bản, gây gián đoạn dịch vụ nghiêm trọng.

**Timeline** (giây ảo — trích xuất từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`):

| t (s) | Sự kiện | Bằng chứng (số liệu từ CSV) |
|---:|---|---|
| 118 | Trạng thái bình thường trước sự cố | attempts_alpha = 5, calls_ok = 10, calls_done = 10 (100% thành công) |
| 120 | alpha mất 75% năng lực xử lý | attempts_alpha = 5, calls_ok bắt đầu giảm (t=121 calls_ok=2) |
| 136 | Tải retry dồn ứ bắt đầu bùng nổ | attempts_alpha nhảy vọt từ 21 lên 53 req/s |
| 144 | Đỉnh bão retry storm (amplification cực đại) | attempts_alpha đạt đỉnh 254 req/s, calls_ok = 0/4 (0% thành công) |
| 200 | Hệ thống kẹt cứng trong quá tải | attempts_alpha duy trì 170–190 req/s, calls_ok = 0 |
| 300 | alpha hồi phục hoàn toàn năng lực phần cứng | attempts_alpha = 115 req/s, calls_ok = 0/5 |
| 350 | Hàng đợi server vẫn nghẽn vì request zombie | attempts_alpha = 16 req/s, nhưng calls_ok vẫn = 0/2 (chưa hồi phục) |
| 455 | Hàng đợi cũ tiêu tán hết, hệ thống bắt đầu phục hồi | t=453: calls_ok = 4/8; t=456: calls_ok = 8/8 (100% thành công) |

**Trigger vs root cause:**
- **Trigger:** Provider alpha bị mất 75% số lượng GPU trong 3 phút (t=120–300 s).
- **Root cause:** Client naive xếp chồng 3 lớp retry (app, gateway, SDK) với delay cố định, không có jitter, không có circuit breaker và không có trần retry budget. Khi provider chậm lại, các request bị timeout client vẫn tiếp tục chạy trên server trong khi client liên tục gửi lại các request mới, tạo ra vòng lặp phản hồi dương (positive feedback loop) khuếch đại tải lên tới 254 rps (vượt xa sức chứa 20 rps của tenant).
- **Metastable failure:** Sau t=300 s, năng lực server đã phục hồi nhưng hàng đợi FIFO vẫn chứa đầy các request cũ đã quá hạn (zombie requests) mà client đã bỏ cuộc; server lãng phí 100% công suất để xử lý các request vô ích này trong khi từ chối các request mới, khiến trạng thái sập tự duy trì thêm 155 giây.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* 3 tầng retry nhân số lượng attempt lên tối đa 36 lần; delay cố định gây hiện tượng sóng tập trung; server vẫn tiếp tục xử lý các request mà client đã timeout; không có cơ chế failover sang alpha-eu hay beta.
- *Giúp khắc phục (ở client resilient):* Circuit breaker ngắt mạch khi tỷ lệ lỗi/chậm tăng cao; fallback chuyển bớt tải sang alpha-eu và beta; retry budget giới hạn retry <= 10%; full jitter phân tán đều các lần thử lại.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient với sliding-window circuit breaker và retry budget 10% trên toàn bộ gateway | Prevent | Core Infra Team | Kiểm tra amplification <= 1.10 và retry_ratio <= 0.05 trên kịch bản G2 trong CI pipeline |
| 2 | Cấu hình burn-rate alerting đa cửa sổ (1h/5m, 6h/30m) cho SLO availability | Detect | SRE Team | Tự động phát hiện và kích hoạt page trong vòng 5 phút khi xảy ra sự cố sập tương tự G2 |
| 3 | Thêm cờ hủy xử lý (cancellation propagation/deadline propagation) từ client xuống server khi client timeout | Mitigate | Backend Platform | Đo lường recovery_s <= 5 s ngay sau khi khôi phục tài nguyên server trên môi trường staging |

## 7. Một điểm yếu còn lại

**Tình huống client vẫn xử lý kém:**
Trong kịch bản cả 4 provider cùng bị suy giảm hiệu năng đồng thời (all-provider brown-out/slowdown) hoặc khi provider dự phòng `beta` phải gánh toàn bộ tải nhưng có chi phí cao (1.4× so với alpha), chi phí tài chính sẽ tăng vọt nếu lưu lượng bị giữ ở beta quá lâu mà không thăm dò chủ động để quay về alpha. Ngoài ra, việc dùng chung một retry budget tĩnh (token bucket) cho tất cả các provider có thể khiến việc retry trên provider phụ bị từ chối dù provider đó hoàn toàn khỏe mạnh.

**Cách khắc phục:**
1. Áp dụng **Adaptive Concurrency Limits** (ví dụ thuật toán Vegas hoặc AIMD) trên từng provider client để điều chỉnh động số lượng request in-flight thay vì chỉ dựa vào timeout cố định.
2. Tách riêng **Retry Budget theo từng provider** để tránh việc một provider bị lỗi làm cạn kiệt ngân sách retry của toàn hệ thống.
3. Thiết lập cơ chế **Probabilistic Failback (Canary Routing)** chủ động gửi 1-2% traffic về alpha sau thời gian ngắn để kiểm tra độ trễ thực tế trước khi chờ đợi hết timeout open của breaker.

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9120 | 0.9645 | 0.635 | 174 | 0 | 6 | 0.927 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Với ngưỡng 0.80, có tới 142 câu hỏi bị trả lời sai (`other_question: 142`); với 0.92, `wrong_answers` triệt để về 0 trong khi vẫn giữ `cache_hit_ratio` cao ở mức 0.635 (vượt chuẩn >= 0.50). |
| ttl_s | 45.0 | Khi bảng phí ship đổi âm thầm ở t=480 s, TTL 45 s giới hạn số lượng `outdated_answers` chỉ còn 6 (yêu cầu <= 15); nếu để TTL 300 s thì số câu lạc hậu vọt lên 95. |
| max_stale_s | 300.0 | Đợt cúp provider toàn diện kéo dài 120 s (t=240–360 s); `max_stale_s: 300` cho phép phục vụ 174 câu trả lời stale an toàn, kéo `faq_availability` từ 86.34% lên 96.45% (vượt chuẩn >= 93%). |

**Thí nghiệm:**

1. **Ngưỡng similarity:**
   - Tại `0.80`: hit_ratio = 0.630, `wrong_answers` = 142 (toàn bộ do `other_question`), cost = 0.985, faq_avail = 0.8820. Các cặp câu hỏi có embedding gần nhau nhưng ý nghĩa hoàn toàn khác nhau (ví dụ: "Thời gian xử lý hoàn tiền là bao lâu?" vs "Chính sách hoàn tiền thế nào?") bị nhầm lẫn nghiêm trọng.
   - Tại `0.92`: hit_ratio = 0.635, `wrong_answers` = 0, cost = 0.927, faq_avail = 0.9645. Không có bất kỳ câu trả lời nhầm lẫn nào.
   - Tại `0.97`: hit_ratio tụt xuống 0.550, `wrong_answers` = 0, cost tăng lên 0.985 do nhiều câu diễn đạt khác nghĩa chuẩn (paraphrase) bị trượt cache và phải gọi lại provider tốn tiền.

2. **TTL (Time-To-Live):**
   - Với `ttl_s = 30`: `outdated_answers` = 2, hit_ratio = 0.592, cost = 0.955.
   - Với `ttl_s = 60`: `outdated_answers` = 3, hit_ratio = 0.652, cost = 0.915.
   - Với `ttl_s = 300`: `outdated_answers` = 95 (vi phạm nặng yêu cầu <= 15), hit_ratio = 0.731, cost = 0.863.
   - Webhook chỉ có thể gửi khi hệ thống CMS chủ động thông báo sự kiện thay đổi. Khi bên thứ ba (hãng vận chuyển) âm thầm đổi biểu phí mà không thông báo, client hoàn toàn không nhận được webhook invalidation; do đó, TTL ngắn hạn là chốt chặn phòng thủ duy nhất buộc cache phải hết hạn và kéo dữ liệu mới về.

3. **Stale-if-error:**
   - Khi chạy với `max_stale_s: 0`: `stale_served` giảm về 0, `faq_availability` tụt từ 96.45% xuống **86.34%** (không đạt chuẩn >= 93%), và availability tổng thể tụt xuống 85.01%.
   - Điều này chứng minh tính năng stale-if-error là vị cứu tinh sống còn giúp giữ trải nghiệm FAQ của người dùng ổn định ở mức cao trong suốt thời gian các provider sập hoàn toàn.

**Trả lời câu hỏi:**
- **Vì sao câu hỏi cá nhân không được cache:** Dữ liệu cá nhân (ví dụ: trạng thái đơn hàng, số tiền hoàn, điểm tích lũy) là dữ liệu động mang tính riêng tư cao và thay đổi liên tục theo thời gian thực. Nếu cache câu hỏi cá nhân, ngay cả khi phân theo `user_id`, người dùng sẽ nhận thông tin trạng thái cũ (stale status). Nguy hiểm hơn, nếu có lỗi về key hoặc rò rỉ phân quyền, dữ liệu nhạy cảm của khách hàng này sẽ bị hiển thị cho khách hàng khác, vi phạm nghiêm trọng quy định bảo mật dữ liệu.
- **Alert khi đang phục vụ stale trong lúc provider sập:** Nếu chỉ nhìn vào dashboard mức ứng dụng (User Facing Availability), biểu đồ vẫn xanh do cache đang trả kết quả stale. Do đó, đội ngũ SRE bắt buộc phải thiết lập cảnh báo dựa trên **Backend Provider Error Rate** (tỷ lệ 5xx/timeout của provider) và metric **Stale Response Serving Rate / Cache Stale Ratio** (`stale_served / total_requests`). Khi tỷ lệ trả về stale vượt quá 0% hoặc lưu lượng gọi provider thất bại, hệ thống giám sát phải lập tức phát cảnh báo (page) cho on-call dù availability của người dùng vẫn đang được che chở.
