# Sổ tay AWS — tra cứu theo chủ đề

Bộ tài liệu sâu nhất trong repo này. Không đọc từ đầu đến cuối — mở đúng một
mục, đọc, đóng lại.

## Sổ tay này khác gì hai thứ kia

Bạn có ba tầng tài liệu. Đừng để chúng chồng lấn:

| | Trục | Trả lời câu hỏi | Cách đọc |
|---|---|---|---|
| [`../aws/`](../aws/) — 12 tuần | **thời gian** | "Tuần này học gì?" | một lần, theo thứ tự |
| **`notebook/`** — file này | **chủ đề** | "Cái này chạy thế nào? Chọn cái nào?" | nhảy vào giữa |
| [`../../aws-saa-c03/`](../../aws-saa-c03/) — submodule | nguồn | "Người khác tóm tắt ra sao?" | đối chiếu, **không sửa** |

Sổ tay sâu hơn phần 12 tuần, nhưng **không rộng hơn**. Vẫn đúng một kỷ luật ghi
trong [`../CONVENTIONS.md`](../CONVENTIONS.md): thà sâu 15 dịch vụ ra thi còn
hơn lướt 60 dịch vụ.

## Bản đồ 20 file

### Nền tảng

| File | Mở khi bạn cần |
|---|---|
| [`00-nen-tang.md`](00-nen-tang.md) | cái khung mà mọi dịch vụ nằm trong đó: Region/AZ, bốn ranh giới, shared responsibility, AWS là một tập API |

### Theo dịch vụ — "cái này chạy thế nào"

| File | Mở khi bạn cần |
|---|---|
| [`01-compute.md`](01-compute.md) | chọn EC2 / Lambda / container nào, cấu hình ra sao, con số nào phải nhớ |
| [`02-storage.md`](02-storage.md) | chọn S3 / EBS / EFS / FSx nào, lớp lưu trữ nào, tiền chảy đi đâu |
| [`03-database.md`](03-database.md) | bài toán lưu trữ có cấu trúc — engine nào, bật tính năng nào |
| [`04-networking.md`](04-networking.md) | gói tin đi từ đâu tới đâu, ai chặn nó ở chặng nào |
| [`05-security.md`](05-security.md) | ai được làm gì, quyền tính ra sao, mã hoá bằng khoá nào |
| [`06-tich-hop.md`](06-tich-hop.md) | hai thành phần cần nói chuyện — hàng đợi, pub/sub, bus, orchestrator hay API |
| [`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) | bốn dịch vụ quan sát khác nhau chỗ nào, vào máy bằng đường nào khi không có SSH |
| [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md) | dữ liệu đi từ nguồn tới biểu đồ qua sáu chặng — ingest, lưu, ETL, truy vấn |
| [`09-ai-ml-media.md`](09-ai-ml-media.md) | tám dịch vụ AI dựng sẵn, SageMaker, media — nhận vào gì trả ra gì, đồng bộ hay không |
| [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) | hạ tầng lai và biên: Outposts, Storage Gateway, Directory Service, và ràng buộc ngoài kỹ thuật |

### Theo bài toán — "chọn cái nào cho tình huống này"

Bốn file này ứng với bốn miền thi. Đây là **phần nguồn thiếu hẳn** và được viết mới.

| File | Miền | Mở khi đề có chữ |
|---|---|---|
| [`10-chi-phi.md`](10-chi-phi.md) | Cost **20%** | `cost-effective`, `minimize cost`, `most economical` |
| [`11-san-sang-cao.md`](11-san-sang-cao.md) | Resilient **26%** | `highly available`, `fault-tolerant`, `minimal downtime` |
| [`12-hieu-nang.md`](12-hieu-nang.md) | High-Performing **24%** | `lowest latency`, `improve performance`, `scale to` |
| [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) | Resilient | `RTO`, `RPO`, `disaster recovery`, `migrate` |

Miền nặng nhất — Secure **30%** — nằm ở [`05-security.md`](05-security.md).

### Tra nhanh — dùng khi đang làm đề

| File | Mở khi |
|---|---|
| [`20-cay-quyet-dinh.md`](20-cay-quyet-dinh.md) | biết bài toán thuộc nhóm nào, cần một chuỗi câu hỏi để chốt dịch vụ |
| [`21-tu-khoa-de-thi.md`](21-tu-khoa-de-thi.md) | cần giải mã thứ tiếng Anh có mã của đề, gồm cả **bẫy từ khoá** |
| [`22-bang-so-sanh.md`](22-bang-so-sanh.md) | còn đúng hai đáp án, cần một dòng để chốt |
| [`23-de-cuong-chinh-thuc.md`](23-de-cuong-chinh-thuc.md) | AWS tuyên bố hỏi gì: 14 task statement, 118 service in-scope, danh sách ngoài phạm vi |

Cột cuối mỗi bảng trong `22` là **"đề thi phân biệt bằng từ nào"** — đó là thứ
làm nó khác một bảng thuộc tính khô khan.

### Thiết kế — "tự nghĩ ra kiến trúc, không chỉ chọn đáp án"

| File | Mở khi |
|---|---|
| [`30-thiet-ke-he-thong.md`](30-thiet-ke-he-thong.md) | có một đề bài nghiệp vụ mơ hồ và phải ra kiến trúc: bảy bước, bốn kiến trúc tham chiếu, năm đề thiết kế mở |

Đây là chương khác biệt nhất trong sổ tay. Mọi chương khác trả lời *"cái này là
gì"*; chương này trả lời *"làm sao biết mình đang hỏi đúng câu hỏi"*. Bốn miền thi
đều bắt đầu bằng động từ **Design** — xem
[`23-de-cuong-chinh-thuc.md`](23-de-cuong-chinh-thuc.md) mục 5.

## Ba lối vào

**Đang học theo tuần.** Đọc [`../aws/wXX-*.md`](../aws/) trước. Gặp chỗ muốn
đào sâu thì mở file chủ đề tương ứng ở đây. Đừng đọc sổ tay thay cho phần tuần —
nó không có thứ tự sư phạm.

**Đang làm lab.** Mở file chủ đề của lab đó. Mục **Nối với thực hành** ở cuối
mỗi file trỏ thẳng tới lab tương ứng, cả bản có lời giải lẫn bản tự viết.

**Đang làm đề, có 90 giây.** Chỉ ba file: `21` để giải mã từ khoá → `20` để chốt
nhánh → `22` để phân biệt hai đáp án cuối.

## Quan hệ với submodule nguồn

Sổ tay dựng trên [`aws-saa-c03/`](../../aws-saa-c03/) của vntechies. Nguồn là
cheat sheet: đúng nhưng nông. Sổ tay giữ cấu trúc chủ đề đó rồi thêm **cơ chế**,
**con số**, và **bẫy** — chỗ nào chỉ chép lại được thì đã cắt.

Hai điều cần biết về nguồn:

**Nó rỗng cho tới khi bạn init.** Submodule chỉ lưu con trỏ tới một commit,
không lưu nội dung:

```bash
git submodule update --init --recursive --depth 1
```

**README của nó liệt kê 7 file không tồn tại** — F, G, H, I, J, N, O. Tức là
toàn bộ phần "giải bài toán theo domain" bị mất, mà đó chính là cách SAA-C03 ra
đề. Bốn file [`10`](10-chi-phi.md)–[`13`](13-khoi-phuc-tham-hoa.md) là phần
viết mới để lấp chỗ đó.

Chỗ nào nguồn sai hoặc cũ, file tương ứng có mục **Nguồn nói khác** kèm link docs
chính thức. Nếu bạn đọc nguồn trước rồi đọc sổ tay sau và thấy hai bên vênh
nhau — đọc mục đó.

## Dùng cùng lab

| | Bản chất | Khi nào |
|---|---|---|
| [`learn-aws/labs/`](../../learn-aws/labs/) | Terraform viết sẵn, chạy là xong | học cách đọc code hạ tầng |
| [`learn-aws/labs-self/`](../../learn-aws/labs-self/) | **đề bài**, bạn tự viết từ file trống | học cách nghĩ ra nó |

Thứ tự dùng mạnh nhất: đọc lý thuyết → **tự viết** ở `labs-self/` → verify xanh →
đọc `DOI-CHIEU.md` → rồi mở `labs/` xem người khác giải cách nào. Bước cuối
thường dạy nhiều hơn cả bước tự viết.

`labs-self/` có hàng rào an toàn ba tầng (permission boundary + ngân sách + kỷ
luật destroy). Xem [`labs-self/_boundary/README.md`](../../learn-aws/labs-self/_boundary/README.md) —
bản thân cái hàng rào đó là bài học Domain 1.

## Hai tuần cuối

Ngừng đọc file chủ đề. Chỉ còn ba việc:

1. `21` → `20` → `22`, lặp cho tới khi phản xạ
2. Làm đề, mỗi câu sai thì mở đúng file chủ đề của nó, đọc mục **Bẫy đề thi**
3. Mục **Tự kiểm tra** cuối mỗi file — câu hỏi buộc giải thích, không phải nhớ tên
