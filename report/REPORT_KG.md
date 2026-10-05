# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Đức Anh  **MSSV:** 2A202602888  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     91.1
graph       196     91958     4717   0.00933    186.6

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     2.45
graph       0.83   1.83     5926       77   0.00093     3.98
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00933 | ×8.33 |
| Indexing giây | 91.1 | 186.6 | ×2.05 |
| Mỗi câu: USD | $0.00013 | $0.00093 | ×7.15 |
| Mỗi câu: giây | 2.45 | 3.98 | ×1.62 |
| Mỗi câu: in_tok | 694 | 5926 | ×8.54 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> - **Lúc Indexing:** Chi phí và thời gian tăng thêm chủ yếu do GraphRAG phải gọi thêm 20 lượt LLM chat (JSON mode) để đọc hiểu và trích xuất thực thể/quan hệ từ 20 bài báo tin tức (tốn thêm 35,886 input tokens và 4,717 output tokens), trong khi Flat RAG chỉ tính toán embedding một chiều cho các chunks.
> - **Lúc Querying:** Chi phí mỗi câu hỏi của GraphRAG cao hơn gấp ~7.15 lần là do prompt đầu vào được mở rộng thêm danh sách dữ kiện (facts) phong phú từ đồ thị Neo4j (trung bình ~5,232 tokens ngữ cảnh bổ sung từ các node kề, tóm tắt vụ việc và các điều khoản liên quan), đẩy số lượng input tokens trung bình từ 694 lên 5,926 tokens.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều trích xuất chính xác định nghĩa tiền chất từ Điều 2 Luật Phòng chống ma túy trong vector store. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai pipeline đều lấy đúng 2 bị cáo nhận án tử hình (Trần Thanh Tuấn, Trần Minh Tâm) từ văn bản bài báo vụ 36kg. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG trả lời "Không đủ thông tin" do thiếu Điều 251 BLHS, trong khi GraphRAG đi từ bị cáo qua tội danh để lấy trọn vẹn Điều luật và mức phạt khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat RAG thiếu thông tin chế tài Điều 255 BLHS, còn GraphRAG kết nối được hành vi của Hoàng Nato sang Điều 255 và khung phạt tối đa 20 năm/chung thân. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat RAG nhầm thành "khoản b)" do thiếu ngữ cảnh định lượng, trong khi GraphRAG liên kết đúng khối lượng >9.6kg MDMA vào khoản 4 Điều 250. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai đều tìm ra đúng 3 vụ án có MDMA nhưng mô tả theo sự kiện nên không khớp đúng các tên riêng cụ thể được hardcode trong từ khóa chấm recall. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật khi lọc khoản quy định

- **Hiện tượng:** Khi truy vấn các câu hỏi yêu cầu khung hình phạt tối đa (như câu Q4 đối với hành vi của Dương Minh Tuấn "Hoàng Nato"), nếu hệ thống lọc khoản theo cơ chế: chỉ giữ khoản 1 và các khoản nhắc đến chất ma túy (`MENTIONS Substance`), mô hình có nguy cơ bị thiếu khung hình phạt tăng nặng cao nhất của Điều luật.
- **Bằng chứng:**
Truy vấn các khoản của Điều 255 BLHS và liên kết chất:
```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
RETURN cl.number AS clause, cl.penalty AS penalty, s.name AS substance;
```
Kết quả thực tế trong Neo4j:
```
clause | penalty                                            | substance
1      | "phạt tù từ 02 năm đến 07 năm"                     | null
2      | "phạt tù từ 07 năm đến 15 năm"                     | null
3      | "phạt tù từ 15 năm đến 20 năm"                     | null
4      | "phạt tù 20 năm hoặc tù chung thân"                | null
5      | "phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng"| null
```
- **Nguyên nhân:** Nằm ở quy tắc lọc khoản tại bước Cypher retrieval (KG-3). Trong BLHS, một số tội danh (như Điều 255 "Tội tổ chức sử dụng trái phép chất ma túy") không phân chia khung phạt theo khối lượng chất cụ thể mà phân chia theo tình tiết (đối với trẻ em, gây chết người...). Vì vậy, không có quan hệ `MENTIONS` nào được tạo giữa các `Clause` của Điều 255 với `Substance`. Nếu áp dụng bộ lọc chỉ giữ `khoản 1` và `khoản có substance_match`, các khoản 2, 3, 4 (chứa mức phạt tối đa 20 năm hoặc chung thân) sẽ bị lược bỏ hoàn toàn khỏi facts.
- **Đề xuất sửa:** Trong logic `Neo4jGraph.context`, kiểm tra xem Điều luật đích có chứa bất kỳ khoản nào có liên kết `MENTIONS` hay không. Nếu Điều luật hoàn toàn không có `substance_match` (như Điều 255), hệ thống cần giữ lại toàn bộ các khoản định khung của Điều luật đó để cung cấp đầy đủ phổ hình phạt cho LLM. Đánh đổi: số lượng facts tăng nhẹ thêm 3-4 dòng, nhưng đảm bảo 100% độ chính xác cho các câu hỏi về mức phạt cao nhất.

---

### Lỗi E4: Phép đo sai (Mâu thuẫn giữa Keyword Recall và LLM-as-judge)

- **Hiện tượng:** Tại câu hỏi tổng hợp Q6 (*"Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"*), cả hai pipeline đều bị chấm `recall = 0.00` nhưng điểm thẩm định `judge = 1` (được LLM công nhận trả lời đúng các vụ việc thực tế).
- **Bằng chứng:**
Câu trả lời thực tế của GraphRAG trong `ket_qua_benchmark_kg.txt`:
```
Các vụ việc trong tin tức có liên quan đến ma túy MDMA bao gồm:
1. Vụ vận chuyển ma túy từ Đức về Việt Nam: Trong vụ này, có tổng khối lượng hơn 9,6kg MDMA được vận chuyển.
2. Vụ góp tiền mua ma túy tại Hà Nội: Vụ này liên quan đến 5 viên MDMA mà các bị cáo đã góp tiền để mua.
3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Vụ này liên quan đến việc thu giữ 0,686g ma túy MDMA trong quá trình khám xét.
Tất cả các vụ việc này đều có liên quan đến ma túy MDMA.
```
So sánh với yêu cầu từ khóa trong `data/benchmark_kg.json`:
```json
"must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
```
- **Nguyên nhân:** Lỗi nằm ở chính thiết kế của bộ công cụ đo lường (`bench_kg.py`). Hàm `keyword_recall` sử dụng so khớp chuỗi ký tự cứng (`k.lower() in answer.lower()`). Khi trả lời câu hỏi tổng hợp, LLM có xu hướng diễn đạt tự nhiên theo tên gọi hoặc bối cảnh sự việc ("Vụ vận chuyển ma túy từ Đức về Việt Nam" chính là vụ Cái Quang Huy; "Vụ góp tiền mua ma túy tại Hà Nội" chính là vụ Lê Minh Thành; "Vụ tổ chức sử dụng ma túy tại Sầm Sơn" chính là vụ án liên quan Viện Pháp y tâm thần). Câu trả lời hoàn toàn chính xác về mặt nội dung nghiệp vụ nhưng bị đánh điểm 0% recall do không chứa đúng các danh từ riêng được ấn định trước.
- **Đề xuất sửa:** 
  1. Cải tiến `benchmark_kg.json`: Mở rộng danh sách từ khóa hợp lệ thành các nhóm từ khóa lựa chọn (ví dụ: `["Cái Quang Huy" OR "Đức về Việt Nam"], ["Lê Minh Thành" OR "Hà Nội"], ["Pháp y tâm thần" OR "Sầm Sơn"]`).
  2. Bổ sung phương pháp đánh giá Semantic Recall / Entity F1 thay vì chỉ dùng phép so khớp chuỗi thuần túy.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> - **Khi nào bắt buộc nên dùng Knowledge Graph (GraphRAG):** Khi nghiệp vụ đòi hỏi suy luận xuyên tài liệu (Cross-document / Cross-KB Reasoning), kết nối các thực thể xuất hiện rời rạc ở nhiều nguồn dữ liệu khác nhau (như từ hồ sơ vụ án tin tức liên kết sang điều khoản xử phạt trong bộ luật hình sự ở Q3, Q4, Q5). Số liệu thực tế chứng minh rõ: Flat RAG hoàn toàn thất bại ở các câu hỏi cross-kb với **recall = 0.00 và judge = 0** (do không có đoạn trích đơn lẻ nào chứa đồng thời cả tình tiết vụ án lẫn điều luật), trong khi GraphRAG đạt **recall = 1.00 và judge = 2**.
> - **Khi nào Flat RAG là hoàn toàn đủ và tối ưu:** Khi câu hỏi thuộc dạng cục bộ (single-hop), câu trả lời nằm trọn vẹn trong một phân đoạn văn bản độc lập (như tra cứu định nghĩa tiền chất ở Q1, danh sách bị cáo tử hình ở Q2). Ở các trường hợp này, Flat RAG đạt kết quả hoàn hảo (recall 1.00, judge 2) trong khi chi phí rẻ hơn **gấp 7.15 lần** ($0.00013 so với $0.00093 mỗi câu) và độ trễ nhanh hơn **1.62 lần** (2.45s so với 3.98s) so với GraphRAG. Do đó, việc ứng dụng Knowledge Graph cần được cân nhắc dựa trên bản chất câu hỏi để tối ưu hóa bài toán chi phí và hiệu năng.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.08s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00065. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Dương Minh Tuấn (Hoàng Nato).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Không có lỗi kỹ thuật nào tồn đọng. Quá trình triển khai giải quyết triệt để vấn đề mã hóa ký tự UTF-8 trên PowerShell Windows (`$env:PYTHONIOENCODING="utf-8"`), cấu hình linh hoạt chuyển đổi provider sang OpenRouter khi tài khoản OpenAI hết hạn mức (`insufficient_quota`), và hoàn thành xuất sắc 100% các tiêu chí kiểm thử hợp đồng và benchmark.
