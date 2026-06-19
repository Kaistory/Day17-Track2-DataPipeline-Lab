<!-- DRAFT — đọc lại, thêm trải nghiệm/ý kiến cá nhân và chỉnh giọng trước khi nộp.
     Rubric chấm theo lập luận & tradeoff, không theo độ dài. Hiện ~950 từ. -->

# Bonus Design — Flywheel dữ liệu cho Agent CSKH tiếng Việt

## Bài toán & ràng buộc

Một sàn TMĐT Việt Nam vận hành **agent chăm sóc khách hàng (CSKH)** trả lời về đổi/trả,
bảo hành, vận chuyển. Mỗi ngày ~80k hội thoại, phần lớn **tiếng Việt có dấu, xen teencode,
tên sản phẩm tiếng Anh, và sai chính tả** ("sài đc ko", "ship hoả tốc"). Ta muốn dựng
**flywheel**: biến log hội thoại production thành (a) eval set để chấm chất lượng và
(b) preference pairs để fine-tune/DPO vòng sau — đúng tinh thần lab.

Ràng buộc cứng:
- **Ngân sách nhỏ**: team 3 người, không có cloud data warehouse đắt tiền.
- **Tuân thủ PDPL** (Nghị định 13/2023): hội thoại chứa SĐT, địa chỉ, số đơn → phải khử
  PII trước khi đưa vào tập huấn luyện.
- **Độ trễ phân tích**: eval set cần làm mới hằng ngày là đủ; **không cần real-time**.
- **Chống self-poisoning**: agent vòng N tạo ra log; nếu dùng nguyên log đó train vòng N+1
  mà không lọc, model tự khuếch đại lỗi của chính nó.

## Kiến trúc đề xuất

```
                          ┌─────────────────────────────────┐
  Agent CSKH (prod) ──► OTel traces (gen_ai.*) ──► Kafka topic "chat.traces"
                          │  partition by conversation_id    │ (idempotent key = span_id)
                          └─────────────────────────────────┘
                                         │  (batch hằng ngày)
                                         ▼
   ┌────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐
   │  BRONZE    │──►│   GATE       │──►│   SILVER     │──►│   GOLD / DATASETS│
   │ 1 row/span │   │ PII scrub +  │   │ dedup span,  │   │ eval_golden.jsonl│
   │ (raw spans)│   │ schema check │   │ rollup trace │   │ preference.jsonl │
   └────────────┘   └──────┬───────┘   └──────────────┘   └────────┬────────┘
                           │ bad → quarantine.csv (DLQ)            │ decontam vs eval
                           ▼                                       ▼
                     human review queue                    ASOF feature join
                                                          (point-in-time, no leak)
```

Nền tảng: **DuckDB + dbt** cho lớp Medallion (rẻ, chạy local/CI), **Kafka/Redpanda** cho
thu thập trace, **batch Airflow** chạy 1 lần/ngày. Cố tình *không* dùng Spark/Snowflake.

## Các câu hỏi mở & tradeoff

**1. Batch vs streaming?** Chọn **batch hằng ngày** thay vì streaming real-time. Eval set và
DPO pairs là tài sản *huấn luyện*, không phải tín hiệu serving — độ trễ vài giờ vô hại. Batch
cho phép dedup/decontam trên *toàn bộ* lô (cần nhìn cả tập), điều mà cửa sổ streaming làm
khó. Đánh đổi: mất khả năng cảnh báo lỗi tức thời → bù bằng một luồng metric riêng (xem Q4).

**2. PII: khử ở Bronze hay Silver?** Chọn khử **ngay tại Gate (trước Silver)** bằng regex VN
(SĐT `0|+84`, CCCD 12 số, địa chỉ) + thay token `<PHONE>`, `<ADDR>`. Đổi lại mất một phần
ngữ cảnh (đôi khi địa chỉ là nội dung câu hỏi vận chuyển). Lý do chọn: PDPL coi *lưu* PII
thô đã là rủi ro; khử sớm thu hẹp bề mặt tuân thủ. X (scrub sớm) vs Y (giữ thô, mask lúc
export) → chọn X vì DLQ/Bronze hay bị copy đi nơi khác, khó kiểm soát.

**3. Gán nhãn chosen/rejected thế nào?** Hai nguồn tín hiệu: (a) **outcome ngầm** —
hội thoại kết thúc bằng "cảm ơn shop" / không tạo ticket = chosen; bị khách phản hồi xấu /
escalate = rejected; (b) **chấm bằng LLM-judge**. Chọn (a) làm chính, (b) làm phụ. Tradeoff:
tín hiệu ngầm *rẻ và không nịnh* nhưng nhiễu (khách bỏ ngang ≠ hài lòng); LLM-judge sạch hơn
nhưng tốn tiền và có bias. Dùng (a) lọc thô, (b) chỉ chấm các cặp biên → cắt 80% chi phí judge.

**4. Chống self-poisoning?** Chỉ nhận vào preference pairs những lượt mà **outcome độc lập
với chính model** (khách xác nhận giải quyết, hoàn tất đơn) — không lấy "agent tự tin" làm
nhãn. Thêm **decontamination** như lab: drop mọi cặp có prompt trùng eval. Phát hiện trôi
chất lượng bằng metric production (tỷ lệ escalate) thay vì điểm offline tự-chấm.

**5. Train/serve parity với feature gì?** Feature `khach_tong_chi_tieu_90_ngay` và
`so_lan_doi_tra_truoc_do` phải join **ASOF theo thời điểm hội thoại**. Nếu join "ngây thơ"
lấy giá trị hiện tại, model học được "khách này hay đổi trả" *bằng thông tin tương lai* →
serve sập. Đây đúng bài học ASOF trong lab, áp vào dữ liệu thật.

## Phương án bị loại (rejected alternative)

**Đẩy thẳng toàn bộ log vào fine-tune theo lô lớn, bỏ Gate + decontam.** Hấp dẫn vì nhanh và
nhiều dữ liệu. **Bị loại** vì: (1) self-poisoning — model khuếch đại chính lỗi của nó;
(2) rò rỉ eval làm điểm ảo, che regression thật; (3) vi phạm PDPL khi PII lọt vào trọng số
model — gần như không thể "xoá" sau khi đã train. Rủi ro tuân thủ + chất lượng vượt xa lợi
ích tốc độ.

## Nhận thức tiếng Việt / chi phí / failure-semantics

- **Tiếng Việt**: chuẩn hoá Unicode NFC (dấu tổ hợp vs dựng sẵn) *trước* khi dedup/embed —
  nếu không, "đổi" và "đổi" thành hai chuỗi khác nhau, dedup trật và KG tách nhầm node.
- **Chi phí**: 80% tiền nằm ở LLM-judge → giới hạn judge cho cặp biên là đường cắt 80/20.
- **Failure-semantics**: pipeline **idempotent theo `span_id`** (replay Kafka không nhân đôi);
  một hội thoại lỗi rơi vào DLQ, *không* làm sập lô; backfill an toàn vì Gold sinh lại được
  từ Bronze bất biến.
