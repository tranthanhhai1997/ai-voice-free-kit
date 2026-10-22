# ai-voice-free-kit

Giọng đọc (TTS) **miễn phí / giá rẻ** cho app, kèm kinh nghiệm thực chiến. Phần tiếng Việt được chú trọng riêng.

| Provider | Chi phí | Ghi chú |
|---|---|---|
| **Microsoft Edge Read Aloud** | Miễn phí, không key | Có mốc thời gian từng từ để làm phụ đề (dịch vụ không chính thức, gọi qua gói Python `edge-tts`) |
| **Gemini TTS** | Key Google của bạn, có tầng free | 30 giọng, giọng Bắc/Nam, khoá giới tính, xoay nhiều key |
| **ai33.pro** | Trả phí, giá rẻ | Vbee, ElevenLabs, MiniMax… qua một API |
| **OmniVoice** | Miễn phí, chạy trên máy (GPU) | Clone giọng. ⚠ trọng số CC-BY-NC, không dùng thương mại |
| **Kokoro** | Miễn phí, chạy trên máy (CPU) | Tiếng Anh |

Ngoài TTS, kit có sẵn:
- **chuẩn hoá chữ tiếng Việt**: số, ngày, giờ, tiền, %, đơn vị;
- **lồng tiếng nhiều nhân vật**: tách lời dẫn/lời thoại, chọn giọng theo giới tính/tuổi, nghỉ khi đổi người nói;
- **phụ đề SRT/VTT** theo mốc từng từ, căn lại theo chữ gốc.

Rút ra từ hai app desktop đang chạy thật (sản xuất video AI và video bài giảng), mỗi lỗi đã gặp được ghi lại thành "bẫy" kèm cách tránh.
Mã là JavaScript (Node ≥ 22) **không dependency**; phần model chạy máy là Python.

## Dùng với AI coding agent

Đưa repo này cho Claude Code / Codex / Cursor… để khỏi tốn token research lại:

```text
Đọc docs/INTEGRATION.md và skills/ai-voice-free/SKILL.md trong repo ai-voice-free-kit,
rồi thêm giọng đọc tiếng Việt (Edge mặc định, Gemini dự phòng) + phụ đề SRT vào dự án này.
```

## Bắt đầu nhanh

```bash
git clone https://github.com/tranthanhhai1997/ai-voice-free-kit && cd ai-voice-free-kit
npm test                    # test offline
pip install -U edge-tts     # cho giọng Edge
```

```js
const fs = require('node:fs');
const edge = require('./src/voice/edge-tts');
const { normalizeVi } = require('./src/voice/normalize-vi');
const subs = require('./src/voice/subtitles');

const text = 'Giảm 50% đến ngày 12/5, chỉ còn 199.000đ.';
const r = await edge.synthesize({ text: normalizeVi(text), voice: 'vi-VN-HoaiMyNeural' });
fs.writeFileSync('a.mp3', r.buffer);
fs.writeFileSync('a.srt', subs.toSrt(subs.cuesFromWords(r.words, { text })));
```

Hàng loạt: `node examples/voice-batch.js examples/voice.sample.json`. Đặt `GEMINI_API_KEY` thì tự chuyển sang Gemini.

**Hướng dẫn tích hợp đầy đủ** (Node/Electron/Python, đa giọng, clone, lưu key, checklist, gỡ lỗi): [`docs/INTEGRATION.md`](docs/INTEGRATION.md)

## Cấu trúc

```
docs/INTEGRATION.md            ← hướng dẫn tích hợp từng bước (đọc đầu tiên)
docs/LIVE_TEST.md              ← kết quả chạy thật + số đo
AGENTS.md / CLAUDE.md          ← chỉ dẫn cho AI agent
skills/ai-voice-free/          ← Claude Code skill + tài liệu tham khảo
  references/  providers.md  edge-tts.md  gemini-tts.md  ai33.md  vietnamese-text.md
               voice-clone.md  multi-voice-dubbing.md  voicestudio-notes.md
src/voice/     edge-tts.js  gemini-tts.js  ai33.js  normalize-vi.js  chunk.js  audio.js  multi-voice.js  subtitles.js
src/core/      queue.js (chạy hàng loạt: xoay luồng/key, cooldown, retry)
python/local_tts/  omnivoice_clone/ (clone giọng: làm sạch mẫu, worker tự tắt, CLI)  kokoro_tts.py
examples/      voice-batch.js  voice.sample.json
test/          voice.test.js (105 ca, nhiều ca lấy từ app thật)  queue.test.js
```

## Mức độ kiểm chứng (22/10/2026)

| Phần | Trạng thái |
|---|---|
| Edge: câu ngắn, văn bản dài 2.219 ký tự, danh sách giọng | ✅ chạy thật |
| Gemini TTS: kiểm key, giọng Bắc/Nam, khoá giới tính (đo F0) | ✅ chạy thật |
| Batch đa giọng + SRT | ✅ chạy thật |

Chi tiết: [`docs/LIVE_TEST.md`](docs/LIVE_TEST.md).

## Repo liên quan

Tạo **ảnh / video** AI hàng loạt (Google Flow Veo 3, Muse, ChatGPT, Gemini, Grok qua Chrome CDP):
[tranthanhhai1997/ai-media-batch-kit](https://github.com/tranthanhhai1997/ai-media-batch-kit)

## Lưu ý pháp lý

- **Edge Read Aloud** là endpoint không chính thức của trình duyệt Edge: có thể bị đổi/chặn, không có SLA. Dùng thương mại ổn định thì chọn Azure AI Speech.
  Gói `edge-tts` là LGPL-3.0; repo này chỉ gọi gói bạn tự cài, không chứa mã của nó.
- **Gemini TTS**: tuân hạn mức và điều khoản Google. Chỉ xoay vòng các key bạn sở hữu hợp lệ.
- **OmniVoice**: mã Apache-2.0 nhưng **trọng số CC-BY-NC, không dùng thương mại**. Chỉ clone giọng của chính bạn hoặc người đã đồng ý.
- **ai33.pro** là dịch vụ trả phí của bên thứ ba, theo điều khoản của họ.
- Ghi chú về VoiceStudio (AGPL-3.0) chỉ là tóm tắt kiến thức, không chứa mã của dự án đó.

Repo không chứa key, cookie hay dữ liệu tài khoản nào.

## Liên hệ

Hỗ trợ / tư vấn tích hợp: **tranthanhhai.it1997@gmail.com**
Cần nguồn voice ElevenLabs Free TTS hơn 9000 voice nước ngoài liên hệ riêng mình nhé.
## Giấy phép

[MIT](LICENSE) © 2026 Hai Tran. Dùng, sửa, phân phối tự do (kể cả thương mại), miễn giữ lại thông báo bản quyền.
Phần mềm cung cấp "như hiện trạng", không bảo hành. Model/dịch vụ bên thứ ba theo giấy phép riêng của chúng (xem trên).
