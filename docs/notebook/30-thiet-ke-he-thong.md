# Thiết kế hệ thống — từ mô tả nghiệp vụ tới kiến trúc có lý do

> **Tra nhanh:** bạn có một đoạn văn mô tả nghiệp vụ và một trang giấy trắng.
> File này cho bạn quy trình bảy bước biến đoạn văn đó thành kiến trúc có con số,
> có ranh giới hỏng, có hoá đơn ước tính, và có cách kiểm chứng.

`Domain 1 · Secure (30%)` · `Domain 2 · Resilient (26%)` · `Domain 3 · High-Performing (24%)` · `Domain 4 · Cost-Optimized (20%)`

Chương này chạm cả bốn miền vì thiết kế là chỗ duy nhất bốn miền gặp nhau. Một
kiến trúc chỉ tối ưu một miền là kiến trúc sai — rẻ nhất thường là kém sẵn sàng
nhất, nhanh nhất thường là đắt nhất.

## Chương này đứng ở đâu trong sổ tay

| File | Trả lời câu hỏi | Giả định ngầm |
|---|---|---|
| [`21-tu-khoa-de-thi.md`](21-tu-khoa-de-thi.md) | "đề đang nói về cái gì" | bạn đã có đề |
| [`20-cay-quyet-dinh.md`](20-cay-quyet-dinh.md) | "chọn cái nào trong ba cái" | **ai đó đã liệt kê sẵn ba cái** |
| [`22-bang-so-sanh.md`](22-bang-so-sanh.md) | "còn hai đáp án, chốt cái nào" | **ai đó đã loại xuống còn hai** |
| **file này** | "tôi đang hỏi đúng câu hỏi chưa" | không ai liệt kê gì cả |

Ba file kia giả định danh sách lựa chọn đã có sẵn. Ngoài phòng thi không ai đưa
bạn bốn phương án. Bạn nhận một đoạn văn kiểu *"chúng tôi cần một hệ thống đặt
lịch cho 40 phòng khám, phải nhanh, không được mất dữ liệu, và rẻ"* — rồi tự
sinh ra danh sách lựa chọn.

Đó là kỹ năng file này dạy. Nó cũng là kỹ năng đề SAA thật sự kiểm tra: đề cho
bốn đáp án **đều chạy được**, và câu hỏi luôn là "cái nào TỐT NHẤT cho ràng buộc
này". Muốn trả lời được, bạn phải đọc ra được ràng buộc — không phải nhớ dịch vụ.

---

## Bản đồ

| Mục | Đọc khi bạn cần |
|---|---|
| [Phần 1 — Phương pháp bảy bước](#phan-1--phuong-phap-bay-buoc) | có một đề bài mơ hồ, không biết bắt đầu từ đâu |
| [Bước 1. Rút ràng buộc ra khỏi văn xuôi](#buoc-1-rut-rang-buoc-ra-khoi-van-xuoi) | đề toàn tính từ: "nhanh", "rẻ", "an toàn" |
| [Bước 2. Xác định trục mở rộng](#buoc-2-xac-dinh-truc-mo-rong) | biết hệ thống sẽ "lớn lên" nhưng chưa biết lớn theo chiều nào |
| [Bước 3. Chọn mô hình dữ liệu trước](#buoc-3-chon-mo-hinh-du-lieu-truoc-compute-sau) | đang định chọn EC2 hay Lambda trước khi chọn database |
| [Bước 4. Vẽ đường đi của một request](#buoc-4-ve-duong-di-cua-mot-request) | cần biết độ trễ và hoá đơn đến từ chặng nào |
| [Bước 5. Đặt ranh giới hỏng](#buoc-5-dat-ranh-gioi-hong) | cần trả lời "cái gì hỏng thì cái gì còn sống" |
| [Bước 6. Tính tiền trước khi dựng](#buoc-6-tinh-tien-truoc-khi-dung) | cần một con số $/tháng để mang đi thuyết phục |
| [Bước 7. Định nghĩa cách biết nó hỏng](#buoc-7-dinh-nghia-cach-biet-no-hong) | thiết kế trông đã xong nhưng chưa có alarm nào |
| [Chạy cả bảy bước trên một đề](#chay-ca-bay-buoc-tren-mot-de) | muốn xem quy trình chạy thật một lần |
| [Phần 2 — Bốn kiến trúc tham chiếu](#phan-2--bon-kien-truc-tham-chieu) | cần một điểm xuất phát thay vì trang trắng |
| [A. Web ba tầng sẵn sàng cao](#a-web-ba-tang-san-sang-cao) | ứng dụng có trạng thái, dữ liệu quan hệ, tải đều |
| [B. API serverless hướng sự kiện](#b-api-serverless-huong-su-kien) | tải gai, đội nhỏ, không muốn trực đêm |
| [C. Xử lý dữ liệu theo lô và theo luồng](#c-xu-ly-du-lieu-theo-lo-va-theo-luong) | log, clickstream, IoT, báo cáo |
| [D. Nội dung tĩnh và động toàn cầu](#d-noi-dung-tinh-va-dong-toan-cau) | người dùng ở nhiều châu lục |
| [Phần 3 — Đọc đề bài như một kiến trúc sư](#phan-3--doc-de-bai-nhu-mot-kien-truc-su) | khách hàng nói một đằng, ràng buộc thật một nẻo |
| [Phần 4 — Sai lầm thiết kế điển hình](#phan-4--sai-lam-thiet-ke-dien-hinh) | muốn biết mình đang mắc lỗi nào phổ biến |

Bốn chương theo bài toán là chỗ đào sâu từng trục: [`10-chi-phi.md`](10-chi-phi.md),
[`11-san-sang-cao.md`](11-san-sang-cao.md), [`12-hieu-nang.md`](12-hieu-nang.md),
[`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md). File này không chép lại
chúng — nó dạy thứ tự áp dụng chúng.

---

## Phần 1 — Phương pháp bảy bước

Bảy bước này chạy theo đúng thứ tự. Đảo thứ tự là nguồn gốc của phần lớn kiến
trúc sai: chọn compute trước dữ liệu, vẽ sơ đồ trước khi có con số, dựng xong
mới tính tiền.

| Bước | Câu hỏi trung tâm | Đầu ra cụ thể |
|---|---|---|
| 1 | Ràng buộc nào là con số, ràng buộc nào là mong muốn? | bảng ràng buộc, mỗi dòng một con số và một nguồn |
| 2 | Trong 18 tháng tới, con số nào nhân 10? | tên **một** trục chính + hai trục phụ |
| 3 | Mẫu truy cập dữ liệu là gì? | tên engine + schema key + 10 câu truy vấn thật |
| 4 | Một request đi qua bao nhiêu chặng? | bảng chặng + ngân sách độ trễ cộng lại bằng p99 |
| 5 | Cái gì hỏng thì cái gì còn sống? | sơ đồ có ranh giới AZ/Region/account vẽ rõ |
| 6 | Hoá đơn ở 0% tải là bao nhiêu? | bảng $/tháng ở ba mức tải |
| 7 | Ai bị gọi lúc 3 giờ sáng, và họ làm gì? | bảng alarm + runbook một dòng mỗi alarm |

Một thiết kế thiếu bước 6 là thiết kế chưa được duyệt. Một thiết kế thiếu bước 7
là thiết kế chưa xong — nó chỉ chạy được, chưa vận hành được.

---

### Bước 1. Rút ràng buộc ra khỏi văn xuôi

Mục tiêu: biến mọi tính từ thành số. Ràng buộc không có số thì chưa phải ràng
buộc — nó là mong muốn, và mong muốn không loại được đáp án nào.

#### Bảng dịch tính từ sang số

| Khách hàng viết | Ràng buộc đo được | Hỏi thêm câu này để lấy số |
|---|---|---|
| "phải nhanh" | p99 của `GET /feed` dưới **300 ms**, đo ở client | "Chậm bao nhiêu thì người dùng bỏ đi? Đo ở máy họ hay ở server?" |
| "không được mất dữ liệu" | **RPO = 0** cho bảng `orders`, RPO 15 phút cho `analytics` | "Mất 5 phút dữ liệu cuối thì thiệt hại tính bằng tiền là bao nhiêu?" |
| "phải luôn sẵn sàng" | **99,9%/tháng** = 43,2 phút chết mỗi tháng | "Đo trên endpoint nào? Deploy có tính là downtime không?" |
| "khôi phục nhanh" | **RTO ≤ 4 giờ** | "Bốn giờ tính từ lúc nào — lúc hỏng hay lúc có người phát hiện?" |
| "rẻ" | trần **$2.000/tháng** ở 100.000 MAU | "Ngân sách hạ tầng năm nay là bao nhiêu, ai duyệt vượt?" |
| "nhiều người dùng" | **5.000 phiên đồng thời**, đỉnh 11:00–13:00 | "Đỉnh gấp mấy lần trung bình? Đỉnh dự đoán được không?" |
| "phải an toàn" | dữ liệu mã hoá at rest bằng **CMK riêng**, audit trail giữ 1 năm | "Ai kiểm toán, theo chuẩn nào, họ hỏi gì?" |
| "co giãn tự động" | từ 4 lên **40 instance trong 5 phút** | "Tải tăng nhanh cỡ nào — 5 phút hay 5 giây?" |

Chú ý cột giữa: mỗi ràng buộc có **phạm vi**. "RPO = 0" cho toàn hệ thống là đắt
gấp nhiều lần "RPO = 0 cho đúng bảng đơn hàng". Kiến trúc sư giỏi thu hẹp phạm vi
ràng buộc trước khi thu hẹp lựa chọn kỹ thuật.

#### Ràng buộc ẩn — thứ không ai viết ra

Sáu loại dưới đây không bao giờ nằm trong bản mô tả yêu cầu, nhưng chúng loại
đáp án mạnh hơn mọi ràng buộc kỹ thuật.

| Loại | Dấu hiệu | Hệ quả kiến trúc |
|---|---|---|
| **Chủ quyền dữ liệu** | khách hàng ở EU, ngành y tế, ngân hàng | dữ liệu không rời `eu-*`; backup cross-Region **bị cấm**, không phải "nên có"; loại Aurora Global Database nếu nó sao chép sang Region ngoài khối |
| **Giấy phép** | Oracle, SQL Server, phần mềm tính theo socket | BYOL cần **Dedicated Host** (nhìn thấy socket vật lý), không phải Dedicated Instance; có thể làm EC2 rẻ hơn RDS dù RDS ít việc hơn |
| **Kỹ năng đội** | 3 kỹ sư, không ai từng vận hành cluster production | EKS là nợ kỹ thuật, không phải tài sản; Fargate hoặc Lambda trả lại thời gian |
| **Hệ thống cũ không sửa được** | app ghi vào đường dẫn NFS cứng trong mã nguồn | **EFS**, không phải S3 — dù S3 rẻ hơn 4 lần. Đổi mã nguồn là dự án riêng |
| **Lịch** | "phải live trước mùa cao điểm" | cắt **phạm vi tính năng**, không cắt HA. HA thêm sau luôn đắt hơn HA làm từ đầu |
| **Hợp đồng** | SLA có điều khoản phạt | mức sẵn sàng không còn là lựa chọn kỹ thuật, nó là điều khoản tài chính |

Cách moi ràng buộc ẩn ra: hỏi *"điều gì khiến dự án này thất bại kể cả khi hệ
thống chạy hoàn hảo?"* Câu trả lời gần như luôn là một ràng buộc ẩn.

#### Đầu ra bước 1

Một bảng, mỗi dòng năm cột:

| Ràng buộc | Con số | Nguồn | Độ cứng | Vi phạm thì sao |
|---|---|---|---|---|
| Độ trễ đọc lịch khám | p99 < 300 ms | trưởng phòng vận hành | mềm | lễ tân phàn nàn, không mất tiền |
| Không mất đơn đã xác nhận | RPO = 0 cho `appointments` | giám đốc | **cứng** | mất niềm tin bệnh nhân, rủi ro pháp lý |
| Ngân sách | ≤ $1.500/tháng | tài chính | cứng đến hết quý 2 | phải xin duyệt lại |
| Dữ liệu bệnh nhân ở trong nước | 100% | pháp chế | **cứng tuyệt đối** | vi phạm luật |

Cột **độ cứng** là cột quan trọng nhất và là cột hay bị bỏ. Khi hai ràng buộc mâu
thuẫn — và chúng luôn mâu thuẫn — bạn hy sinh cái mềm. Không có cột này thì mọi
đánh đổi đều thành tranh cãi cảm tính.

> Kiểm tra bước 1: đọc bảng ràng buộc của bạn, xoá hết tên dịch vụ AWS nếu lỡ
> viết vào. Bảng ràng buộc nói về **nghiệp vụ**, không nói về AWS. Nếu xoá xong
> mà bảng trống thì bạn chưa làm bước 1, bạn đã nhảy sang bước chọn dịch vụ.

---

### Bước 2. Xác định trục mở rộng

"Hệ thống phải scale được" là câu vô nghĩa cho tới khi bạn nói **scale theo trục
nào**. Năm trục dưới đây dẫn tới năm kiến trúc khác nhau, và tối ưu nhầm trục là
cách tốn tiền mà không giải quyết được gì.

| Trục | Câu hỏi đo | Thành phần chạm trần đầu tiên | Kiến trúc mà nó dẫn tới |
|---|---|---|---|
| **Phiên đồng thời** | bao nhiêu người online cùng lúc? | tầng app, số kết nối tới DB | stateless + ASG/Lambda, tách session ra ElastiCache |
| **Đọc/giây** | bao nhiêu query đọc mỗi giây? | database primary | cache trước, rồi read replica; Aurora tới 15 reader |
| **Ghi/giây** | bao nhiêu ghi mỗi giây? | **database primary — trần cứng nhất** | đổi engine, phân vùng theo key, hoặc queue để làm phẳng đỉnh |
| **Dung lượng** | dữ liệu tăng bao nhiêu GB/tháng? | chi phí lưu trữ, thời gian backup | lifecycle sang lớp lạnh, tách nóng/lạnh, partition theo thời gian |
| **Số thực thể** | bao nhiêu tenant / thiết bị / file? | quota theo tài nguyên, hot partition | phân vùng theo tenant, ranh giới account, shuffle sharding |

Trục **ghi** là trục nguy hiểm nhất. Đọc thì cache và replica giải quyết được gần
như mọi lúc. Ghi thì không có cách nào "thêm vào" — bạn phải đổi mô hình dữ liệu,
và đổi mô hình dữ liệu là bước 3, tức là phải quyết trước khi viết dòng code đầu.

#### Cách chọn trục chính

Hỏi đúng một câu: *"trong 18 tháng tới, con số nào nhân 10?"*

- Nhân 10 **đọc**, ghi giữ nguyên → cache tầng ứng dụng, read replica, CloudFront.
  Chi phí tăng tuyến tính và nhẹ.
- Nhân 10 **ghi** → đây là dự án viết lại tầng dữ liệu. Bắt đầu bằng việc hỏi
  ghi đó có cần đồng bộ không: nếu không, một queue trước database biến đỉnh 10x
  thành hàng đợi dài hơn thay vì database chết.
- Nhân 10 **dung lượng**, đọc không đổi → lifecycle rule và storage class. Xem
  ngưỡng hoà vốn trong [`10-chi-phi.md`](10-chi-phi.md) mục 4. Đừng đụng vào compute.
- Nhân 10 **phiên đồng thời**, cùng lượng dữ liệu → tầng app phải stateless. Đây
  là trục rẻ nhất để giải nếu bạn đã tách state ra ngoài từ đầu.
- Nhân 10 **số tenant** → bài toán cô lập, không phải bài toán capacity. Một
  tenant nặng làm hỏng trải nghiệm 999 tenant còn lại nếu không có ranh giới.

#### Trục KHÔNG tăng cũng là thông tin

Viết ra cả những trục đứng yên. Một hệ thống có 200 GB dữ liệu và mãi mãi 200 GB
thì không cần bàn về sharding — và mọi phút bàn về sharding là phút lấy đi từ
việc thật. Đây là cách rẻ nhất để loại bỏ over-engineering: chứng minh rằng trục
đó không nhúc nhích.

> Kiểm tra bước 2: viết được một câu dạng *"trong 18 tháng, X đi từ A lên B, còn
> Y và Z giữ nguyên"* với A, B là số. Không viết được thì đừng vẽ sơ đồ vội.

---

### Bước 3. Chọn mô hình dữ liệu trước, compute sau

Đây là bước bị đảo thứ tự nhiều nhất, và cũng là bước đảo thứ tự đắt nhất.

#### Vì sao dữ liệu đi trước

Chi phí sửa một lựa chọn sai không đều nhau chút nào:

| Chọn sai cái gì | Cách sửa | Thời gian thật |
|---|---|---|
| Instance type | sửa launch template, instance refresh | **1 giờ** |
| Storage class S3 | thêm lifecycle rule | vài giờ + phí request chuyển tier |
| Compute platform (EC2 → Fargate) | viết Dockerfile, sửa pipeline, sửa cách đọc config | **1–2 tuần** |
| Database engine (DynamoDB → PostgreSQL) | viết lại tầng truy cập, migrate dữ liệu đang sống, dual-write, cutover | **1–6 tháng** |
| Partition key DynamoDB | **không đổi tại chỗ được** — tạo bảng mới, backfill, dual-write, đổi đọc, xoá bảng cũ | 1–3 tháng, có rủi ro mất dữ liệu |

Dòng cuối là lý do của cả bước này. Partition key là quyết định gần như không thể
hoàn tác, và nó phải được đưa ra ở ngày đầu tiên, khi bạn biết ít nhất về hệ thống.
Cách bù lại sự thiếu thông tin đó là bước 3 dưới đây.

#### Bốn câu hỏi chốt mô hình dữ liệu

1. **Truy vấn có biết trước không?** Nếu bạn liệt kê được toàn bộ đường truy cập
   và chúng ổn định → key-value/document (DynamoDB). Nếu người dùng, nhà phân
   tích, hoặc tính năng tương lai sẽ hỏi những câu bạn chưa nghĩ ra → SQL.
2. **Có cần giao dịch nhiều bảng và ràng buộc toàn vẹn không?** Chuyển tiền, trừ
   kho, đặt chỗ có giới hạn → quan hệ. DynamoDB có transaction nhưng giới hạn
   **100 item hoặc 4 MB** một transaction và không có foreign key.
3. **Hình dạng dữ liệu là gì?** Quan hệ nhiều-nhiều → SQL. Tài liệu lồng nhau đọc
   nguyên khối → document. Chuỗi thời gian ghi nhiều đọc theo khoảng → chuyên
   dụng hoặc SQL có partition theo thời gian. Đồ thị nhiều bậc → graph.
4. **Working set có vừa RAM không?** Nếu tập dữ liệu nóng vừa trong bộ nhớ một
   node thì cache giải quyết được gần như mọi vấn đề hiệu năng, và bạn không cần
   đổi engine. Nếu không vừa, cache chỉ làm giảm chứ không xoá được tải đọc.

Bảng chi tiết theo mẫu truy vấn nằm ở [`12-hieu-nang.md`](12-hieu-nang.md) mục 4;
cây quyết định engine nằm ở [`20-cay-quyet-dinh.md`](20-cay-quyet-dinh.md) mục 4;
cơ chế từng engine nằm ở [`03-database.md`](03-database.md).

#### Luật mười câu truy vấn

Trước khi gõ tên engine, viết ra **mười câu truy vấn thật** mà hệ thống sẽ chạy,
bằng tiếng Việt, kèm tần suất ước tính:

```
1.  Lấy lịch khám của bác sĩ X trong ngày D                  ~2.000 lần/giờ
2.  Lấy lịch sử khám của bệnh nhân P, 20 lần gần nhất        ~500 lần/giờ
3.  Đếm số slot trống của phòng khám C trong tuần W          ~800 lần/giờ
4.  Đặt một slot — phải thất bại nếu ai đó vừa đặt trước     ~200 lần/giờ
5.  Huỷ một lịch và trả slot về trạng thái trống             ~30 lần/giờ
...
9.  Báo cáo: tỉ lệ huỷ theo phòng khám theo tháng            2 lần/tháng
10. Truy vết: ai đã sửa lịch khám này, lúc nào               vài lần/tuần
```

Câu 4 là câu quyết định: nó cần **điều kiện ghi có kiểm tra** (conditional write
hoặc transaction). Câu 9 là câu thứ hai: nó là truy vấn ad-hoc nhóm theo hai
chiều, thứ DynamoDB không làm được nếu bạn không thiết kế index cho nó trước.

Hai câu đó dẫn tới hai kết luận khác nhau, và đó là điểm mấu chốt: **truy vấn
hiếm nhưng ad-hoc là thứ quyết định engine, không phải truy vấn nhiều nhất**.
Đường giải thoát chuẩn là tách: dữ liệu vận hành ở một engine tối ưu cho câu 1–5,
đẩy bản sao sang S3 rồi truy vấn bằng Athena cho câu 9–10.

Viết không nổi mười câu nghĩa là bạn chưa hiểu nghiệp vụ đủ để chọn engine. Quay
lại bước 1.

#### Compute chọn sau, và chọn nhanh

Khi mô hình dữ liệu đã chốt, compute gần như tự lộ ra:

| Dữ liệu và mẫu tải | Compute hợp lý | Vì sao |
|---|---|---|
| Ghi có điều kiện, đọc theo key, tải gai | Lambda | mỗi request độc lập, scale theo request |
| Tải đều 24/7, kết nối DB dài | EC2/Fargate với connection pool | Lambda mở/đóng kết nối liên tục làm cạn `max_connections` |
| Job dài, dùng nhiều RAM, chạy theo lịch | Fargate task hoặc EC2 Spot | vượt trần 15 phút của Lambda |
| Xử lý theo lô đêm, chịu được gián đoạn | EC2 Spot trong ASG nhiều instance type | tiết kiệm tới ~90% |

Đổi compute sau này tốn một đến hai tuần. Đổi database tốn một đến sáu tháng. Đó
là toàn bộ lý do của thứ tự này.

---

### Bước 4. Vẽ đường đi của một request

Vẽ đúng **một** request, từ lúc người dùng gõ tên miền tới lúc byte cuối cùng về
màn hình. Mỗi chặng trong đường đi đó là ba thứ cùng lúc: một chỗ tốn thời gian,
một chỗ có thể hỏng, và một dòng trên hoá đơn.

#### Bảng chặng — độ trễ, chế độ hỏng, cách tính tiền

| Chặng | Độ trễ điển hình | Hỏng thì triệu chứng | Tính tiền theo |
|---|---|---|---|
| Resolve DNS (Route 53) | 0 ms nếu cache client, 20–50 ms nếu không | tên không resolve, lỗi rất khó chẩn đoán | $0,50/hosted zone/tháng + $0,40/triệu query |
| Bắt tay TLS | 1 RTT với TLS 1.3; 30–100 ms xuyên lục địa | chứng chỉ hết hạn → sập toàn bộ, không cảnh báo trước | ACM cho ELB/CloudFront **miễn phí** |
| CloudFront | hit 10–30 ms; miss = 10 ms + RTT tới origin | cache key sai → hit ratio sập, origin chết theo | ~$0,085/GB ra internet, 1 TB đầu/tháng miễn phí |
| ALB | +1–3 ms | target group rỗng → 503 | ~$0,0225/giờ (~$16,4/tháng) + ~$0,008/LCU-giờ |
| App compute | p50 vs p99 chênh 5–20 lần khi có GC hoặc cold start | timeout, 5xx | EC2 theo giờ; Lambda theo GB-giây |
| Gọi ElastiCache cùng AZ | **dưới 1 ms** | cache miss bão → database chết | theo giờ node, chạy là tính |
| Query database cùng AZ | 1–10 ms | connection pool cạn → xếp hàng, p99 nổ | theo giờ instance + IOPS/storage |
| Một chặng cross-AZ | **+0,5–1 ms** | thường không hỏng, nhưng **luôn tốn tiền** | $0,01/GB **mỗi chiều** |
| Một chặng cross-Region `us-east-1` ↔ `eu-west-1` | **~75–90 ms RTT** | không thể sửa bằng tối ưu code | ~$0,02/GB |
| NAT Gateway (app gọi ra internet) | +1 ms | port exhaustion ở tải rất cao | ~$0,045/giờ + ~$0,045/GB xử lý |

#### Ngân sách độ trễ — phân bổ p99 cho từng chặng

Ràng buộc "p99 < 300 ms" không dùng được cho tới khi bạn chia nó ra:

| Chặng | Ngân sách | Ghi chú |
|---|---|---|
| DNS + TLS (kết nối mới) | 60 ms | keep-alive làm chặng này về 0 cho request thứ hai trở đi |
| CloudFront → origin khi miss | 40 ms | chỉ tính cho 10% request miss |
| ALB | 3 ms | |
| Ứng dụng, gồm mọi lệnh gọi nội bộ | 120 ms | đây là chỗ code của bạn sống |
| Database + cache | 30 ms | vượt số này nghĩa là thiếu index hoặc thiếu cache |
| Dự phòng | 47 ms | **luôn để dự phòng ít nhất 15%** |
| **Tổng** | **300 ms** | |

Bảng này biến một câu hỏi mơ hồ ("hệ thống có nhanh không?") thành một loạt câu
hỏi kiểm chứng được ("chặng nào đang vượt ngân sách của nó?"). Nó cũng là thứ bạn
dùng để từ chối yêu cầu: nếu người dùng ở Singapore và database ở `us-east-1`,
một chặng cross-Region đã ăn 90 ms trong 300 ms, và không dòng code nào cứu được.

#### Khuếch đại đuôi — vì sao nhiều chặng làm p99 xấu đi nhanh hơn bạn tưởng

Nếu một request gọi 5 dịch vụ downstream tuần tự, mỗi dịch vụ "chậm" 1% số lần,
thì xác suất cả năm đều nhanh là 0,99⁵ ≈ **95%**. Tức là p99 của bạn không còn
được quyết định bởi p99 của từng dịch vụ nữa — nó bị quyết định bởi **số chặng**.
Hệ quả thiết kế: giảm số lệnh gọi nội bộ trên đường đi đồng bộ có tác dụng lên
p99 mạnh hơn tối ưu từng lệnh gọi. Chi tiết p50/p99 ở
[`12-hieu-nang.md`](12-hieu-nang.md) mục 12.

#### Đếm chặng cross-AZ — đó chính là hoá đơn data transfer

Một kiến trúc "chatty" trải trên 2 AZ có khoảng **một nửa** số lệnh gọi nội bộ đi
cross-AZ. Với 10 TB traffic nội bộ mỗi tháng, một nửa cross-AZ, tính hai chiều:
5.120 GB × $0,02 = **~$102/tháng** cho thứ không tạo ra giá trị nào. Cách sửa
không phải bỏ multi-AZ (vi phạm ràng buộc HA) mà là giảm số lệnh gọi: gộp request,
thêm cache, hoặc đặt các thành phần nói chuyện nhiều với nhau vào cùng một AZ và
nhân bản cả cụm ra AZ khác.

> Kiểm tra bước 4: đếm số chặng. Trên 8 chặng cho một request đọc đơn giản là dấu
> hiệu kiến trúc đang có tầng thừa. Mỗi tầng phải trả lời được câu "nó làm gì mà
> tầng bên cạnh không làm được".

---

### Bước 5. Đặt ranh giới hỏng

Câu hỏi của bước này: **cái gì hỏng thì cái gì còn sống?** Trả lời bằng cách vẽ
ranh giới lên sơ đồ, không phải bằng cách thêm chữ "HA" vào tên thành phần.

Bốn ranh giới trong [`00-nen-tang.md`](00-nen-tang.md) mục 2 là bộ công cụ:

| Ranh giới | Cô lập được loại hỏng nào | Giá phải trả | Dấu hiệu bạn cần nó |
|---|---|---|---|
| **AZ** | mất điện, mất mạng, hỏng phần cứng của một toà nhà | dư thừa capacity + $0,01/GB cross-AZ mỗi chiều | mọi hệ thống production, không có ngoại lệ |
| **Region** | thiên tai vùng, sự cố diện rộng của một Region | nhân đôi hạ tầng + độ trễ 75–90 ms + phức tạp dữ liệu | RTO/RPO buộc phải thế, hoặc người dùng ở châu lục khác |
| **Account** | **lỗi con người**, quota cạn, rò rỉ quyền, sự cố lan từ dev sang prod | quản trị nhiều account (Organizations, SSO, billing) | có môi trường prod thật, có nhiều đội |
| **VPC** | lộ đường mạng giữa hai hệ thống không nên nói chuyện | routing và endpoint phức tạp hơn | nhiều hệ thống độc lập trong cùng account |

Ranh giới **account** là ranh giới bị đánh giá thấp nhất. Nó là ranh giới duy nhất
chặn được lỗi con người và là ranh giới của **quota** — service quota tính theo
account theo Region, nên một job chạy loạn trong dev có thể làm prod không launch
nổi instance nếu hai môi trường chung account.

#### Blast radius — viết ra bằng câu, không bằng sơ đồ

Với mỗi thành phần, viết một câu theo mẫu: *"nếu X chết thì Y vẫn chạy, Z thì không"*.

```
Nếu một EC2 app chết        -> ALB rút khỏi rotation trong ~90 giây, ASG thay máy mới. Không ai thấy gì.
Nếu AZ-a chết               -> mất 50% capacity app; RDS failover 60-120 giây; có 1-2 phút lỗi ghi.
Nếu ElastiCache chết        -> mọi đọc dồn vào RDS. RDS chịu được 3x tải đọc? Nếu không, cả hệ thống chết theo.
Nếu RDS primary + standby chết -> hệ thống chết. RTO = thời gian restore snapshot = 30-90 phút.
Nếu người ta xoá nhầm bảng  -> Multi-AZ KHÔNG cứu được. Chỉ PITR cứu được. RPO = 5 phút.
Nếu Region us-east-1 chết   -> hệ thống chết cho tới khi có người dựng lại ở Region khác.
```

Dòng thứ ba là dòng hay bị bỏ sót nhất và là nguồn của phần lớn sự cố thật: **một
thành phần phụ trợ chết làm lộ ra rằng thành phần chính chưa bao giờ được thiết kế
để chịu tải đầy đủ**. Cache là thủ phạm kinh điển. Nếu database không sống nổi khi
cache trống, thì cache không phải tối ưu hiệu năng — nó là một single point of
failure đội lốt.

Dòng thứ năm là ranh giới giữa HA và DR: nhân bản đồng bộ nhân bản luôn cả lệnh
`DROP TABLE`. Xem [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) mục 2.

#### Ba câu hỏi chốt ranh giới

1. **Một lần deploy hỏng ảnh hưởng bao nhiêu phần trăm người dùng?** 100% nghĩa là
   bạn chưa có canary hay blue/green. Đây là chế độ hỏng phổ biến hơn AZ chết
   nhiều lần.
2. **Một tenant nặng làm chậm bao nhiêu tenant khác?** Nếu là "tất cả", bạn cần
   giới hạn theo tenant (throttle, hạn ngạch, hoặc phân vùng).
3. **Mất bao lâu để phát hiện?** Ranh giới hỏng chỉ có giá trị nếu ai đó biết nó
   vừa bị chạm. Đó là bước 7.

> Kiểm tra bước 5: chỉ vào một chỗ bất kỳ trên sơ đồ và hỏi "chết cái này thì sao".
> Nếu bạn phải suy nghĩ quá 10 giây, chỗ đó chưa được thiết kế, nó chỉ được vẽ.

---

### Bước 6. Tính tiền trước khi dựng

Chi phí không phải bước cuối. Nó là bước có quyền phủ quyết kiến trúc, nên nó
phải chạy **trước** khi bạn dựng, không phải sau khi hoá đơn về.

#### Chia hoá đơn thành ba loại

| Loại | Đặc điểm | Ví dụ | Rủi ro |
|---|---|---|---|
| **Theo giờ** | tồn tại là tính tiền, dùng hay không cũng thế | EC2, RDS, ElastiCache, ALB, NAT Gateway, Interface Endpoint | **chi phí ở 0% tải** — dòng nguy hiểm nhất |
| **Theo lượng** | tỉ lệ với traffic thật | Lambda GB-giây, DynamoDB RRU/WRU, request S3, data transfer | hoá đơn nổ khi có sự cố hoặc bị lạm dụng |
| **Theo lưu trữ** | tích luỹ, không bao giờ tự giảm | S3, EBS, snapshot, CloudWatch Logs | tăng đều đặn cho tới khi ai đó nhìn |

Kiến trúc "theo giờ" rẻ hơn khi tải đều và cao. Kiến trúc "theo lượng" rẻ hơn khi
tải gai và thấp. Ngưỡng hoà vốn thường rơi vào khoảng **40–50% utilization liên
tục** — dưới ngưỡng đó serverless thắng, trên ngưỡng đó container/EC2 thắng.

#### Ba khoản người mới luôn quên

**Một — data transfer ra internet.** Ingress $0, egress ~$0,09/GB sau 100 GB miễn
phí mỗi tháng cho cả account. Một ứng dụng phát 5 TB/tháng ra internet trực tiếp:
5.120 GB × $0,09 = **$460/tháng** — thường lớn hơn cả tiền compute. Qua CloudFront
(~$0,085/GB, 1 TB đầu miễn phí, và chặng origin→CloudFront **$0**) con số về
khoảng $350 và nhanh hơn. Bảng đầy đủ chiều nào tốn tiền ở
[`10-chi-phi.md`](10-chi-phi.md) mục 6.

**Hai — NAT Gateway.** ~$0,045/giờ mỗi NAT + ~$0,045/GB xử lý. Kiến trúc 3 AZ
đúng chuẩn HA có 3 NAT Gateway: 3 × $0,045 × 730 = **$98,55/tháng trước khi truyền
một byte hữu ích nào**. Cộng phí xử lý cho traffic đi S3 là bài toán kinh điển
$559/tháng so với $0 của Gateway Endpoint — tính chi tiết ở
[`10-chi-phi.md`](10-chi-phi.md) mục 7.

**Ba — tài nguyên nhàn rỗi tính theo giờ.** Cộng lại thành một khoản lớn hơn bạn nghĩ:

| Thứ quên | $/tháng | Ghi chú |
|---|---|---|
| ALB không có traffic | **$16,4** | tính từ giây đầu, không có bậc miễn phí |
| RDS Multi-AZ `db.m6g.large` chạy 24/7 ở môi trường staging | **~$250** | staging không ai dùng ban đêm và cuối tuần |
| Địa chỉ IPv4 công cộng | **~$3,6** mỗi địa chỉ | từ 01/02/2024, tính **kể cả khi đang gắn vào instance** |
| EBS volume mồ côi 500 GB gp3 | **$40** | instance xoá rồi, volume ở lại |
| Snapshot tích luỹ nhiều năm | tăng đều | không có lifecycle thì không bao giờ giảm |
| CloudWatch log group không đặt retention | tăng đều | mặc định của AWS là **giữ vĩnh viễn** |
| 3 NAT Gateway ở môi trường dev | **$98,55** | dev không cần HA |

#### Đầu ra bước 6 — bảng ba mức tải

Luôn tính ba cột, không phải một:

| Thành phần | 0% tải | Tải hiện tại | Tải × 10 |
|---|---|---|---|
| ALB | $16 | $45 | $300 |
| EC2 app (4 → 4 → 24 instance) | $121 | $121 | $730 |
| RDS Multi-AZ | $250 | $250 | $500 (lên size lớn hơn) |
| NAT Gateway (2 AZ) | $66 | $70 | $110 |
| Egress | $0 | $90 | $900 |
| **Tổng** | **$453** | **$576** | **$2.540** |

Cột **0% tải** là cột dùng để ra quyết định kiến trúc. Ở ví dụ trên, 79% hoá đơn
tồn tại kể cả khi không có người dùng nào. Nếu đây là hệ thống nội bộ chỉ chạy
giờ hành chính, kiến trúc này sai loại — cùng nghiệp vụ đó dựng bằng API Gateway +
Lambda + DynamoDB có chi phí 0% tải gần **$0**.

Quy tắc kiểm tra: **nếu chi phí ở 0% tải lớn hơn 30% chi phí ở tải đỉnh, hãy xét
lại xem có nên dùng mô hình theo lượng không.** Ngược lại, nếu tải gần như phẳng
24/7, mô hình theo giờ cộng Savings Plans (~66–72%) gần như luôn rẻ hơn.

---

### Bước 7. Định nghĩa cách biết nó hỏng

Thiết kế chưa có bước này là thiết kế chưa xong. Nó chạy được, nhưng không vận
hành được — và khoảng cách giữa hai thứ đó là toàn bộ nghề này.

#### Bảng alarm — bốn cột, không được thiếu cột nào

| Thành phần | Metric | Ngưỡng | Ai bị gọi và làm gì |
|---|---|---|---|
| ALB | `HTTPCode_ELB_5XX_Count` | > 10 trong 5 phút | on-call · kiểm tra target health, xem deploy gần nhất |
| ALB | `UnHealthyHostCount` | ≥ 1 trong 2 chu kỳ | on-call · xem log instance, health check path còn đúng không |
| ALB | `TargetResponseTime` p99 | > 0,3 giây trong 10 phút | on-call · so với ngân sách độ trễ ở bước 4 |
| ASG | `GroupInServiceInstances` | < 2 | on-call · quota? AZ hết capacity? launch template hỏng? |
| RDS | `DatabaseConnections` | > 80% `max_connections` | on-call · pool rò rỉ hay thiếu RDS Proxy |
| RDS | `FreeStorageSpace` | < 20% | **cảnh báo sớm, không phải sự cố** · bật storage autoscaling |
| RDS | `ReplicaLag` | > 30 giây | on-call · đọc từ replica đang trả dữ liệu cũ |
| ElastiCache | `CacheHitRate` | < 80% | không gọi ai · vé backlog, xem lại cache key |
| SQS | `ApproximateAgeOfOldestMessage` | > 15 phút | on-call · consumer chết hay đang bị throttle |
| SQS DLQ | `ApproximateNumberOfMessagesVisible` | ≥ 1 | on-call · **luôn luôn có alarm này** |
| Lambda | `Throttles` | ≥ 1 | on-call · concurrency limit account hay reserved concurrency |
| Chi phí | AWS Budgets, dự báo | > 100% ngân sách tháng | chủ sở hữu hệ thống · không phải on-call |

Cột cuối là cột phân biệt hệ thống giám sát với đống tiếng ồn. **Alarm không có
runbook là spam**, và spam thì sau ba tuần sẽ bị tắt thông báo — lúc đó bạn còn
tệ hơn không có alarm, vì bạn tin là mình đang được bảo vệ.

#### Ba cái bẫy của bước 7

**Alarm im lặng khi metric ngừng phát.** Nếu instance chết hẳn, nó không phát
metric nữa, và alarm cấu hình mặc định sẽ vào trạng thái `INSUFFICIENT_DATA` chứ
không phải `ALARM`. Đặt `treat_missing_data = breaching` cho mọi alarm mà việc
"không có dữ liệu" tự nó là tin xấu. Chi tiết ở
[`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md).

**Đo trung bình thay vì đo đuôi.** `Average` của `TargetResponseTime` có thể
tuyệt đẹp trong khi 1% người dùng chờ 5 giây. Với 1 triệu request/ngày, 1% là
10.000 người. Luôn đặt alarm trên p99, không phải trên trung bình.

**Ngưỡng không bắt nguồn từ ràng buộc.** Ngưỡng 0,3 giây trong bảng trên không
phải con số đẹp — nó là p99 mà bước 1 đã ghi vào bảng ràng buộc. Ngưỡng tự nghĩ
ra thì hoặc quá nhạy (ồn) hoặc quá lỏng (vô dụng).

#### Kiểm chứng thiết kế — game day

Alarm chỉ được coi là hoạt động sau khi bạn tự gây ra sự cố và thấy nó kêu. Ba
bài tối thiểu, chạy trước khi lên production:

1. **Giết một instance** — ALB phải rút target trong ~90 giây, ASG phải thay máy,
   người dùng không thấy lỗi. Đo thời gian thật.
2. **Failover database** — `reboot-db-instance --force-failover`, bấm giờ, so với
   60–120 giây của RDS Multi-AZ và dưới 30 giây của Aurora.
3. **Làm rỗng cache** — flush ElastiCache lúc tải trung bình và xem RDS có sống
   không. Đây là bài hay trượt nhất.

> Kiểm tra bước 7: với mỗi ràng buộc trong bảng bước 1, chỉ ra **một** alarm chứng
> minh nó đang được giữ. Ràng buộc không có alarm tương ứng là ràng buộc bạn sẽ
> phát hiện đã vi phạm qua lời phàn nàn của khách hàng.

---

### Chạy cả bảy bước trên một đề

> *"Chúng tôi có 40 phòng khám. Cần hệ thống đặt lịch cho bệnh nhân đặt qua web
> và cho lễ tân đặt hộ qua điện thoại. Phải nhanh, không được đặt trùng slot,
> không được mất dữ liệu, và rẻ. Dữ liệu bệnh nhân phải ở trong nước."*

| Bước | Kết quả |
|---|---|
| **1. Ràng buộc** | p99 đọc lịch < 300 ms (mềm) · **không đặt trùng: cứng tuyệt đối** · RPO = 0 cho `appointments`, RPO 24 giờ cho phần còn lại · ≤ $1.500/tháng (cứng tới hết quý) · **dữ liệu ở Region trong nước: cứng tuyệt đối, loại mọi kiến trúc multi-Region ra ngoài khối** |
| **2. Trục** | Trục chính là **phiên đồng thời** (giờ cao điểm 08:00–10:00, gấp 8 lần trung bình). Ghi rất thấp: 40 phòng khám × ~60 lịch/ngày ≈ **2.400 ghi/ngày**, tức dưới 1 ghi/giây. Dung lượng đứng yên: vài GB/năm |
| **3. Dữ liệu** | Câu truy vấn số 4 ("đặt slot, thất bại nếu vừa có người đặt") cần khoá và điều kiện. Câu số 9 là báo cáo ad-hoc theo hai chiều. Ghi dưới 1/giây nên **không có lý do gì rời khỏi quan hệ** → PostgreSQL trên RDS, unique constraint trên `(doctor_id, slot_start)` giải bài đặt trùng bằng đúng một dòng schema. Chọn DynamoDB ở đây là chọn khó |
| **4. Đường đi** | Client → CloudFront (static) / ALB (API) → app 2 AZ → RDS. 5 chặng. Ngân sách: 60 ms mạng + 3 ms ALB + 120 ms app + 30 ms DB + dự phòng |
| **5. Ranh giới** | AZ: app 2 AZ, RDS Multi-AZ. Region: **cấm**, ràng buộc pháp lý. Account: tách `prod` và `dev`, lỗi con người là chế độ hỏng có thật ở đội 4 người. Cache chết → RDS phải chịu được 100% tải đọc, và với dưới 1 ghi/giây thì nó chịu được |
| **6. Tiền** | 2 × `t3.small` $30 · ALB $45 · RDS `db.t4g.medium` Multi-AZ ~$120 · 1 NAT Gateway (dev, không HA) hoặc **0 NAT + Gateway Endpoint cho S3** · CloudFront $5 · **≈ $210/tháng**, còn xa trần $1.500. Không cần tối ưu thêm — tối ưu ở đây là lãng phí thời gian |
| **7. Alarm** | `HTTPCode_ELB_5XX_Count` > 10/5 phút · `TargetResponseTime` p99 > 0,3 s · `DatabaseConnections` > 80% · `FreeStorageSpace` < 20% · **thêm một alarm nghiệp vụ: số lượt đặt lịch trong 1 giờ = 0 vào giờ hành chính** — đây là alarm duy nhất bắt được lỗi "hệ thống chạy nhưng nút Đặt bị hỏng" |

Điều đáng chú ý: chữ "rẻ" trong đề hoá ra không ràng buộc gì cả — kiến trúc đơn
giản nhất đã nằm dưới trần bảy lần. Chữ ràng buộc thật là "trong nước" và "không
đặt trùng", và cả hai được giải bằng những thứ rất tầm thường. **Phần lớn công
việc thiết kế là phát hiện ra ràng buộc nào không ràng buộc.**

---

## Phần 2 — Bốn kiến trúc tham chiếu

Bốn kiến trúc này là điểm xuất phát, không phải đáp án. Dùng chúng như sau: nhận
đề → chạy bước 1 và 2 → nhận ra đề gần với kiến trúc nào nhất → lấy nó làm nền →
rồi sửa theo ràng buộc riêng. Tự vẽ từ trang trắng mỗi lần là lãng phí.

Mỗi kiến trúc có bốn phần: sơ đồ, bảng chi phí, **cái gì gãy trước khi mở rộng 10
lần**, và **khi nào không chọn nó**. Phần thứ ba là phần đáng đọc nhất — nó cho
bạn biết trần của kiến trúc trước khi bạn đâm vào trần đó lúc 2 giờ sáng.

---

### A. Web ba tầng sẵn sàng cao

**Dấu hiệu chọn:** dữ liệu quan hệ, có phiên đăng nhập, tải tương đối đều trong
giờ làm việc, đội đã quen Linux và SQL, có sẵn ứng dụng chạy trên máy chủ.

**Giả định để tính:** 50.000 MAU, đỉnh 300 request/giây, 200 GB dữ liệu quan hệ,
2 TB egress/tháng, tỉ lệ đọc/ghi 20:1.

```
                        Route 53  (alias record + health check)
                                       |
                        CloudFront  (/static/* cache, /api/* no-cache)
                                       |
                    +------------------+------------------+
                    |          ALB  (public subnet, 2 AZ)  |
                    +------------------+------------------+
                                       |
             +-------------------------+-------------------------+
             |                                                   |
   [AZ-a  private subnet]                             [AZ-b  private subnet]
     app t3.medium x2  (ASG)                            app t3.medium x2  (ASG)
             |                                                   |
             +-------------------------+-------------------------+
                                       |
                       ElastiCache Redis  (primary AZ-a, replica AZ-b)
                                       |
                       RDS PostgreSQL  primary AZ-a
                                    +- standby AZ-b   (đồng bộ, không đọc được)
                                    +- read replica AZ-c (bất đồng bộ)
                                       |
                       S3 (ảnh, file đính kèm) qua Gateway Endpoint  -> $0
```

| Thành phần | Cấu hình | $/tháng |
|---|---|---|
| ALB | 1 ALB, ~5 LCU | $16 + $29 = **$45** |
| EC2 app | 4 × `t3.medium` On-Demand ($0,0416/giờ) | **$121** |
| RDS | `db.m6g.large` Multi-AZ ($0,171/giờ × 2) | **$250** |
| RDS storage | 200 GB gp3, nhân đôi vì Multi-AZ | **$46** |
| Read replica | `db.m6g.large` single-AZ | **$125** |
| ElastiCache | 2 × `cache.t4g.small` ($0,034/giờ) | **$50** |
| NAT Gateway | 2 AZ + ~100 GB xử lý | **$70** |
| CloudFront | 2 TB, 1 TB đầu miễn phí | **$87** |
| Route 53 + S3 + CloudWatch Logs | | **$25** |
| **Tổng** | | **≈ $819** |
| **Cùng kiến trúc, 0% tải** | bỏ egress và LCU | **≈ $690** |

Áp Compute Savings Plans 1 năm cho tầng app đưa $121 xuống ~$73; RDS Reserved
Instance 1 năm đưa $250 xuống ~$160. Tổng còn khoảng **$660** — nhưng chỉ làm sau
khi kiến trúc ổn định ít nhất một quý, vì cam kết là cam kết.

**Cái gì gãy trước khi mở rộng 10 lần** (300 → 3.000 request/giây):

| Thứ tự gãy | Triệu chứng | Cách sửa | Trần mới |
|---|---|---|---|
| 1. Số kết nối tới RDS | 40 app instance × pool 20 = 800 kết nối, vượt `max_connections` | **RDS Proxy** gộp kết nối | ~vài nghìn client |
| 2. Tải đọc trên primary | CPU primary 90%, `ReplicaLag` tăng | thêm read replica (RDS tối đa **5**), đẩy đọc sang replica | 5 replica |
| 3. Trần ghi của primary | không có cách nào thêm writer vào RDS | chuyển Aurora (15 reader, storage tự lớn tới 128 TiB) hoặc phân vùng theo tenant | phải đổi engine |
| 4. Data transfer cross-AZ | hoá đơn tăng nhanh hơn traffic | giảm chattiness, gộp request, thêm cache | |
| 5. Thời gian launch instance | ASG không kịp phản ứng với đỉnh 2 phút | AMI đã nướng sẵn, warm pool, hoặc scale theo lịch | |

Trần thật của kiến trúc này là **trần ghi của một writer duy nhất**. Mọi thứ khác
mua thêm được. Nếu bước 2 nói trục chính là ghi, đừng bắt đầu từ đây.

**Khi nào không chọn:** tải rất gai và nhàn rỗi phần lớn thời gian — bạn trả
$690/tháng cho lúc không ai dùng. Hệ thống nội bộ 200 người dùng giờ hành chính
nên dùng kiến trúc B, rẻ hơn 10 lần cho cùng nghiệp vụ.

---

### B. API serverless hướng sự kiện

**Dấu hiệu chọn:** tải gai hoặc thấp, đội nhỏ không muốn trực đêm, mẫu truy cập
dữ liệu biết trước, mỗi request độc lập, cần đi từ 0 lên production trong vài tuần.

**Giả định để tính:** 20 triệu request/tháng (5 triệu ghi, 15 triệu đọc), 100 GB
dữ liệu, thời gian chạy trung bình 50 ms ở 256 MB, 10.000 người dùng đăng nhập.

```
   Client
     |
   API Gateway (HTTP API)  --- JWT authorizer (Cognito user pool)
     |
   Lambda "api"  256 MB, p50 40 ms, reserved concurrency 200
     |                    \
     |                     +--> DynamoDB  on-demand
     |                            PK = TENANT#<id>   SK = ORDER#<ts>
     |                            GSI1 = trạng thái + thời gian
     |
     +--> EventBridge bus "orders"
                 |
                 +--> SQS chính --> Lambda "worker" --> DynamoDB
                 |        |
                 |        +--> DLQ (giữ 14 ngày, có alarm)
                 |
                 +--> Step Functions (Standard) cho luồng dài hơn 15 phút

   DynamoDB Streams --> Firehose --> S3 (parquet)  --> Athena cho truy vấn ad-hoc
```

| Thành phần | Cách tính | $/tháng |
|---|---|---|
| API Gateway HTTP API | 20 triệu × $1,00/triệu | **$20** |
| Lambda request | 20 triệu × $0,20/triệu | **$4** |
| Lambda compute | 20M × 0,05 s × 0,25 GB = 250.000 GB-giây × $0,0000166667 | **$4** |
| DynamoDB ghi | 5 triệu WRU × $0,625/triệu | **$3** |
| DynamoDB đọc | 15 triệu RRU × $0,125/triệu | **$2** |
| DynamoDB storage | 100 GB × $0,25 | **$25** |
| EventBridge | 5 triệu event × $1,00/triệu | **$5** |
| SQS | 10 triệu request × $0,40/triệu | **$4** |
| Cognito user pool | 10.000 MAU nằm trong bậc miễn phí | **$0** |
| CloudWatch Logs | 20 GB ingest × $0,50 | **$10** |
| **Tổng** | | **≈ $77** |
| **Cùng kiến trúc, 0% tải** | chỉ còn storage | **≈ $25** |

So sánh thẳng với kiến trúc A cho cùng nghiệp vụ: **$77 so với $819**, và chi phí
0% tải **$25 so với $690**. Đó là lý do kiến trúc B thắng gần như mọi lúc ở giai
đoạn đầu — và cũng là lý do nó thua khi tải lên rất cao và rất đều.

**Cái gì gãy trước khi mở rộng 10 lần** (20 → 200 triệu request/tháng, đỉnh ~1.000/giây):

| Thứ tự gãy | Triệu chứng | Cách sửa |
|---|---|---|
| 1. Concurrency limit account | `Throttles` > 0, 429 trả về client | mặc định **1.000 concurrent mỗi account mỗi Region** — xin tăng quota trước, không phải sau |
| 2. Cold start ở đỉnh đột ngột | p99 nhảy lên 1–3 giây | burst concurrency 500–3.000 tuỳ Region rồi chỉ tăng ~500/phút → **provisioned concurrency** cho hàm trên đường đi đồng bộ |
| 3. Hot partition DynamoDB | `ThrottlingException` trong khi capacity tổng còn dư | trần cứng **3.000 RCU / 1.000 WCU mỗi partition**. Nếu một tenant chiếm 80% traffic thì `PK = TENANT#id` là thiết kế sai — thêm hậu tố shard |
| 4. Truy vấn ad-hoc | sếp hỏi một câu không có index | không sửa được trên DynamoDB → phải có sẵn nhánh Streams → S3 → Athena từ đầu |
| 5. Chi phí đảo chiều | hàm nặng CPU 500 ms thay vì 50 ms | 200M × 0,5 s × 1 GB = 100.000 GB-giây/tháng… ở mức này Fargate chạy đều rẻ hơn. Ngưỡng hoà vốn quanh **40–50% utilization** |

**Khi nào không chọn:** cần truy vấn quan hệ ad-hoc ngay trong đường đi chính; cần
transaction nhiều bảng phức tạp; job chạy quá 15 phút trên đường đồng bộ; p99 phải
dưới 50 ms ổn định với tải gai; hoặc đội chưa có kỷ luật quan sát — debug một hệ
phân tán 12 thành phần khó hơn debug một monolith nhiều lần, và đó là chi phí thật
mà bảng $77 ở trên không thể hiện.

---

### C. Xử lý dữ liệu theo lô và theo luồng

**Dấu hiệu chọn:** log, clickstream, telemetry IoT, dữ liệu giao dịch cần phân
tích. Hai nhánh trong cùng một kiến trúc vì hai câu hỏi khác nhau: "có gì bất
thường ngay bây giờ không" và "tháng trước ra sao".

**Giả định để tính:** 500 GB/ngày (≈15 TB/tháng), đỉnh 200.000 event/phút
(≈3.300/giây), bản ghi trung bình 25 KB, giữ 13 tháng.

```
  Nguồn: app log · clickstream · thiết bị IoT
                 |
     +-----------+------------------------------+
     |                                          |
  [LUỒNG - trả lời trong vài giây]        [LÔ - trả lời trong vài giờ]
     |                                          |
  Kinesis Data Streams  (8 shard)          app ghi thẳng S3 (raw, JSON gzip)
     |        \                                   |
     |         +--> Lambda: phát hiện bất thường  |
     |                  -> SNS -> on-call    Glue Crawler -> Data Catalog
     |                                            |
     +--> Data Firehose  (buffer 5 phút / 128 MB) |
              |                              Glue ETL job (Spark, 10 DPU)
              +--> S3 curated (parquet, phân vùng dt=YYYY-MM-DD/)
                                |
                                +--> Athena (ad-hoc, $5/TB quét)
                                +--> lifecycle: 90 ngày -> Glacier Instant Retrieval
```

| Thành phần | Cách tính | $/tháng |
|---|---|---|
| Kinesis provisioned | 8 shard × $0,015/giờ × 730 | **$88** |
| Kinesis PUT payload | 15 TB ÷ 25 KB = 600 triệu unit × $0,014/triệu | **$8** |
| *(Kinesis on-demand thay thế)* | $0,04/giờ/stream + 15.360 GB × $0,08/GB | *($1.258)* |
| Data Firehose | 15.360 GB × $0,029/GB | **$445** |
| S3 curated (parquet, nén ~5:1) | 3 TB tích luỹ × $0,023 | **$69** |
| Glue ETL | 10 DPU × 1 giờ/ngày × 30 × $0,44 | **$132** |
| Athena | 200 query × 5 GB quét = 1 TB × $5 | **$5** |
| Lambda cảnh báo | | **$10** |
| **Tổng** | | **≈ $757** |

Hai con số cần nhớ từ bảng này. **Kinesis on-demand đắt hơn provisioned 13 lần ở
mức tải cao và đều** ($1.258 so với $96) — on-demand đúng cho tải không đoán được,
sai cho tải ổn định. **Firehose là dòng lớn nhất** ($445): nếu ứng dụng có thể tự
ghi parquet vào S3 theo lô, bạn tiết kiệm gần một nửa hoá đơn và mất đi sự tiện lợi.

**Cái gì gãy trước khi mở rộng 10 lần** (5 TB/ngày):

| Thứ tự gãy | Triệu chứng | Cách sửa |
|---|---|---|
| 1. Trần shard | `ProvisionedThroughputExceeded` khi ghi | mỗi shard **1 MB/s hoặc 1.000 record/giây ghi, 2 MB/s đọc** → cần ~80 shard, phải resharding và chọn lại partition key |
| 2. Vấn đề file nhỏ | Athena chậm dần rồi đắt dần | Firehose buffer 1 phút tạo hàng triệu file vài MB. Tăng buffer lên 5–15 phút, hoặc chạy job nén định kỳ |
| 3. Athena quét toàn bảng | một query $50 thay vì $0,25 | phân vùng theo `dt=` **và** ghi parquet. Không phân vùng thì mọi query quét 100% dữ liệu |
| 4. Firehose tuyến tính | $445 → $4.450 | đây là lúc ghi thẳng từ ứng dụng bắt đầu đáng công |
| 5. Glue job vượt cửa sổ | job đêm không kịp xong trước giờ làm | tăng DPU (tuyến tính về tiền), hoặc chia job theo phân vùng |

**Khi nào không chọn:** dưới ~1 GB/ngày thì toàn bộ kiến trúc này là
over-engineering. Ghi thẳng vào S3 rồi query bằng Athena tốn khoảng **$5/tháng**
và giải quyết đúng nghiệp vụ đó. Kinesis chỉ đáng khi bạn thật sự cần đọc lại
(replay) và cần nhiều consumer độc lập đọc cùng một luồng — nếu chỉ có một consumer
và không cần replay, SQS đơn giản hơn và rẻ hơn. So sánh ở
[`22-bang-so-sanh.md`](22-bang-so-sanh.md) mục Tích hợp.

---

### D. Nội dung tĩnh và động toàn cầu

**Dấu hiệu chọn:** người dùng ở nhiều châu lục, tỉ trọng nội dung tĩnh cao, độ trễ
là ràng buộc nghiệp vụ chứ không phải mong muốn.

**Giả định để tính:** người dùng ở Bắc Mỹ, châu Âu, châu Á; 20 TB egress/tháng;
90% request là nội dung tĩnh; 500 GB dữ liệu quan hệ; ghi tập trung, đọc phân tán.

```
                            người dùng toàn cầu
                                     |
                     Route 53  (latency-based hoặc geolocation)
                                     |
                            CloudFront  (600+ PoP)
                     cache behavior tách theo đường dẫn
                                     |
        +----------------------------+----------------------------+
        |  /static/*                                     /api/*   |
        |                                                          |
   S3 bucket us-east-1  (OAC, chặn public)          Origin group (failover)
        |                                             |                  |
   replication -> S3 eu-west-1                  ALB us-east-1      ALB eu-west-1
                                                     |                  |
                                                app tier ASG      app tier ASG
                                                     |                  |
                                            Aurora Global Database
                                       writer us-east-1  ->  reader eu-west-1
                                          (lag điển hình dưới 1 giây)
```

| Thành phần | Cách tính | $/tháng |
|---|---|---|
| CloudFront egress | 20 TB, 1 TB đầu miễn phí, ~$0,085/GB | **≈ $1.630** |
| *(so sánh: đi thẳng từ ALB)* | 20.480 GB × $0,09 + độ trễ tệ hơn | *($1.843)* |
| S3 storage 2 Region | 2 TB × $0,023 × 2 | **$94** |
| S3 replication | ~$0,02/GB cross-Region cho phần thay đổi | **$20** |
| Aurora writer | `db.r6g.large` $0,29/giờ | **$212** |
| Aurora reader Region 2 | `db.r6g.large` | **$212** |
| Aurora storage | 500 GB × $0,10 × 2 Region | **$100** |
| ALB × 2 Region | | **$90** |
| EC2 app 2 Region | 4 + 2 instance | **$180** |
| Route 53 health check + query | | **$15** |
| **Tổng** | | **≈ $2.553** |

**Cái gì gãy trước khi mở rộng 10 lần** (200 TB egress):

| Thứ tự gãy | Triệu chứng | Cách sửa |
|---|---|---|
| 1. Cache hit ratio | origin traffic tăng nhanh hơn traffic tổng | biến quan trọng nhất trong toàn kiến trúc: đi từ 90% lên 95% **giảm một nửa** tải origin. Đo `CacheHitRate`, xem cache key có đang chứa header/cookie/query không cần thiết không |
| 2. Chi phí invalidation | hoá đơn có dòng lạ | 1.000 path đầu mỗi tháng miễn phí, sau đó ~$0,005/path → dùng **URL có version** (`app.v3.js`) thay vì invalidate |
| 3. Độ trễ ghi từ xa | người dùng APAC thấy nút Lưu chậm 400 ms | mọi ghi vẫn phải về writer Region: RTT `ap-southeast-1` ↔ `us-east-1` ~180–200 ms. Không tối ưu code nào sửa được — hoặc chấp nhận, hoặc phân vùng dữ liệu theo Region |
| 4. Ràng buộc chủ quyền dữ liệu | pháp chế chặn go-live | nếu dữ liệu EU không được rời EU thì Aurora Global Database bị loại → hai hệ thống độc lập theo Region, không đồng bộ dữ liệu người dùng |
| 5. Failover Region | RTO thực tế dài hơn dự kiến | promote Aurora Global secondary mất khoảng 1 phút cho RPO ~1 giây; cộng thời gian đổi DNS/Route 53 và thời gian **người ra quyết định** |

**Khi nào không chọn:** người dùng thật sự chỉ ở một quốc gia. CloudFront một mình
(không multi-Region origin) đã giải quyết được phần lớn bài toán độ trễ và chi phí
egress với giá vài chục đô. Nhân đôi tầng dữ liệu ra Region thứ hai chỉ đáng khi
bước 1 có một ràng buộc **cứng** buộc phải thế — hoặc RTO/RPO, hoặc người dùng ở
châu lục khác, hoặc luật. Xem [`11-san-sang-cao.md`](11-san-sang-cao.md) mục 8 về
cái giá thật của multi-Region.

---

### Bốn kiến trúc cạnh nhau

| | A. Web ba tầng | B. Serverless | C. Dữ liệu | D. Toàn cầu |
|---|---|---|---|---|
| Chi phí **0% tải** | $690 | **$25** | $220 | $810 |
| Chi phí ở tải giả định | $819 | **$77** | $757 | $2.553 |
| Trục scale tự nhiên | đọc, phiên đồng thời | request/giây | throughput byte/giây | egress, địa lý |
| Trần cứng đầu tiên | **ghi của một writer** | concurrency + hot partition | shard | độ trễ ghi xuyên lục địa |
| p99 điển hình | 100–300 ms | 50–200 ms (cold start 1–3 s) | không áp dụng | 30–80 ms cho nội dung tĩnh |
| Kỹ năng đội cần | Linux, SQL, Terraform | mô hình sự kiện, quan sát phân tán | SQL, Spark, mô hình cột | tất cả những thứ trên |
| Thời gian dựng lần đầu | 2–4 tuần | **1–2 tuần** | 3–6 tuần | 6–12 tuần |
| Rủi ro lớn nhất | trần ghi database | debug hệ phân tán | hoá đơn Athena/Firehose | phức tạp vận hành |

---

## Phần 3 — Đọc đề bài như một kiến trúc sư

Khách hàng mô tả **giải pháp họ tưởng tượng ra**, không mô tả ràng buộc. Việc của
bạn là dịch ngược. Cùng một câu nói có thể có hai nghĩa dẫn tới hai kiến trúc khác
hẳn nhau, nên cột thứ ba trong bảng dưới — câu hỏi phân biệt — là cột làm việc.

| Khách hàng nói | Thật ra có thể là | Hỏi câu này để phân biệt | Hệ quả kiến trúc |
|---|---|---|---|
| "Chúng tôi không muốn quản máy chủ" | (a) ngân sách vận hành thấp · (b) **không có ai trực đêm** · (c) đội không biết Linux | "Hiện ai vá OS, mất bao lâu mỗi tháng? Ai bị gọi lúc 3 giờ sáng?" | (a) → Fargate/Beanstalk là đủ · (b) → phải là managed **và** có alarm định tuyến được, kiến trúc B · (c) → tránh mọi thứ cần SSH |
| "Phải nhanh" | (a) p99 API · (b) cảm nhận tải trang · (c) báo cáo chạy xong sớm | "Nhanh với ai, đo ở đâu — máy người dùng hay server?" | (b) thường được giải bằng CloudFront và ảnh nhẹ hơn, **không** bằng instance to hơn |
| "Không được mất dữ liệu" | (a) RPO = 0 thật · (b) không mất **đơn đã xác nhận** · (c) sợ xoá nhầm | "Mất 5 phút dữ liệu cuối thì thiệt hại bằng tiền là bao nhiêu?" | (b) chỉ cần queue bền + xử lý idempotent, rẻ hơn (a) nhiều lần · (c) là bài toán **backup/PITR**, Multi-AZ không cứu |
| "Cần 99,99%" | (a) con số trong hợp đồng có phạt · (b) một con số nghe cho oai | "Đo trên endpoint nào? Deploy có tính là downtime không? Ai đo?" | (a) → multi-AZ mọi tầng, deploy không downtime, ngân sách lỗi 4,3 phút/tháng · (b) → 99,9% là đủ và rẻ hơn nhiều |
| "Phải scale tự động" | (a) tải theo mùa **dự đoán được** · (b) đột biến bất ngờ · (c) chỉ là không muốn sửa capacity bằng tay | "Đỉnh đến trong bao lâu — 5 phút hay 5 giây? Có lặp lại theo lịch không?" | (a) → scheduled scaling, rẻ nhất · (b) → warm pool hoặc serverless, vì ASG cần 2–4 phút để có instance sẵn sàng |
| "Chúng tôi muốn multi-Region" | (a) DR · (b) độ trễ người dùng · (c) tuân thủ | "Nếu Region chính chết 4 giờ, thiệt hại là bao nhiêu?" | Ba câu trả lời ra ba kiến trúc **khác hẳn nhau**: (a) pilot light, (b) read replica + CloudFront, (c) hai hệ thống độc lập không đồng bộ |
| "Hệ thống hiện tại chậm" | chưa ai đo | "Chậm ở đâu — chỉ tôi biểu đồ p99 theo endpoint" | Không có số đo thì việc đầu tiên là **đo**, không phải thêm cache. Xem [`12-hieu-nang.md`](12-hieu-nang.md) mục 1 |
| "Chi phí quá cao" | cao so với ngân sách, hoặc so với kỳ vọng mơ hồ | "Ba dòng lớn nhất trên Cost Explorer là gì?" | 80% hoá đơn thường nằm ở 2–3 dòng. Tối ưu chỗ khác là lãng phí thời gian |
| "Dữ liệu phải mã hoá" | (a) ô tick tuân thủ · (b) kiểm soát và audit khoá · (c) khoá không được rời tổ chức | "Ai kiểm toán, và họ yêu cầu chứng minh điều gì?" | (a) SSE-S3 đủ · (b) SSE-KMS với CMK riêng + CloudTrail · (c) CloudHSM hoặc import key material |
| "Cần realtime" | (a) dưới 100 ms · (b) "trong vòng vài giây" · (c) "cùng ngày" | "Chuyện gì xảy ra nếu chậm 10 giây?" | (a) WebSocket/streaming, đắt · (b) **SQS + Lambda là đủ**, đây là nghĩa của 90% lần nói "realtime" · (c) batch đêm |
| "Đội chúng tôi biết Kubernetes" | (a) đã vận hành cluster production · (b) đã làm tutorial | "Ai từng bị gọi lúc 3 giờ sáng vì một cluster, và chuyện gì đã xảy ra?" | (b) → EKS là nợ kỹ thuật; ECS on Fargate cho 90% lợi ích với 10% bề mặt vận hành |
| "Chúng tôi muốn microservice" | (a) nhiều đội cần deploy độc lập · (b) nghe nói nó tốt | "Có bao nhiêu đội deploy độc lập hôm nay?" | Dưới 3 đội thì microservice thêm chi phí phân tán mà không thêm lợi ích tổ chức |
| "Cần audit đầy đủ" | (a) ai gọi API AWS nào · (b) **ai đổi bản ghi nghiệp vụ nào** | "Kiểm toán viên sẽ hỏi câu gì đầu tiên?" | (a) CloudTrail · (b) audit log ở tầng ứng dụng — CloudTrail **không** thấy được. Thường họ cần (b) |
| "Chúng tôi có hệ thống on-prem không bỏ được" | (a) dữ liệu lớn khó chuyển · (b) giấy phép · (c) chính trị nội bộ | "Hệ thống đó gọi cái gì, và độ trễ chấp nhận được là bao nhiêu?" | Băng thông và độ trễ quyết định VPN hay Direct Connect. Xem [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) mục 11 |

### Những câu hỏi phải hỏi lại trước khi vẽ

Mười hai câu, chia bốn nhóm. Mỗi câu chọn được vì câu trả lời của nó **đổi kiến
trúc**, không chỉ đổi kích thước tài nguyên.

**Về số**
1. Số người dùng đồng thời ở giờ cao điểm, và đỉnh gấp mấy lần trung bình?
2. Bao nhiêu ghi mỗi giây ở đỉnh? *(trục khó nhất — hỏi sớm)*
3. Dữ liệu tăng bao nhiêu GB mỗi tháng, và giữ bao lâu?

**Về dữ liệu**
4. Kể tôi nghe mười câu truy vấn hệ thống sẽ chạy nhiều nhất.
5. Có truy vấn nào ad-hoc, do người dùng hoặc nhà phân tích tự đặt câu hỏi không?
6. Thao tác nào phải hoặc thành công trọn vẹn hoặc không xảy ra gì?

**Về ranh giới**
7. Nếu mất 5 phút dữ liệu cuối, thiệt hại là bao nhiêu? *(ra RPO)*
8. Nếu hệ thống chết 4 giờ, thiệt hại là bao nhiêu? *(ra RTO)*
9. Dữ liệu có bị cấm rời khỏi một quốc gia hoặc khối nào không?

**Về con người**
10. Bao nhiêu người sẽ vận hành cái này, và họ đã vận hành gì trước đây?
11. Có ai trực ngoài giờ không, và họ được trả tiền cho việc đó chứ?
12. Điều gì làm dự án này thất bại kể cả khi hệ thống chạy hoàn hảo?

Câu 12 moi ra ràng buộc ẩn nhanh hơn mọi câu khác. Câu 11 quyết định mức tự động
hoá đáng đầu tư nhiều hơn mọi con số kỹ thuật.

### Khi không có ai để hỏi

Trong phòng thi, trong bài tập, hoặc khi người ra đề đã nghỉ việc, dùng bộ mặc
định an toàn này và **ghi rõ đó là giả định**:

| Chưa biết | Mặc định an toàn | Vì sao |
|---|---|---|
| Mức sẵn sàng | **99,9%**, multi-AZ, single-Region | mức mà multi-AZ đạt được tự nhiên; 99,99% phải có người yêu cầu rõ |
| RPO/RPO của DR | RPO 5 phút (PITR), RTO 4 giờ | backup & restore, rẻ nhất trong bốn chiến lược |
| Mô hình dữ liệu | **quan hệ** | sai sang NoSQL đắt hơn sai sang SQL, vì SQL luôn trả lời được câu hỏi bạn chưa nghĩ ra |
| Compute | managed trước, tự quản sau | Fargate/Lambda trước EC2 |
| Mã hoá | bật hết, SSE-KMS với khoá do AWS quản | không bao giờ là đáp án sai |
| Ranh giới môi trường | account riêng cho prod | ranh giới rẻ nhất chặn được lỗi con người |
| Ngân sách | tính ba mức tải và trình bày cả ba | người duyệt cần cột 0% tải hơn bạn nghĩ |
| Số AZ | **3** | dư thừa 150% thay vì 200% của 2 AZ — rẻ hơn cho cùng mức chịu lỗi |

---

## Phần 4 — Sai lầm thiết kế điển hình

Đây không phải bẫy đề thi. Đây là những thứ đã tốn tiền thật và thời gian thật.

| Sai lầm | Vì sao nó hấp dẫn | Cái giá thật | Làm đúng là gì |
|---|---|---|---|
| **Multi-Region khi chưa multi-AZ cho đúng** | nghe như mức cao nhất của HA | nhân đôi chi phí và nhân ba độ phức tạp để chống lại chế độ hỏng hiếm hơn nhiều lần so với chế độ hỏng bạn **chưa** chống | multi-AZ mọi tầng trước, gồm cả tầng bạn quên: NAT, cache, và quy trình deploy |
| **Microservice khi đội có ba người** | mọi bài blog đều nói thế | 12 service = 12 pipeline, 12 alarm set, 12 chỗ đặt secret, và một cuộc gọi mạng ở chỗ trước đây là một lời gọi hàm | monolith có module rõ ràng; tách ra khi có **đội** cần deploy độc lập, không khi có **service** cần tách |
| **Chọn DynamoDB rồi phát hiện cần truy vấn ad-hoc** | "scale vô hạn, không cần quản server" | không thêm được truy vấn mới nếu không thiết kế index trước; đổi partition key là dự án hàng tháng | luật mười câu truy vấn ở bước 3; nếu có bất kỳ câu ad-hoc nào, hoặc chọn SQL, hoặc dựng sẵn nhánh Streams → S3 → Athena từ ngày đầu |
| **Đặt cache trước một bài toán chưa đo** | cache luôn làm mọi thứ nhanh hơn | thêm một chế độ hỏng (dữ liệu cũ), thêm một SPOF (cache chết → DB chết), thêm $50–200/tháng, và **giấu đi** query thiếu index thay vì sửa nó | đo trước: `TargetResponseTime` p99 theo endpoint, `slow query log`. Thêm index thường nhanh hơn cache và không tốn gì |
| **Tự dựng cái AWS đã quản lý** | "chúng tôi kiểm soát được nhiều hơn" | tự dựng NAT instance, tự dựng Redis trên EC2, tự dựng scheduler — mỗi cái là một hệ thống phải vá, phải backup, phải giám sát, phải có người biết | dùng managed trước; tự dựng chỉ khi có ràng buộc **cứng** viết ra được (giấy phép, phiên bản engine, tính năng thiếu) |
| **Thiết kế cho đỉnh, quên đáy** | ai cũng hỏi "chịu được bao nhiêu" | trả tiền đỉnh 24/7; ví dụ kiến trúc A trả $690/tháng cho lúc không ai dùng | luôn tính cột 0% tải ở bước 6; nếu nó vượt 30% chi phí đỉnh, xét lại loại kiến trúc |
| **Coi HA là thứ bật sau** | "cứ chạy được đã, HA thêm sau" | session trong RAM, file ghi vào EBS local, IP cứng trong config — ba thứ này biến việc thêm instance thứ hai thành dự án viết lại | stateless từ ngày đầu; state ra ElastiCache/S3/RDS. Chi phí ngày đầu gần bằng 0, chi phí thêm sau là hàng tuần |
| **Không có cách kiểm chứng** | sơ đồ trông đúng | "chúng tôi có Multi-AZ" mà chưa ai từng failover; alarm chưa ai từng thấy kêu | ba bài game day ở bước 7, chạy **trước** khi lên production và lặp mỗi quý |

Ba sai lầm đầu có chung một gốc: **chọn theo danh tiếng của công nghệ thay vì
theo ràng buộc của bài toán**. Cách kiểm tra rẻ nhất là bắt bản thân viết một câu
"chúng tôi chọn X vì ràng buộc số N trong bảng bước 1". Viết không nổi câu đó thì
lựa chọn đang đến từ thói quen, không từ thiết kế.

---

## Bảng số phải nhớ

Con số dùng để **thiết kế**, không phải để nhớ tên dịch vụ. Giá tham chiếu
`us-east-1`, tính đến **2026-08**.

| Con số | Giá trị | Dùng ở bước nào |
|---|---|---|
| Chi phí giờ của ALB nhàn rỗi | **$16,4/tháng** | bước 6, cột 0% tải |
| Chi phí giờ của 3 NAT Gateway | **$98,55/tháng** trước khi truyền byte nào | bước 6, quyết định số AZ cho môi trường dev |
| Địa chỉ IPv4 công cộng | **~$3,6/tháng** mỗi địa chỉ, kể cả đang dùng | bước 6, đếm số EIP và số instance public |
| Egress internet | ~$0,09/GB, 100 GB đầu/tháng miễn phí | bước 6, dòng lớn nhất của hệ thống nhiều nội dung |
| CloudFront egress | ~$0,085/GB, **1 TB đầu miễn phí**, origin→CloudFront $0 | bước 4 và 6 |
| Cross-AZ | **$0,01/GB mỗi chiều** = $0,02 một lượt đi-về | bước 4, đếm chặng cross-AZ |
| RTT cross-Region `us-east-1` ↔ `eu-west-1` | **~75–90 ms** | bước 4, ngân sách độ trễ |
| RTT cross-Region `us-east-1` ↔ `ap-southeast-1` | **~180–200 ms** | bước 4, quyết định có ghi từ xa được không |
| Độ trễ cùng AZ tới ElastiCache | **dưới 1 ms** | bước 4 |
| Chặng cross-AZ | **+0,5–1 ms** | bước 4 |
| Lambda concurrency mặc định | **1.000 mỗi account mỗi Region** | bước 2, xin tăng quota trước khi lên production |
| Lambda trần chạy | **15 phút**, 10.240 MB | bước 3, loại Lambda cho job dài |
| DynamoDB trần mỗi partition | **3.000 RCU / 1.000 WCU** | bước 2, thiết kế partition key |
| Kinesis trần mỗi shard | **1 MB/s hoặc 1.000 record/s** ghi; 2 MB/s đọc | bước 2, tính số shard |
| RDS read replica | tối đa **5**; Aurora **15 reader** | bước 2, trần trục đọc |
| RDS Multi-AZ failover | **60–120 giây**; Aurora dưới 30 giây | bước 5, ngân sách RTO |
| Dư thừa capacity cho N AZ | N/(N−1): 2 AZ **200%**, 3 AZ **150%** | bước 5, chọn số AZ |
| 99,9% / 99,99% | 43,2 phút/tháng · **4,3 phút/tháng** | bước 1, dịch mức sẵn sàng ra số |
| Ngưỡng hoà vốn serverless ↔ container | ~**40–50% utilization** liên tục | bước 6, chọn loại kiến trúc |
| Khuếch đại đuôi 5 chặng, mỗi chặng 1% chậm | p99 tổng hỏng ở **~5%** request | bước 4, giới hạn số chặng đồng bộ |

---

## Bẫy đề thi

**Bẫy 1 — thêm tài nguyên cho một bài toán chưa đo**

> *Ứng dụng chậm dần trong giờ cao điểm. Giải pháp nào cải thiện hiệu năng?* —
> "Tăng instance size" và "thêm ElastiCache" đều hấp dẫn. Đáp án đúng gần như luôn
> là thứ **xác định nghẽn trước**: bật Performance Insights, xem `slow query log`,
> hoặc bật detailed monitoring. Vì sao: đề mô tả triệu chứng, không mô tả nguyên
> nhân; ba nguyên nhân khác nhau cho ba đáp án khác nhau.

**Bẫy 2 — multi-Region cho một yêu cầu multi-AZ**

> *Ứng dụng phải chịu được sự cố của một data center.* — "Deploy sang Region thứ
> hai" là bẫy. Một AZ **là** một hoặc nhiều data center; đáp án là multi-AZ. Region
> chỉ vào cuộc khi đề nói `Region-wide outage`, `natural disaster`, hoặc có RTO/RPO
> mà multi-AZ không đạt.

**Bẫy 3 — "no server management" bị đọc thành "phải là Lambda"**

> *Đội không muốn quản lý máy chủ; workload chạy 4 giờ mỗi lần.* — Lambda là bẫy
> vì trần 15 phút. Đáp án: **Fargate**. Vì sao: "no server management" là ràng buộc
> vận hành, thời lượng chạy là ràng buộc kỹ thuật, và ràng buộc kỹ thuật loại
> trước.

**Bẫy 4 — tối ưu chi phí bằng cách vi phạm ràng buộc HA**

> *Hoá đơn tăng sau khi chuyển sang multi-AZ. Giảm chi phí thế nào?* — "Gộp về một
> AZ" là bẫy: nó vi phạm ràng buộc đã nêu ở câu trước trong đề. Đáp án: giảm traffic
> cross-AZ (cache, gộp request), hoặc dùng Gateway Endpoint thay NAT.

**Bẫy 5 — chọn DynamoDB vì đề có chữ "scale"**

> *Ứng dụng cần scale tới hàng triệu người dùng và cho phép nhà phân tích tự đặt
> câu hỏi.* — "DynamoDB" là bẫy. Vế thứ hai là truy vấn ad-hoc. Đáp án: Aurora, hoặc
> DynamoDB kèm đường xuất sang S3 + Athena nếu đề liệt kê phương án đó.

**Bẫy 6 — thiết kế xong mà không có cách phát hiện hỏng**

> *Kiến trúc nào đáp ứng yêu cầu và cho phép đội phản ứng khi có sự cố?* — Ba đáp
> án dựng đúng hạ tầng nhưng không có phần giám sát. Vế sau của câu hỏi là vế tính
> điểm. Vì sao: đề SAA thường gài yêu cầu vận hành vào mệnh đề phụ cuối câu.

---

## Cây quyết định

**Trang giấy trắng, bắt đầu từ đâu?**

**Câu hỏi loại nửa: dữ liệu chính có cần truy vấn ad-hoc hoặc transaction nhiều
bảng không?**

- **Có** → tầng dữ liệu là quan hệ. Tiếp: **tải có đều 24/7 không?**
  - Đều, và trên ~40% utilization → **kiến trúc A**, RDS/Aurora Multi-AZ, cộng
    Savings Plans sau một quý.
  - Gai hoặc thấp → vẫn Aurora nhưng compute serverless (Fargate hoặc Lambda có
    RDS Proxy). Chi phí 0% tải giảm mạnh, trần ghi giữ nguyên.
- **Không**, mẫu truy cập biết trước và ổn định → tiếp: **một request có độc lập
  với request khác không?**
  - Độc lập, mỗi lần chạy dưới 15 phút → **kiến trúc B**.
  - Không độc lập, hoặc chạy lâu → Fargate task, Step Functions cho luồng dài.
- **Dữ liệu là dòng sự kiện, giá trị nằm ở tổng hợp chứ không ở từng bản ghi** →
  **kiến trúc C**. Tiếp: **có cần đọc lại (replay) và nhiều consumer độc lập không?**
  - Có → Kinesis. Tải đều và cao → provisioned, không phải on-demand.
  - Không → SQS đơn giản hơn và rẻ hơn; dưới 1 GB/ngày thì ghi thẳng S3 + Athena.
- **Ràng buộc chính là địa lý người dùng hoặc khối lượng egress** → **kiến trúc D**.
  Tiếp: **ghi có cần xảy ra gần người dùng không?**
  - Không (ghi tập trung, đọc phân tán) → CloudFront + read replica cross-Region.
  - Có → phân vùng dữ liệu theo Region, chấp nhận hai hệ thống không đồng bộ. Đây
    cũng là đáp án bắt buộc khi có ràng buộc chủ quyền dữ liệu.

Sau khi chốt nhánh, chạy tiếp bước 5, 6, 7 — chúng không phụ thuộc vào nhánh nào.

---

## Nối với thực hành

Quy trình khác với các chương khác: ở đây bạn **thiết kế trước, viết Terraform sau**.
Với mỗi lab, dành 20 phút chạy bảy bước trên giấy trước khi mở `main.tf`.

| Lab | Bước nào của chương này | Việc cụ thể |
|---|---|---|
| [`labs-self/w02-vpc-networking/`](../../learn-aws/labs-self/w02-vpc-networking/README.md) | Bước 5, 6 | Tự chọn số AZ rồi tính cột 0% tải trước khi `apply`. So sánh 2 AZ và 3 AZ về cả dư thừa capacity lẫn tiền NAT |
| [`labs-self/w03-ec2-alb-asg/`](../../learn-aws/labs-self/w03-ec2-alb-asg/README.md) | Bước 4, 5, 7 | Đây là kiến trúc A thu nhỏ. Vẽ bảng chặng, viết ba câu blast radius, rồi giết một instance và bấm giờ |
| [`labs-self/w04-s3-cloudfront/`](../../learn-aws/labs-self/w04-s3-cloudfront/README.md) | Bước 4, 6 | Kiến trúc D thu nhỏ. Đo `CacheHitRate`, thử đưa query string vào cache key và xem hit ratio sập |
| [`labs-self/w05-databases/`](../../learn-aws/labs-self/w05-databases/README.md) | **Bước 3** | Viết mười câu truy vấn trước khi chọn giữa DynamoDB và RDS. Thử ép một câu ad-hoc lên DynamoDB để thấy nó khó ở đâu |
| [`labs-self/w06-serverless-api/`](../../learn-aws/labs-self/w06-serverless-api/README.md) | Bước 2, 6 | Kiến trúc B. So chi phí 0% tải của lab này với lab w03 — hai con số đó là toàn bộ luận điểm của Phần 2 |
| [`labs-self/w07-decoupling/`](../../learn-aws/labs-self/w07-decoupling/README.md) | Bước 2, 5 | Đề chỉ nói "backend không được sập khi traffic tăng 10 lần". Chọn trục và chọn cơ chế là việc của bạn |
| [`labs-self/w10-observability-iac/`](../../learn-aws/labs-self/w10-observability-iac/README.md) | **Bước 7** | Bảng alarm bốn cột. Đặc biệt chú ý `treat_missing_data` và alarm trên p99 thay vì trung bình |
| [`labs-self/w11-dr-hybrid/`](../../learn-aws/labs-self/w11-dr-hybrid/README.md) | Bước 1, 5 | Chọn **một** chiến lược DR rồi chứng minh hạ tầng khớp con số RTO/RPO đã cam kết — đúng vòng lặp ràng buộc → kiến trúc → kiểm chứng |
| [`labs-self/w12-exam-review/`](../../learn-aws/labs-self/w12-exam-review/README.md) | Toàn bộ | Chạy bảy bước trên kiến trúc capstone, rồi đối chiếu với bốn kiến trúc tham chiếu ở Phần 2 |

Luật chung của bộ lab và ba tầng hàng rào an toàn:
[`labs-self/README.md`](../../learn-aws/labs-self/README.md).

---

## Nguồn nói khác

| Chỗ | Nguồn `aws-saa-c03/` nói | Thực tế (2026-08) |
|---|---|---|
| Chương thiết kế | `README.md` của nguồn liệt kê các file F, G, H, I, J, N, O cho phần "giải bài toán theo kiến trúc" | **Không file nào tồn tại.** Toàn bộ phần phương pháp thiết kế bị thiếu; file bạn đang đọc lấp chỗ đó |
| Cách học | Nguồn tổ chức theo dịch vụ: học hết EC2 rồi sang S3 | Đề SAA không hỏi theo dịch vụ, nó hỏi theo tình huống. Học theo dịch vụ cho bạn từ vựng, không cho bạn phương pháp |
| DynamoDB on-demand | `Q-service-comparisons.md` ghi $1,25/triệu ghi và $0,25/triệu đọc | AWS **giảm 50% cuối 2024**: **$0,625/triệu WRU**, **$0,125/triệu RRU**. Bảng chi phí kiến trúc B ở trên dùng giá mới ([pricing](https://aws.amazon.com/dynamodb/pricing/)) |
| Well-Architected | Nguồn nhắc 5 trụ cột | **6 trụ cột** từ 2021, thêm Sustainability. Xem [`00-nen-tang.md`](00-nen-tang.md) mục 4 ([WA Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)) |

---

## Ngoài phạm vi

- **Cell-based architecture và shuffle sharding** — cách thu nhỏ blast radius xuống dưới mức AZ. Mức Professional. [Cell-based](https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html)
- **Domain-Driven Design và cách chia bounded context** — quyết định ranh giới service, không phải kiến thức AWS.
- **Lý thuyết hàng đợi cho capacity planning** (định luật Little, hệ số sử dụng) — hữu ích ngoài đời, không ra SAA.
- **FinOps như một chức năng tổ chức** — showback, chargeback, unit economics. [FinOps trên AWS](https://aws.amazon.com/aws-cost-management/)
- **AWS Fault Injection Service** — công cụ chạy game day tự động. Biết tên là đủ. [FIS](https://docs.aws.amazon.com/fis/latest/userguide/what-is.html)
- **Service mesh (App Mesh, Istio)** — chỉ đáng bàn khi đã có hàng chục service.
- **Mô hình cô lập multi-tenant** (silo, pool, bridge) — mức SaaS Lens. [SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html)

---

## Tự kiểm tra

Năm đề thiết kế mở. Không có đáp án đúng duy nhất — chấm bằng việc bạn có nêu được
ràng buộc, đánh đổi, và con số hay không. Dành 15 phút cho mỗi đề **trước** khi mở
lời giải, và viết ra bảng ràng buộc của bước 1 bằng tay.

**1.** Một chuỗi 12 rạp chiếu phim muốn bán vé online. Cao điểm là 30 phút sau khi
mở bán một bom tấn: khoảng 20.000 người vào cùng lúc, phần lớn chỉ xem sơ đồ ghế.
Không được bán trùng ghế. Ngoài các đợt mở bán, hệ thống gần như không có ai dùng.
Thiết kế hệ thống và giải thích từng lựa chọn.

<details><summary>Lời giải mẫu</summary>

**Bước 1 — ràng buộc.** Không bán trùng ghế: **cứng tuyệt đối**. Đỉnh 20.000 phiên
đồng thời trong 30 phút, phần còn lại của tháng gần như 0 — tỉ lệ đỉnh/trung bình
cỡ 100:1. Đọc/ghi rất lệch: hàng trăm nghìn lượt xem sơ đồ ghế, vài nghìn lượt đặt.

**Bước 2 — trục.** Trục chính là **phiên đồng thời trong cửa sổ ngắn và dự đoán
được**. Trục ghi thấp tuyệt đối: một rạp 200 ghế × 12 rạp × vài suất = dưới 100
ghi/giây kể cả lúc điên nhất.

**Bước 3 — dữ liệu.** Ghi thấp và cần ràng buộc toàn vẹn mạnh → **quan hệ**.
Unique constraint trên `(suat_id, ghe_id)` giải bài trùng ghế bằng một dòng schema,
và database từ chối lượt thứ hai. Aurora hoặc RDS PostgreSQL đều được.

**Bước 4 và 6 — đường đi và tiền.** Tách hai đường: sơ đồ ghế đọc rất nhiều →
CloudFront + cache ngắn (5–15 giây) hoặc ElastiCache; đặt vé đi thẳng vào database.
Vì tỉ lệ đỉnh/trung bình là 100:1, chi phí 0% tải là tiêu chí quyết định → compute
serverless hoặc ASG scale theo lịch (**giờ mở bán biết trước** — đây là món quà).

**Đánh đổi chính:** hàng đợi trước database (SQS + trang "đang xử lý") làm hệ thống
không bao giờ sập nhưng thêm độ trễ và một trạng thái người dùng phải hiểu. Chấp
nhận được cho bán vé, không chấp nhận được cho tra cứu.

**Chỗ lời giải khác cũng đúng:** (a) giữ ghế tạm bằng Redis với TTL 10 phút rồi mới
ghi database — nhanh hơn, nhưng thêm một nguồn sự thật thứ hai và bạn phải xử lý
trường hợp Redis chết giữa chừng; (b) scheduled scaling thay vì serverless — rẻ hơn
nếu đội đã quen EC2, và giờ mở bán biết trước nên rủi ro thấp; (c) DynamoDB với
conditional write cũng chặn được trùng ghế — đúng về kỹ thuật, nhưng đổi lấy khó
khăn ở báo cáo doanh thu ad-hoc, và ở tải này SQL không hề chật.
</details>

**2.** Một công ty logistics có 8.000 xe tải, mỗi xe gửi vị trí 10 giây một lần.
Điều phối viên cần thấy vị trí gần thời gian thực trên bản đồ. Cuối tháng, bộ phận
phân tích cần tính quãng đường và thời gian dừng của từng xe trong 24 tháng qua.
Thiết kế đường dữ liệu.

<details><summary>Lời giải mẫu</summary>

**Bước 1 và 2.** 8.000 xe ÷ 10 giây = **800 bản ghi/giây**, đều đặn 24/7, không có
đỉnh. Dung lượng: 800 × 86.400 × 30 ≈ 2 tỉ bản ghi/tháng — trục **dung lượng** là
trục chính, không phải throughput. "Gần thời gian thực" cần hỏi lại: điều phối viên
chịu được trễ 30 giây hay cần 1 giây? Hai câu trả lời ra hai kiến trúc.

**Bước 3.** Hai bài toán khác nhau, đừng ép một engine gánh cả hai:
- **Vị trí hiện tại** — 8.000 bản ghi, ghi đè liên tục, đọc theo key. DynamoDB một
  item mỗi xe, hoặc ElastiCache. Đây là dữ liệu **nhỏ và nóng**.
- **Lịch sử 24 tháng** — chỉ ghi thêm, đọc theo khoảng thời gian, truy vấn ad-hoc
  hàng tháng. S3 parquet phân vùng theo `dt=` và `xe_id`, truy vấn bằng Athena.

**Bước 4 và 6.** Thiết bị → Kinesis (800 rec/s → 1 shard đủ theo record rate, lấy
2–4 shard để dự phòng) → Lambda cập nhật vị trí hiện tại **và** Firehose ghi
lịch sử. Chi phí chính là Firehose theo GB và S3 tích luỹ; Athena rẻ nếu phân vùng
đúng, đắt gấp 20 lần nếu không.

**Đánh đổi chính:** viết lịch sử vào database vận hành (một bảng 48 tỉ dòng) là
lựa chọn tự nhiên nhất và sai nhất — nó làm chậm truy vấn nóng và làm backup thành
cơn ác mộng. Tách nóng/lạnh là quyết định quan trọng nhất của bài này.

**Chỗ lời giải khác cũng đúng:** (a) nếu "gần thời gian thực" nghĩa là 30 giây, bỏ
Kinesis, cho thiết bị ghi thẳng qua API Gateway → Lambda; rẻ hơn và ít bộ phận hơn;
(b) Timestream thay cho S3+Athena nếu truy vấn chuỗi thời gian là chính — đổi tiện
lợi lấy chi phí cao hơn và một dịch vụ nữa phải học; (c) giữ 90 ngày gần nhất trong
Aurora cho truy vấn nhanh, phần cũ hơn ở S3 — kiến trúc hai tầng, phức tạp hơn nhưng
phục vụ được cả điều phối lẫn phân tích gần.
</details>

**3.** Một SaaS quản lý nhân sự có 300 khách hàng doanh nghiệp, khách lớn nhất
chiếm 40% lưu lượng. Ba tháng gần đây, cứ khi khách lớn chạy báo cáo cuối tháng thì
299 khách còn lại bị chậm. Ngân sách không tăng. Sửa kiến trúc thế nào?

<details><summary>Lời giải mẫu</summary>

**Chẩn đoán trước, kiến trúc sau.** Đây là bài toán **cô lập**, không phải bài toán
capacity — thêm máy chỉ đẩy ngày tái phát ra xa. Bước 5 (ranh giới hỏng) là bước
đang thiếu: blast radius của một tenant hiện là 100% tenant.

**Ba cách sửa, theo thứ tự chi phí tăng dần:**
1. **Tách đường đọc nặng khỏi đường vận hành.** Báo cáo cuối tháng chạy trên read
   replica hoặc trên bản sao ở S3+Athena. Rẻ nhất, sửa được đúng triệu chứng đã mô
   tả, và không đụng vào mô hình dữ liệu.
2. **Giới hạn theo tenant.** Throttle ở API Gateway theo usage plan, hoặc hàng đợi
   riêng cho job nặng với concurrency trần. Chặn được cả những kiểu quá tải chưa
   xảy ra.
3. **Phân vùng tenant lớn ra hạ tầng riêng** (bridge model): khách chiếm 40% lưu
   lượng có database riêng. Đắt nhất, nhưng cũng là thứ khách trả tiền cao sẵn sàng
   trả thêm.

**Đánh đổi chính:** cách 1 rẻ và nhanh nhưng chỉ chữa loại tải đã biết. Cách 3
triệt để nhưng biến một hệ thống thành N hệ thống phải vận hành, và ngân sách không
tăng nên nó phải đi kèm một cuộc nói chuyện về giá bán.

**Chỗ lời giải khác cũng đúng:** (a) nếu đo ra nghẽn là một vài query thiếu index
thì sửa index là câu trả lời đúng và tốn $0 — luôn đo trước; (b) chuyển báo cáo
sang chạy bất đồng bộ (bấm nút, nhận email khi xong) đổi trải nghiệm lấy sự ổn
định, thường được chấp nhận cho báo cáo cuối tháng; (c) lên lịch cho báo cáo nặng
chạy ngoài giờ cao điểm — thô sơ nhưng giải quyết được ngay trong tuần này.
</details>

**4.** Một cơ quan nhà nước có ứng dụng nội bộ 2.000 nhân viên dùng, chỉ trong giờ
hành chính, chỉ từ mạng nội bộ. Hạ tầng hiện tại là 6 máy ảo on-prem sắp hết vòng
đời. Yêu cầu: lên cloud trong 3 tháng, dữ liệu không được rời lãnh thổ, chi phí phải
thấp hơn phương án mua máy mới. Thiết kế.

<details><summary>Lời giải mẫu</summary>

**Bước 1 — ràng buộc quyết định là ràng buộc ẩn.** Ba tháng là lịch cứng. "Dữ liệu
không rời lãnh thổ" loại mọi kiến trúc multi-Region xuyên biên giới, và cũng ràng
buộc cả nơi để backup. "Chỉ từ mạng nội bộ" là ràng buộc kiến trúc mạng, không phải
ràng buộc bảo mật chung chung: không cần endpoint public.

**Bước 2 — trục.** Không trục nào tăng. 2.000 người dùng giờ hành chính, dữ liệu
vài trăm GB, tăng chậm. Đây là hệ thống **không cần scale**, và mọi thiết kế cho
scale ở đây là lãng phí ngân sách đang bị soi.

**Bước 3 và lựa chọn chiến lược.** Với lịch 3 tháng, **rehost** (lift-and-shift)
là lựa chọn đúng: EC2 tương ứng 6 máy ảo, database sang RDS nếu engine tương thích.
Refactor sang serverless nghe hấp dẫn nhưng không vừa lịch và không có ràng buộc
nào đòi hỏi nó. Xem 7R ở [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) mục 8.

**Bước 5 và 6.** Truy cập qua Site-to-Site VPN hoặc Direct Connect, ALB internal,
không có gì public. Multi-AZ cho database. Vì chỉ chạy giờ hành chính, **tắt tài
nguyên ngoài giờ** cắt khoảng 65% chi phí compute — đây là đòn bẩy chi phí lớn
nhất của bài này và nó chỉ dùng được vì bước 1 ghi rõ "chỉ giờ hành chính".

**Đánh đổi chính:** rehost không tận dụng được cloud, và sau 12 tháng sẽ có người
hỏi vì sao vẫn phải vá OS. Đó là món nợ có ý thức, đổi lấy việc kịp lịch — và
phương án đúng là ghi nó vào backlog ngay lúc quyết định, kèm một mốc thời gian.

**Chỗ lời giải khác cũng đúng:** (a) VPN thay Direct Connect nếu 2.000 người dùng
chủ yếu làm việc với văn bản — DX mất hàng tuần đến hàng tháng để cung cấp và có
thể không kịp lịch 3 tháng; (b) Elastic Beanstalk hoặc ECS on Fargate nếu ứng dụng
đóng gói được dễ, cho ít việc vá OS hơn mà vẫn kịp lịch; (c) giữ một máy on-prem
làm dự phòng trong 6 tháng đầu — không đẹp về kiến trúc, nhưng là quản trị rủi ro
hợp lý cho một lần di trú có hạn chót cứng.
</details>

**5.** Bạn được giao một kiến trúc do người khác vẽ: CloudFront → ALB → ECS Fargate
→ Aurora Multi-AZ, nhân bản sang Region thứ hai bằng Aurora Global Database, tất cả
trong một AWS account, không có alarm nào ngoài `CPUUtilization`. Nêu ba vấn đề lớn
nhất theo thứ tự ưu tiên và cách sửa.

<details><summary>Lời giải mẫu</summary>

**Vấn đề 1 — không có cách biết nó hỏng (bước 7).** `CPUUtilization` là metric ít
nói nhất trong toàn bộ danh sách: hệ thống có thể trả 100% lỗi 5xx với CPU 15%.
Thiếu tối thiểu: `HTTPCode_ELB_5XX_Count`, `TargetResponseTime` p99,
`UnHealthyHostCount`, `DatabaseConnections`, `ReplicaLag`, và alarm trên DLQ nếu
có hàng đợi. Sửa trước tiên vì nó rẻ nhất và vì mọi vấn đề khác đều **vô hình** cho
tới khi có nó.

**Vấn đề 2 — một account cho tất cả (bước 5).** Ranh giới account là ranh giới duy
nhất chặn lỗi con người và là ranh giới của service quota. Multi-Region mà chung
account nghĩa là một `terraform apply` sai, một khoá bị lộ, hoặc một quota cạn có
thể lấy đi **cả hai** Region cùng lúc — tức là toàn bộ khoản đầu tư multi-Region
bị vô hiệu bởi một chế độ hỏng mà nó không hề chống.

**Vấn đề 3 — multi-Region có thể chưa được biện minh (bước 1).** Aurora Global
Database khoảng gấp đôi chi phí tầng dữ liệu cộng phí sao chép. Câu hỏi phải hỏi:
RTO/RPO nào buộc phải có nó, và ai ký con số đó? Nếu không có câu trả lời bằng số,
đây là chi phí lớn nhất trong kiến trúc mà không ràng buộc nào đòi.

**Đánh đổi và chỗ lời giải khác cũng đúng:** thứ tự trên đúng khi hệ thống **đã**
chạy production — quan sát trước, ranh giới sau, chi phí sau nữa. Nếu hệ thống chưa
lên production, đảo vấn đề 2 lên đầu là hợp lý hơn, vì tách account sau khi đã có
dữ liệu thật khó hơn nhiều lần. Một người khác có thể xếp "chưa biết cache behavior
của CloudFront có đúng không" lên trên vấn đề 3 nếu chi phí egress là dòng lớn nhất
trên hoá đơn — đó cũng là một lập luận có số đỡ lưng, và nó đúng.
</details>
