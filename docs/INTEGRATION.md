# Hướng dẫn tích hợp giọng đọc (TTS) vào dự án

Tài liệu này dành cho **người** lẫn **AI coding agent**: đọc từ trên xuống, chỉ làm phần mình cần.
Mọi API dưới đây là API thật của mã trong `src/voice/` (Node ≥ 22, CommonJS, không dependency).

---

## 0. Chọn provider trong 30 giây

| Bạn cần | Dùng | Chi phí | Tiếng Việt |
|---|---|---|---|
| Miễn phí, không đăng ký, có phụ đề theo từng từ | **Edge** (`edge-tts.js`) | 0 đ (dịch vụ không chính thức) | 2 giọng: HoaiMy (nữ), NamMinh (nam) |
| Giọng tự nhiên hơn, nhiều giọng, chỉnh giọng miền/cảm xúc | **Gemini TTS** (`gemini-tts.js`) | Key Google của bạn, có tầng free (hạn mức thấp) | 30 giọng gốc × giọng Bắc/Nam |
| Nhiều giọng thương mại (Vbee, ElevenLabs, MiniMax…) giá rẻ | **ai33.pro** (`ai33.js`) | Trả phí theo credit | Vbee + giọng thư viện |
| Clone giọng riêng, chạy trên máy | **OmniVoice** (`python/local_tts/omnivoice_clone`) | 0 đ, cần GPU ~6 GB | Tốt. ⚠ **Trọng số CC-BY-NC: không dùng thương mại** |
| Chạy offline tiếng Anh, máy yếu | **Kokoro** (`python/local_tts/kokoro_tts.py`) | 0 đ, CPU được | ❌ Không có tiếng Việt |

Nên có **ít nhất 2 provider**: provider chính + dự phòng. Edge có thể bị chặn bất cứ lúc nào, còn Gemini free hết lượt theo phút/ngày.

---

## 1. Cài đặt

```bash
git clone https://github.com/tranthanhhai1997/ai-voice-free-kit
cd ai-voice-free-kit
npm test                         # 105+ test offline, không gọi mạng
pip install -U edge-tts          # chỉ khi dùng Edge (gói Python mã nguồn mở, LGPL)
```

Phần cần chép vào dự án của bạn:

| Dự án của bạn | Chép | Ghi chú |
|---|---|---|
| Node / Electron (main process) | `src/voice/` (+ `src/core/queue.js` nếu chạy hàng loạt) | Không cần `npm install` gì thêm |
| Electron renderer / web | **Không** gọi trực tiếp từ renderer | Gọi qua IPC tới main process: key không lộ ra renderer, và Edge cần chạy tiến trình con |
| Python | Port theo mã Node, **giữ nguyên ca test** trong `test/voice.test.js` | Edge: dùng thẳng gói `edge_tts`. Gemini/ai33: chỉ là REST |
| Ngôn ngữ khác (Go, C#, PHP…) | Port như trên | Hằng số thời gian/giới hạn ký tự đều đã đo, giữ nguyên |

Yêu cầu phụ:
- **ffmpeg** trong PATH: chỉ cần khi ghép nhiều file mp3 (kịch bản đa giọng Edge/ai33) hoặc đổi định dạng. Ghép WAV (Gemini) không cần ffmpeg.
- Python ≥ 3.8 + `edge-tts`: chỉ cho Edge. Nếu máy có nhiều bản Python, đặt `EDGE_TTS_PYTHON=C:\path\python.exe`.

---

## 2. Đọc một câu

### Edge (miễn phí)

```js
const fs = require('node:fs');
const edge = require('./src/voice/edge-tts');
const { normalizeVi } = require('./src/voice/normalize-vi');
const subs = require('./src/voice/subtitles');

const goc = 'Giảm 50% đến ngày 12/5, chỉ còn 199.000đ.';
const r = await edge.synthesize({ text: normalizeVi(goc), voice: 'vi-VN-HoaiMyNeural', rate: -4 });
fs.writeFileSync('cau1.mp3', r.buffer);                                       // mp3 24 kHz
fs.writeFileSync('cau1.srt', subs.toSrt(subs.cuesFromWords(r.words, { text: goc })));   // phụ đề theo CHỮ GỐC
```

- `rate`/`volume` tính bằng %, `pitch` tính bằng Hz. Giọng bé: giọng người lớn + `pitch: 18`.
- `r.words = [{ text, offsetMs, durationMs }]`.

### Gemini TTS (key của bạn)

```js
const { createGeminiTts } = require('./src/voice/gemini-tts');
const tts = createGeminiTts({ keys: [process.env.GEMINI_API_KEY] });  // nhiều key bạn sở hữu → tự xoay vòng
console.log(await tts.checkKeys());                                    // kiểm key, không tốn quota TTS
const r = await tts.synthesize({ text: normalizeVi(goc), voice: 'Fenrir', accent: 'vi-s', gender: 'male', style: 'vui tươi, rõ ràng' });
fs.writeFileSync('cau1.wav', r.buffer);                                // WAV 24 kHz; không có mốc từ
fs.writeFileSync('cau1.srt', subs.toSrt(subs.cuesFromText(goc, r.durationMs)));
```

- `accent`: `vi-n` (Bắc), `vi-s` (Nam), `en-us`, `en-gb`.
- **Luôn truyền `gender`.** Không khoá thì vài giọng nam đọc tiếng Việt miền Nam ra giọng nữ (đo được 205–253 Hz); có khoá thì Fenrir đo 115 Hz.
- Hết lượt: `GEMINI_TTS_QUOTA` (key nghỉ 65 s; hết lượt ngày thì nghỉ 1 h).

### ai33.pro (trả phí rẻ)

```js
const { createAi33 } = require('./src/voice/ai33');
const ai33 = createAi33({ apiKey: process.env.AI33_API_KEY });
console.log(await ai33.credits());
const voices = await ai33.listVoices({ provider: 'vbee' });            // cache 24 h
const r = await ai33.synthesize({ voiceId: voices[0].id, text: normalizeVi(goc), speed: 1.0 });
```

- Tạo giọng là tác vụ bất đồng bộ: tạo task → chờ → tải file. Code đã tự lo phần này.
- `AI33_NO_CREDIT` nghĩa là hết tiền.
- Chỉ gửi lại khi chắc server **chưa nhận** request (429/503), để tránh bị trừ tiền hai lần.

---

## 3. Văn bản tiếng Việt: bắt buộc chuẩn hoá

```js
normalizeVi('Ngày 22/10/2026, giá 150.000đ, tăng 15%, nhiệt độ 32°C.')
// → "Ngày hai mươi hai tháng mười năm hai nghìn không trăm hai mươi sáu, giá một trăm năm mươi nghìn đồng, …"
```

- Chuẩn hoá **trước** khi gửi TTS. Phụ đề thì làm từ **chữ gốc**: `cuesFromWords(words, { text: goc })` tự căn lại.
- Chỉ chuẩn hoá khi văn bản là tiếng Việt (`looksVietnamese(text)`). Chạy trên tiếng Anh sẽ biến "3.5" thành chữ Việt.
- Văn bản dán từ Word/PDF thường ở dạng Unicode tách dấu: mã đã chuyển sang NFC.
- Chi tiết + bẫy: `skills/ai-voice-free/references/vietnamese-text.md`.

---

## 4. Kịch bản nhiều nhân vật (lồng tiếng)

```js
const mv = require('./src/voice/multi-voice');
const audio = require('./src/voice/audio');

const characters = [{ name: 'Lan', description: 'cô gái hai mươi tuổi' }, { name: 'Ông Ba', description: 'cụ ông hiền lành' }];
const voices = [{ id: 'vi-VN-HoaiMyNeural', gender: 'female' }, { id: 'vi-VN-NamMinhNeural', gender: 'male' }];  // {id, gender, desc?}
const cast = mv.castVoices(characters, voices, { narratorVoice: 'vi-VN-HoaiMyNeural' });
// → Map { 'Lan' → HoaiMy, 'Ông Ba' → NamMinh }; hoặc tự gán bằng new Map([...])
// Edge chỉ có 2 giọng Việt nên người kể và nhân vật nữ dùng chung HoaiMy. Muốn tách tai: dùng Gemini (30 giọng) hoặc lệch pitch.

const script = 'Chiều muộn, Lan về đến cổng. Ông Ba mỉm cười: "Về rồi đấy à con?" "Dạ, con về rồi ông ơi," Lan đáp.';
const lines = mv.parseScript(script, characters);
const chunks = mv.planChunks(lines, (who) => cast.get(who), 'vi-VN-HoaiMyNeural', { maxChars: 220, transform: normalizeVi });
const pieces = [];
for (const c of chunks) pieces.push(await edge.synthesize({ text: c.text, voice: c.voice }));
const gaps = mv.gapsAfter(chunks);              // nghỉ 0,45 s khi đổi người nói, 0,12 s cùng người
// mp3 → ghép bằng ffmpeg; WAV (Gemini) → audio.concatWav(pieces.map(p => p.buffer), { gapsSec: gaps })
```

- Nhãn kiểu `Lan ôm hộp quà: "…"`: phần dẫn do người kể đọc, phần trong ngoặc do Lan đọc. Tên nhãn không bị đọc lên.
- Giới tính/tuổi suy ra từ mô tả tiếng Việt ("ba mươi tuổi", "cụ ông", đực/cái…), nên kiểm lại khi mô tả mơ hồ.
- Chi tiết: `skills/ai-voice-free/references/multi-voice-dubbing.md`.

---

## 5. Chạy hàng loạt

Dùng `examples/voice-batch.js` làm mẫu:

```bash
node examples/voice-batch.js examples/voice.sample.json out/voice      # Edge mặc định
GEMINI_API_KEY=... node examples/voice-batch.js examples/voice.sample.json   # tự chuyển sang Gemini
```

Nguyên tắc (đã có trong ví dụ):
1. **Ghi file ngay khi xong từng câu**; chạy lại thì bỏ qua câu đã có file.
2. Phân loại lỗi cho `runBatch` (`src/core/queue.js`):
   - `*_NETWORK / *_TIMEOUT / *_NO_AUDIO` → thử lại;
   - `*_QUOTA / *_RATE_LIMIT` → cho key/luồng nghỉ;
   - còn lại → bỏ câu đó.
3. Edge chậm hơn thời gian thực (đo: 169 s cho 136 s audio), nên chạy 2–4 luồng. Gemini free thì giữ 1–2 luồng mỗi key.

---

## 6. Clone giọng chạy máy (OmniVoice)

```bash
cd python/local_tts
pip install -r requirements.txt            # torch CUDA + omnivoice + faster-whisper (tuỳ chọn)
python -m omnivoice_clone.cli --ref mau_giong.mp4 --text-file kichban.txt --out ra.wav --lang vi --asr small
```

- Mẫu tốt: 6–15 s lời nói rõ, không nhạc. Có nhạc thì thêm `--demucs`. Biết chính xác lời trong mẫu thì truyền `--ref-text`.
- Tích hợp vào app: chạy `worker.py` như tiến trình con (JSON lines qua stdin/stdout). Model chỉ nạp khi cần và **tự tắt sau N phút rảnh**; giữ model thường trú tốn 2–3 GB RAM.
- ⚠ **Trọng số OmniVoice là CC-BY-NC: không dùng cho sản phẩm thương mại.** Muốn dùng thương mại, xem VoxCPM2 / Confucius4-TTS (Apache-2.0, có tiếng Việt) trong `skills/ai-voice-free/references/voicestudio-notes.md`.
- **Chỉ clone giọng của chính bạn hoặc người đã đồng ý.**

---

## 7. Lưu key an toàn

- Không đặt key trong URL, log hay thông báo lỗi. Mã đã dùng header và tự che key khi log.
- Electron: lưu key bằng `safeStorage.encryptString` trong `userData`, giải mã ở main process.
- Server/CLI: biến môi trường (`GEMINI_API_KEY`, `AI33_API_KEY`) hoặc kho bí mật của hệ thống.
- Gemini: chỉ xoay vòng các key **bạn sở hữu hợp lệ**. Tạo hàng loạt tài khoản để lách hạn mức là vi phạm điều khoản của Google.

---

## 8. Checklist nghiệm thu

- [ ] `npm test` đạt.
- [ ] Mỗi provider đã bật: 1 câu ngắn tiếng Việt có số/ngày/tiền, nghe đọc đúng.
- [ ] 1 đoạn > 1 500 ký tự: không bị cắt, phụ đề không lệch dần.
- [ ] 1 kịch bản 2 nhân vật: đúng giọng, có khoảng nghỉ, không đọc tên nhãn.
- [ ] Cố tình dùng key sai / ngắt mạng: app báo lỗi rõ ràng, không treo, không lộ key.
- [ ] Có provider dự phòng khi provider chính lỗi.
- [ ] Đã kiểm giấy phép model/dịch vụ theo mục đích (cá nhân hay thương mại).

---

## 9. Prompt mẫu cho AI coding agent

```text
Đọc docs/INTEGRATION.md và skills/ai-voice-free/SKILL.md trong repo ai-voice-free-kit.
Tích hợp giọng đọc vào dự án này:
- Provider chính: <Edge | Gemini TTS | ai33>, dự phòng: <…>
- Ngôn ngữ: tiếng Việt (chuẩn hoá bằng normalizeVi), phụ đề SRT theo chữ gốc
- Runtime: <Electron main process | Node server | Python>
- <Có/Không> kịch bản nhiều nhân vật
Chép src/voice/ (giữ nguyên test), lưu key bằng <safeStorage | env>, làm theo checklist mục 8.
Hỏi tôi trước mỗi bước tốn quota/credit.
```

---

## 10. Gỡ lỗi nhanh

| Mã lỗi | Nguyên nhân thường gặp | Cách sửa |
|---|---|---|
| `EDGE_NOT_INSTALLED` | Thiếu gói `edge-tts` hoặc sai bản Python | `pip install -U edge-tts`, hoặc đặt `EDGE_TTS_PYTHON` |
| `EDGE_FORBIDDEN` | Microsoft đổi cơ chế xác thực | `pip install -U edge-tts`; kiểm giờ hệ thống |
| `EDGE_NO_AUDIO` | Tên giọng sai, hoặc lỗi tạm thời của server | Kiểm `listVoices()`; mã đã tự thử lại |
| `GEMINI_TTS_QUOTA` | Mọi key đang nghỉ (hết lượt phút/ngày) | Chờ, thêm key hợp lệ, hoặc chuyển provider dự phòng |
| `GEMINI_TTS_AUTH` | Key sai/bị thu hồi | Tạo key mới ở Google AI Studio |
| `AI33_NO_CREDIT` | Hết credit | Nạp thêm |
| `AUDIO_*` | Thiếu ffmpeg khi ghép mp3 | Cài ffmpeg, hoặc dùng WAV + `concatWav` |

Kết quả chạy thật và số đo: `docs/LIVE_TEST.md`.
