# Architecture brief — Multimodal RAG có thể tái lập cho 10 triệu hồ sơ pháp lý

## 1. Problem statement

Một hãng luật Việt Nam muốn triển khai RAG cho **10 triệu PDF** gồm văn bản có text, scan, bảng và phụ lục. Khoảng 30 tỷ tokens sau OCR được chunk ở kích thước mục tiêu 500 tokens, tương đương xấp xỉ 60 triệu chunk. Mỗi embedding 1.024 chiều sẽ được tái tạo ít nhất hai lần khi đổi model. Online retrieval phải có p95 dưới 200 ms; kết quả trích dẫn phải tái lập được sau năm năm theo đúng bản tài liệu, chunk, model embedding và policy đã dùng. Một yêu cầu xóa/thu hồi tài liệu phải ngăn retrieval mới trong 15 phút. Đây không chỉ là bài toán ANN: nếu index lệch với kho tài liệu hoặc một bản án bị sửa, câu trả lời có thể trích dẫn sai nguồn mà không thể audit.

Các số liệu chi phí dưới đây là ước lượng thiết kế cho một region AWS, chưa gồm egress, thuế và chiết khấu hợp đồng. Giá thực tế phải được chốt lại trong AWS Pricing Calculator trước khi mua capacity.

## 2. Kiến trúc đề xuất

```text
 PDF/scan + manifest quyền sử dụng
              |
              v
 [Bronze: object store + immutable ingest manifest]
   sha256, source_uri, received_at, owner/license, raw PDF pointer
              |  malware/OCR/layout extraction; quarantine lỗi
              v
 [Silver Iceberg: document_version, page, chunk, provenance]
   document_id, version_id, chunk_id, text, bbox, language,
   source_hash, legal_hold, subject_id, deleted_at
              |  snapshot + embedding model/version + CDF/outbox
       +------+---------------------+
       |                            |
       v                            v
 [Gold: citation/audit]      [Derived vector index]
 citation manifest,          HNSW shards keyed by
 query policy, metrics       (chunk_id, embedding_version)
       |                            |
       +---------- query -----------+
                      |
                      v
               RAG API / legal reviewer
```

Bronze là bằng chứng bất biến của ingest, không phải nơi analyst đọc nội dung. Silver là system of record: mỗi sửa đổi tạo `document_version` mới, không overwrite im lặng; chunks kế thừa `source_hash`, license và subject ID từ ingest. Gold lưu citation manifest gồm `Iceberg snapshot_id`, `document_version`, `chunk_id`, `embedding_model`, `embedding_version`, index-build id và policy version. Vì vậy một câu trả lời năm 2031 có thể tái chạy trên snapshot đã pin, thay vì “query latest”.

Tôi dùng Apache Iceberg với REST catalog (Apache Polaris hoặc catalog tương thích) trên object storage. Catalog là control plane cho namespace, quyền, snapshot và registration; object store chỉ là data plane. Partition của Silver là `month(ingested_at)` và bucket 128 theo `document_id`; không partition theo `embedding_model` hoặc `subject_id` vì tạo quá nhiều partition nhỏ. File writer gom file 256–512 MB; nightly compaction và manifest rewrite chạy ngoài giờ cao điểm. Các fields truy vấn phổ biến (`jurisdiction`, `court`, `effective_date`) được giữ là columns để lọc trước ANN.

## 3. Các quyết định và alternatives bị loại

### Quyết định 1 — Iceberg là nguồn chuẩn; HNSW chỉ là index dẫn xuất

Tôi chọn **Iceberg Silver/Gold + vector index HNSW dẫn xuất**. Iceberg giữ schema evolution, snapshot và catalog interoperability; HNSW phục vụ p95 dưới 200 ms, nhưng có thể rebuild từ Silver.

- Loại **chỉ dùng vector database**: nhanh cho similarity search nhưng khó giữ OCR text, quyền sử dụng, legal hold, lineage và deletion history làm nguồn chuẩn.
- Loại **brute-force SQL trên lakehouse**: tốt cho audit/offline recall như NB7, nhưng 60 triệu vector không đáp ứng SLA online.
- Loại **nhét PDF blob + index vào cùng một engine online**: làm storage/serving coupling và random read trở thành chi phí theo row group.

### Quyết định 2 — version hóa bất biến theo `document_version`

Tôi chọn append version mới, gắn `supersedes_version`, và pin `snapshot_id` trong citation manifest. Delete logic tách `deleted_at` (chặn query hiện tại) khỏi physical purge theo retention/hold.

- Loại **UPDATE đè cùng một row**: không chứng minh được bản tài liệu nào đã được trích dẫn.
- Loại **chỉ dùng timestamp của object store**: không liên kết được text, chunk và embedding vào cùng một atomic table snapshot.
- Loại **giữ mọi phiên bản online vĩnh viễn**: vừa đắt vừa tăng khả năng trả nhầm bản cũ; version cũ phải chỉ đọc cho audit và chịu retention policy.

### Quyết định 3 — embedding là dữ liệu có version, index nhận CDF/outbox delete

Tôi chọn bảng `embedding(chunk_id, embedding_version, vector, source_snapshot)` và outbox CDF cho `INSERT`, `UPDATE`, `DELETE`. Indexer lưu checkpoint theo Iceberg snapshot/outbox sequence; query service chỉ nhận index build đã có watermark đủ mới.

- Loại **nightly full sync**: deletion có thể tồn tại tới ngày hôm sau; upsert-only thường bỏ quên delete, đúng lifecycle bug của NB7.
- Loại **overwrite vector cũ tại chỗ**: không đánh giá được recall của model mới hay reproduce retrieval cũ.
- Loại **dual write trực tiếp từ API vào table và index**: một nhánh có thể thành công, nhánh còn lại thất bại; outbox idempotent tách retry khỏi transaction nguồn.

### Quyết định 4 — metadata filter trước ANN, không sau ANN

Tôi chọn filter `jurisdiction`, access entitlement, `legal_hold=false`, `deleted_at IS NULL`, `embedding_version` trước hoặc trong vector pre-filter. Gold citation kiểm lại entitlement sau retrieval.

- Loại **ANN trước rồi filter**: top-k có thể bị loại sạch hoặc rò kết quả không được phép vào telemetry/prompt.
- Loại **một index cho mọi tenant/quyền**: đơn giản vận hành nhưng filter bitmap lớn và rủi ro isolation.
- Loại **index riêng cho mọi tenant**: an toàn hơn nhưng 10 triệu PDF tạo quá nhiều shard nhỏ; chỉ tách physical index cho tenant lớn hoặc biên giới dữ liệu bắt buộc.

### Quyết định 5 — lifecycle theo giá trị truy hồi và legal hold

Tôi chọn hot Silver/current index trên S3 Standard, bản OCR/PDF ít truy cập chuyển tầng archive, và Gold citation manifest giữ online năm năm. Lifecycle rule có prefix/tag cho `legal_hold=true`; không expire khi hold còn hiệu lực. S3 Lifecycle có hỗ trợ transition và expiration, nhưng với bucket versioned, expiration current version có thể tạo delete marker và để bản noncurrent tồn tại; quy tắc phải quản lý cả noncurrent version. [AWS Lifecycle documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)

- Loại **một tier Standard cho mọi PDF**: query đơn giản nhưng trả tiền hot storage cho chứng cứ hiếm khi đọc.
- Loại **archive mọi raw PDF ngay khi ingest**: OCR/reprocessing và incident review chậm, không phù hợp SLA.
- Loại **xóa raw ngay sau OCR**: mất chain-of-custody và không thể tái OCR khi model/layout parser cải thiện.

## 4. Chi phí back-of-the-envelope

Giả định 10 triệu PDF trung bình 5 MB = 50 TB raw; 5 TB hot/current + OCR derivatives, 45 TB warm raw, 10 TB archive của version cũ/derivatives; 0.6 TB vector index đã gồm graph overhead và replica; 2 TB metadata/chunk/citation. Tôi dùng $23/TB-tháng cho S3 Standard và xấp xỉ $1/TB-tháng cho Glacier Deep Archive làm mốc đơn giản; AWS công bố Deep Archive ở mức $0.00099/GB-tháng và lưu ý retrieval có độ trễ/chi phí riêng. [AWS Glacier storage classes](https://aws.amazon.com/s3/storage-classes/glacier/) Các con số Standard-IA và compute dưới đây là planning assumptions cần báo giá lại theo region.

| Hạng mục | Phép tính | USD/tháng |
|---|---:|---:|
| Hot Silver/current index | `(5 + 0.6 + 2) TB × $23` | $175 |
| Warm raw PDFs (Standard-IA assumption) | `45 TB × $12.50` | $563 |
| Archive derivative/version cũ | `10 TB × $1` | $10 |
| OCR + embedding batch | `2,500 GPU-h × $0.60` | $1,500 |
| ETL/compaction/catalog compute | `2,000 vCPU-h × $0.08` | $160 |
| Managed vector serving + observability reserve | fixed planning allowance | $450 |
| **Tổng planning estimate** | storage $748 + compute $2,110 | **$2,858/tháng** |

Không coi `$2,858` là báo giá. Cost driver lớn nhất là GPU re-embedding, không phải metadata. Lần migration embedding thứ hai cần ngân sách one-off: `60M chunks × 2,048 B` là khoảng 123 GB raw float16 trước graph/replica, nhưng GPU encoding và index rebuild mới là phần đắt. Tôi throttle rebuild theo shard, chạy canary 1% corpus trước, và chỉ cut over khi recall, freshness watermark và filter parity đạt chuẩn.

AWS hỗ trợ lifecycle transition/expiration, nhưng Glacier Deep Archive có minimum duration và restore Standard có thể mất đến khoảng 12 giờ; vì vậy archive không được đặt trên đường retrieval online. [AWS lifecycle considerations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html) [AWS archive retrieval options](https://docs.aws.amazon.com/AmazonS3/latest/userguide/restoring-objects-retrieval-options.html)

## 5. Failure modes lúc 3 giờ sáng

| Failure mode | Detection | Rollback / xử lý |
|---|---|---|
| Index chưa nhận delete nên trả chunk đã bị xóa | Monitor `outbox_lag`, đối soát `deleted_at` với index sample, synthetic erasure probe phải có 0 hit trong 15 phút | Block query theo deny-list từ Silver, replay outbox từ checkpoint, tombstone shard; nếu không đạt watermark thì route sang lexical/SQL fallback có policy filter |
| Embedding model v2 giảm chất lượng cho tiếng Việt hoặc bảng scan | Canary 1% corpus; recall@10, citation accuracy và reviewer acceptance so với v1 theo jurisdiction | Giữ v1 index và version trong manifest; không overwrite vector; route traffic về v1, giữ v2 để debug |
| Bad OCR/parser tạo sai citation hoặc schema thay đổi làm job fail | Data contract: page count, language, empty-text rate, bbox null rate; schema compatibility check tại catalog CI | Quarantine batch, pin reader về snapshot trước; rollback table pointer/snapshot trong catalog, rerun parser đã sửa |
| Compaction/manifest rewrite thất bại hoặc tạo small-file storm | Alert file count, median file size, planning latency, commit retry rate | Dừng writer mới theo shard, use last known snapshot, rerun idempotent rewrite; không VACUUM cho đến khi reader watermark qua retention |
| Legal hold bị lifecycle transition/expiry xử lý sai | Audit mẫu object tags + catalog hold field; policy test trước deploy | Disable lifecycle rule theo prefix/tag, restore noncurrent copy nếu còn, mở incident review; do đó hold được kiểm trước physical purge |

## 6. MVP một tuần và tiêu chí nghiệm thu

**Slice MVP:** 100.000 PDF (bao gồm scan và text), một jurisdiction, 1 triệu chunks; một embedding model, Iceberg REST catalog, HNSW index nhỏ và RAG endpoint chỉ cho legal reviewer nội bộ.

Ngày 1–2: ingest Bronze kèm hash/provenance; OCR vào Silver và contract tests. Ngày 3: chunk + embedding version v1, build index và lưu build manifest. Ngày 4: metadata/entitlement pre-filter, citation manifest có snapshot pin. Ngày 5: CDF/outbox delete worker, synthetic erasure và dashboard lag. Ngày 6: benchmark/recovery drill. Ngày 7: reviewer test và design review.

Acceptance criteria:

1. P95 retrieval dưới 200 ms ở 50 QPS với metadata filter; ít nhất 99% truy vấn trả citation chứa đúng `document_version` và snapshot pin.
2. Recall@10 của ANN so với exact baseline tối thiểu 0.90 trên tập đánh giá do reviewer gán nhãn.
3. Synthetic delete không còn hit ở Silver lẫn index trong 15 phút; outbox consumer có thể restart mà không duplicate/lost event.
4. Một citation đã ghi có thể replay đúng snapshot, embedding version và top-k manifest sau khi có ingest mới.
5. Compaction tạo file trong khoảng 256–512 MB và không làm query reader đang dùng snapshot cũ lỗi.

Mechanism khó nhất là đồng bộ delete giữa table và derived index. Tôi kiểm chứng bằng một PoC nhỏ: tạo 1.000 chunks, xóa một subject từ bảng nguồn, phát CDF/outbox, cố ý dừng worker, chứng minh index còn stale hit, khởi động worker từ checkpoint, rồi chứng minh hit giảm về 0. PoC này trực tiếp đo lifecycle bug của NB7 thay vì chỉ nói “eventual consistency”.

## References

- [AWS S3 Lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html)
- [AWS lifecycle actions and versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html)
- [AWS S3 Glacier storage classes and reference pricing](https://aws.amazon.com/s3/storage-classes/glacier/)
- [AWS archive retrieval options](https://docs.aws.amazon.com/AmazonS3/latest/userguide/restoring-objects-retrieval-options.html)
