# Kết quả chạy thật — giọng đọc

Ngày 22/10/2026. Máy test: Windows 11, Node 24, Python 3.12, edge-tts 7.3.1. Mọi số đo là đo thật.


| Provider | Bước | Kết quả | Thời gian | Ghi chú |
|---|---|---|---|---|
| Edge (gói Python `edge-tts`) | câu tiếng Việt có số/ngày/tiền/%/°C (đã `normalizeVi`) | ✅ | 12,5 s cho 11,8 s audio | 47 mốc từ; SRT căn lại theo chữ gốc giữ "22/10/2026", "150.000đ", "32°C" |
| Edge | văn bản 2.219 ký tự | ✅ | **169 s cho 136 s audio** | 480 mốc từ liền mạch. Chậm hơn thời gian thực, nên batch lớn phải chạy song song |
| Edge | `listVoices` | ✅ | 0,8 s | 322 giọng; tiếng Việt chỉ có HoaiMy (nữ), NamMinh (nam) |
| Edge | gói **edge-tts 7.2.8** | ⛔ 403 `EDGE_FORBIDDEN` | — | `pip install -U edge-tts` → 7.3.1 thì chạy |
| Gemini TTS | `checkKeys` 3 key | ✅ | 0,6 s | không tốn quota TTS |
| Gemini TTS | Fenrir giọng Nam (vi-s), khoá giới tính nam | ✅ | 5,5 s cho 7,2 s audio | **F0 = 115 Hz** (giọng nam). Lỗi cũ không khoá ra 205–253 Hz |
| Gemini TTS | Kore giọng Bắc (vi-n) | ✅ | 6,6 s cho 7,6 s audio | F0 = 218 Hz |
| `examples/voice-batch.js` | 2 câu + 1 kịch bản 2 nhân vật (Edge) | ✅ | ~12 s | mp3 + SRT; đổi người nói nghỉ 0,45 s, lời dẫn/lời thoại tách đúng |
| ai33 | — | chưa chạy | — | cần key ai33 của người dùng |
| OmniVoice clone (`python/local_tts`) | — | chưa chạy | — | runtime trên máy test trộn 2 bản torch, chưa dựng được môi trường sạch; API đối chiếu mã nguồn gói `omnivoice` 0.2.1 |

Sửa sau khi chạy thật:
- `EDGE_NO_AUDIO` trên văn bản dài có lúc chỉ là lỗi tạm: cùng văn bản chạy lại thì được. Giờ mã thử lại thay vì dừng ngay.
- Bỏ phần tự cài lại token của Edge (bản port từ mã LGPL), thay bằng gọi gói `edge-tts` người dùng tự cài.
