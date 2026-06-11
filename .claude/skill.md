# PTI Insurance Calculator PRO — Project Skills

## Tổng quan dự án
Mini web app tính phí bảo hiểm vật chất xe ô tô (VCX) theo quy tắc biểu phí PTI.
Toàn bộ logic nằm trong **một file duy nhất**: `index.html`.

---

## Skill 1: Cập nhật bảng tỷ lệ phí (RATES)

**Khi nào dùng:** PTI ban hành biểu phí mới, cần cập nhật tỷ lệ % các loại xe.

**Vị trí trong code:** Đối tượng `RATES` trong `<script>` của `index.html`.

**Cấu trúc:**
```js
RATES = {
  [loại xe 1-14]: {
    n: 'Tên loại xe',
    r: {
      a03:   [rate_<200M, 200-300M, 300-400M, 400-500M, 500-700M, 700M-1B, >1B],  // dưới 3 năm
      a36:   [...],  // 3–6 năm
      a610:  [...],  // 6–10 năm
      a1015: [...],  // 10–15 năm
    }
  }
}
```

**Lưu ý:** Tỷ lệ tính theo **% của giá trị xe**. Ví dụ: `1.42` = 1.42%.

---

## Skill 2: Thêm điều khoản bổ sung (BS)

**Khi nào dùng:** Có BS mới hoặc thay đổi tỷ lệ BS hiện tại.

**Vị trí:** Mảng `BS[]` trong `<script>`.

```js
BS = [
  { c: 'BS##', n: 'Tên điều khoản', r: 0.XX },  // r = tỷ lệ % thêm vào
  ...
]
```

---

## Skill 3: Điều chỉnh logic tăng/giảm phí

**Vị trí:** Hàm `_calc()` — phần "Adjustments".

**Các điều chỉnh hiện tại:**
| Điều kiện | Hệ số | Biến trong code |
|-----------|-------|-----------------|
| Mercedes | ×1.10 | `brand.includes('mercedes')` |
| HINO (type 8) | ×1.05 | `typeId===8 && brand.includes('hino')` |
| Doanh nghiệp | ×0.90 | `ctype==='business'` |
| HCSN | ×0.90 | `ctype==='hcsn'` |
| Đoàn 10–30 xe | ×0.90 | `count>=10 && count<=30` |
| Đoàn 31–50 xe | ×0.85 | `count>30 && count<=50` |
| Đoàn >50 xe | ×0.80 | `count>50` |
| Khấu trừ 1M | ×0.95 | `deduct===1` |
| Khấu trừ 2M | ×0.90 | `deduct===2` |
| Khấu trừ 5M | ×0.675 | `deduct===5` |

**Từ v1.3.0, `_calc()` còn có:**
- Khấu trừ tách hệ số riêng (`DEDUCT_OPTS` + `factorExclDeduct`) để `feeForDeduct()` tính bảng
  so sánh 4 mức khấu trừ (`#r_deduct_compare`)
- Phí cuối **làm tròn nghìn đồng**, sau đó cộng **VAT 10%** (`vat`, `feeWithVat`)
- Mảng `warns[]` — cảnh báo nghiệp vụ: xe >15 năm, KDVT lệch loại xe, STBH ≥5 tỷ
- `window._last` lưu thêm `baseFee, durFee, vat, feeWithVat` (chart đổi theme + CSV cần)

---

## Skill 4: Thêm loại xe mới

1. Thêm `<option value="15">` vào `<select id="f_type">`
2. Thêm entry `15: { n:'...', r:{ a03:[...], a36:[...], a610:[...], a1015:[...] }}` vào `RATES`

---

## Skill 5: Thêm pattern phiên bản xe mới

**Vị trí:** Mảng `patterns[]` trong hàm `detectVersion()`.

```js
{ brand:'TOYOTA', re:/CROSS.*AWD/i, ver:'Corolla Cross 1.8 AWD' }
```

- `brand`: tên hãng in hoa
- `re`: regex khớp với model code đầy đủ hoặc số khung
- `ver`: chuỗi phiên bản hiển thị cho người dùng

---

## Skill 6: Điều chỉnh logic tìm giá tự động

**Vị trí:** Hàm `autoSearchPrice(brand, model, year, version)` + `fetchJSONWithProxy(url, onStatus)`.

- Nguồn dữ liệu duy nhất: `https://gateway.chotot.com/v1/public/ad-listing?cg=2010&q=...&limit=50`
- ⚠️ **API Chợ Tốt KHÔNG trả CORS header** — browser không fetch trực tiếp được.
  `fetchJSONWithProxy()` xử lý theo chuỗi: trực tiếp (7s) → **r.jina.ai** (proxy chính) → allorigins (dự phòng)
- r.jina.ai bọc JSON sau dòng `Markdown Content:` → parse bằng cách cắt từ `{` đầu đến `}` cuối
- 2 lượt query: `"{brand} {model} {year}"` → nếu <3 tin thì `"{brand} {model}"` (rộng hơn)
- Field `ads[].year` thường rỗng → `adYear()` trích năm từ tiêu đề tin (`subject`), lọc ±2 năm
- Giá lọc: `price_min=100000000` (200M nếu >9 chỗ), `price_max=8000000000`
- Công thức: trung bình (min+max)/2 sau khi cắt 10% ngoại lệ hai đầu (`calcIQRMedian`)
- **Giá tìm được LUÔN tự điền vào ô Số tiền bảo hiểm (`f_value`)** trong `showPriceResult()` + toast
- Khi thêm proxy mới: chèn vào giữa chuỗi trong `fetchJSONWithProxy`, giữ timeout hợp lý (15–20s)
- **Bố cục card "Tra cứu giá trị xe"** (từ v1.4.0): `#priceDataPanel` (dữ liệu) trên →
  `#autoPriceStatus` (trạng thái) → nút `#btnTraGia` full-width dưới cùng
- **Tra cứu thủ công đã xóa** (modal `priceModal`, `openPriceModal`, `checkPriceLookup`,
  `priceLookupTrigger`) — không tìm được giá thì người dùng nhập tay vào ô Số tiền bảo hiểm

---

## Skill 7: Tinh chỉnh OCR không cần API key (ocrSmart)

**Vị trí:** `ocrSmart()`, `preprocessSoft()`, `preprocessBinary()`, `scoreExtraction()`, `vinValid()`, `textFromWords()`.

**Kiến trúc best-of-2 passes:**
1. Lượt 1 — `preprocessSoft`: grayscale **giảm trọng số kênh đỏ** (0.15R + 0.55G + 0.30B — làm mờ
   dấu mộc đỏ trên giấy đăng ký) + kéo giãn tương phản percentile 2–98%, phóng to ≥2000px
2. Lượt 2 — `preprocessBinary` (Otsu): chỉ chạy nếu lượt 1 chưa đạt `GOOD_ENOUGH = 9` điểm
3. Mỗi lượt parse 2 bản: văn bản lọc confidence ≥30 (`textFromWords`, ưu tiên) + bản thô (bù trường thiếu)
4. `scoreExtraction()`: trọng số trường (plate/chassis = 2, owner = 1.5...) + bonus 2đ nếu VIN qua
   checksum ISO 3779 + 1đ nếu biển số đúng chuẩn `00X-000.00`
5. Merge giữa lượt/ảnh: số khung ưu tiên bản qua `vinValid()` (chỉ là tín hiệu DƯƠNG — VIN châu Á
   không bắt buộc check digit, không dùng để loại)

**Tinh chỉnh thường gặp:**
- OCR sót trường → giảm ngưỡng confidence trong `textFromWords` (30 → 20)
- Chạy chậm → giảm `minDim` trong `loadScaledGray` (2000 → 1600)
- Lượt 2 chạy quá thường xuyên → hạ `GOOD_ENOUGH` (9 → 7)

---

## Quy tắc khi chỉnh sửa dự án

- **Không** tách CSS/JS ra file riêng — giữ nguyên single-file
- **Không** thêm framework (React, Vue...) — dùng vanilla JS
- **Không** thêm backend — tất cả chạy client-side
- **Không** có nút Demo / dữ liệu mẫu cứng trong UI — đã bỏ từ v1.2.0
- **Không dùng `alert()`** — dùng `toast(msg, 'ok'|'warn'|'err')` (từ v1.3.0); `confirm()` vẫn dùng cho hành động xóa
- OCR mặc định là `ocrSmart` (không cần API key) — Claude Vision chỉ là tùy chọn khi có key
- Khi cập nhật Chart.js, dùng CDN: `https://cdn.jsdelivr.net/npm/chart.js`
- OCR dùng Tesseract.js v5: `https://unpkg.com/tesseract.js@5/dist/tesseract.min.js`
- Test bằng cách mở thẳng `index.html` trên trình duyệt
- Upload nhiều ảnh cùng lúc để test OCR multi-file merge
- Kiểm tra cú pháp JS sau khi sửa: tách phần `<script>` cuối ra file tạm rồi chạy `node --check`
