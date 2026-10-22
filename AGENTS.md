# AGENTS.md — hướng dẫn cho AI (Claude Code, Codex, Cursor, Gemini CLI…) đọc repo này

Người dùng đưa repo này để bạn **ghép giọng đọc (TTS) miễn phí / giá rẻ vào dự án của họ**: Edge, Gemini TTS, ai33, clone giọng chạy máy,
chuẩn hoá tiếng Việt, lồng tiếng nhiều nhân vật, phụ đề. Đừng research lại từ đầu: bẫy và số đo ở đây lấy từ app production.

## Thứ tự đọc

1. `docs/INTEGRATION.md`: hướng dẫn tích hợp từng bước, có mã mẫu chạy được và checklist nghiệm thu.
2. `skills/ai-voice-free/SKILL.md` → `references/providers.md` (chọn provider) → `references/vietnamese-text.md` nếu đọc tiếng Việt.
3. Tài liệu và mã của provider cần dùng:
   - `edge-tts.md` → `src/voice/edge-tts.js`
   - `gemini-tts.md` → `gemini-tts.js`
   - `ai33.md` → `ai33.js`
   - `voice-clone.md` → `python/local_tts/`
   - Nhiều nhân vật: `multi-voice-dubbing.md` → `multi-voice.js`
4. `docs/LIVE_TEST.md`: phần nào đã chạy thật, phần nào chưa.

## Khi tích hợp

- Hỏi người dùng nếu chưa rõ:
  - provider chính và provider dự phòng;
  - ngôn ngữ/vùng miền;
  - runtime đích;
  - có dùng thương mại không (nếu có: loại OmniVoice, vì trọng số CC-BY-NC);
  - máy có GPU không.
- Node/Electron: copy `src/voice/` (+ `src/core/queue.js` nếu chạy hàng loạt), giữ nguyên API. Electron: chỉ gọi TTS ở **main process**, renderer gọi qua IPC.
- Ngôn ngữ khác: port, **giữ nguyên các hằng số đã đo** (giới hạn ký tự/byte, thời gian nghỉ key, khoảng lặng 0,45/0,12/0,18 s) và **giữ nguyên ca test** trong `test/voice.test.js`.
- Trước bước tốn quota/credit (Gemini, ai33), hỏi người dùng. Không đưa key vào URL/log/thông báo lỗi.
- Phần ghi "Chưa kiểm chứng" là giả thuyết: kiểm thật, rồi cập nhật tài liệu nếu được phép.

## Lệnh

```bash
npm run check   # node --check + require mọi module (không mạng)
npm test        # test offline: giọng đọc + hàng đợi batch
pip install -U edge-tts                                          # cho Edge
node examples/voice-batch.js examples/voice.sample.json          # TTS hàng loạt (gọi mạng thật)
cd python/local_tts && python -m py_compile kokoro_tts.py omnivoice_clone/*.py
```

## Quy ước mã

- CommonJS, Node ≥ 22 (fetch toàn cục), không dependency. Comment tiếng Anh, giải thích "vì sao".
- Lỗi có `code` dạng `PROVIDER_REASON` (`EDGE_FORBIDDEN`, `GEMINI_TTS_QUOTA`, `AI33_NO_CREDIT`…).
- Mốc từ thống nhất `{ text, offsetMs, durationMs }`.
- Python (`python/local_tts/`): 3.10–3.12; import nặng (torch, omnivoice) đặt trong hàm.
