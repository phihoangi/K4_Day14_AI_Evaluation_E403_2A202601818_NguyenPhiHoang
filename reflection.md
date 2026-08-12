# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 15.0%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.803 | 0.385 | 1.000 | Retriever hoạt động tốt, thường xuyên lấy đúng context chứa câu trả lời. |
| Context Precision | 0.931 | 0.333 | 1.000 | Retriever xếp hạng context rất tốt, chunk đúng thường nằm trong top 2. |
| Faithfulness | 0.475 | 0.107 | 0.875 | Model sinh câu trả lời bịa đặt hoặc thiếu từ khoá từ context. |
| Relevance | 0.549 | 0.105 | 1.000 | Câu trả lời đôi khi chưa tập trung vào câu hỏi. |
| Completeness | 0.375 | 0.133 | 0.625 | MockGenerator chỉ ghép câu hỏi và context nên thiếu sót thông tin quan trọng. |
| Overall Score | 0.501 | 0.141 | 0.723 | Nhìn chung RAG pipeline hỏng ở generation. |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Recall, Context Precision
- Metrics/cases ở mức Needs Work (0.6–0.8): -
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness, Relevance, Completeness

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 20.0% |
| irrelevant | 4 | 20.0% |
| incomplete | 2 | 10.0% |
| off_topic | 7 | 35.0% |
| refusal | 0 | 0.0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:* Vấn đề nằm 100% ở generation. Context Recall (0.803) và Context Precision (0.931) đều rất cao, nhưng Faithfulness (0.475) và Completeness (0.375) lại quá thấp do MockGenerator không hiểu câu hỏi mà chỉ ghép nối chuỗi một cách máy móc, dẫn đến off_topic và hallucination.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:** H01 - I ordered an unopened NovaBook 14 on August 20, 2026. Can I return it on September 22?

**Expected answer:** No, you cannot return it on September 22. The 30-day return window for unopened standard products ends on September 19.

**Actual answer:** I ordered an unopened NovaBook 14 on August 20, 2026. Can I return it on September 22? You cannot return it on September 22 because the 30-day return window ends on September 19.

**Scores:** Context Recall: 0.909 | Context Precision: 0.950 | Faithfulness: 0.182 | Relevance: 0.105 | Completeness: 0.136 | Overall: 0.141

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> *Câu trả lời:* Lấy đúng chunk về chính sách đổi trả 30 ngày.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Relevance và Faithfulness rất thấp. |
| Why 1 | Tại sao symptom xảy ra? | Answer có lặp lại toàn bộ câu hỏi (MockGenerator behavior) |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Word-overlap bị loãng do quá nhiều từ không liên quan. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Không có filter / rewriter cho output. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Đang dùng MockGenerator. |
| Why 5 | Root cause có thể hành động được là gì? | Thay thế MockGenerator bằng LLM thực thụ. |

**Root cause từ `find_root_cause()`:**

> *Paste output:* The generation logic is hallucinating or producing irrelevant text.

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> *Câu trả lời:* Đồng ý. Actual answer lặp lại toàn bộ câu hỏi khiến Relevance score cực thấp vì overlap ratio bị pha loãng.

**Proposed fix cụ thể:** Thay MockGenerator bằng OpenAI/Gemini xịn.

### Failure 2

**ID và question:** M03 - If my account is compromised...

**Expected answer:** If your account is compromised, we will freeze it. OrbitTech is not liable for unauthorized purchases.

**Actual answer:** If my account is compromised... If your account is compromised, we will freeze it. OrbitTech is not liable for unauthorized purchases.

**Scores:** Context Recall: 0.533 | Context Precision: 1.000 | Faithfulness: 0.250 | Relevance: 0.267 | Completeness: 0.200 | Overall: 0.239

**Evidence inspection:**

> *Câu trả lời:* Lấy đủ context liên quan đến account security.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Điểm Overall rất thấp. |
| Why 1 | Tại sao symptom xảy ra? | MockGenerator lặp câu hỏi. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Lỗi hardcode. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Chưa thay model. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | RAGAS Word overlap đánh giá khắt khe. |
| Why 5 | Root cause có thể hành động được là gì? | Thay Generator model. |

**Root cause và proposed fix:**

> *Câu trả lời:* Generator kém. Fix: Thay Generator model.

### Failure 3

**ID và question:** H02 - I bought a phone on September 5...

**Expected answer:** You cannot return it on November 2. The 30-day window ended in October.

**Actual answer:** I bought a phone on September 5... You cannot return it on November 2. The 30-day window ended in October.

**Scores:** Context Recall: 0.933 | Context Precision: 1.000 | Faithfulness: 0.500 | Relevance: 0.105 | Completeness: 0.133 | Overall: 0.246

**Evidence inspection:**

> *Câu trả lời:* Context lấy đúng về thời hạn 30 ngày.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Relevance thấp (0.105). |
| Why 1 | Tại sao symptom xảy ra? | Câu hỏi quá dài được nối vào câu trả lời, pha loãng keyword. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Do logic MockGenerator. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | |
| Why 5 | Root cause có thể hành động được là gì? | |

**Root cause và proposed fix:**

> *Câu trả lời:* Fix: dùng proper LLM thay cho MockGenerator.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Lỗi generation (MockGenerator trả lời lặp từ, cắt ghép sai) | E01, E02, M02, M03, H01, H02 | High |
| 2 | Thiếu guardrails (Không có cơ chế filter các câu irrelevant/hallucinated trước khi trả về user) | A01, A03 | Medium |
| 3 | Retrievel chưa tối ưu cho các câu phức tạp | M05, M06, H03 | Low |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:* Chọn Cluster 1 (Lỗi Generation). Vì context đã được retrieve rất tốt (Recall/Precision cao), root cause lớn nhất hiện tại đang nằm ở khâu tổng hợp câu trả lời từ context. Nếu thay model tốt hơn, hầu hết các lỗi này sẽ biến mất.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Multiple issues detected | Increase chunk size in RAG pipeline | Open |
| F002 | off_topic | Answer is missing key information | Add few-shot examples | Open |
| F003 | off_topic | Multiple issues detected | Implement hallucination checker | Open |
| F004 | off_topic | Answer is missing key information | Refine intent classification | Open |
| F006 | hallucination | Multiple issues detected | Add few-shot examples | Open |
| F007 | off_topic | Context is missing or irrelevant | Implement hallucination checker | Open |
```

**Ba improvement suggestions ưu tiên**

1. Thay thế MockGenerator bằng LLM thực thụ (OpenAI/Gemini).
2. Thêm Few-shot examples vào prompt.
3. Tích hợp Hallucination checker (Self-reflection step).

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Thay thế MockGenerator bằng LLM xịn | Completeness, Relevance | Chạy lại benchmark (python evaluate_answers.py) và xem average score tăng không. |
| Thêm Few-shot examples | Faithfulness | Kiểm tra tỷ lệ lỗi Hallucination giảm bao nhiêu %. |
| Self-reflection step | Overall pass rate | Đo lượng bad responses bị chặn lại trước khi trả về. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:* Chạy `run_regression()` trong CI/CD pipeline mỗi khi có thay đổi về Prompt, Retriever config (chunk size, K), hoặc đổi Model. Nó chạy tự động trên tập Golden Dataset trước khi merge code lên nhánh chính.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:* Phù hợp. Do các metrics heuristic (Word-overlap) thường có dao động (noise) khi thay đổi wording, nên cho phép biên độ giảm nhỏ (0.05). Nếu dùng LLM-as-a-judge, threshold này cũng dung sai cho độ ngẫu nhiên của LLM.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:* 
> - **Block deployment:** Faithfulness giảm sâu hoặc xuất hiện nhiều Hallucination (rủi ro cao cho Customer Support).
> - **Alert:** Completeness hoặc Context Recall giảm nhẹ (có thể là lỗi nhỏ, user có thể hỏi lại).

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Offline Benchmark (Golden Dataset)] → [Regression Analysis (run_regression)] → [Manual Human Review (Edge cases)] → Deploy
```

> *Giải thích:* Trước tiên chạy toàn bộ test trên tập cố định (Offline benchmark), so sánh kết quả với baseline trước đó (Regression). Nếu pass, đưa cho QA duyệt các case nhạy cảm (Human review) rồi mới Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thay thế generator (Sử dụng OpenAI thay vì MockGenerator) | Completeness, Relevance | Khắc phục 100% lỗi do word-matching máy móc. |
| 2 | Refine system prompt để focus trả lời trực tiếp | Relevance | Cải thiện độ liên quan của các câu hỏi dài. |
| 3 | Thêm Re-ranker vào pipeline | Context Precision | Tăng Context Precision lên mức tuyệt đối. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:* 
> 1. Case có reasoning phức tạp kết hợp nhiều ngày tháng (vd như H01).
> 2. Các case prompt injection (để test system prompt guardrails).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:* Context Recall và Context Precision cao đến không ngờ (cả hai đều ~0.9). Tôi từng nghĩ BM25 retriever đơn giản sẽ gặp khó khăn, nhưng nó hoạt động cực kì tốt, chứng tỏ lỗi nằm chủ yếu ở khâu Generation.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:* Word-overlap heuristics rất cứng nhắc, nó phạt các câu trả lời đồng nghĩa (synonyms) hoặc có lối hành văn khác dù ý nghĩa đúng. Nếu đưa vào production, tôi sẽ thay bằng LLM-as-a-judge (như `LLMJudge` trong Task 3) để chấm điểm bằng semantic similarity thay vì exact lexical match.
