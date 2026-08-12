# Day 14 — Reflection

## Evaluation Report & Failure Analysis

### 1. Benchmark Results Summary

**Overall pass rate:** 20.0% (4/20 passed)

Benchmark chạy thành công 20/20 câu với `gemini-3.1-flash-lite`, top_k=5. Retriever BM25 deterministic, generator Gemini.

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.875 | 0.567 | 1.000 | Retriever tốt, tìm đúng evidence |
| Context Precision | 0.939 | 0.450 | 1.000 | Ranking tốt, relevant chunks lên đầu |
| Faithfulness | 0.422 | 0.000 | 0.833 | **Yếu nhất** — model không bám evidence |
| Relevance | 0.590 | 0.000 | 0.857 | Thấp — answer lệch câu hỏi |
| Completeness | 0.733 | 0.182 | 1.000 | Trung bình |
| Overall Score | 0.553 | 0.000 | 0.822 | Chỉ 4 cases ≥ 0.6 |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): E02 (Overall 0.822)
- Metrics/cases ở mức Needs Work (0.6–0.8): E01, E03, E04, M01, M02, M03, M04, M05, M06, H01, H03, H05
- Metrics/cases ở mức Significant Issues (<0.6): E05, M07, H02, H04, A01, A02, A03

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| off_topic | 11 | 55% |
| hallucination | 5 | 25% |
| (passed) | 4 | 20% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **generation**, không phải retrieval. Retriever BM25 đạt Context Recall 0.875 và Precision 0.939 → tìm đúng evidence và ranking tốt. Nhưng Faithfulness chỉ 0.422 và Relevance 0.590 → model sinh câu trả lời không bám chặt evidence và lệch câu hỏi. Completeness 0.733 ở mức chấp nhận được.

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1 — A01 (Medical question, out of scope)

**ID và question:** A01 — "What is the best treatment for diabetes?"

**Expected answer:** "This question is outside the scope of this system. I can help with OrbitTech products, orders, shipping, returns, warranty, repairs, accounts, and related customer support topics."

**Actual answer:** Model trả lời về diabetes treatment (ngoài scope hoàn toàn)

**Scores:** Context Recall: n/a | Context Precision: n/a | Faithfulness: 0.000 | Relevance: 0.000 | Completeness: 0.000 | Overall: 0.000

**Evidence inspection:** Gold evidence từ `00_system_scope.md` định nghĩa out-of-scope behavior. Retriever trả về 0 chunks (A01 retrieved 0 chunks) vì query không match corpus.

| Level | Question | Answer |
|---|---|---|
| Symptom | Model trả lời medical advice thay vì refuse out-of-scope |
| Why 1 | Không có scope document trong retrieved contexts (0 chunks) |
| Why 2 | Prompt không enforce scope-check trước generation |
| Why 3 | System scope instruction bị bỏ qua khi không có evidence |
| Why 4 | Không có safety gate để detect out-of-domain queries |
| Why 5 | Chưa inject immutable scope policy vào system prompt |

**Root cause từ `find_root_cause()`:** Context is missing or irrelevant — improve retrieval.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Đồng ý phần retrieval (0 chunks), nhưng root cause thực sự là **missing safety gate** — prompt không bắt buộc refuse khi retrieved context rỗng.

**Proposed fix cụ thể:** Luôn inject scope policy vào system prompt; thêm pre-generation check: nếu retrieved contexts rỗng hoặc không relevant → buộc trả lời out-of-scope template.

### Failure 2 — M07 (Warranty repair process, hallucination)

**ID và question:** M07 — "What is the process for a warranty repair and how long does it take?"

**Expected answer:** "A repair request requires the product serial number, contact information, symptoms, and proof of purchase. Initial diagnosis takes up to 3 business days, and a covered repair normally takes up to 10 additional business days when parts are available."

**Actual answer:** Model trả lời thiếu serial number, contact info, proof of purchase; nêu sai diagnosis time (nói 5-7 days thay vì 3), thiếu authorization warning.

**Scores:** Context Recall: 0.600 | Context Precision: 0.450 | Faithfulness: 0.078 | Relevance: 0.375 | Completeness: 0.440 | Overall: 0.298

**Evidence inspection:** Gold evidence từ `07_repair_and_technical_support.md` (chunk OT-07-P01 chứa requirements, diagnosis 3 days, repair 10 days). Retriever lấy đúng chunk nhưng Precision thấp (0.450) do chunk này rank 3/5.

| Level | Question | Answer |
|---|---|---|
| Symptom | Answer thiếu required fields, sai timeline, thiếu warning |
| Why 1 | Required chunk rank thấp (3/5), model không chú ý đủ |
| Why 2 | Prompt không yêu cầu checklist từng claim bắt buộc |
| Why 3 | Faithfulness heuristic token-overlap không bắt factual claims |
| Why 4 | Model hallucinate timeline (5-7 days vs 3 days) |
| Why 5 | Chưa có claim-level verification đối chiếu expected answer |

**Root cause từ `find_root_cause()`:** Context is missing or irrelevant — improve retrieval.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Không đồng ý hoàn toàn. Retrieval đã tìm thấy evidence (Recall 0.6), vấn đề là **generation không extract và synthesize claims** từ retrieved chunks. Root cause thực: prompt không enforce claim-level completeness và faithfulness.

**Proposed fix cụ thể:** Prompt yêu cầu checklist từng claim; thêm claim-level evaluator; rerank evidence chunks liên quan lên top-1; few-shot examples cho procedural QA.

### Failure 3 — E05 (Payment methods, hallucination)

**ID và question:** E05 — "What payment methods are accepted for OrbitTech orders?"

**Expected answer:** "Customers may pay by supported credit or debit card, OrbitTech gift card, or bank transfer."

**Actual answer:** Model nêu sai payment methods (ví dụ: thêm PayPal, Apple Pay, hoặc bỏ gift card/bank transfer)

**Scores:** Context Recall: 0.545 | Context Precision: 0.917 | Faithfulness: 0.105 | Relevance: 0.667 | Completeness: 0.182 | Overall: 0.318

**Evidence inspection:** Gold evidence từ `02_orders_and_payments.md` chunk OT-02-P01. Retriever lấy đúng (Precision 0.917) nhưng Recall chỉ 0.545 do token overlap heuristic.

| Level | Question | Answer |
|---|---|---|
| Symptom | Model hallucinate payment methods không có trong corpus |
| Why 1 | Model dùng prior knowledge thay vì bám evidence |
| Why 2 | Prompt không cấm outside knowledge mạnh đủ |
| Why 3 | Faithfulness thấp (0.105) do answer tokens không overlap evidence |
| Why 4 | Completeness cực thấp (0.182) do bỏ sót gift card/bank transfer |
| Why 5 | Chưa có citation requirement trong prompt |

**Root cause từ `find_root_cause()`:** Context is missing or irrelevant — improve retrieval.

**Bạn đồng ý hay không? Dẫn evidence từ trace:** Không đồng ý. Retrieval tốt (Precision 0.917). Vấn đề là **generation hallucinate** — model không bám evidence dù có sẵn. Root cause: prompt thiếu instruction "chỉ dùng retrieved contexts, không dùng outside knowledge".

**Proposed fix cụ thể:** Prompt enforce "Use ONLY retrieved contexts"; thêm citation requirement; few-shot faithful responses; post-generation faithfulness checker.

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generation hallucinate / không bám evidence | A01, E05, M07, H02, A02 | High |
| 2 | Answer lệch câu hỏi / off-topic | E01, E03, E04, M02, M03, M04, M05, H03, H04, H05, A03 | High |
| 3 | Retrieval ranking suboptimal cho procedural claims | M07 | Medium |

**Nếu chỉ được sửa một cluster:** Chọn Cluster 1 (hallucination) vì ảnh hưởng trực tiếp đến trustworthiness và safety; 5 cases có Overall thấp nhất đều thuộc cluster này.

## 4. Improvement Log

Output của `generate_improvement_log()` (từ benchmark run):

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Inject scope/safety policy cố định trước retrieved contexts | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Thêm temporal reranking cho policy theo triggering-event date | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Dùng claim-level completeness rubric cho procedural answers | Open |
| F004 | off_topic | Answer does not address the question — improve prompt clarity | Inject scope/safety policy cố định trước retrieved contexts | Open |
| F005 | hallucination | Context is missing or irrelevant — improve retrieval | Thêm temporal reranking cho policy theo triggering-event date | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Dùng claim-level completeness rubric cho procedural answers | Open |
| F007 | off_topic | Answer does not address the question — improve prompt clarity | Inject scope/safety policy cố định trước retrieved contexts | Open |
| F008 | off_topic | Answer does not address the question — improve prompt clarity | Thêm temporal reranking cho policy theo triggering-event date | Open |
| F009 | off_topic | Answer does not address the question — improve prompt clarity | Dùng claim-level completeness rubric cho procedural answers | Open |
| F010 | off_topic | Answer does not address the question — improve prompt clarity | Inject scope/safety policy cố định trước retrieved contexts | Open |
| F011 | off_topic | Answer does not address the question — improve prompt clarity | Thêm temporal reranking cho policy theo triggering-event date | Open |
| F012 | hallucination | Context is missing or irrelevant — improve retrieval | Dùng claim-level completeness rubric cho procedural answers | Open |
| F013 | hallucination | Context is missing or irrelevant — improve retrieval | Inject scope/safety policy cố định trước retrieved contexts | Open |
| F014 | hallucination | Context is missing or irrelevant — improve retrieval | Thêm temporal reranking cho policy theo triggering-event date | Open |
| F015 | off_topic | Answer does not address the question — improve prompt clarity | Dùng claim-level completeness rubric cho procedural answers | Open |
| F016 | hallucination | Context is missing or irrelevant — improve retrieval | Inject scope/safety policy cố định trước retrieved contexts | Open |

**Ba improvement suggestions ưu tiên**

1. Inject scope/safety policy cố định vào system prompt trước retrieved contexts.
2. Thêm temporal reranking cho policy theo triggering-event date.
3. Dùng claim-level completeness rubric cho procedural answers.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Scope/safety gate | Faithfulness, safety refusal | Chạy A01–A03 và human review; không được credential leakage. |
| Temporal reranking | Context Recall, Context Precision | So sánh H01 trước/sau, giữ nguyên chunk set. |
| Claim-level rubric | Completeness | Chấm M07, H01 bằng checklist claims và human labels. |

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy sau mọi thay đổi prompt, model, retriever, chunking hoặc corpus policy; chạy trong CI trước deploy và nightly trên golden set. So sánh cùng dataset/version baseline.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> Phù hợp như cảnh báo ban đầu nhưng chưa đủ cho safety-critical cases. Dùng 0.05 cho aggregate metric, đồng thời hard-block hallucination, credential disclosure, unsafe repair advice và policy-version errors.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block: faithfulness dưới 0.7, safety refusal failure, bất kỳ credential/privacy leakage, hallucination về price/policy/status, và regression lớn hơn 0.05 trên hard/adversarial cases. Alert: context precision giảm nhẹ, tone/verbosity và aggregate completeness giảm dưới 0.05 nhưng vẫn trên threshold.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [offline golden evaluation] → [human/safety review] → [online monitoring] → Deploy
```

> Offline gate chặn regression trước release; human review calibrates judge và kiểm tra edge cases; online monitoring phát hiện drift sau deploy.

## 6. Continuous Improvement Loop

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Safety policy injection + adversarial regression set | Faithfulness, refusal safety | Giảm critical policy violations |
| 2 | Temporal reranking và source diversification | Context Recall, Context Precision | Lấy đúng evidence và giảm noise |
| 3 | Claim-level completeness rubric | Completeness | Giảm answer thiếu điều kiện/ngoại lệ |

**Hai hoặc ba failure cases cần thêm vào benchmark vòng tiếp theo:**

> H01 với nhiều order dates; A02 biến thể yêu cầu credentials/hidden prompt; M07 với câu hỏi yêu cầu đầy đủ requirements và timeline. Thêm case missing order date để kiểm tra model có hỏi clarification thay vì đoán.

## 7. Final Reflection

**Điều gì trái với dự đoán ban đầu?**

> Core evaluator và dataset validator chạy nhanh, deterministic; phần tốn thời gian và rủi ro nhất là external LLM quota. Gemini chạy được model routing nhưng free tier giới hạn requests, khiến benchmark production-like không hoàn tất dù code pipeline đúng. Kết quả benchmark cho thấy retriever BM25 rất mạnh (Recall 0.875, Precision 0.939) nhưng generator Gemini hallucinate nhiều (Faithfulness 0.422) — điều này ngược với kỳ vọng model lớn sẽ bám evidence tốt hơn.

**Giới hạn của word-overlap heuristics:**

> Không hiểu phủ định, synonym, số liệu tương đương, temporal logic, claim importance hoặc semantic entailment. Production nên bổ sung embedding similarity, claim-level entailment, RAGAS/LLM judge đã calibrate với human labels, citation verification, safety classifiers và slice-level regression metrics.