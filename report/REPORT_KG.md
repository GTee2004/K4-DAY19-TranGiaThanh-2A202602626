# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Gia Thành  **MSSV:** 2A202602626  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    104.4
graph       196     91958     4547   0.00923    194.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.04
graph       0.89   1.83     6490       85   0.00102     3.50
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00923 | ×8.24 |
| Indexing giây | 104.4 | 194.3 | ×1.86 |
| Mỗi câu: USD | 0.00013 | 0.00102 | ×7.85 |
| Mỗi câu: giây | 2.04 | 3.50 | ×1.72 |
| Mỗi câu: in_tok | 694 | 6490 | ×9.35 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Khi indexing, GraphRAG dùng cùng 176 lượt embedding như Flat RAG nhưng gọi thêm LLM để trích xuất thực thể và quan hệ từ 20 bài báo, nên tổng số lượt gọi tăng từ 176 lên 196 và chi phí tăng khoảng 8,24 lần. Khi truy vấn, GraphRAG đưa cả các chunk vector lẫn dữ kiện multi-hop từ graph vào prompt, khiến input token trung bình tăng từ 694 lên 6490 và chi phí mỗi câu tăng khoảng 7,85 lần. Đổi lại, recall trung bình tăng từ 0,43 lên 0,89 và judge tăng từ 1,00 lên 1,83.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Một đoạn luật được vector search lấy đúng đã đủ trả lời định nghĩa “tiền chất”; GraphRAG chỉ bổ sung cách dẫn Điều 2, khoản 4. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Câu trả lời nằm trọn trong một bài báo nên cả hai pipeline đều xác định đúng Trần Thanh Tuấn và Trần Minh Tâm. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG thiếu dữ kiện, còn GraphRAG đi từ Lê Minh Thành qua vụ án và tội danh tới Điều 251, khoản 1 với khung 02–07 năm. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | GraphRAG nối biệt danh Hoàng Nato với vụ án rồi tới Điều 255, khoản 4 để tìm được mức cao nhất là 20 năm hoặc tù chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | GraphRAG kết hợp người, vụ án, MDMA, khối lượng và Điều 250 để chọn đúng khoản 4, trong khi Flat RAG không nêu đúng Điều và khoản. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph (một phần) | GraphRAG tìm được ba nhóm vụ MDMA và nêu đúng Viện Pháp y tâm thần nhưng thiếu hai tên người bắt buộc, đồng thời thêm một vụ không được chứng minh có MDMA. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E4: Phép đo sai — `recall` và `judge` phản ánh khác nhau

- **Hiện tượng:** Ở Q6, Flat RAG có `recall=0.00` nhưng vẫn được LLM judge chấm `1` (đúng một phần). Câu trả lời nhận ra ba vụ việc nhưng gọi người bằng tên rút gọn, nên phép đo recall theo chuỗi không ghi nhận được.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt` có dòng `Q6 [aggregation] flat recall=0.00 judge=1`. Câu trả lời nguyên văn là:

```
1. Vụ việc của Đức liên quan đến số viên nén hình tam giác màu hồng - xám được xác định là MDMA.
2. Vụ việc của Thành liên quan đến 5 viên nén màu trắng được xác định là ma túy MDMA.
3. Vụ việc của Đông liên quan đến 0,686g ma túy MDMA được thu giữ trong buồng chữa bệnh.
```

- **Nguyên nhân:** Lỗi nằm ở bước đánh giá. `keyword_recall` chỉ kiểm tra sự xuất hiện nguyên văn của `Cái Quang Huy`, `Lê Minh Thành` và `Pháp y tâm thần`; các từ “Đức”, “Thành”, “Đông” không khớp dù câu trả lời có một phần nội dung liên quan. Judge đánh giá ngữ nghĩa nên vẫn cho 1 điểm, nhưng judge cũng có thể sai và không thay thế việc đọc câu trả lời.
- **Đề xuất sửa:** Yêu cầu mô hình nêu đầy đủ tên người/tổ chức trong câu tổng hợp; đồng thời có thể bổ sung phép đo dựa trên alias hoặc entity matching. Khi báo cáo vẫn phải dùng cả recall, judge và kiểm tra thủ công câu trả lời nguyên văn.

### Lỗi E5: LLM lệch với dữ kiện trong graph

- **Hiện tượng:** Ở Q6, GraphRAG đưa “Vụ mua bán hơn 36kg ma túy tại TP.HCM” vào danh sách vụ liên quan đến MDMA, nhưng chính câu trả lời thừa nhận vụ này không nêu cụ thể MDMA. Đây là một kết luận không được phần mô tả đi kèm chứng minh.
- **Bằng chứng:** `ket_qua_benchmark_kg.txt` ghi `Q6 [aggregation] graph recall=0.33 judge=1` và có đoạn nguyên văn:

```
4. Vụ mua bán hơn 36kg ma túy tại TP.HCM - Mặc dù không nêu cụ thể về MDMA,
nhưng vụ việc này cũng liên quan đến ma túy.
```

Trong cùng câu trả lời, ba vụ có bằng chứng MDMA là vụ tại Viện Pháp y tâm thần Trung ương, vụ góp tiền mua ma túy tại Hà Nội và vụ vận chuyển ma túy từ Đức về Việt Nam. Tuy vậy, câu trả lời không nêu hai tên bắt buộc `Lê Minh Thành` và `Cái Quang Huy`.
- **Nguyên nhân:** Lỗi nằm ở bước retrieval và sinh câu trả lời. `seed_facts` có thể đưa vào các `Case` xuất hiện do vector search dù chúng không có quan hệ `INVOLVES` trực tiếp với node `MDMA`; LLM sau đó tổng hợp cả vụ thừa. Mặt khác, ontology lưu người ở node `Person`, còn tên `Case` và `summary` do LLM đặt có thể không chứa tên người, nên câu tổng hợp dễ bỏ mất tên riêng.
- **Đề xuất sửa:** Với câu hỏi tổng hợp theo chất, chỉ lấy các vụ thỏa `(:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})`, rồi truy vấn thêm `(:Person)-[:INVOLVED_IN]->(:Case)` để đưa đầy đủ tên người vào fact. Prompt cần yêu cầu không liệt kê vụ việc nếu graph không có cạnh trực tiếp tới chất được hỏi.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> Flat RAG phù hợp với câu hỏi single-hop khi câu trả lời nằm trong một đoạn văn: Q1 và Q2 đều đạt `recall=1.00`, `judge=2`, trong khi chi phí mỗi câu chỉ là 0.00013 USD và độ trễ 2,04 giây. GraphRAG phù hợp với câu hỏi cần nối nhiều nguồn hoặc tổng hợp quan hệ: ở Q3–Q5, recall lần lượt đạt 1.00 thay vì 0.00, 0.00 và 0.60 của Flat RAG; recall trung bình toàn bộ tăng từ 0,43 lên 0,89. Đánh đổi là GraphRAG dùng 6490 input token và 0.00102 USD mỗi câu, cao hơn Flat RAG khoảng 9,35 lần token và 7,85 lần chi phí. Vì vậy nên dùng Flat RAG cho tra cứu trực tiếp, còn GraphRAG cho câu hỏi cần đi từ người/vụ án sang tội danh, Điều luật, khoản và khung hình phạt; với câu tổng hợp vẫn cần kiểm soát false positive như lỗi Q6.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q

................................................                                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check

[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Q6 chưa đạt đầy đủ: GraphRAG có `recall=0.33`, `judge=1`, thiếu tên `Lê Minh Thành` và `Cái Quang Huy`, đồng thời đưa thêm vụ mua bán hơn 36kg ma túy dù không có bằng chứng MDMA trong phần trả lời. Các câu cross-KB Q3–Q5 đã đạt `recall=1.00`, `judge=2`; lỗi còn lại tập trung ở truy vấn tổng hợp theo `Substance` và cách LLM giữ tên thực thể trong câu trả lời.
