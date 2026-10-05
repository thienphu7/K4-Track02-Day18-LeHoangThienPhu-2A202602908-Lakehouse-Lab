# Reflection — Lakehouse anti-pattern

## Anti-pattern: coi vector database là system of record

RAG đồng bộ một chiều từ lakehouse sang vector database rồi coi index là nguồn dữ liệu chính. Khi tài liệu bị sửa hoặc có yêu cầu xóa dữ liệu cá nhân, embedding cũ vẫn có thể được truy hồi. NB7 tái hiện rủi ro này: bảng lakehouse đã xóa tài liệu nhưng external index vẫn trả kết quả.

Tôi giữ văn bản, metadata, quyền sử dụng, subject ID và embedding version trong lakehouse làm nguồn chuẩn; vector index chỉ là derived, rebuildable index. Mọi thay đổi đi qua Change Data Feed/outbox có idempotency key; consumer checkpoint theo version và đối soát với snapshot bảng. Với erasure request, hệ thống xóa ở nguồn, phát delete event, đo backlog đến khi index trả 0 hit, rồi lưu audit record. Time travel hữu ích cho audit nhưng không thay thế retention, VACUUM và chính sách xóa bản sao.

## Phạm vi sử dụng AI

Tôi dùng Codex để tóm tắt checklist, tổ chức ảnh screenshot để tăng tốc phần thu thập bằng chứng. Tôi tự chạy notebook, kiểm tra output/ảnh và rà soát nội dung trước khi nộp.
