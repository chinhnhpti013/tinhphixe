# Dự án: PTI Insurance Calculator PRO

## Mô tả
Mini web app tính phí bảo hiểm vật chất xe ô tô (VCX) theo quy tắc và biểu phí của PTI (Bảo hiểm Bưu điện Việt Nam). Chạy hoàn toàn trên trình duyệt, không cần backend.

## File hệ thống

| File | Mô tả |
|------|-------|
| `index.html` | File duy nhất — toàn bộ HTML + CSS + JS nhúng trong một file |
| `docs/Quy-trinh.docx` | Tài liệu quy trình tính phí PTI (nguồn logic nghiệp vụ) |
| `docs/CV2896- Triển khai quy tắc và biểu phí sản phẩm tự nguyện XCG.pdf` | Công văn triển khai quy tắc & biểu phí |
| `docs/Hướng dẫn Quy Tắc Và Biểu Phí Bảo Hiểm Tự Nguyện Xe Cơ Giới Mới.pdf` | Hướng dẫn quy tắc & biểu phí mới |
| `docs/Phụ lục 01- Biểu phí bảo hiểm VCX ô tô_10.2025.xlsx` | Biểu phí gốc PTI (Excel, dùng để tham chiếu tỷ lệ phí) |
| `input/thong_tin_giam_dinh_xe.xlsx` | Dữ liệu giám định xe mẫu |
| `.claude/settings.json` | Cấu hình dự án: kiến trúc, nguồn tra giá, quy tắc nghiệp vụ, OCR |
| `.claude/skill.md` | Hướng dẫn chỉnh sửa từng module (rates, BS, tra giá, OCR...) |

## Cấu trúc index.html

### CSS
- Font: `Sora` (Google Fonts) + `JetBrains Mono`
- Theme: dark/light toggle (CSS variable `data-theme`)
- Layout: 2 cột sticky (form trái, kết quả phải)
- Responsive mobile: 1 cột khi < 1024px
- Form fields compact: label 9px, input 11px/padding 5px 9px

### JavaScript — Modules chính

```
RATES{}              — Bảng tỷ lệ phí: 14 loại xe × 4 nhóm tuổi × 7 mức giá trị
BS[]                 — 17 điều khoản bổ sung BS01–BS27
bsEffective()        — Tỷ lệ BS theo THỜI GIAN SỬ DỤNG XE (Phụ lục 01): BS02 0/0.10/0.20/0.30%
                       theo nhóm tuổi; BS05 0.45/0.50/0.75%, BS26 0.13/0.15/0.22% (xe ≥10 năm
                       chặn, trình TCT); BS07 chỉ xe mới 100%; mã khác giữ tỷ lệ cố định
bsSelected           — Set<string> lưu mã BS đang được chọn (không dùng checkbox)
calculate()          — Engine tính phí: làm tròn nghìn ₫, VAT 10%, cảnh báo nghiệp vụ
                       (xe >15 năm, KDVT lệch loại xe, STBH ≥5 tỷ), so sánh 4 mức khấu trừ
renderChart()        — Biểu đồ Chart.js (CDN)
toast()              — Thông báo nổi thay alert() (#toastWrap, kiểu ok/warn/err)
copySummary()        — Sao chép tóm tắt báo giá vào clipboard
exportHistoryCSV()   — Xuất lịch sử ra file CSV (BOM UTF-8, mở được bằng Excel)
saveHistory()        — Lưu localStorage key: 'pti_h'
processOCRFiles()    — Xử lý NHIỀU ảnh tuần tự: Claude Vision (nếu có key) hoặc ocrSmart
ocrSmart()           — OCR KHÔNG CẦN API KEY: best-of-2 passes Tesseract —
                       preprocessSoft (giảm kênh đỏ, giữ chữ dưới mộc đỏ) +
                       preprocessBinary (Otsu), chấm điểm scoreExtraction, merge trường tốt nhất
vinValid()           — Checksum VIN ISO 3779 (chỉ dùng làm tín hiệu dương khi chấm điểm)
textFromWords()      — Lọc từ Tesseract theo confidence ≥30, giữ cấu trúc dòng
ocrWithClaude()      — Gọi Anthropic API Vision để đọc ảnh giấy tờ xe → JSON
parseOCRText()       — Phân tích văn bản OCR → các trường thông tin xe
fillFromOCR()        — Điền hidden inputs từ OCR → render bảng xe → autoSearchPrice
renderVehicleTable() — Render bảng thông tin xe có thể edit inline (#vehicleInfoTable)
syncVitField()       — Đồng bộ edit inline bảng → hidden input
detectVersion()      — Phát hiện phiên bản xe từ model code + số khung
modelFromChassis()   — Giải mã tên model từ VIN (KIA ND5 → CARNIVAL...)
autoSearchPrice()    — Tìm giá thị trường qua API Chợ Tốt (nguồn duy nhất)
calcMarketPrice()    — Giá thị trường tại thời điểm tra cứu: lọc ngoại lệ IQR (hàng rào 1.5×IQR),
                       lấy trung vị của tối đa 20 tin rao mới nhất (theo list_time)
resetOCRZone()       — Reset khung upload + xóa bảng + xóa hidden inputs
getApiKey()          — Đọc Anthropic API key từ localStorage key: 'pti_anthropic_key' (đặt key qua DevTools)
loadTesseract()      — Lazy-load Tesseract.js khi cần OCR lần đầu (không chặn render lúc mở trang)
correctVIN()         — Sửa lỗi OCR cho số khung (S→5 tại pos digit, B→8, I→1...)
fuzzyModel()         — Fuzzy match tên model (Levenshtein ≤25%) → sửa CARNFVAL→CARNIVAL
```

### Tính năng OCR — Multi-file, KHÔNG cần API key
- Hỗ trợ: kéo thả / chọn nhiều file đồng thời (`<input multiple>`) / `Ctrl+V` clipboard
- **Mặc định: ocrSmart (Tesseract.js v5, vie+eng) — chạy hoàn toàn trên máy, không cần API key**
  - Lượt 1 `preprocessSoft`: grayscale giảm trọng số kênh đỏ (mờ dấu mộc đỏ) + kéo giãn tương phản percentile 2–98%
  - Lượt 2 `preprocessBinary`: nhị phân hóa Otsu — chỉ chạy nếu lượt 1 chưa đủ điểm (`GOOD_ENOUGH = 9`)
  - Mỗi lượt parse 2 bản văn bản: bản lọc confidence ≥30 (ưu tiên) + bản thô (bù trường thiếu)
  - `scoreExtraction` chấm điểm theo trọng số trường + bonus VIN qua checksum + biển số đúng chuẩn
  - Merge giữa các lượt và giữa các ảnh: ưu tiên số khung qua được `vinValid`
- **Tùy chọn: Claude Vision API** (Anthropic Haiku) khi người dùng tự cài key — chính xác hơn nữa
  - API key lưu trong `localStorage['pti_anthropic_key']`, nhập qua nút 🔑 trong upload zone
  - `anthropic-dangerous-direct-browser-access: true` header cho phép gọi từ browser
- Sau OCR: hiển thị `#vehicleInfoTable` — bảng 10 trường có thể edit inline (click vào giá trị)
- Checkbox "Kinh doanh vận tải" hiển thị riêng bên dưới bảng
- Form fields (f_plate, f_owner...) là hidden inputs, dùng cho tính toán

### Phát hiện phiên bản xe
- `detectVersion(brand, modelCode, chassis)` chạy sau OCR
- Bảng patterns cho: Mitsubishi, Toyota, Honda, Hyundai, Kia, Ford, Mazda, VinFast, Isuzu, Hino
- Phân tích: transmission (CVT/AT/MT), drivetrain (AWD/4WD), trim (Ultimate/Premium/Sport…)
- `f_version` hidden input, không hiển thị trong bảng xe (dùng nội bộ cho price search)

### Tự động tìm giá thị trường
- Query dựa trên: brand + model (ưu tiên decode từ VIN qua `modelFromChassis`) + year
- `parseOCRText` ghép nhãn hiệu + cụm chữ đầu Mã kiểu loại thành brand ("KIA" + "SONET QY PE..." → "KIA SONET");
  chỉ ghép khi cụm đầu là chữ thuần (bỏ qua mã nhà máy như KA4). Query tránh lặp từ khi brand đã chứa model
  → từ khóa "KIA SONET 2024"
- Nguồn dữ liệu duy nhất: API Chợ Tốt `gateway.chotot.com/v1/public/ad-listing?cg=2010` (limit=50)
  - ⚠️ API này KHÔNG trả CORS header → browser không fetch trực tiếp được
  - `fetchJSONWithProxy()`: thử trực tiếp → **r.jina.ai** (proxy chính, có CORS, JSON nằm sau
    "Markdown Content:" → cắt từ `{` đầu đến `}` cuối) → allorigins (dự phòng)
  - 2 lượt query: "brand model year" → "brand model" (rộng hơn nếu lượt 1 <3 tin)
  - Lọc năm ±2: `adYear()` đọc field `year` hoặc trích từ tiêu đề tin rao
  - (Đã bỏ bonbanh/oto.com.vn/xegiatot — trang không chứa giá tĩnh, proxy cũ chết)
- **Công thức giá**: trung vị của tối đa 20 tin rao mới nhất sau khi lọc ngoại lệ IQR (`calcMarketPrice`) — bám sát giá đang rao tại thời điểm tra cứu
- **Giá tìm được LUÔN tự điền vào ô Số tiền bảo hiểm (`f_value`)** + toast thông báo
- UI: `#autoPriceStatus` → "Đã điền vào Số tiền BH: X ₫ · N xe · Dao động: Y–Z triệu"

### Điều khoản bổ sung
- Mỗi card 1 dòng: `[MÃ]  [tên điều khoản]  [+x.xx%]`
- **Tỷ lệ hiển thị trên card đổi theo tuổi xe** (`renderBS` đọc `f_year` → `bsEffective`);
  render lại khi sửa năm trong bảng xe (`syncVitField`), sau OCR (`fillFromOCR`) và khi Reset
- Xe ≥10 năm: BS05/BS26 hiện "Trình TCT" (vẫn click được nhưng `_calc` loại khỏi phí + cảnh báo)
- Không dùng checkbox — click card để toggle
- Khi chọn: card chìm xuống (`translateY(1px)` + `inset box-shadow` + viền 2px `--primary`)
- Badge đếm số điều khoản đã chọn hiển thị trên card-header

## Logic nghiệp vụ

### A. Phí gốc
```
Phí = (tỷ lệ cơ bản + tổng tỷ lệ BS) × giá trị xe
Tỷ lệ cơ bản = RATES[loại xe].r[nhóm tuổi][mức giá trị]
Tỷ lệ BS     = bsEffective(mã BS, tuổi xe) — BS02/BS05/BS26 tăng theo tuổi xe
```

### A2. Thời gian sử dụng xe (tuổi xe = năm hiện tại − năm sản xuất)
| Nhóm tuổi | Tỷ lệ cơ bản | BS02 | BS05 | BS26 |
|-----------|--------------|------|------|------|
| < 3 năm | RATES a03 | +0% | +0.45% | +0.13% |
| 3–6 năm | RATES a36 | +0.10% | +0.45% | +0.13% |
| 6–9 năm | RATES a610 | +0.20% | +0.50% | +0.15% |
| 9–10 năm | RATES a610 | +0.20% | +0.75% | +0.22% |
| 10–15 năm | RATES a1015 | +0.30% | Trình TCT | Trình TCT |
| > 15 năm | a1015 + cảnh báo thẩm định | | | |

- BS07 chỉ áp dụng xe mới 100% (tuổi < 1 năm), xe cũ bị loại + cảnh báo
- Xe KDVT/taxi (type 5/7/11/12/13 hoặc checkbox KDVT) ≥10 năm: ngoài biểu phí, cảnh báo trình TCT

### B. Tăng / Giảm phí
| Điều kiện | Hệ số |
|-----------|-------|
| Xe Mercedes | ×1.10 |
| Xe tải HINO (type 8) | ×1.05 |
| Doanh nghiệp / HCSN | ×0.90 |
| Đoàn 10–30 xe | ×0.90 |
| Đoàn 31–50 xe | ×0.85 |
| Đoàn > 50 xe | ×0.80 |
| Khấu trừ 1 triệu | ×0.95 |
| Khấu trừ 2 triệu | ×0.90 |
| Khấu trừ 5 triệu | ×0.675 |

### C. Phí sàn, làm tròn & VAT
```
Phí sàn = hệ số sàn (default 0.70) × phí gốc lần đầu
Phí cuối = max(phí tính, phí sàn), làm tròn đến nghìn đồng
Tổng thanh toán = phí cuối + VAT 10%
```
Kết quả hiển thị thêm: bảng so sánh phí theo 4 mức khấu trừ (giữ nguyên thông số khác).

### D. Tái tục
| TLBT | Hệ số |
|------|-------|
| < 30% | ×0.82 |
| 30–60% | ×0.95 |
| 60–100% | ×1.00 |
| > 100% | ×1.10–1.20 |
| Ngắt quãng > 12 tháng | Tính lại như xe lần đầu |

### E. Thời hạn
| Tháng | Tỷ lệ |
|-------|-------|
| 3 tháng | 40% |
| 6 tháng | 60% |
| 9 tháng | 75% |
| 12 tháng | 100% |
| 2–3 năm | ×0.90 |
| 3–5 năm | ×0.85 |

## Tra cứu giá xe
- Chỉ có **tra cứu tự động** (sau OCR hoặc nút 💹 "Tra giá trị xe thị trường"): `autoSearchPrice()` → API Chợ Tốt
  qua `fetchJSONWithProxy()` (trực tiếp → r.jina.ai → allorigins) → **luôn tự điền vào ô Số tiền bảo hiểm** + toast
- Bố cục card "Tra cứu giá trị xe": dữ liệu tra cứu (`#priceDataPanel`) ở trên → trạng thái (`#autoPriceStatus`) → nút tra giá ở dưới
- Đã xóa tra cứu thủ công (modal `priceModal`, `openPriceModal`, `checkPriceLookup`, `priceLookupTrigger`) — không tìm được giá thì nhập trực tiếp vào ô Số tiền bảo hiểm

## Phụ thuộc bên ngoài
- Chart.js: `https://cdn.jsdelivr.net/npm/chart.js` (tải `defer`)
- Tesseract.js v5: `https://unpkg.com/tesseract.js@5/dist/tesseract.min.js` (OCR — lazy-load qua `loadTesseract()` khi OCR lần đầu, không nằm trong `<head>`)
- Google Fonts: Sora + JetBrains Mono
- Không có backend, không có framework, không cần build

## Cách mở
Mở thẳng `index.html` trên trình duyệt. Có thể deploy lên Vercel / GitHub Pages.
