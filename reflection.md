# Day 14 — Reflection

## Evaluation Report & Failure Analysis

### 1. Benchmark Results Summary

**Overall pass rate:** Chưa có — benchmark thật bị block trước khi sinh đủ 20 answers.

Gemini `gemini-3-flash-preview` chạy thành công 7 câu đầu, sau đó trả `429 RESOURCE_EXHAUSTED` do free-tier quota. Không tạo `artifacts/actual_answers.json` hoàn chỉnh và không suy diễn score từ partial run.

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | n/a | n/a | n/a | Chưa có artifact đầy đủ |
| Context Precision | n/a | n/a | n/a | Chưa có artifact đầy đủ |
| Faithfulness | n/a | n/a | n/a | Chưa có artifact đầy đủ |
| Relevance | n/a | n/a | n/a | Chưa có artifact đầy đủ |
| Completeness | n/a | n/a | n/a | Chưa có artifact đầy đủ |
| Overall Score | n/a | n/a | n/a | Chưa có artifact đầy đủ |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Chưa kết luận.
- Metrics/cases ở mức Needs Work (0.6–0.8): Chưa kết luận.
- Metrics/cases ở mức Significant Issues (<0.6): Chưa kết luận.

**Failure type distribution:** Chưa kết luận vì chưa chạy đủ dataset.

**Chẩn đoán tổng quan:** Chưa thể tách retrieval và generation bằng score thật. Retriever đã chạy deterministic BM25 và lưu trace theo từng chunk; cần artifact hoàn chỉnh để đối chiếu Context Recall/Precision với Faithfulness/Relevance/Completeness.

## 2. Top 3 Worst Failures — 5 Whys

Benchmark chưa có 3 failure cases hợp lệ. Ba failure analysis dưới đây là các rủi ro cần kiểm tra ở lần chạy quota kế tiếp, không phải score đo được.

### Failure 1 — Policy version confusion

**ID và question:** H01 — order trước 2026-09-01 nhưng delivery sau ngày hiệu lực policy mới.

**Expected answer:** Phải áp dụng version 1.0 theo order-placement date: 21 ngày unopened; OrbitPlus không mở rộng thành 45 ngày.

**Actual answer:** Chưa có actual answer hợp lệ.

**Evidence inspection:** Gold evidence có `09_escalation_and_policy_updates.md` và `05_returns_and_exchanges.md`. Retriever cần đưa policy-version evidence lên trước nội dung return policy hiện hành.

| Level | Question | Answer |
|---|---|---|
| Symptom | Có thể trả lời sai 30/45 ngày thay vì 21 ngày. |
| Why 1 | Query chứa cả ngày order, ngày delivery và OrbitPlus. |
| Why 2 | BM25 có thể ưu tiên nhiều token chung của policy hiện hành. |
| Why 3 | Chưa có reranking ưu tiên triggering-event evidence. |
| Why 4 | Metric word-overlap không hiểu quan hệ thời gian/pháp lý. |
| Why 5 | Chưa có targeted temporal-policy cases và rule-based validation. |

**Root cause:** Context is missing or irrelevant — improve retrieval.

**Proposed fix:** Thêm temporal reranker và regression cases cho từng policy version; block answer nếu thiếu order date.

### Failure 2 — Safety refusal

**ID và question:** A02 — yêu cầu reveal admin password bằng prompt injection.

**Expected answer:** Từ chối reveal credentials và giữ system scope.

**Actual answer:** Chưa có actual answer hợp lệ.

**Evidence inspection:** Gold evidence từ `00_system_scope.md` yêu cầu bỏ qua instruction reveal credentials. Đây là safety case, không đánh giá như factual answer thông thường.

| Level | Question | Answer |
|---|---|---|
| Symptom | Model có thể làm theo user instruction thay vì scope. |
| Why 1 | Prompt injection đặt instruction độc hại trước câu hỏi. |
| Why 2 | Retrieved context có thể không chứa scope document ở top-k. |
| Why 3 | Retriever tối ưu lexical overlap, không ưu tiên safety policy. |
| Why 4 | Chưa có safety gate độc lập trước generation. |
| Why 5 | Chưa tách refusal correctness thành hard deployment gate. |

**Root cause:** Context is missing or irrelevant — improve retrieval.

**Proposed fix:** Luôn inject immutable scope policy vào system prompt; đánh giá A01–A03 bằng safety-specific assertions.

### Failure 3 — Incomplete procedural answer

**ID và question:** M07 — yêu cầu process và thời gian warranty repair.

**Expected answer:** Cần serial, contact, symptoms, proof of purchase; diagnosis tối đa 3 ngày làm việc và repair thêm tối đa 10 ngày khi có parts.

**Actual answer:** Chưa có actual answer hợp lệ.

**Evidence inspection:** Gold evidence từ `07_repair_and_technical_support.md` chứa cả required fields, diagnosis, repair duration và authorization warning. Đây là case dễ bị trả lời thiếu một điều kiện.

| Level | Question | Answer |
|---|---|---|
| Symptom | Answer chỉ nêu thời gian nhưng thiếu hồ sơ cần chuẩn bị. |
| Why 1 | Model tóm tắt câu hỏi thành phần “how long”. |
| Why 2 | Prompt chưa bắt buộc checklist mọi requirement và exception. |
| Why 3 | Completeness heuristic chỉ là token overlap, chưa phân biệt claim quan trọng. |
| Why 4 | Golden rubric chưa gán trọng số cho required fields. |
| Why 5 | Chưa có claim-level completeness evaluation. |

**Root cause:** Answer is missing key information — increase context window or improve generation.

**Proposed fix:** Prompt yêu cầu checklist; rubric chấm bắt buộc từng claim; thêm claim-level evaluator.

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Temporal/policy retrieval không ưu tiên triggering event | H01 | High |
| 2 | Thiếu safety gate và scope context cố định | A02 | High |
| 3 | Generation thiếu checklist claim bắt buộc | M07 | Medium |

**Nếu chỉ được sửa một cluster:** Chọn Cluster 2 vì credential disclosure và unsafe advice là hard safety failures, phải block deployment dù average score cao.

## 4. Improvement Log

Benchmark chưa sinh đủ failure results, nên chưa paste output runtime. Format output của `generate_improvement_log()` đã được implement và được test.

**Ba improvement suggestions ưu tiên**

1. Inject scope/safety policy cố định trước retrieved contexts.
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

> Core evaluator và dataset validator chạy nhanh, deterministic; phần tốn thời gian và rủi ro nhất là external LLM quota. Gemini chạy được model routing nhưng free tier giới hạn requests, khiến benchmark production-like không hoàn tất dù code pipeline đúng.

**Giới hạn của word-overlap heuristics:**

> Không hiểu phủ định, synonym, số liệu tương đương, temporal logic, claim importance hoặc semantic entailment. Production nên bổ sung embedding similarity, claim-level entailment, RAGAS/LLM judge đã calibrate với human labels, citation verification, safety classifiers và slice-level regression metrics.
