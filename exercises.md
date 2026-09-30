# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu hỏi ngoài phạm vi (out-of-scope/adversarial) hoặc chào hỏi xã giao, trợ lý trả lời lịch sự hoặc từ chối đúng quy định bằng kiến thức tổng quát chứ không dựa vào retrieved context (context rỗng). | Các câu hỏi về chính sách bảo hành, hoàn tiền, giá bán, hoặc thông số kỹ thuật quan trọng của OrbitTech nhưng câu trả lời bịa đặt (hallucination), sai lệch so với context tài liệu. | Thắt chặt system prompt ("Chỉ trả lời dựa trên context được cung cấp, nếu không có thông tin hãy nói không biết"), giảm temperature về 0, bổ sung grounding check / guardrails trước khi xuất câu trả lời. |
| Answer Relevance | Khách hỏi câu hỏi phức tạp hoặc mơ hồ, trợ lý cần giải thích các điều kiện liên quan hoặc đưa thêm câu hỏi làm rõ/hướng dẫn liên hệ hỗ trợ thay vì chỉ trả lời cộc lốc một ý. | Khách hỏi một đằng trả lời một nẻo (off-topic), ví dụ hỏi về "Thời gian bảo hành tai nghe" nhưng trợ lý trả lời về "Chính sách đổi trả bàn phím". | Tinh chỉnh prompt hướng dẫn trả lời tập trung vào trọng tâm câu hỏi, bổ sung intent classification hoặc rewrite query để tránh đưa ngữ cảnh gây xao nhãng vào prompt. |
| Context Recall | Expected answer chứa các chi tiết nền tảng chung ngoài tài liệu hỗ trợ cửa hàng mà retriever không cần lấy về hết vẫn đảm bảo trả lời đúng câu hỏi cốt lõi. | Retriever bỏ sót các điều khoản quan trọng trong tài liệu chính sách (ví dụ: điều kiện từ chối bảo hành, thời hạn 30 ngày) khiến trợ lý không đủ thông tin để trả lời. | Tăng số lượng chunks lấy về (`top_k`), tối ưu hóa chiến lược chunking (kích thước chunk và overlap), nâng cấp retriever (kết hợp Dense Retrieval + BM25 Hybrid Search). |
| Context Precision | Cài đặt `top_k` lớn (ví dụ k=10) trong pha thăm dò; chunk liên quan xếp thứ 2 hoặc 3 nhưng generator vẫn tổng hợp và trả lời chính xác. | Chunk chứa thông tin cốt lõi bị xếp ở cuối danh sách hoặc bị chìm giữa các chunk nhiễu (lost in the middle), làm generator bỏ sót hoặc bị phân tâm. | Tích hợp reranker (Cross-encoder reranking hoặc keyword overlap reranking) để đẩy chunk quan trọng lên đầu; lọc bỏ các chunks có điểm tương đồng thấp trước khi đưa vào context. |
| Completeness | Câu hỏi tóm tắt nhanh hoặc câu hỏi định nghĩa ngắn gọn, khách hàng chỉ cần câu trả lời cô đọng, không cần liệt kê toàn bộ danh mục chi tiết như expected_answer. | Khách hỏi quy trình nhiều bước (ví dụ: các bước gửi bảo hành hoặc điều kiện đổi trả), nhưng câu trả lời chỉ nêu 1 bước rồi dừng, thiếu sót nghiêm trọng thông tin hướng dẫn. | Yêu cầu trong prompt trả lời đầy đủ theo cấu trúc checklist/step-by-step; tăng `max_tokens`; tinh chỉnh chunking để không làm đứt đoạn các quy trình liền mạch. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế thực nghiệm (Pairwise Comparison):** Chuẩn bị một tập câu hỏi kiểm thử (ví dụ: 30-50 câu hỏi từ golden dataset) cùng hai phương án trả lời A và B có chất lượng tương đương.
> - **Condition 1 (Original Order):** Prompt LLM Judge đánh giá với Answer A ở vị trí 1 (Option 1) và Answer B ở vị trí 2 (Option 2).
> - **Condition 2 (Swapped Order):** Prompt LLM Judge với thứ tự đảo ngược: Answer B ở vị trí 1 (Option 1) và Answer A ở vị trí 2 (Option 2), giữ nguyên hoàn toàn rubric và câu hỏi.
> - **Đánh giá bias:** So sánh tỷ lệ thắng của Option 1 ở cả hai condition. Nếu Option 1 luôn thắng với tỷ lệ áp đảo (> 60% bất kể là Answer A hay B), LLM Judge có biểu hiện Position Bias rõ rệt. Khi đánh giá thực tế cần dùng kỹ thuật đảo vị trí (bidirectional evaluation) và lấy điểm trung bình của cả hai chiều để triệt tiêu bias này.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - **Định nghĩa tiêu chí theo mật độ thông tin và độ súc tích:** Trong rubric, nêu rõ tiêu chí chấm dựa trên tính chính xác, tính đầy đủ của thông tin cốt lõi (information density) thay vì độ dài văn bản; quy định rõ ràng rằng câu trả lời dài dòng, lan man, lặp ý sẽ bị trừ điểm.
> - **Cung cấp Few-shot Anchor Examples:** Đưa các ví dụ mẫu cụ thể trong prompt của Judge, trong đó câu trả lời ngắn gọn, đúng trọng tâm được gán điểm tối đa (5/5), còn câu trả lời dài dòng nhưng ít giá trị thông tin chỉ nhận điểm trung bình/thấp (2/5 hoặc 3/5).
> - **Chuẩn hóa input hoặc đặt ràng buộc độ dài:** Hướng dẫn Judge tập trung kiểm tra danh sách sự thật (fact checklist) thay vì cảm nhận văn phong tổng thể, hoặc tóm tắt/chuẩn hóa cấu trúc câu trả lời trước khi gửi vào LLM Judge.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - **Đảm bảo tính tin cậy và căn chỉnh theo tiêu chuẩn thực tế:** LLM Judge có thể có các thiên kiến nội tại (như severity bias - quá khắt khe, leniency bias - quá dễ dãi, hoặc self-preference - thiên vị văn phong của chính nó). Calibrate với human labels (nhãn đánh giá từ chuyên gia miền nghiệp vụ) giúp đo lường mức độ tương quan (Correlation như Pearson/Spearman, Cohen's Kappa) giữa điểm số của AI Judge và con người.
> - **Xác định threshold chính xác cho Production/CI-CD:** Chỉ khi LLM Judge được chứng minh là có độ đồng thuận cao với con người (e.g. agreement rate > 80%), các ngưỡng threshold (như 0.8 hay 0.5) trong pipeline tự động mới có giá trị thực tiễn, tránh việc hệ thống từ chối oan bản phát hành tốt (false positives) hoặc bỏ lọt lỗi nghiêm trọng lên production (false negatives).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | Đối với trợ lý CSKH OrbitTech, tính trung thực chống hallucination là ưu tiên số một. Thông tin sai lệch về chính sách bảo hành, giá bán hoặc hoàn tiền sẽ trực tiếp gây tổn thất tài chính và rủi ro pháp lý cho cửa hàng. |
| Answer Relevance | 0.80 | Câu trả lời bắt buộc phải trả lời đúng thắc mắc của khách hàng. Điểm relevance dưới 0.80 khiến khách hàng thất vọng, phải hỏi lại nhiều lần hoặc làm tăng tỷ lệ chuyển sang nhân viên tổng đài (tăng chi phí vận hành). |
| Completeness | 0.70 | Cần đảm bảo cung cấp đủ các bước hướng dẫn và điều kiện then chốt, tuy nhiên ngưỡng có thể linh hoạt hơn (0.70) để chấp nhận cách diễn đạt ngắn gọn, súc tích thay vì bắt buộc dài như ground-truth. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment):** Dùng trong giai đoạn phát triển (development), pre-commit, hoặc pre-merge trong CI/CD pipeline trên tập Golden Dataset (20-100+ test cases). Mục tiêu là phát hiện hồi quy chất lượng (regression testing) nhanh chóng, chi phí thấp, an toàn tuyệt đối trước khi release phiên bản mới.
> - **Online Evaluation (Post-deployment / Production):** Dùng liên tục khi hệ thống đang phục vụ người dùng thực tế. Đo lường qua tín hiệu telemetry thời gian thực: implicit signals (thời gian xem, tỷ lệ copy, tỷ lệ escalate) và explicit feedback (thumbs up/down), kết hợp lấy mẫu ngẫu nhiên (e.g. 5% request) để chạy LLM Judge. Mục tiêu là phát hiện concept drift, edge cases chưa có trong golden dataset và theo dõi trải nghiệm thực tế.
> - **Human Review (Auditing & Calibration):** Dùng định kỳ hoặc có chủ đích: (1) Kiểm định và cập nhật golden dataset khi có chính sách mới; (2) Calibrate LLM Judge để duy trì độ chuẩn xác; (3) Audit các trường hợp điểm số thấp, các đoạn chat bị khách hàng bấm thumbs down hoặc các giao dịch nhạy cảm có khiếu nại nghiêm trọng.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
