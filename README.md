# kiem-tra-file — tool rà soát file biểu mẫu

Trả lời 3 câu hỏi cho từng dòng biểu mẫu trong bảng rà soát Excel:

1. **Tải xuống có vấn đề không?** — link còn sống, tải được, không bị đòi đăng nhập.
2. **Mở lên có chết file không?** — file mở được bằng Word/Excel/PDF, không hỏng, không rỗng.
3. **Mở lên có nhầm file không?** — file Google trỏ tới có đúng là mẫu của dòng đó.

## Chạy

Nhận cả file trên máy và link Google Sheet. Với link, tool tự tải bản mới nhất và
lưu thành `bang_ra_soat.xlsx` trong thư mục `--ra`, nên chạy lại là luôn lấy bản mới.

```bash
python -m kiem_tra_file.cli "D:\duong\dan\bieu mau gtel nho ra soat.xlsx"
```

```bash
python -m kiem_tra_file.cli "https://docs.google.com/spreadsheets/d/<id>/edit"
```

Mặc định kiểm 4 sheet: `DÂN SỰ ĐÃ RÀ SOÁT`, `HÔN NHÂN GIA ĐÌNH ĐÃ RÀ SOÁT`,
`KINH DOANH THƯƠNG MẠI ĐÃ RÀ SOÁ`, `LAO ĐỘNG ĐÃ RÀ SOÁT`.

| Tham số | Ý nghĩa |
|---|---|
| `--sheet "HÀNH CHÍNH"` | chọn sheet khác (lặp lại được để kiểm nhiều sheet) |
| `--limit 20` | chỉ kiểm 20 dòng đầu, dùng để thử nhanh |
| `--luong 6` | số file tải song song (mặc định 6) |
| `--chi-loi` | báo cáo chỉ gồm dòng có lỗi |
| `--ra ket_qua` | thư mục xuất `bao_cao.csv` + `bao_cao.html` |

Kết thúc trả mã lỗi `1` nếu có dòng mức `LOI`, nên cắm được vào CI.

## Các mã lỗi

**Mức LOI — phải sửa**

| Mã | Nghĩa |
|---|---|
| `THIEU_LINK` | ô link trống, hoặc không phải link Google Docs/Drive |
| `TAI_LOI` | HTTP 404/5xx, timeout — file đã xoá hoặc link sai |
| `KHONG_CO_QUYEN` | HTTP 403 — file không được chia sẻ |
| `CAN_DANG_NHAP` | bị chuyển sang trang đăng nhập Google |
| `FILE_RONG` | tải về 0 byte |
| `KHONG_PHAI_FILE` | Google trả về trang HTML/JSON thay vì tệp |
| `CHET_FILE` | tệp không mở được (zip hỏng, XML sai, PDF lỗi) |
| `LECH_DINH_DANG` | đuôi file không khớp nội dung thật |
| `NHAM_FILE` | tên file Google khác **số hiệu mẫu** của dòng (80-DS trỏ file 81-DS) |

**Mức CANH_BAO — cần người xem lại**

| Mã | Nghĩa |
|---|---|
| `KHAC_PHAN_HE` | cùng số hiệu nhưng file thuộc phân hệ khác (93-HNGD dùng file 93-DS) |
| `KHAC_HAU_TO` | cùng số hiệu, cùng phân hệ, khác hậu tố nghị quyết |
| `MA_MOI_KHONG_HOP_LE` | cột *mã mới* không phải mã (ghi "Bỏ", ghi cả câu tên) |
| `KHOP_YEU` | tên file khớp một phần tên biểu mẫu (40–80%) |
| `TRUNG_LINK` | nhiều mã biểu mẫu khác nhau trỏ cùng một file |
| `NOI_DUNG_KHAC` | chữ trong file không giống tên biểu mẫu |
| `KHONG_DOC_DUOC_CHU` | file không có chữ nào để đối chiếu (ảnh scan?) |

## Cách tool kết luận "nhầm file"

Biểu mẫu toà án là **mẫu trắng**: tiêu đề trong file bị thay bằng `..........`,
nên không thể chỉ dựa vào chữ trong file. Bằng chứng chính là **tên file thật**
Google trả về ở header `Content-Disposition`:

- Google Docs thường trả về chính mã biểu mẫu mới, ví dụ `01-DS.TC01-2026.docx`
  → so với cột *Gia Nam - Mã biểu mẫu mới*, rồi tới *MÃ BIỂU MẪU*.
- Drive trả về tên biểu mẫu → so theo từ khoá tên (bỏ dấu, bỏ mã trong ngoặc).

Chỉ khi không lấy được tên file thì tool mới đối chiếu chữ trong file, và khi đó
chênh lệch chỉ là `NOI_DUNG_KHAC` ở mức cảnh báo.

Mã biểu mẫu được tách thành **số hiệu – phân hệ – hậu tố** (`45A` / `DS` /
`NQ01-2017`) rồi so từng phần, nên phân biệt được lệch số hiệu (nhầm file thật)
với dùng chung mẫu của phân hệ khác (chỉ cần biết). Hậu tố bản copy Drive tự
thêm (`_2`, ` (1)`) không bị tính là lệch.

## Chạy test

```bash
python -m pytest tests/
```

100 test, tự sinh file mẫu (docx hợp lệ / docx hỏng / pdf / file rỗng), không cần mạng.

## Cấu trúc

| File | Việc |
|---|---|
| `nguon_excel.py` | đọc bảng rà soát (file hoặc link Sheet), lấy mã/tên/link, bỏ dòng ghi chú |
| `tai_xuong.py` | dựng URL tải, tải có thử lại, đọc tên file từ header |
| `kiem_tra.py` | nhận dạng định dạng thật, mở thử file để bắt file hỏng |
| `doi_chieu.py` | so tên file & chữ trong file với mã/tên biểu mẫu, bắt trùng link |
| `chay.py` | ghép các phép kiểm, mỗi file chỉ tải một lần |
| `bao_cao.py` | xuất CSV (dán lại vào sheet) + HTML (xem nhanh) |
| `cli.py` | tham số dòng lệnh |
