# Báo cáo ngày 19: So sánh Flat RAG và GraphRAG

**Họ tên:** Trần Tuấn Hoàng  
**MSSV:** 2A202602832  
**Ngày:** 05/10/2026

Báo cáo được trình bày theo yêu cầu trong `SUBMISSION.md`. Các số liệu lấy từ `ket_qua_benchmark_kg.txt`. Bản thiết kế mô hình dữ liệu được nộp riêng tại `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Kết quả đo chi phí lập chỉ mục và trả lời câu hỏi được tổng hợp từ `ket_qua_benchmark_kg.txt` như sau.

**Chi phí lập chỉ mục, thực hiện một lần:**

```text
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    106.1
graph       196     91958     4587   0.00926    216.6
```

**Chi phí trả lời, tính trung bình cho mỗi câu hỏi:**

```text
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.80
graph       0.63   1.33     3736       79   0.00060     3.24
```

| Chỉ số | Flat RAG | GraphRAG | Tỉ lệ GraphRAG so với Flat RAG |
| --- | --- | --- | --- |
| Chi phí lập chỉ mục, USD | 0.00112 | 0.00926 | 8.27 lần |
| Thời gian lập chỉ mục, giây | 106.1 | 216.6 | 2.04 lần |
| Chi phí mỗi câu hỏi, USD | 0.00013 | 0.00060 | 4.62 lần |
| Thời gian mỗi câu hỏi, giây | 2.80 | 3.24 | 1.16 lần |
| Số token đầu vào mỗi câu hỏi | 694 | 3736 | 5.38 lần |

**Nguyên nhân làm tăng chi phí**

Chi phí lập chỉ mục của GraphRAG cao gấp khoảng 8.3 lần Flat RAG. Nguyên nhân chính là GraphRAG phải dùng mô hình ngôn ngữ để trích xuất thực thể và quan hệ từ 20 bài báo. Bước này sử dụng thêm khoảng 36 nghìn token đầu vào và 4,587 token đầu ra. Trong khi đó, Flat RAG chỉ cần tạo biểu diễn vector cho các đoạn văn bản, không có bước trích xuất thực thể và quan hệ.

Khi trả lời câu hỏi, GraphRAG có chi phí cao gấp khoảng 4.6 lần và sử dụng số token đầu vào cao gấp khoảng 5.4 lần. Phần tăng thêm chủ yếu đến từ các dữ kiện lấy từ đồ thị qua nhiều bước liên kết. Những dữ kiện này được bổ sung vào ngữ cảnh gửi cho mô hình ngôn ngữ.

Thời gian trả lời tăng từ 2.80 giây lên 3.24 giây, tương đương khoảng 1.16 lần. Mức tăng này khá nhỏ so với mức tăng chi phí và số token đầu vào. Truy vấn Cypher trên Neo4j mất chưa đến 15 mili giây nên chỉ chiếm một phần nhỏ trong tổng thời gian xử lý.

## 2. Kết quả từng câu hỏi (10 điểm)

Trong bảng dưới đây, `recall` là chỉ số thu hồi từ khóa theo bộ đánh giá, còn `judge` là điểm do mô hình ngôn ngữ chấm.

| Câu hỏi | Loại câu hỏi | Flat RAG: recall / judge | GraphRAG: recall / judge | Kết quả | Giải thích |
| --- | --- | --- | --- | --- | --- |
| Q1 | Tra cứu trực tiếp trong luật | 1.00 / 2 | 1.00 / 2 | Hòa | Câu trả lời nằm trong Điều 2 Luật Phòng, chống ma túy năm 2021 nên cả hai hệ thống đều tìm thấy và trả lời đúng. |
| Q2 | Tra cứu trực tiếp trong tin tức | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai hệ thống đều xác định đúng hai bị cáo bị tuyên án tử hình là Trần Thanh Tuấn và Trần Minh Tâm. |
| Q3 | Kết hợp tin tức và luật | 0.00 / 0 | 1.00 / 2 | GraphRAG tốt hơn | Flat RAG không nối được thông tin trong bài báo với Điều 251 Bộ luật Hình sự, còn GraphRAG tìm được liên kết giữa hai nguồn thông qua tội danh. |
| Q4 | Kết hợp tin tức và luật | 0.00 / 0 | 0.00 / 0 | Hòa | Cả hai hệ thống đều không tìm được thông tin cần thiết vì biệt danh “Hoàng Nato” không xuất hiện trong ba đoạn văn bản được truy xuất bằng vector. |
| Q5 | Kết hợp nhiều bước giữa tin tức và luật | 0.60 / 1 | 0.80 / 1 | GraphRAG tốt hơn về recall | GraphRAG thu hồi được nhiều thông tin hơn về các khoản quy định khung hình phạt và mức án tử hình, nhưng điểm chấm câu trả lời của hai hệ thống vẫn bằng nhau. |
| Q6 | Tổng hợp thông tin | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai hệ thống đều liệt kê ba vụ việc liên quan đến MDMA, nhưng recall bằng 0 vì câu trả lời dùng tên vụ việc và địa điểm thay cho các tên riêng mà bộ đánh giá yêu cầu. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Vụ án chưa có liên kết với dữ liệu luật

**Hiện tượng**

Một vụ án trong nguồn tin tức chưa được liên kết với nguồn dữ liệu luật vì không có quan hệ `CHARGED_WITH`.

**Bằng chứng**

Truy vấn Cypher dưới đây tìm các nút `Case` chưa có quan hệ `CHARGED_WITH`:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name AS name, k.doc_id AS doc_id;
```

Kết quả trả về:

| name | doc_id |
| --- | --- |
| Vụ tông cảnh sát giao thông ở An Giang | news-100260926112415229 |

**Nguyên nhân**

Lỗi xuất phát từ bước trích xuất thông tin và phạm vi tội danh mà hệ thống hỗ trợ. Bài báo `news-100260926112415229` đề cập một tài xế xe tải dương tính với ma túy, nhưng tội danh bị khởi tố là “Chống người thi hành công vụ” theo Điều 330 Bộ luật Hình sự. Tội này không thuộc nhóm tội phạm về ma túy trong Chương XX.

Danh sách `known_crimes` hiện chỉ chứa các tội danh về ma túy thuộc Chương XX. Vì vậy, hàm `link_entity` trả về `None` khi xử lý tội danh trên. Vụ án vẫn được đưa vào đồ thị nhưng không có liên kết với tội danh trong nguồn dữ liệu luật.

**Đề xuất sửa**

Với các bài báo chỉ đề cập việc sử dụng ma túy như một tình tiết của vụ việc, cần hướng dẫn mô hình phân biệt thông tin này với tội danh bị khởi tố. Hệ thống có thể gắn nhãn bổ sung hoặc tạo quan hệ `(:Case)-[:RELATED_TO]->(:Substance)` để lưu mối liên hệ với chất ma túy.

Ngoài ra, có thể mở rộng mô hình dữ liệu để hỗ trợ những tội danh liên quan nằm ngoài Chương XX. Cách này giúp giữ lại thông tin của vụ án mà không phải gán một tội danh về ma túy khi bài báo không nêu tội danh đó.

### Lỗi E3: Trùng thực thể

**Hiện tượng**

Cùng một chất ma túy được lưu thành nhiều nút khác nhau trong đồ thị do cách viết tên không thống nhất.

**Bằng chứng**

Truy vấn danh sách các nút `Substance`:

```cypher
MATCH (s:Substance) RETURN s.name AS name ORDER BY toLower(s.name);
```

Kết quả:

```text
['Amphetamine', 'chất ma túy', 'Cocaine', 'côca', 'cần sa', 'etomidate', 'Heroine', 'Ketamine', 'ketamine', 'ma túy', 'ma túy tổng hợp', 'MDMA', 'Methamphetamine', 'methamphetamine', 'thuốc lắc', 'thuốc phiện', 'XLR-11']
```

**Nguyên nhân**

Mô hình dữ liệu chưa có quy tắc thống nhất để chuẩn hóa tên thực thể trước khi tạo nút.

Neo4j phân biệt chữ hoa và chữ thường. Vì vậy, `Ketamine` trong dữ liệu luật và `ketamine` trong tin tức được lưu thành hai nút riêng. Trường hợp `Methamphetamine` và `methamphetamine` cũng tương tự.

Bên cạnh đó, tin tức có thể dùng tên gọi thông dụng như “thuốc lắc”, còn dữ liệu luật dùng tên chất như `MDMA`. Khi chưa có bảng đối chiếu tên gọi, hệ thống không nhận diện được các cách gọi tương ứng và có thể tạo thêm nút.

**Đề xuất sửa**

- Chuẩn hóa tên bằng `trim(toLower(s.name))` trước khi dùng `MERGE (sub:Substance {name: lower_name})`. Cách này xử lý các trường hợp khác nhau về chữ hoa, chữ thường hoặc khoảng trắng.
- Xây dựng bảng đối chiếu tên gọi cho chất ma túy, tương tự cách chuẩn hóa tội danh. Những cách gọi như “thuốc lắc” và “hàng đá” cần được đối chiếu với tên chất tương ứng dựa trên thông tin trong nguồn trước khi ghi vào Neo4j.

### Lỗi E4: Chỉ số đánh giá chưa phản ánh đúng câu trả lời

**Hiện tượng**

Ở câu Q6, cả Flat RAG và GraphRAG đều liệt kê các vụ việc liên quan đến MDMA và được chấm `judge = 1`. Tuy nhiên, chỉ số thu hồi từ khóa của cả hai hệ thống đều bằng `recall = 0.00`.

**Bằng chứng**

Trong `ket_qua_benchmark_kg.txt`, kết quả của GraphRAG ở câu Q6 là `recall = 0.00`, `judge = 1`, với thời gian trả lời 4.19 giây. Câu trả lời liệt kê ba vụ việc:

1. Vụ tổ chức sử dụng ma túy tại Sầm Sơn, liên quan đến 0,686g MDMA.
2. Vụ góp tiền mua ma túy tại Hà Nội, liên quan đến 5 viên MDMA.
3. Vụ vận chuyển ma túy từ Đức về Việt Nam, liên quan đến tổng khối lượng hơn 9,6kg MDMA.

Trong khi đó, `data/benchmark_kg.json` quy định các cụm từ bắt buộc như sau:

```json
{
  "id": "Q6",
  "must_include": ["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]
}
```

**Nguyên nhân**

Hàm tính `recall` kiểm tra xem câu trả lời có chứa đúng ba cụm từ trong `must_include` hay không. Các cụm từ này là tên người hoặc đơn vị, trong khi câu hỏi chỉ yêu cầu: “Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?”.

Mô hình trả lời bằng cách mô tả vụ việc và địa điểm, nhưng không nêu các tên riêng mà bộ đánh giá yêu cầu. Vì vậy, câu trả lời vẫn chứa thông tin về các vụ việc nhưng không được ghi nhận ở chỉ số thu hồi từ khóa. Trường hợp này cho thấy `recall` đang phụ thuộc vào cách diễn đạt và chưa phản ánh đầy đủ nội dung câu trả lời.

**Đề xuất sửa**

- Cho phép nhiều cách gọi tương đương cho mỗi vụ việc. Chẳng hạn, vụ án liên quan đến Cái Quang Huy có thể được nhận diện qua tên người, cụm “Đức về Việt Nam” hoặc “Nội Bài”, nếu thông tin trong câu trả lời đủ để xác định đúng vụ án.
- Bổ sung cách đánh giá dựa trên ý nghĩa của câu trả lời, chẳng hạn dùng mô hình ngôn ngữ để kiểm tra các vụ việc được nêu có khớp với đáp án hay không.
- Thêm hướng dẫn cho mô hình: “Khi liệt kê vụ việc, nêu rõ tên bị cáo chính và cơ quan hoặc đơn vị liên quan nếu nguồn dữ liệu có thông tin này”.

## 4. Kết luận (5 điểm)

**Khi nào Flat RAG là đủ?**

Flat RAG phù hợp với các câu hỏi có thể trả lời trực tiếp từ một điều luật hoặc một bài báo. Trong thí nghiệm, Q1 hỏi về định nghĩa tiền chất và Q2 hỏi về các bị cáo bị tuyên án tử hình. Cả hai hệ thống đều đạt `recall = 1.00` và `judge = 2` ở hai câu này.

Tính trên toàn bộ phép đo, chi phí lập chỉ mục của Flat RAG chỉ bằng khoảng 1/8.3 so với GraphRAG, còn chi phí trung bình mỗi câu hỏi bằng khoảng 1/4.6. Thời gian phản hồi cũng thấp hơn. Vì vậy, khi thông tin cần tìm nằm trong một đoạn văn bản và không đòi hỏi liên kết nhiều nguồn, Flat RAG là lựa chọn tiết kiệm và đáp ứng tốt yêu cầu.

**Khi nào nên dùng đồ thị tri thức?**

Đồ thị tri thức hữu ích khi câu hỏi cần kết nối thông tin giữa nhiều nguồn hoặc đi qua nhiều bước liên kết. Q3 là ví dụ rõ nhất trong thí nghiệm: Flat RAG đạt `recall = 0.00` và `judge = 0`, còn GraphRAG đạt `recall = 1.00` và `judge = 2`.

Ở câu này, bài báo không nêu đầy đủ khung hình phạt, còn dữ liệu luật không chứa tên bị cáo. GraphRAG kết nối hai nguồn thông qua nút tội danh `Crime`, từ đó lấy được thông tin cần thiết để trả lời.

Kết quả cho thấy chi phí tăng thêm của GraphRAG có thể hợp lý với các bài toán cần truy vết căn cứ pháp lý, đối chiếu hồ sơ hoặc phân tích quan hệ giữa các đối tượng. Tuy nhiên, lợi ích còn phụ thuộc vào từng loại câu hỏi và chất lượng đồ thị. Q4 và Q6 cho thấy việc bổ sung đồ thị chưa tự động giải quyết được mọi hạn chế của quá trình truy xuất và đánh giá.

## 5. Tự kiểm (5 điểm)

Kết quả chạy kiểm thử và kiểm tra hệ thống:

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.09s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00077.
```

Các ảnh minh chứng trên Neo4j được lưu tại:

- `report/img/kg_count.png`
- `report/img/kg_cross_kb.png`
- `report/img/kg_my_case.png`

Ảnh `kg_my_case.png` minh họa trường hợp **Cái Quang Huy** trong vụ vận chuyển trái phép hơn 9,6kg MDMA từ Đức về Việt Nam qua sân bay Nội Bài. Vụ án được liên kết với Điều 250 Bộ luật Hình sự.

## Vấn đề gặp phải (không tính điểm)

Không có vấn đề cản trở việc hoàn thành bài. Các bước thiết lập Neo4j bằng Docker, chuẩn hóa thực thể, xây dựng đồ thị, truy vấn Cypher qua nhiều bước liên kết và đánh giá hai hệ thống đều đã thực hiện được. Các kiểm tra tự động cũng đã chạy thành công.

Những hạn chế về liên kết vụ án, trùng thực thể và cách tính chỉ số đánh giá đã được phân tích ở phần 3, kèm theo hướng cải thiện.