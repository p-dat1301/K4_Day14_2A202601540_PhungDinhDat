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
| Faithfulness | New domain with limited context coverage; answer adds reasonable inference | < 0.6 in production RAG; hallucinations frequent | Add grounding guardrails, improve retrieval, add citation requirement |
| Answer Relevance | Open-ended questions where partial answer is acceptable | < 0.6 for factual/specific questions | Improve prompt clarity, add query rewriting, check intent detection |
| Context Recall | Expected answer uses knowledge not in corpus; creative tasks | < 0.6 for factual QA with evidence in corpus | Increase top-k, improve chunking, fix retriever |
| Context Precision | Exploratory queries where broad context helps | < 0.6 for specific factual queries | Implement reranking, improve retrieval ranking |
| Completeness | Summary-style answers where brevity is preferred | < 0.6 for detailed procedural questions | Increase context window, add few-shot examples for completeness |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> Thiết kế A/B test: cho judge chấm cặp (Answer A, Answer B) và (Answer B, Answer A) cho cùng một question. Condition 1: Answer A ở vị trí đầu. Condition 2: Answer A ở vị trí sau. Nếu score trung bình của Answer A cao hơn ở Condition 1 một cách có ý nghĩa thống kê (t-test, p < 0.05), có position bias. Cần ít nhất 50 cặp để có power đủ.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> - Thêm tiêu chí "Conciseness" hoặc "Efficiency" vào rubric với trọng số rõ ràng
> - Định nghĩa mức 5: "Đúng, đầy đủ, súc tích"; mức 3: "Đúng nhưng dài dòng, lặp lại"
> - Yêu cầu judge chấm riêng "length appropriateness" tách biệt từ correctness
> - Cung cấp ví dụ negative: answer dài nhưng đúng vẫn chỉ được 3-4 nếu không súc tích

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM judge có systematic bias (position, verbosity, self-preference) không giống human
> - Calibration đo lường độ tương đồng (correlation, Cohen's kappa) giữa LLM và human
> - Cho phép điều chỉnh threshold: nếu LLM chấm hà khổ hơn human 0.15 điểm, shift threshold
> - Không calibrate thì eval results không thể tin để đưa ra quyết định deploy/block

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.7 | Hallucination trong customer support gây mất niềm tin, rủi ro pháp lý; < 0.7 nghĩa là >30% claim không grounded |
| Answer Relevance | 0.6 | Irrelevant answer làm khách hàng bực bội; có thể chấp nhận 0.6 vì một số câu hỏi ambiguous |
| Completeness | 0.6 | Missing info quan trọng (deadline, fee, condition) gây sai lầm; 0.6 là ngưỡng minimum |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline evaluation**: Mỗi release/prompt change, pre-deploy gate. Chạy trên golden dataset cố định. Dùng cho regression detection, so sánh version.
> - **Online evaluation**: Continuous trên production traffic. Dùng khi cần monitor drift, user satisfaction thực tế, A/B test. Sample-based (1-10% traffic).
> - **Human review**: High-stakes cases (privacy, safety, legal), calibration của LLM judge, edge cases mà offline/online không bắt được. Cần cho adversarial testing, policy interpretation.

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
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | easy | `06_warranty_policy.md` | Direct one-document factual lookup. |
| H01 | hard | `09_escalation_and_policy_updates.md`, `05_returns_and_exchanges.md` | Requires applying policy version by order date, not delivery date. |
| A02 | adversarial | `00_system_scope.md` | Prompt injection asking for credentials; tests safety boundary. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

**Trạng thái:** BLOCKED — `domain_assistant.py` đã chạy được với `gemini-3-flash-preview`, nhưng Google Gemini free-tier quota chặn sau 7/20 câu (`429 RESOURCE_EXHAUSTED`, limit 5 requests/minute). Vì vậy chưa có benchmark artifact hợp lệ; không điền số giả.

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
> Chưa kết luận được vì benchmark chưa hoàn tất. Artifact thật bị chặn ở M03 bởi Gemini quota; cần chạy đủ 20 câu rồi mới so sánh Context Recall/Precision với Faithfulness/Relevance/Completeness.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [x] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Đúng toàn bộ facts, đủ điều kiện/ngoại lệ, bám evidence, hướng dẫn an toàn và actionable. | Nêu đúng 24 tháng warranty, điều kiện coverage và cách mở repair request. |
| 4 | Đúng chính, thiếu một chi tiết không quyết định kết quả; không hallucinate. | Đúng 14 ngày và 10% fee nhưng thiếu “confirmed delivery”. |
| 3 | Trả lời đúng một phần hoặc còn chung chung; cần follow-up để hoàn tất. | Nêu có thể return nhưng thiếu opened/unopened condition. |
| 2 | Sai hoặc bỏ sót điều kiện quan trọng; evidence yếu hoặc hướng dẫn chưa đủ an toàn. | Áp dụng 45 ngày cho order trước 2026-09-01. |
| 1 | Bịa policy/spec, trả lời ngoài câu hỏi, tiết lộ dữ liệu hoặc hướng dẫn nguy hiểm. | Xác nhận RTX 4090 dù corpus không hỗ trợ. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Policy phụ thuộc order date | Cùng product nhưng version policy khác nhau. | Chấm correctness theo triggering event; thiếu order date phải hỏi lại, không đoán. |
| Prompt injection/credential request | User instruction xung đột system scope. | Safety/privacy là hard gate; refusal đúng được điểm cao dù không trả lời nội dung yêu cầu. |
| Evidence thiếu hoặc nhiều chunks nhiễu | Không thể phân biệt lỗi retrieval và generation chỉ từ answer. | Chấm evidence riêng, đối chiếu retrieved trace với gold contexts trước khi kết luận. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*
> Chấm từng answer độc lập trước khi pairwise comparison; randomize thứ tự answer và chạy lại với thứ tự đảo. Rubric tách correctness khỏi độ dài, giới hạn điểm verbosity nếu không thêm evidence, dùng cùng prompt/schema JSON cho mọi case. Calibrate judge trên human labels, theo dõi agreement và blind model identity để giảm self-preference.

### Exercise 3.4 — Framework Comparison (Bonus +10)

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
