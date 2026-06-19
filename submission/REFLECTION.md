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

1. **Flywheel.** Bước dễ hỏng ngầm nhất là `flatten()` đệ quy span-tree trong
   `pipeline/traces.py`. Gán nhầm `trace_id` hay sót span con thì pipeline không báo lỗi —
   dataset vẫn sinh ra, chỉ thiếu/lệch. Tôi bắt bằng invariant: tổng span Bronze khớp trace
   gốc (21/8), mỗi `trace_id` roll-up đúng một dòng.

2. **Decontamination.** Bỏ bước này thì prompt eval rò vào train (lần chạy của tôi loại 2/3
   cặp). Model học thuộc đúng câu sắp bị chấm nên điểm eval đẹp giả tạo, chất lượng thật không
   đổi. Lời nói dối lộ ra khi điểm offline cao mà metric production (escalate, thắng/thua thật)
   đứng im.

3. **Point-in-time.** Tôi nghĩ tới `lifetime_spend`/`account_balance` của khách. Join không
   khoá thời điểm bằng `ASOF` sẽ kéo giá trị tương lai vào hàng huấn luyện quá khứ — model
   "nhìn trộm", train đẹp nhưng serve sập vì lúc inference giá trị đó chưa có.

4. **Graph vs vector.** Knowledge graph thắng câu đa bước "widget ship từ đâu?"
   (widget → accessory → kho Hà Nội): hai fact ở hai chunk nên retrieval phẳng không bắc cầu
   được. Nhưng tra một fact ("gadget bảo hành bao lâu?") thì graph là thừa — một chunk đã đủ.
