# Giải thích kết quả 8 notebook

Các đoạn dưới đây sử dụng đúng số liệu đã lưu trong `submission/notebooks/`.

## NB1 — Delta basics

File JSON trong `_delta_log` ghi protocol, metadata và các action tạo nên trạng thái bảng. Lần append `age="thirty"` bị chặn vì không ép được chuỗi sang `Int64`, chứng minh schema enforcement hoạt động bằng lỗi thực tế. Khi ghi với `schema_mode="merge"`, cột `tier` được thêm có chủ đích; ba dòng cũ nhận `null`, còn dòng mới có `premium`.

## NB2 — OPTIMIZE và Z-order

Compaction/Z-order giảm số file từ 200 xuống 55, nên giảm chi phí mở file và đọc metadata. Sau Z-order theo `user_id`, chỉ 1/55 file có khoảng min/max chứa `user_id=4242`, tương ứng pruning 55×. Truy vấn đo được nhanh hơn 7,9×, nhưng wall-clock phụ thuộc cache và máy; pruning ratio là bằng chứng ổn định hơn cho file skipping của delta-rs.

## NB3 — MERGE, time travel và RESTORE

MERGE xử lý 100.000 dòng nguồn trong 0,38 giây, gồm update các khóa có sẵn và insert khóa mới. RESTORE về version 2 tạo thêm commit `v4 RESTORE`; nó không xóa lịch sử cũ. Sau restore, số dòng `score < 0` bằng 0 nhưng history vẫn có đủ 5 version, nên rollback vừa khôi phục trạng thái hiện tại vừa giữ audit trail.

## NB4 — Medallion

Bronze giữ 200.000 log thô; Silver còn 190.052 dòng sau khi loại 9.948 bản ghi trùng theo khóa nghiệp vụ. Gold tạo 24 dòng cho 8 ngày × 3 model, gồm p50/p95 latency, token, error rate và chi phí. `p50 ≤ p95`, chi phí dương và error rate trong [0,1], nên bảng phù hợp cho dashboard tổng hợp mà không phải quét lại Bronze.

## NB5 — Iceberg catalog

Bảng được quản lý qua catalog và partition bằng transform `day(ts)`. Khi lọc trên cột nguồn `ts`, `plan_files()` giảm lượng file cần đọc 10×; người dùng không phải tự thêm cột partition. Rename `latency_ms` thành `latency_millis` vẫn giữ `field_id=4`, vì Iceberg đổi tên trong metadata thay vì rewrite data. Hai `spec_id` cùng tồn tại và toàn bộ 5.500 dòng vẫn đọc được.

## NB6 — Maintenance

Compaction giảm 200 file xuống 11; clustering làm phần lớn file có thể bị bỏ qua cho point query. Delta vacuum thu hồi 16,1 MB nhưng không nhìn thấy file do writer hỏng trước commit, nên notebook phải đối chiếu file vật lý với tập file đang được log tham chiếu và xóa đúng 3 orphan. Với PyIceberg trong lab, expiry giảm 20 snapshot xuống 3 nhưng các manifest list không còn được tham chiếu vẫn tồn tại vật lý; sweep riêng mới thu hồi 37,3 KB. Checkpoint giúp reader không phải replay toàn bộ JSON log.

## NB7 — Vector và multimodal

Đọc ngẫu nhiên blob inline gây amplification 200× do granularity của row group. Vector int8 dùng 256 B thay cho 1.024 B mỗi dòng và nhỏ hơn 5,8× trên đĩa, trong khi recall@10 đạt 0,904 và topic fidelity đạt 1,000. Sau khi xóa, bảng trả 0 hit nhưng external index cũ vẫn trả 8 hit; 8 delete event trong CDF cung cấp cơ chế để đồng bộ việc xóa sang index dẫn xuất.

## NB8 — Agent trajectory và provenance

Silver lưu 1.578 bước trajectory, partition theo hai `agent_version`; Gold tổng hợp cả hai policy. Run huấn luyện pin table version 0, và replay ở version đó khớp 1.578 bước đã ghi nhận. Đây là kiểm tra số lượng bước, chưa chứng minh nội dung từng bước hoàn toàn giống nhau. Cache làm 5 lần `list_tables` chỉ đọc catalog 1 lần. `input_required` và task polling chỉ là mô phỏng offline; cờ xác nhận do caller truyền, không phải ranh giới phân quyền. Training filter chọn 1.666/2.000 dòng và loại 334 dòng `UNCLASSIFIED`; bốn bucket còn lại là quy tắc minh họa của lab, không phải kết luận pháp lý.
