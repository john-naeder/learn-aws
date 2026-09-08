---
ngày: 2026-09-08
chủ đề: Đọc đề cương chính thức, lấp 5 chương sổ tay, và chuyển trọng tâm từ luyện thi sang thiết kế
liên quan: docs/notebook/, docs/solutions-architect-associate-03.pdf
---

## Câu hỏi đã đặt ra

1. Lý thuyết cũ (`docs/aws/w00`, `w01`) đã đọc rồi — có gì thay đổi không, có cần đọc lại không?
2. Thực hành được chưa?
3. Dựa vào các domain và service trong PDF đề cương, chi tiết hoá từng service hoạt động thế nào.
4. Sửa theo hướng **hiểu và thiết kế được**, không chỉ tập trung vào việc thi.

## Chốt lại được gì

**Lý thuyết cũ không sai, không cần đọc lại.** Đã đối chiếu từng con số: managed
policy gắn vào role **20** (không phải 10), MFA root **35 ngày bắt buộc**, S3
**strong read-after-write từ 12/2020**, và luồng đánh giá quyền có đủ chỗ tinh
(resource-based policy trỏ thẳng user ARN cắt ngang implicit deny). Sổ tay không
phủ nhận bản cũ — nó đào sâu và mở rộng.

**Đề cương chính thức là nguồn chuẩn, và nó nói nhiều hơn danh sách service.**
Bóc PDF ra được 40.994 ký tự (máy không có `pdftotext`, phải dựng venv với `pypdf`).
Bốn thứ quan trọng rút ra:

| Phát hiện | Hệ quả |
|---|---|
| **Task 3.5 "data ingestion and transformation"** là task statement đầy đủ, ngang hàng compute/database | sổ tay thiếu hẳn → viết `08-phan-tich-du-lieu.md` |
| **Lightsail nằm trong out-of-scope** | `01-compute.md` §11 đang dạy nó như nội dung chính → gắn nhãn "phương án nhiễu" |
| Cả họ **Elemental ngoài phạm vi**, nhưng **Elastic Transcoder in-scope** | chỗ hiếm hoi *đáp án đúng của đề* khác *lựa chọn đúng của kiến trúc thật* |
| **Không có điểm liệt theo miền** — *"You need to pass only the overall exam"* | rất nhiều nguồn dạy ngược lại |

**Cả bốn miền đều bắt đầu bằng động từ "Design".** Không miền nào tên là Operate
hay Troubleshoot. Và trong mục "Skills in" của cả 14 task statement, ba động từ
chiếm đa số: *Designing, Determining, Selecting*. Đó là lý do sổ tay thiên về
"Bẫy đề thi" là thiên sai hướng, và là lý do có chương `30-thiet-ke-he-thong.md`.

**Hai mức độ trong đề cương phải đọc khác nhau:** "Knowledge of" = nhận ra và
giải thích được (ôn bằng đọc); "Skills in" = ra được quyết định trong tình huống
mới (chỉ ôn được bằng **làm**).

**Năm chương mới, 5.510 dòng:** `08` phân tích dữ liệu · `09` AI/ML/media ·
`14` hybrid và biên · `23` đề cương chính thức · `30` thiết kế hệ thống.
Sổ tay từ 15 lên **20 file**.

## Chỗ tôi hiểu sai

**Tưởng remote GitHub đã trống nên `--force-with-lease` sẽ chạy.** Nó báo
"stale info" ba lần. Nguyên nhân: `git ls-remote` trả về **rỗng** — repo trống
thật, còn `origin/main` ở máy chỉ là **ref cũ sót lại** từ trước khi xoá repo.
Lease so với một thứ không còn tồn tại nên luôn thất bại. Sửa bằng
`git fetch --prune` rồi push thường.

**Suýt xoá mất PDF đề cương.** Nó có trên remote cũ nhưng **không có trong thư
mục làm việc**. Force push sẽ xoá vĩnh viễn. Bắt được nhờ so
`git ls-tree origin/main` với commit mới trước khi đẩy. Bài học: **trước khi
force push, luôn diff cây file remote với cây file sắp đẩy** — force push không
chỉ ghi đè lịch sử, nó ghi đè cả những file bạn không biết là mình có.

**Viết "130 phút" vào bảng số theo trí nhớ.** Kiểm lại thì PDF **không hề nêu
thời lượng**. Cả chương đặt tiền đề "mọi con số trích thẳng từ PDF" nên lẫn kiến
thức ngoài vào là làm hỏng chính tiền đề đó. Đã đổi thành "đề cương này không nêu".

## Còn treo

- Chưa có lab cho **task 3.2 (compute hiệu năng cao)** và **task 3.5 (nạp và biến
  đổi dữ liệu)**. Lý do thật: Kinesis, Glue, Redshift đều tính tiền theo giờ hoặc
  theo shard, không lọt hàng rào **$0/giờ** của `labs-self/`. Athena là ngoại lệ
  rẻ nhất nếu muốn thử.
- Vẫn chưa chạy `verify.sh` thật lần nào — 12 lab mới chỉ được kiểm **tĩnh**.
- Chưa áp `_boundary/`, chưa xác nhận mail Budgets, chưa tạo profile `lab-builder`.
- Bắt đầu tuần 1 (IAM) sau khi `aws sts get-caller-identity` trả về ARN dạng user.
