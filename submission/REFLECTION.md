# Reflection

**Anti-pattern lựa chọn:** Small Files Problem.

Hệ thống tôi quan tâm là nền tảng quan sát LLM nhận log theo micro-batch. Nếu mỗi client hoặc lần retry ghi một file, Bronze nhanh chóng có hàng nghìn file nhỏ. Khi đó chi phí mở file, liệt kê object và đọc metadata có thể lớn hơn chi phí xử lý dữ liệu. NB2 tái hiện hiện tượng này: 200 file ban đầu làm truy vấn điểm chậm rõ rệt.

Tôi sẽ đặt kích thước batch và chu kỳ flush hợp lý tại writer, theo dõi số file cùng kích thước trung vị, rồi chạy compaction theo ngưỡng. Với cột lọc thường xuyên như `user_id`, Z-order thu hẹp khoảng min/max để tăng file skipping. Không nên gom toàn bộ thành một file vì sẽ mất lợi ích pruning và tăng tranh chấp ghi.

**Phạm vi sử dụng AI:** Tôi dùng Gemini và OpenAI Codex để hỗ trợ đọc yêu cầu, chạy và kiểm tra notebook, phân tích lỗi, rà rubric và trình bày bài nộp. Tôi đã kiểm tra output và chịu trách nhiệm về nội dung.
