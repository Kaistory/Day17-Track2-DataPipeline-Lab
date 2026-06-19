# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

_Write your answers below._

<!-- DRAFT — đọc lại và chỉnh theo giọng của bạn trước khi nộp -->

1. **Flywheel.** `flatten()` đệ quy span-tree (`pipeline/traces.py`) hỏng ngầm nhất: bỏ sót
   span con hoặc gán nhầm `trace_id` thì eval set và DPO pairs vẫn *sinh bình thường*, không
   báo lỗi — chỉ thiếu/lệch dữ liệu. Phát hiện bằng invariant: tổng span Bronze = tổng span
   trace gốc (21/8), mỗi trace_id roll-up đúng 1 dòng.

2. **Decontamination.** Bỏ qua → prompt eval rò vào tập train (2/3 cặp). Model "học thuộc"
   chính câu sẽ bị chấm → **eval score cao giả tạo**, trong khi chất lượng thật không tăng.
   Lời nói dối lộ ra khi điểm offline đẹp nhưng metric production (thắng/thua thật) đứng im
   hoặc tệ đi.

3. **Point-in-time.** `account_balance`/`lifetime_spend`: join không ASOF kéo *giá trị tương
   lai* vào hàng huấn luyện quá khứ → model "nhìn trộm", train đẹp nhưng serve sập vì lúc
   inference giá trị đó chưa tồn tại.

4. **Graph vs vector.** KG mạnh ở câu **đa bước** (widget→accessory→kho Hà Nội) vì fact nằm
   ở hai chunk khác nhau, retrieval phẳng không bắc cầu được. Graph là *thừa* cho tra cứu
   1 fact ("bảo hành gadget bao lâu?") — một chunk đã trả lời, KG chỉ thêm chi phí dựng đồ thị.
