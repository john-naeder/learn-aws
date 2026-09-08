# Hạ tầng lai, biên, và danh tính doanh nghiệp

> **Tra nhanh:** đề mô tả một ràng buộc **nằm ngoài kỹ thuật** — luật bắt dữ liệu ở lại
> trong toà nhà, dây chuyền chịu tối đa 10 ms, giấy phép Oracle tính theo lõi vật lý,
> thiết bị cũ chỉ nói được SFTP — và bạn phải chọn đúng mảnh hạ tầng AWS đặt **ngoài
> Region** để thoả ràng buộc đó, rồi nối nó về Region cho đúng.

`Domain 1 · Design Secure Architectures (30% đề)` · `Domain 3 · Design High-Performing Architectures (24% đề)`

Đây là nhóm dịch vụ kỳ lạ nhất của AWS: gần như **không cái nào được chọn vì nó nhanh
hơn hay rẻ hơn**. Bạn chọn Outposts vì luật cấm dữ liệu rời khỏi toà nhà. Bạn chọn
Storage Gateway vì phần mềm backup mua năm 2011 chỉ biết ghi băng từ. Ràng buộc luôn là
**con người, hợp đồng, hoặc vật lý**.

Hệ quả cho cách đọc: với mỗi dịch vụ, đừng hỏi "nó làm gì". Hỏi **"nó gỡ ràng buộc nào,
và nó tạo ra ràng buộc mới nào"**. Outposts gỡ ràng buộc trú ngụ dữ liệu nhưng tạo ràng
buộc mới — phải giữ một đường mạng sống 24/7 về Region cha, nếu không control plane
chết. Đó mới là câu hỏi thiết kế thật.

Phần **đường ống** (Direct Connect, VPN, DX Gateway) đã nằm ở
[`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md#11-hybrid--direct-connect-vpn-và-dùng-cả-hai);
phần **VPC và Transit Gateway** ở [`04-networking.md`](04-networking.md#4-peering-và-transit-gateway).
File này lo **những gì cắm vào hai đầu đường ống**.

---

## Bản đồ

| Mục | Khi nào bạn cần đọc mục này |
|---|---|
| [1. Năm ràng buộc](#1-năm-ràng-buộc-sinh-ra-cả-nhóm-dịch-vụ-này) | Đề mô tả một hoàn cảnh và bạn cần biết nó thuộc nhóm nào |
| [2. Outposts](#2-outposts--giá-máy-của-aws-đặt-trong-phòng-máy-của-bạn) | "must remain on-premises", "data residency in our own facility" |
| [3. Local Zones và Wavelength](#3-local-zones-và-wavelength--cùng-cơ-chế-khác-chủ-chỗ-đặt) | Ba thứ biên hay bị nhầm, phân biệt bằng ai sở hữu chỗ đặt |
| [4. VMware Cloud on AWS](#4-vmware-cloud-on-aws--nguyên-cụm-vsphere-chuyển-chỗ) | "hundreds of VMs", "keep the same vSphere tooling" |
| [5. Container lai](#5-container-lai--ecs-anywhere-eks-anywhere-eks-distro) | Chạy container ở on-prem mà vẫn quản lý từ AWS |
| [6. Bốn loại Storage Gateway](#6-storage-gateway--cột-giao-thức-là-cột-quyết-định) | Đề nói NFS / SMB / iSCSI / tape và bạn phải chọn đúng loại |
| [7. Bốn cách chuyển dữ liệu](#7-datasync-vs-storage-gateway-vs-transfer-family-vs-snow) | Đề cho dung lượng, băng thông, deadline, giao thức |
| [8. Sơ đồ kiến trúc lai](#8-một-kiến-trúc-lai-thật--đặt-cái-gì-ở-đâu) | Cần thấy cả bức tranh: cái gì ở on-prem, cái gì ở VPC |
| [9. Client VPN](#9-client-vpn--kết-nối-người-không-phải-kết-nối-site) | "remote employees", "laptop", "work from home" |
| [10. Directory Service](#10-directory-service--ba-lựa-chọn-một-câu-hỏi-quyết-định) | Đề nhắc Active Directory, domain join, WorkSpaces, RDS SQL Server |
| [11. License Manager](#11-license-manager--giấy-phép-tính-theo-lõi-vật-lý) | "BYOL", "licensed per physical core", "license audit" |
| [12. RAM](#12-ram--chia-sẻ-tài-nguyên-không-chia-sẻ-quyền) | Nhiều account cần dùng chung subnet, TGW, license, Outpost |
| [13. Hai bài toán thiết kế](#13-hai-bài-toán-thiết-kế-có-lời-giải) | Muốn xem cách ghép các mảnh thành một đáp án |
| [Bảng số phải nhớ](#bảng-số-phải-nhớ) | 30 phút trước giờ thi |

Liên quan: [nền tảng](00-nen-tang.md#2-bốn-ranh-giới-phải-thuộc-lòng),
[storage](02-storage.md#16-storage-gateway-và-aws-backup), [networking](04-networking.md),
[security](05-security.md#7-iam-identity-center), [DR và migration](13-khoi-phuc-tham-hoa.md),
[tuần 11](../aws/w11-dr-hybrid.md).

---

## 1. Năm ràng buộc sinh ra cả nhóm dịch vụ này

Học nhóm này theo ràng buộc, không theo tên dịch vụ. Đề mô tả ràng buộc bằng tiếng Anh
doanh nghiệp; việc của bạn là nhận ra nó thuộc dòng nào.

| Ràng buộc | Nó nghe như thế nào trong đề | Vì sao dịch vụ Region không giải được | Nhóm đáp án |
|---|---|---|---|
| **Trú ngụ dữ liệu** — bit không được rời một toà nhà, một quốc gia | *"data must remain within our facility"*, *"must not leave the country"* | Region gần nhất vẫn là **cơ sở của AWS**, và có thể ở nước khác | Outposts; Local Zone nếu ràng buộc là quốc gia chứ không phải toà nhà |
| **Độ trễ vật lý** — tốc độ ánh sáng không thương lượng | *"sub-10-millisecond latency to factory equipment"*, *"real-time control loop"* | 100 km cáp ≈ **1 ms một chiều**. Region cách 500 km đã là 10–15 ms khứ hồi | Outposts (tới máy trong nhà máy), Local Zone (tới người trong thành phố), Wavelength (tới thiết bị 5G) |
| **Giấy phép** — tính theo lõi vật lý hoặc socket | *"our Oracle licenses are per physical core"*, *"BYOL"*, *"license audit"* | Instance chia sẻ không cho bạn **thấy** hay **cố định** lõi vật lý | Dedicated Host + License Manager |
| **Giao thức thiết bị cũ** | *"the application only supports NFS"*, *"backup software writes to tape"*, *"partners upload via SFTP"* | S3 nói HTTPS/REST. Ứng dụng cũ không sửa được, hoặc sửa thì mất bảo hành | Storage Gateway (NFS/SMB/iSCSI/VTL), Transfer Family (SFTP/FTPS/FTP/AS2) |
| **Băng thông** — đường truyền không đủ, hoặc không có | *"remote research station"*, *"limited connectivity"*, *"petabytes"* | Đường truyền là ràng buộc cứng; xem phép tính ở [mục 7](#phép-tính-quyết-định-mạng-hay-thiết-bị) | Snow Family, DataSync có throttle |

Ràng buộc thứ sáu ít lộ hơn nhưng ra thi nhiều: **danh tính**. Doanh nghiệp đã có Active
Directory chạy 15 năm với chính sách mật khẩu và group được kiểm toán. Không ai bỏ nó để
tạo lại user trong IAM. Directory Service tồn tại vì lý do đó — mục
[10](#10-directory-service--ba-lựa-chọn-một-câu-hỏi-quyết-định).

---

## 2. Outposts — giá máy của AWS đặt trong phòng máy của bạn

### Cơ chế: ai sở hữu cái gì

- **Phần cứng thuộc sở hữu của AWS.** Bạn không mua. AWS giao, lắp, thay ổ hỏng, vá
  firmware từ xa. Bạn cung cấp **chỗ đặt, điện, làm mát, đường mạng**, và quyền cho kỹ
  sư AWS vào thay phần cứng.
- **Data plane chạy tại chỗ.** EC2, EBS, subnet của VPC nằm trên giá máy trong phòng máy
  bạn. Gói tin giữa hai instance cùng Outpost **không rời khỏi toà nhà**.
- **Control plane vẫn ở Region cha.** `RunInstances` cho subnet Outpost đi **về Region**,
  xử lý ở đó, rồi Region ra lệnh ngược xuống. Không có API endpoint cục bộ.
- **Service link** là đường hầm mã hoá về Region cha — qua internet, public VIF hoặc
  private VIF của Direct Connect. Nó mang **lệnh điều khiển, metric, log, và traffic tới
  dịch vụ Region**. AWS khuyến nghị tối thiểu **500 Mbps**, nên thiết kế **1 Gbps trở
  lên** và có hai đường độc lập.
- **Local gateway (LGW)** — chỉ có ở Outposts rack — là cửa ra thẳng LAN của bạn,
  **không qua Region**, dùng **customer-owned IP pool (CoIP)** lấy từ dải địa chỉ của
  chính bạn. Đây là đường mà nhà máy và hệ thống cũ nói chuyện với instance trên Outpost.

```
   trung tâm dữ liệu của BẠN
   máy CNC, ERP cũ ══LAN══▶ Local Gateway (CoIP) ══▶ Outposts rack     ← đường DỮ LIỆU
                                                     EC2 · EBS · subnet   < 1 ms, không rời toà nhà
                                                          │
                                                          │ service link (mã hoá)   ← đường ĐIỀU KHIỂN
                                                          ▼
                                        Region cha: EC2 API · IAM/STS · CloudWatch · S3
```

### Mất service link thì cái gì còn chạy

Đây là câu hỏi thiết kế thật, và tài liệu tiếp thị không trả lời nó.

| Trong lúc service link chết | Trạng thái |
|---|---|
| EC2 đang chạy, EBS volume, local gateway, container ECS, RDS đang chạy | **Chạy tiếp bình thường**, truy cập được qua LAN |
| `RunInstances`, `StartInstances`, `TerminateInstances` | **Hỏng** — mọi lệnh EC2 là lệnh gửi về Region |
| Bất cứ thao tác nào cần **IAM authorization**, kể cả đọc S3 ở Region | **Hỏng** |
| Systems Manager quản lý instance | **Hỏng** |
| RDS backup tự động, tự thay instance hỏng | **Dừng** |
| CloudWatch metric và log | **Cache tại chỗ tối đa 7 ngày**, đẩy lên khi nối lại. Quá 7 ngày là mất |
| Phân giải tên (Route 53 zone ở Region) | **Hỏng**, trừ khi đã đặt **Route 53 Resolver ngay trên Outpost** |
| Instance dùng service link để ra internet | **Mất internet** |

Ba hệ quả thiết kế, và đây là chỗ đề mức khó gài:

1. **Outposts không phải giải pháp cho môi trường mất kết nối.** AWS nói thẳng điều này.
   Đề mô tả *"a site with no reliable network connectivity"* thì Outposts là đáp án
   **sai** — đáp án là Snow Family hoặc kiến trúc tự chủ.
2. **Đừng để đường sống còn của ứng dụng đi qua control plane.** Auto Scaling trên
   Outpost cần API ở Region, nên nó không cứu bạn khi mất link. Ứng dụng chịu được mất
   link là ứng dụng **đã có sẵn đủ instance đang chạy**.
3. **Phân giải tên là điểm hỏng bị quên nhiều nhất.** Ứng dụng nối database bằng tên
   DNS, DNS ở Region, mất link là mất DNS — dù database vẫn chạy cách đó hai mét.

### Hai form factor và con số

| | Outposts rack | Outposts server (1U / 2U) |
|---|---|---|
| Kích thước, điện | **42U**; gen 1 **5–15 kVA**, gen 2 compute rack **10–30 kVA** + network rack **8,89 kVA** | 1U/2U lắp vào tủ 19″ có sẵn, điện như một server thường |
| Uplink | gen 1: 1/10/40/100 Gbps · gen 2: 10/40/100 Gbps | Ethernet thường |
| Mở rộng | tới **96 rack**; từ **4 compute rack** trở lên bắt buộc có **ACE rack** (Aggregation, Core, Edge) làm điểm gom mạng | Không mở rộng, mỗi server độc lập |
| Local gateway, EBS | **Có cả hai** | Không LGW (dùng **LNI**), không EBS — chỉ **instance store NVMe** |
| Trạng thái 2026-08 | gen 2 đang bán | **AWS đã ngừng bán cho khách hàng mới** |

Khách dùng Outposts rack **bắt buộc có Enterprise Support** — một lý do Outposts hiếm
khi là đáp án của câu có chữ `cost-effective`.

Dịch vụ chạy được **trên** Outpost là tập con hẹp: EC2, EBS, ECS, EKS worker node, RDS,
ElastiCache, EMR, S3 on Outposts, Application Load Balancer. Lambda, DynamoDB, SQS,
CloudFront chỉ tồn tại ở **Region**, và gọi chúng từ Outpost là gọi qua service link.

---

## 3. Local Zones và Wavelength — cùng cơ chế, khác chủ chỗ đặt

Ba thứ Outposts / Local Zone / Wavelength dùng **cùng một cơ chế**: một mảnh hạ tầng AWS
ngoài Region, control plane ở Region cha, bạn **mở rộng VPC** xuống đó bằng cách tạo
subnet trong zone. Khác nhau ở đúng hai trục, và đó là hai trục đề dùng để phân biệt.

| | Outposts | Local Zone | Wavelength Zone |
|---|---|---|---|
| **Ai sở hữu chỗ đặt** | **Bạn** — phòng máy, nhà máy, chi nhánh của bạn | **AWS** — cơ sở AWS đặt ở một thành phố lớn | **Nhà mạng** — bên trong trung tâm dữ liệu 5G của carrier |
| **Độ trễ thấp tới ai** | Máy móc **trong chính toà nhà đó** | Người dùng **trong vùng đô thị đó** | Thiết bị **đang bám vào mạng 5G của carrier đó** |
| Mã định danh | ARN Outpost, subnet gắn `--outpost-arn` | `us-east-1-bos-1a`, `us-west-2-den-1a` | `us-east-1-wl1-bos-wlz-1` |
| Cửa ra mạng ngoài | **Local gateway** ra LAN của bạn | Internet gateway như AZ thường | **Carrier gateway**, dùng **carrier IP** |
| Trả tiền kiểu gì | Thuê nguyên giá máy theo tháng, dùng hay không cũng trả | Theo giờ như EC2, **bật zone miễn phí** | Theo giờ, **chỉ On-Demand**, không có Reserved Instance |
| Giải ràng buộc nào | Trú ngụ dữ liệu mức **toà nhà**; độ trễ tới máy móc | Độ trễ tới người dùng một thành phố; trú ngụ mức **quốc gia** | Độ trễ tới ứng dụng di động, không qua internet công cộng |

**Local Zone không phải edge location.** Edge location chạy CloudFront, Route 53, Global
Accelerator — không chạy EC2 của bạn. Local Zone **chạy EC2, EBS, ALB, ECS, EKS thật**,
và ở một số thành phố có thêm RDS, ElastiCache, FSx, EMR, S3. Đề nói *"dựng hình video
cần GPU, độ trễ dưới 10 ms tới studio ở Los Angeles"* → Local Zone, vì CloudFront không
chạy được phần mềm của bạn. Xem
[`00-nen-tang.md`](00-nen-tang.md#local-zone-wavelength-outposts--chỉ-cần-nhận-diện-từ-khóa).

**Wavelength chỉ nhanh cho thiết bị trên mạng của đúng nhà mạng đó.** Gói tin từ điện
thoại **không phải trèo qua internet** — nó dừng ngay tại trạm biên của carrier. Người
dùng vào bằng Wi-Fi nhà mình thì Wavelength **không giúp gì**. Đề không nhắc `5G` hoặc
`mobile` thì Wavelength gần như chắc chắn sai.

Cả Local Zone lẫn Wavelength Zone đều phải **opt-in**; chúng không tự xuất hiện trong
`describe-availability-zones`.

---

## 4. VMware Cloud on AWS — nguyên cụm vSphere chuyển chỗ

Cơ chế: AWS cấp **EC2 bare metal** (máy vật lý, không có hypervisor của AWS ở giữa),
VMware cài nguyên bộ SDDC lên đó — ESXi, vSAN, NSX, vCenter. Bạn nhận được một vCenter y
hệt cái đang có, và **VM chuyển qua không cần convert**: cùng định dạng, cùng công cụ,
cùng license Windows đang có. Cụm nối vào VPC qua ENI tốc độ cao, nên VM gọi được RDS,
S3, Lambda như tài nguyên nội bộ.

Ràng buộc nó giải: **thời gian và kỹ năng, không phải kỹ thuật.** 800 VM và hợp đồng
thuê phòng máy hết hạn sau 4 tháng — rehost từng máy bằng MGN mất hàng tháng và phải
kiểm thử từng ứng dụng, còn chuyển nguyên tầng hạ tầng thì đội vận hành giữ nguyên kỹ
năng. Đây đúng là chữ **Relocate** trong 7R, xem
[`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md#8-7r--bảy-cách-đưa-một-ứng-dụng-lên-cloud).

Dấu hiệu trong đề: *"hundreds of VMs"* + *"keep using the same tooling"* + *"minimal
application changes"* + *"datacenter lease expires"*. Ba trong bốn dấu hiệu cùng lúc thì
đó là VMware Cloud on AWS chứ không phải MGN.

Trạng thái thật, tính đến 2026-08: **từ 30/04/2024 AWS và các đối tác kênh không còn bán
lại VMware Cloud on AWS**; dịch vụ tiếp tục tồn tại nhưng do Broadcom bán và hỗ trợ. Đề
SAA-C03 vẫn hỏi theo mô hình cũ.

---

## 5. Container lai — ECS Anywhere, EKS Anywhere, EKS Distro

Ba cái tên gần giống nhau, khác nhau ở một trục duy nhất: **control plane nằm ở đâu**.

| | ECS Anywhere | EKS Anywhere | EKS Distro |
|---|---|---|---|
| Control plane ở đâu | **Ở Region AWS** — cluster ECS thật, máy của bạn chỉ là capacity | **Ở hạ tầng của bạn** — cụm Kubernetes hoàn chỉnh tại chỗ | Không có gì cả — đây chỉ là **bộ binary và container image** |
| AWS chịu trách nhiệm gì | Vận hành control plane, lập lịch task, khởi động lại container chết | Cung cấp phần mềm, bản vá, hỗ trợ có đăng ký | **Chỉ dựng và kiểm thử bản phân phối.** Không có support plan |
| Cần gì trên máy của bạn | **SSM Agent** + **ECS container agent** + Docker | Máy chủ theo provider: bare metal (Tinkerbell), vSphere, CloudStack, Nutanix | Tuỳ bạn |
| Đăng ký kiểu gì | Launch type **`EXTERNAL`**, sau khi máy đã thành SSM managed instance | `eksctl anywhere create cluster` từ một máy quản trị | Không đăng ký với AWS |
| Chịu được mất mạng | **Không hoàn toàn** — cần kết nối ổn định tới control plane. Agent khởi động lại container theo restart policy, nhưng agent chết là hết | **Có** — cụm tự đủ | Có |
| ELB, awsvpc networking, Fargate | **Không.** Task chạy `bridge` hoặc `host`, không có ENI trong VPC, không gắn vào ALB/NLB | Không, dùng thành phần Kubernetes tại chỗ | Không |

- **ECS Anywhere là "AWS lập lịch, máy của bạn chạy".** Hợp khi bạn đã dùng ECS trên
  cloud và muốn **một mặt phẳng quản lý duy nhất** cho cả hai nơi. Cái giá là phụ thuộc
  kết nối: mất link là mất khả năng triển khai và tự phục hồi ở mức lập lịch.
- **EKS Anywhere là "bạn chạy tất cả, AWS bán phần mềm và hỗ trợ".** Hợp khi ràng buộc
  là **phải tự chủ hoàn toàn** — trạm ngoài khơi, tàu biển, cơ sở air gap.
- **EKS Distro là nguyên liệu, không phải món ăn.** Chính những binary Kubernetes, etcd,
  CNI, CSI mà EKS dùng, đã kiểm thử tương thích, phát hành trên GitHub/ECR/S3. EKS
  Anywhere được xây **trên** nó.

Thứ tư dễ lẫn: **EKS Connector** chỉ **đăng ký một cụm Kubernetes bất kỳ để nhìn thấy nó
trong console EKS** — không quản lý, không thay đổi được gì. Đề nói *"view all our
clusters in one place"* → EKS Connector; *"run and manage"* → EKS Anywhere. Phần container
ở Region: [`01-compute.md`](01-compute.md#10-ecs-eks-fargate).

---

## 6. Storage Gateway — cột giao thức là cột quyết định

### Cơ chế chung

Storage Gateway là một **appliance đặt tại chỗ**: VM trên VMware ESXi, Hyper-V hoặc KVM;
hoặc **thiết bị phần cứng** của AWS; hoặc **EC2 instance** (khi nguồn dữ liệu đã ở trong
AWS). Appliance đó (1) **nói giao thức của máy tại chỗ**, (2) **giữ cache cục bộ** để dữ
liệu nóng đọc ở tốc độ LAN, (3) **đẩy phần còn lại lên AWS** qua HTTPS bất đồng bộ, có
**upload buffer** chịu được lúc đường truyền chậm hoặc đứt tạm thời.

Ba đặc điểm này là lý do Storage Gateway **không phải công cụ di trú**. Nó là **thành
phần thường trực**: sau khi dữ liệu lên cloud, appliance vẫn ở đó và ứng dụng cũ vẫn nói
chuyện qua nó, mãi mãi.

### Bốn loại

Đọc bảng theo **cột đầu tiên**. Đề hầu như luôn cho bạn giao thức, và giao thức quyết
định loại — ba cột còn lại chỉ để xác nhận.

| Máy tại chỗ nói giao thức gì | Loại gateway | Dữ liệu **chính** nằm ở đâu | Cache nằm ở đâu | Bài toán nó giải |
|---|---|---|---|---|
| **NFS** hoặc **SMB** (file) | **Amazon S3 File Gateway** | **S3** — mỗi file thành **một object đọc thẳng được** từ S3 | Đĩa local, chỉ dữ liệu nóng | Ứng dụng cũ chỉ biết file share nhưng bạn muốn dữ liệu vào data lake để Athena/Glue dùng được. Backup, archive, ingest |
| **SMB** (chỉ SMB) | **Amazon FSx File Gateway** | **FSx for Windows File Server** — file share Windows thật, có ACL và DFS | Đĩa local, tối ưu cho truy cập tương tác lặp lại | File share Windows của một chi nhánh: nhiều người mở, sửa, lưu cùng tập tài liệu suốt ngày, cần độ trễ như ổ mạng nội bộ |
| **iSCSI** (block) — **cached** | **Volume Gateway, chế độ cached** | **AWS** | Đĩa local giữ **phần nóng** | Bạn muốn **giảm dung lượng lưu trữ tại chỗ**; dataset lớn hơn đĩa local, chỉ phần hay dùng cần nhanh |
| **iSCSI** (block) — **stored** | **Volume Gateway, chế độ stored** | **On-premises** — toàn bộ dataset trên đĩa của bạn | Không có cache; AWS giữ bản sao bất đồng bộ dạng **EBS snapshot** | Bạn cần **độ trễ thấp cho toàn bộ dataset**, AWS chỉ làm nơi backup và điểm khôi phục sang EC2 |
| **iSCSI VTL** (băng từ ảo) | **Tape Gateway** | **S3 / Glacier Flexible Retrieval / Deep Archive** | Đĩa local cho tape đang ghi | Phần mềm backup (Veeam, NetBackup, Commvault) ghi ra thư viện băng từ; bỏ băng vật lý mà **không đổi phần mềm và quy trình** |

### Stored so với cached — chỗ sai nhiều nhất

Cách nhớ chắc chắn: **tên chế độ mô tả cái nằm ở on-premises.**

- **Stored** = dữ liệu **được lưu (stored) tại chỗ**. Toàn bộ dataset ở on-prem, AWS chỉ
  có bản sao. Đề nói *"low-latency access to the entire dataset"* → stored.
- **Cached** = tại chỗ chỉ có **cache**, dữ liệu chính ở AWS. Đề nói *"reduce
  on-premises storage footprint"*, *"minimize local storage"* → cached.

Hệ quả khôi phục: chế độ stored cho bạn **EBS snapshot** dùng ngay để dựng EC2 ở Region —
kịch bản DR rẻ nhất của Volume Gateway. Với cached, dữ liệu vốn đã ở AWS, nên "khôi phục"
là dựng gateway mới trỏ vào cùng volume.

### Con số ra thi

| Hạng mục | Con số (kiểm tra 2026-08) |
|---|---|
| Cache tối thiểu / tối đa (Volume Gateway cached) | **150 GiB** / **64 TiB** |
| Upload buffer tối thiểu / tối đa | **150 GiB** / **2 TiB** |
| Volume lớn nhất — cached / stored | **32 TiB** / **16 TiB** |
| Số volume mỗi gateway; tổng dung lượng cached / stored | **32**; **1.024 TiB** / **512 TiB** |
| Tape ảo: kích thước; số tape; tổng | **100 GiB → 15 TiB**; **1.500**; **1 PiB**. Tape trong archive: **không giới hạn** |
| S3 File Gateway: file share mỗi gateway; file lớn nhất | **50**; **5 TiB** (bằng trần object của S3) |
| Số file gateway cache metadata cùng lúc | Small **5 triệu** · Medium **10 triệu** · Large **20 triệu** |

Bẫy về snapshot: snapshot tạo từ **cached volume lớn hơn 16 TiB** khôi phục về Storage
Gateway volume được nhưng **không** thành EBS volume được, vì trần EBS là 16 TiB. Nếu kế
hoạch DR là "dựng EC2 từ snapshot" thì volume phải ≤ 16 TiB.

Bản tóm tắt ngắn hơn nằm ở [`02-storage.md`](02-storage.md#16-storage-gateway-và-aws-backup);
mục này bổ sung cơ chế cache/upload buffer và toàn bộ quota.

---

## 7. DataSync vs Storage Gateway vs Transfer Family vs Snow

| | **DataSync** | **Storage Gateway** | **Transfer Family** | **Snow Family** |
|---|---|---|---|---|
| **Một lần hay liên tục** | Một lần **hoặc theo lịch**. Có điểm kết thúc | **Thường trực.** Không bao giờ "xong" | **Thường trực**, nhưng do **bên ngoài** đẩy vào | **Một lần**, dứt điểm |
| **Ai khởi xướng** | Bạn, qua task có lịch | Ứng dụng tại chỗ, mỗi lần đọc/ghi file | **Đối tác, khách hàng, hệ thống bên thứ ba** | Bạn, thủ công |
| **Giao thức phía ngoài AWS** | NFS, SMB, HDFS, object storage (kể cả S3 của cloud khác) | NFS, SMB, iSCSI, iSCSI VTL | **SFTP, FTPS, FTP, AS2**, và trình duyệt | Không có — chép vào thiết bị qua NFS hoặc S3 API cục bộ |
| **Đích** | S3, EFS, FSx (mọi biến thể) | S3, FSx for Windows, EBS snapshot, Glacier | **S3 hoặc EFS** | S3 |
| **Băng thông cần** | Đường truyền tốt; một agent đạt cỡ **10 Gbps** nếu mạng và storage cho phép. Có **throttle** để không bóp nghẹt đường chung | Cần **ổn định** hơn là lớn — cache và upload buffer chịu được lúc chậm | Nhỏ, phụ thuộc đối tác | **Không cần đường truyền nào** |
| **Khi nào đường truyền không khả thi** | Khi phép tính ra số ngày lớn hơn deadline | Không áp dụng — không phải công cụ chuyển khối lượng lớn | Không áp dụng | **Đây chính là lý do nó tồn tại** |
| **Giá (us-east-1, 2026-08)** | **~$0,0125/GB**; task Enhanced mode tính thêm phí mỗi lần chạy | Phí dung lượng lưu ở AWS + phí gateway | **$0,30/giờ mỗi giao thức bật** + **$0,04/GB** lên và xuống | Phí thuê thiết bị + vận chuyển |
| **Từ khoá trong đề** | *"recurring sync"*, *"millions of files"*, *"verify data integrity"*, *"NFS to S3"* | *"continue to access as a file share"*, *"iSCSI volumes"*, *"replace the tape library"* | *"partners upload via SFTP"*, *"legacy FTP workflow"*, *"EDI"* | *"remote location"*, *"limited connectivity"*, *"would take months over the network"* |

Ba câu hỏi loại nửa, hỏi theo đúng thứ tự:

1. **"Sau khi xong, on-prem còn phải đọc dữ liệu đó không?"** Còn → **Storage Gateway**,
   dừng ở đây. Đây là câu mạnh nhất, vì nó phân biệt bằng **kiến trúc sau khi chuyển**,
   không phải bằng dung lượng.
2. **"Ai đẩy dữ liệu — bạn hay người ngoài?"** Người ngoài, nói SFTP/FTPS/AS2 →
   **Transfer Family**. Nó không phải công cụ di trú; nó là **cửa nhận hàng** thường trực.
3. **"Tính ra bao nhiêu ngày qua đường truyền?"** Rồi so với vòng đời thiết bị.

### Phép tính quyết định mạng hay thiết bị

Công thức đầy đủ, bảng tra, và ngân sách thời gian của thiết bị vật lý nằm ở
[`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md#10-chuyển-x-tb-trong-y-ngày--tính-thật-đừng-đoán).
Dạng rút gọn:

```
   Số ngày ≈  Dung lượng(TB) × 132 ÷ Băng thông(Mbps)      (hiệu suất η = 0,7)
   So với vòng đời cố định của Snow Family: khoảng 7–14 ngày, gần như
   không phụ thuộc dung lượng.
```

Điều thuộc riêng file này: **phép tính đổi kết luận khi đường truyền còn phải phục vụ
việc khác.** Đường 1 Gbps của một nhà máy đang chạy MES, camera và ERP thì η thực tế rơi
xuống 0,3, và bạn phải trả lời câu đắt hơn — *"trong suốt N ngày đó, nhà máy có chạy được
không?"* Đây là chỗ **DataSync bandwidth throttle** là câu trả lời đúng thay vì đổi công
cụ: đặt trần 200 Mbps cho task, chấp nhận chậm hơn, đổi lấy dây chuyền không đứng.

- *"12 TB, đường DX 1 Gbps dành riêng, xong trong 1 tháng."* `12 × 132 ÷ 1000` ≈ **1,6
  ngày** → **DataSync**. Chọn Snowball ở đây là chọn phương án chậm hơn 5 lần.
- *"400 TB từ trạm quan trắc, đường 50 Mbps, xong trong quý."* `400 × 132 ÷ 50` ≈
  **1.056 ngày** → **Snow Family**, nhiều thiết bị song song.
- *"3 TB, nhưng ERP vẫn phải mở các file này như ổ đĩa mạng sau khi chuyển."* Dung lượng
  nhỏ, đường truyền thừa, nhưng **câu hỏi 1 đã chốt** → **S3 File Gateway**. Dung lượng
  không tham gia quyết định này chút nào.

### Transfer Family — hai chi tiết đáng tiền

**Nó không lưu dữ liệu.** Endpoint là lớp giao thức đứng trước S3 hoặc EFS. File đối tác
upload xuất hiện trong bucket ngay, và lifecycle, versioning, event notification,
replication áp dụng bình thường. Đó là lý do nó thắng "dựng một EC2 chạy sshd": không
phải vá server, không phải lo HA, dữ liệu vào thẳng data lake.

**Bạn trả tiền theo giao thức được bật, tính theo giờ, kể cả khi không ai dùng.** Bật
SFTP và FTPS là **hai lần** $0,30/giờ ≈ **$432/tháng** chỉ riêng phí endpoint, cộng
$0,04/GB. `stop-server` **không dừng đồng hồ** — chỉ `delete-server` mới dừng. Đề có chữ
`cost-effective` cộng "một đối tác gửi 20 MB mỗi tuần" thì Transfer Family thua một
presigned URL của S3.

Xác thực: thư mục do dịch vụ quản lý, **AWS Directory Service**
([mục 10](#10-directory-service--ba-lựa-chọn-một-câu-hỏi-quyết-định)), hoặc identity
provider tự viết sau API Gateway + Lambda.

---

## 8. Một kiến trúc lai thật — đặt cái gì ở đâu

Doanh nghiệp có trung tâm dữ liệu riêng, đang chuyển dần lên AWS nhưng còn Active
Directory, hệ thống backup băng từ, và một ERP cũ chỉ biết file share. Nét đôi `═══` là
**dữ liệu**, nét đơn `───` là **điều khiển và xác thực**.

```
┌──────────── TRUNG TÂM DỮ LIỆU CỦA BẠN ─────────────┐
│  ERP cũ ══ NFS/SMB ══▶ ┌──────────────────┐        │
│                        │ S3 File Gateway  │        │  appliance chạy như VM
│  phần mềm backup ═VTL═▶│ Tape Gateway     │════╗   │  cache + upload buffer local
│                        └────────┬─────────┘    ║   │
│  Active Directory ─────────┐    │ đăng ký      ║   │
│  laptop nhân viên ─────┐   │    │              ║   │
└────────────────────────┼───┼────┼──────────────╫───┘
            ╔════════════╪═══╪════╪══════════════╝
       Direct Connect (chính) ────────── VPN dự phòng qua internet
       độ trễ ổn định, không mã hoá       mã hoá sẵn, BGP tự chuyển
            ║            │   │    │
┌───────────╫────────────┼───┼────┼──────────────────────────────┐
│  VPC      ▼            ▼   ▼    ▼                              │
│     ┌──────────────────────────────────────┐                   │
│     │ Transit Gateway / Virtual Private GW │                   │
│     └──────┬─────────────────────┬─────────┘                   │
│  ┌─────────▼─────────┐   ┌───────▼──────────────┐              │
│  │ subnet private    │   │ subnet private       │              │
│  │ AWS Managed       │◀──┤ Client VPN endpoint  │◀── laptop    │
│  │ Microsoft AD      │   │ (ENI trong subnet)   │   xác thực AD│
│  │  ├ 2 DC, 2 AZ     │   └──────────────────────┘              │
│  │  └ trust hai chiều├────────▶ AD tại chỗ (qua DX)            │
│  │ FSx for Windows ──┤ domain join                             │
│  │ RDS SQL Server ───┘                                         │
│  └───────────────────┘                                         │
│  VPC endpoint (Gateway) ══════════════════════╗                │
└───────────────────────────────────────────────╫────────────────┘
                                     ┌──────────▼──────────┐
                                     │ S3                  │
                                     │ ├ bucket data lake  │ ← từ File Gateway
                                     │ └ Glacier Deep      │ ← từ Tape Gateway
                                     └─────────────────────┘
```

Bốn quyết định đặt chỗ, và lý do từng cái:

**Storage Gateway đặt ở on-premises, không trong VPC.** Vì cái nó phục vụ — ERP và phần
mềm backup — nằm ở on-premises, và giá trị của nó là **cache ở cạnh ứng dụng**. Đặt
gateway trên EC2 rồi bắt ERP đọc NFS qua DX là phá đúng thứ khiến nó hữu ích. Ngoại lệ
duy nhất: nguồn dữ liệu **đã ở trong AWS**.

**Directory Service đặt trong VPC, không ở on-premises.** Vì cái nó phục vụ — FSx for
Windows, RDS SQL Server, Client VPN, WorkSpaces — nằm trong VPC. Rồi nối về AD gốc bằng
**trust hai chiều**, để user vẫn quản trị ở một nơi duy nhất mà việc xác thực hằng ngày
**không phải chạy qua DX**. DX chết thì tài nguyên trong VPC vẫn xác thực được.

**Client VPN endpoint đặt trong subnet private.** Nó tạo ENI trong subnet đó và thừa
hưởng route table của subnet — nên qua nó nhân viên tới được cả VPC lẫn mạng on-prem qua
TGW. Xác thực gắn vào cùng Managed Microsoft AD.

**Đường tới S3 đi qua VPC endpoint.** Không chỉ vì bảo mật: lưu lượng gateway đẩy lên là
**liên tục và lớn**, cho nó đi qua NAT Gateway là trả $0,045/GB xử lý cho thứ lẽ ra miễn
phí. Xem [`04-networking.md`](04-networking.md#3-vpc-endpoint).

---

## 9. Client VPN — kết nối người, không phải kết nối site

### Cơ chế

Client VPN là endpoint **managed, dựa trên OpenVPN**. Hai thứ bạn khai khi tạo quyết
định mọi thứ còn lại:

- **Client CIDR** — dải IPv4 giữa `/12` và `/22`, cấp cho mỗi phiên một địa chỉ. **Không
  sửa được sau khi tạo.** Không được đè lên VPC lẫn mạng on-prem.
- **Subnet association** — mỗi lần gắn, AWS tạo **ENI trong subnet đó**, và lưu lượng
  client đi ra từ ENI này. Client thừa hưởng **route table của subnet** — muốn client tới
  được mạng on-prem thì subnet phải có route về TGW hoặc VGW.

Hai lớp kiểm soát riêng biệt, lẫn lộn chúng là lỗi cấu hình phổ biến nhất: **route** nói
*"đích này đi qua network association nào"*; **authorization rule** nói *"nhóm AD nào
được tới CIDR nào"*. Có route mà không có authorization rule thì vẫn không đi được — đây
là chỗ bạn cài phân quyền theo nhóm.

**Split-tunnel** quyết định lưu lượng nào chui vào tunnel. Bật: chỉ CIDR trong route
table của endpoint đi qua VPN. Tắt: **toàn bộ** lưu lượng đi qua AWS, kể cả xem video.
Bật là mặc định nên chọn; tắt chỉ khi tuân thủ bắt mọi lưu lượng qua điểm kiểm tra
tập trung.

### Ba kiểu xác thực

| Kiểu | Cơ chế | Chọn khi |
|---|---|---|
| **Mutual authentication** | Chứng chỉ hai chiều; bạn tự vận hành CA, thu hồi bằng CRL | Ít người, không có IdP, hoặc cần buộc **thiết bị cụ thể** chứ không phải người |
| **Active Directory** | Qua Directory Service — Managed Microsoft AD hoặc **AD Connector** trỏ về AD tại chỗ. Hỗ trợ **MFA qua RADIUS** | Doanh nghiệp đã có AD. Đáp án mặc định cho đề nhắc "corporate credentials" |
| **SAML 2.0 federated** | IdP bên ngoài (Okta, Entra ID, Ping) | Đã dùng IdP cho mọi thứ, muốn một chỗ tắt tài khoản |

Kết hợp được mutual + AD hoặc mutual + SAML — khi đó **cả hai phải qua**, tức vừa đúng
máy vừa đúng người. Dù chọn kiểu nào, bạn **vẫn phải có server certificate trong ACM**.

### Con số

| Hạng mục | Giá trị (2026-08) |
|---|---|
| Client CIDR | `/12` đến `/22`; khuyến nghị **gấp đôi** số kết nối dự kiến |
| Kết nối đồng thời theo số subnet gắn | 1 → **7.000** · 2 → **36.500** · 3 → **66.500** · 4 → **96.500** · 5 → **126.000** |
| Authorization rule / route / endpoint mỗi Region | **200** / **100** / **5** (đều nâng được) |
| Giá | **$0,10/giờ mỗi endpoint association** + **$0,05/giờ mỗi kết nối client** |

Con số kết nối có tính chất phi tuyến đáng chú ý: từ 1 lên 2 subnet, sức chứa nhảy từ
7.000 lên 36.500 — **hơn năm lần**. Gắn hai subnet ở hai AZ vừa là HA vừa là mở rộng
năng lực, gần như luôn đúng.

Về giá: 200 nhân viên online 8 giờ/ngày, 22 ngày/tháng, 2 subnet →
`2 × $0,10 × 730 + 200 × 8 × 22 × $0,05` ≈ **$146 + $1.760 = ~$1.900/tháng**. Phần đắt
là **kết nối**. Nếu nhân viên chỉ cần vào **một EC2 để chạy lệnh**, đáp án rẻ hơn nhiều
là **SSM Session Manager** — không VPN, không bastion, không port mở, xem
[`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md).

Phân biệt ba thứ, một câu mỗi thứ: **Site-to-Site VPN** nối **một mạng** với VPC, luôn 2
tunnel. **Client VPN** nối **một người** với VPC, mỗi phiên một IP. **Session Manager**
không nối mạng gì cả — nó cho bạn một shell trên đúng một instance.

---

## 10. Directory Service — ba lựa chọn, một câu hỏi quyết định

Đừng học ba lựa chọn như ba dịch vụ song song. Hỏi đúng một câu:

> **Bạn cần một Active Directory *thật* chạy trên AWS, hay chỉ cần *chuyển tiếp* yêu cầu
> xác thực về AD đang có ở on-premises?**

Cần AD **thật** — vì cần domain controller sống sót khi DX đứt, cần mở rộng schema, cần
chia sẻ directory cho nhiều account, hoặc **chưa có** AD nào → **Managed Microsoft AD**.
Chỉ cần **chuyển tiếp** — đã có AD, giữ một nguồn sự thật duy nhất, **không muốn dữ liệu
danh tính nào được cache trên AWS** → **AD Connector**.

| | **AWS Managed Microsoft AD** | **AD Connector** | **Simple AD** |
|---|---|---|---|
| Bản chất | **Windows Server AD thật**, AWS vận hành, 2 domain controller ở **2 AZ trong cùng một VPC** | **Proxy**, không lưu, **không cache** thông tin thư mục nào | Máy chủ tương thích AD dựng trên **Samba 4** |
| Nguồn sự thật của user | **Trên AWS** (có thể lập trust về AD tại chỗ) | **On-premises** | Trên AWS |
| Trust với domain khác | **Có** — một chiều, hai chiều, forest trust | **Không**, kể cả transitive trust. Mỗi domain cần **một AD Connector riêng** | **Không** |
| Mở rộng schema; MFA | Có; có (RADIUS) | Theo AD tại chỗ; có (RADIUS) | **Không**; **không** |
| Chia sẻ cho account khác qua RAM | **Có** — Standard tới ~25 share, Enterprise tới 500 | **Không.** Cũng **không multi-VPC aware** | Không |
| Multi-Region | Chỉ **Enterprise Edition** | Không | Không |
| Quy mô | **Standard** ~5.000 user, ~**30.000 object**, 1 GB · **Enterprise** ~100.000 user, ~**500.000 object**, 17 GB | Small / Large, **không có trần user cứng** — chia tải bằng nhiều connector | Small 500 user (~2.000 object) · Large 5.000 user (~20.000 object) |
| Hoạt động khi DX/VPN đứt | **Có** — DC nằm trong VPC | **Không** — mọi lần xác thực phải tới được AD tại chỗ | Có |
| Trạng thái 2026-08 | Đang bán | Đang bán | **Không nhận khách hàng mới** |

**AD Connector là điểm hỏng đơn nếu đường về on-prem là đơn.** Nó không cache gì, nên
mỗi lần nhân viên đăng nhập WorkSpaces là một lần gói tin đi hết DX về DC ở phòng máy.
DX chết là **không ai đăng nhập được**, dù WorkSpaces vẫn chạy. Đây là lập luận mạnh nhất
để chọn Managed Microsoft AD + trust khi hệ thống phải chịu được mất kết nối.

**Nâng Standard lên Enterprise được, hạ xuống thì không.** Ba câu hỏi chọn edition: cần
multi-Region không? Chia sẻ cho hơn ~25 account không? Quá 30.000 object không? Một "có"
là Enterprise.

**Chỉ một số dịch vụ thực sự dùng directory:** WorkSpaces, FSx for Windows File Server,
RDS for SQL Server và RDS for Oracle, EC2 domain join (kể cả seamless), Client VPN,
Transfer Family, QuickSight, IAM Identity Center. Simple AD **không** hỗ trợ FSx, RDS SQL
Server, RDS Oracle, IAM Identity Center — đủ để loại nó khỏi phần lớn câu doanh nghiệp.

**Nối với IAM Identity Center.** Hai thứ giải hai bài toán khác nhau: **Directory Service**
trả lời *"tài nguyên Windows trong VPC xác thực người dùng bằng gì"*; **IAM Identity
Center** trả lời *"người dùng đăng nhập vào Console và CLI của nhiều account bằng gì"* —
cơ chế đầy đủ ở [`05-security.md`](05-security.md#7-iam-identity-center). Ghép lại trong
doanh nghiệp thật: AD tại chỗ là nguồn sự thật → Managed Microsoft AD trong VPC có trust
về đó → **FSx và RDS** dùng nó để xác thực người dùng cuối, còn **IAM Identity Center**
dùng chính thư mục đó làm identity source. Một danh tính, hai đường sử dụng.

---

## 11. License Manager — giấy phép tính theo lõi vật lý

Oracle Database, SQL Server, Windows Server và phần lớn phần mềm doanh nghiệp đắt tiền
tính tiền theo **physical core** hoặc **socket**, không theo vCPU. Hợp đồng ghi "36
cores" nghĩa là bạn được chạy trên tối đa 36 lõi vật lý, và kiểm toán viên sẽ đến kiểm
tra. Trên hạ tầng chia sẻ bạn **không thấy** lõi vật lý nên không chứng minh được gì —
đó là ràng buộc.

Ba mảnh ghép, phải đủ cả ba mới thành cơ chế cưỡng chế:

1. **License configuration** — khai luật của hợp đồng: đơn vị đếm là **vCPU, core,
   socket, hay instance**; số lượng trần; tenancy bắt buộc; và **cưỡng chế cứng hay chỉ
   cảnh báo**. Cưỡng chế cứng nghĩa là `RunInstances` vượt trần bị **từ chối**.
2. **Gắn license configuration vào AMI** (hoặc launch template, hoặc instance). Từ đó mọi
   máy dựng từ AMI đó đều bị đếm.
3. **Host resource group** — nhóm **EC2 Dedicated Host** do License Manager tự quản. Ba
   công tắc: tự cấp host mới khi thiếu chỗ, tự trả host khi rỗng, tự chuyển instance sang
   host khác khi host hỏng. Sau khi một Dedicated Host vào nhóm, bạn **không launch trực
   tiếp lên nó được nữa**.

Nối với phần tenancy ở [`01-compute.md`](01-compute.md#dedicated-host-so-với-dedicated-instance):
**Dedicated Instance** cho bạn máy vật lý riêng nhưng **không cho thấy socket và core** —
vô dụng cho BYOL. **Dedicated Host** mới cho thấy và cố định chúng, nên BYOL luôn kéo
theo Dedicated Host; License Manager là lớp tự động hoá phía trên.

Ba chi tiết ra thi: nó theo dõi được cả license mua qua **AWS Marketplace** lẫn license
tự quản; nó **chia sẻ được qua RAM** cho các account trong Organization nên nhiều đội
dùng chung một hạn mức; và nó báo cáo tập trung cho kiểm toán — đúng thứ đề mô tả bằng
*"track license usage across accounts"*.

---

## 12. RAM — chia sẻ tài nguyên, không chia sẻ quyền

Account là **ranh giới mạnh nhất** của AWS
([`00-nen-tang.md`](00-nen-tang.md#2-bốn-ranh-giới-phải-thuộc-lòng)). Có đúng ba cách
xuyên qua: `sts:AssumeRole`, resource-based policy, và **AWS RAM**.

RAM khác hai cách kia ở một điểm bản chất: nó **không cấp quyền cho bạn gọi API trong
account khác**. Nó làm **tài nguyên xuất hiện trong account của bạn** như thể nó ở đó.
Bạn tạo một **resource share** gồm ba thứ: tài nguyên, principal nhận (account, OU, cả
Organization, và với vài loại thì cả IAM role/user), và **managed permission** quy định
người nhận làm được gì. Bật `enable-sharing-with-aws-organization` thì share trong
Organization được nhận **tự động**; share ra ngoài phải **chấp nhận lời mời**.

| Chia sẻ được | Bài toán nó giải | Chi tiết quan trọng |
|---|---|---|
| **Subnet** (VPC sharing) | Đội mạng trung tâm quản một VPC, các đội ứng dụng launch vào đó | **Chỉ trong cùng Organization.** **Subnet mặc định không chia sẻ được** |
| **Transit Gateway** | Một TGW trung tâm cho hàng chục account | Chia sẻ được ra **ngoài** Organization |
| **Route 53 Resolver rule** và **endpoint** | Một chỗ duy nhất định nghĩa "tên `.corp` phân giải về AD tại chỗ" | Nền tảng của DNS lai đa account |
| **Prefix list** | Một danh sách CIDR của on-prem, dùng lại trong SG và route table mọi account | Sửa một chỗ, mọi nơi cập nhật |
| **License configuration** | Nhiều đội dùng chung một hạn mức license | Xem [mục 11](#11-license-manager--giấy-phép-tính-theo-lõi-vật-lý) |
| **Outpost, Outpost site, local gateway route table** | Nhiều account cùng dùng một giá máy | **Chỉ trong cùng Organization** |
| **Aurora DB cluster** | Account khác **clone** cluster để test trên dữ liệu thật | Chia sẻ để clone, không phải để ghi chung |
| **Security group**, **Backup vault**, **Private CA**, **IPAM pool**, **Network Firewall rule group** | Hạ tầng dùng chung điển hình | Security group: chỉ trong cùng Organization |

Phần ra thi thật là cột ngược lại:

| Không chia sẻ được | Vì sao |
|---|---|
| **IAM user, group, role** | Danh tính là tài nguyên của account. Cross-account làm bằng `AssumeRole` và trust policy |
| **VPC** (nguyên cái) | Chỉ chia sẻ được **subnet**. Người nhận không sửa được VPC, route table, IGW |
| **Subnet mặc định**, **instance profile** | AWS chặn hẳn |
| **S3 bucket**, **KMS key** | Dùng **bucket policy** / **key policy** — resource-based policy, không phải RAM |
| **AD Connector** | Không chia sẻ được. Cần chia sẻ directory thì phải là **Managed Microsoft AD** |
| **EBS volume, EC2 instance** | Chia sẻ được **snapshot** và **AMI** bằng cơ chế riêng của EC2. Snapshot mã hoá bằng AWS managed key thì **không** chia sẻ được — phải là customer managed key, và chia sẻ cả key |

**VPC sharing — ai trả tiền cái gì.** **Owner** sở hữu VPC, là người duy nhất tạo sửa
xoá **subnet, route table, IGW, NAT Gateway, VPC endpoint, peering**, và trả tiền cho
**NAT Gateway và các endpoint**. **Participant** launch tài nguyên vào subnet được chia
sẻ và **trả tiền cho chính tài nguyên đó**; participant tạo được security group riêng
nhưng **không** sửa được gì thuộc về mạng.

Lợi ích thật không phải tiền mà là **địa chỉ IP và số kết nối phải quản**: 40 đội, mỗi
đội một VPC, là 40 attachment TGW và một cơn ác mộng CIDR. 40 đội trong 3 VPC chia sẻ là
3 attachment.

---

## 13. Hai bài toán thiết kế có lời giải

### Bài 1 — nhà máy có ràng buộc độ trễ và ràng buộc trú ngụ dữ liệu

> Nhà máy sản xuất linh kiện thu 50.000 điểm dữ liệu mỗi giây từ dây chuyền. Vòng điều
> khiển yêu cầu **phản hồi dưới 10 ms**. Hợp đồng với khách hàng quốc phòng bắt **dữ
> liệu thô không được rời khỏi khuôn viên nhà máy**; chỉ báo cáo tổng hợp được lên cloud.
> Region gần nhất cách 600 km. Nhà máy có internet 500 Mbps.

**Lời giải.** 600 km cáp quang ≈ **6 ms một chiều**, khứ hồi **12 ms**, chưa cộng thiết
bị mạng — **đã vượt 10 ms trước khi tính bất cứ thứ gì khác**. Ràng buộc này loại mọi
phương án đặt compute ở Region, không phải vì AWS chậm mà vì vật lý. Ràng buộc trú ngụ
ở mức **khuôn viên** loại luôn **Local Zone**, vì đó là cơ sở của **AWS**. Còn lại:
**AWS Outposts rack** đặt trong nhà máy.

- Vòng điều khiển chạy **hoàn toàn trên Outpost**; PLC nói chuyện với instance qua
  **local gateway** và CoIP, độ trễ LAN dưới 1 ms.
- **Dữ liệu thô không bao giờ đi qua service link**; job tổng hợp chạy trên Outpost, chỉ
  đẩy bản tóm tắt lên S3.
- **Route 53 Resolver đặt ngay trên Outpost** — nếu không, mất service link là mất DNS và
  vòng điều khiển đứt dù mọi máy vẫn chạy.
- **Không đặt Auto Scaling vào đường sống còn**; cấp đủ capacity thường trực cho tải đỉnh.
- Service link đi qua **Direct Connect, VPN qua internet làm dự phòng**, BGP tự chuyển.

**Vì sao ba phương án kia sai:** **Local Zone** có thể đạt độ trễ nhưng **vi phạm ràng
buộc trú ngụ** — ràng buộc pháp lý không thương lượng bằng con số kỹ thuật. **EC2 ở
Region + DX băng thông lớn** sai vì DX làm độ trễ **ổn định**, không làm nó **nhỏ hơn**
khoảng cách vật lý. **Snowball Edge Compute Optimized** chạy được EC2 tại chỗ nhưng là
thiết bị **tạm thời**, không có EBS hay ALB, không phải hạ tầng sản xuất chạy nhiều năm.

### Bài 2 — trung tâm dữ liệu đóng cửa trong 6 tháng

> Công ty bảo hiểm đóng trung tâm dữ liệu sau 6 tháng. Ở đó có **300 TB** file mà ứng
> dụng định giá cũ mở qua **SMB** và không sửa được; một thư viện **băng từ** giữ 8 năm
> hồ sơ theo quy định; **Active Directory** 12.000 nhân viên; và quy trình nhận file bồi
> thường từ 40 bệnh viện qua **SFTP**. Đường truyền **1 Gbps** dùng chung toàn công ty.
> Sau khi đóng, nhân viên làm việc từ xa vẫn phải truy cập được mọi thứ.

**Lời giải.** Tách thành bốn bài toán độc lập, vì chúng có bốn đáp án khác nhau.

**300 TB file share qua SMB.** Ứng dụng không sửa được nên nó vẫn phải thấy SMB. Nhưng
sau khi trung tâm dữ liệu **đóng**, chính ứng dụng cũng phải lên EC2 — nên đáp án
**không** phải Storage Gateway mà là **FSx for Windows File Server**, một SMB share thật
trong VPC. Chuyển 300 TB bằng gì: `300 × 132 ÷ 1000` ≈ **40 ngày** với η = 0,7, nhưng
đường **dùng chung toàn công ty** nên η thực tế cỡ 0,3 → **khoảng 92 ngày**, và suốt thời
gian đó mạng công ty nghẹt. Kịp deadline nhưng giá vận hành quá cao → **Snow Family** cho
khối lớn ban đầu, rồi **DataSync** đồng bộ phần thay đổi ở giai đoạn cuối, có throttle.
Mẫu kinh điển: thiết bị cho khối lượng, mạng cho độ tươi.

**Thư viện băng từ, 8 năm hồ sơ.** Phần mềm backup ghi VTL và không đổi được →
**Tape Gateway** trên VM tại chỗ trong giai đoạn chuyển tiếp, đổ vào **Glacier Deep
Archive**. Sau khi đóng, appliance chuyển thành **EC2 gateway** nếu phần mềm backup còn
sống; nếu bỏ luôn thì tape đã ở S3 và **Object Lock** giữ tính bất biến cho 8 năm.

**Active Directory, 12.000 nhân viên.** Vượt xa Simple AD (trần 5.000, và không hỗ trợ
FSx). AD Connector thì sau khi đóng sẽ không còn AD nào để trỏ về. → **Managed Microsoft
AD, Enterprise Edition**. Trình tự: dựng directory → lập **trust hai chiều** với AD tại
chỗ → di trú user → cắt trust khi đóng. FSx và mọi EC2 Windows domain join vào đây.

**SFTP từ 40 bệnh viện.** → **Transfer Family**, endpoint SFTP, backend S3, xác thực qua
chính Managed Microsoft AD ở trên. 40 đối tác giữ nguyên script; trỏ bản ghi Route 53 cũ
sang endpoint mới là xong. Phí một giao thức ≈ **$216/tháng** cộng $0,04/GB.

**Nhân viên từ xa.** → **Client VPN**, 2 subnet ở 2 AZ (36.500 kết nối đồng thời, thừa
cho 12.000 nhân viên), xác thực bằng Managed Microsoft AD, MFA qua RADIUS, bật
**split-tunnel** để không trả data transfer cho lưu lượng Netflix.

**Vì sao vài lựa chọn hấp dẫn khác sai:** **S3 File Gateway** cho 300 TB sai vì ứng dụng
cũng lên cloud — giữ một gateway bắc cầu SMB → S3 là thêm tầng không cần thiết và mất
tính năng file server thật (ACL Windows, DFS, khoá file). **AD Connector** sai về **thời
điểm**: nó phụ thuộc AD tại chỗ, mà AD tại chỗ sắp biến mất. **EC2 chạy sshd** rẻ hơn về
đơn giá nhưng bạn nhận lại việc vá OS, tự làm HA và giám sát cho một hệ thống nhận dữ
liệu y tế. **Chuyển hết 300 TB qua mạng** kịp deadline nhưng bóp nghẹt đường truyền cả
công ty trong ba tháng — đáp án "đúng kỹ thuật" thua đáp án đúng về vận hành.

---

## Bảng số phải nhớ

| Con số | Giá trị | Vì sao ra thi |
|---|---|---|
| Outposts: cache metric/log khi mất service link | **7 ngày** | Câu "mất kết nối thì sao" |
| Outposts: băng thông service link khuyến nghị | tối thiểu **500 Mbps**, nên **1 Gbps+** | Thiết kế đường về Region |
| Outposts rack: kích thước, điện; ACE rack bắt buộc từ | **42U**, **5–15 kVA** (gen 1); **4 compute rack** | Yêu cầu cơ sở vật chất |
| Managed Microsoft AD Standard | ~**5.000 user**, ~**30.000 object** | Chọn edition |
| Managed Microsoft AD Enterprise | ~**100.000 user**, ~**500.000 object**, **có multi-Region** | Chọn edition |
| Simple AD Small / Large | **500** / **5.000** user | Loại nó khi đề nói 12.000 nhân viên |
| Client VPN: kết nối đồng thời theo số subnet | 1 → **7.000** · 2 → **36.500** · 5 → **126.000** | Câu sizing |
| Client VPN: client CIDR; giá | **/12 đến /22**, không sửa được; **$0,10/giờ** endpoint + **$0,05/giờ** mỗi kết nối | Thiết kế địa chỉ và so với Session Manager |
| Volume Gateway: volume lớn nhất | cached **32 TiB** · stored **16 TiB** | Phân biệt hai chế độ |
| Volume Gateway: số volume; cache tối thiểu | **32**; **150 GiB** | Yêu cầu triển khai |
| Tape Gateway: kích thước tape; số tape | **100 GiB – 15 TiB**; **1.500**, tổng **1 PiB** | Câu thay thư viện băng từ |
| S3 File Gateway: file share; file lớn nhất | **50**; **5 TiB** | Trần thiết kế |
| Transfer Family: giá | **$0,30/giờ mỗi giao thức** + **$0,04/GB** | Bẫy chi phí |
| DataSync: giá | **~$0,0125/GB** | So với Snow và Transfer Family |
| Công thức băng thông; vòng đời Snow | `ngày ≈ TB × 132 ÷ Mbps` (η = 0,7); **7–14 ngày** | Bài mạng-hay-thiết-bị |

---

## Bẫy đề thi

**1. Outposts được chọn cho môi trường mất kết nối.**
Đề: *"a remote mining site with intermittent satellite connectivity needs to process data
locally"*. Đáp án sai hấp dẫn: **AWS Outposts**. Đáp án đúng: **Snowball Edge Compute
Optimized** hoặc kiến trúc tự chủ. Vì sao: control plane của Outposts nằm ở Region cha;
mất link là không launch, không terminate, không xác thực IAM được. AWS nói rõ Outposts
**không thiết kế cho vận hành mất kết nối**.

**2. Local Zone dùng để thoả ràng buộc "dữ liệu ở lại cơ sở của chúng tôi".**
Đáp án sai hấp dẫn: **Local Zone gần nhất**. Đáp án đúng: **Outposts**. Vì sao: Local
Zone là **cơ sở của AWS**. Ngược lại, đề nói *"data must remain in country X"* thì Local
Zone hoặc Region trong nước đó đã đủ, và Outposts là đắt quá mức.

**3. Volume Gateway stored và cached bị nhớ ngược.**
Đề: *"reduce the on-premises storage footprint while keeping the most recently used data
fast"*. Đáp án sai hấp dẫn: **stored volumes**. Đáp án đúng: **cached volumes**. Vì sao:
tên chế độ mô tả **cái nằm ở on-premises**. Stored = dữ liệu **lưu** tại chỗ (toàn bộ);
cached = tại chỗ chỉ có **cache**.

**4. DataSync bị chọn cho bài toán truy cập thường trực.**
Đề: *"after the migration, the legacy application must continue to access the files as an
SMB share"*. Đáp án sai hấp dẫn: **DataSync sang FSx**. Đáp án đúng: **FSx File Gateway**
nếu ứng dụng ở lại on-prem, **FSx for Windows** nếu ứng dụng cũng lên cloud. Vì sao:
DataSync chuyển xong là hết, không để lại điểm truy cập nào ở on-prem.

**5. AD Connector được chọn cho hệ thống phải chịu được mất kết nối.**
Đề: *"AWS resources must continue to authenticate users even if the Direct Connect link
fails"*. Đáp án sai hấp dẫn: **AD Connector** (rẻ hơn, không cần di trú user). Đáp án
đúng: **Managed Microsoft AD với trust về AD tại chỗ**. Vì sao: AD Connector là **proxy
không cache**; mọi lần xác thực đều phải tới được DC ở on-prem.

**6. Simple AD được chọn cho tổ chức lớn hoặc cho FSx.**
Đáp án sai hấp dẫn: **Simple AD** khi đề nhấn `low cost`. Đáp án đúng: **Managed Microsoft
AD**. Vì sao: Simple AD trần **5.000 user** và **không hỗ trợ** FSx, RDS for SQL Server,
RDS for Oracle, IAM Identity Center, MFA, trust, schema extension. Nó cũng đã **ngừng
nhận khách hàng mới**.

**7. RAM bị tưởng là chia sẻ được VPC hoặc IAM role.**
Đề: *"share the VPC with other accounts"*. Đáp án sai hấp dẫn: *"share the VPC using
RAM"*. Đáp án đúng: chia sẻ **subnet** — người nhận launch được tài nguyên nhưng không
sửa được route table, IGW, NAT Gateway. Và **IAM role không bao giờ chia sẻ bằng RAM**;
cross-account danh tính là `AssumeRole`.

**8. Client VPN được chọn khi chỉ cần vào một máy.**
Đề: *"administrators need shell access to EC2 instances in private subnets, most
cost-effective"*. Đáp án sai hấp dẫn: **Client VPN** hoặc **bastion host**. Đáp án đúng:
**SSM Session Manager**. Vì sao: Client VPN tính **$0,05/giờ mỗi kết nối**; Session
Manager không tính phí, không mở port, và ghi log đầy đủ.

**9. ECS Anywhere bị coi là chạy được khi mất mạng.**
Đề: *"containers must keep being scheduled and replaced at a site with unreliable
connectivity"*. Đáp án sai hấp dẫn: **ECS Anywhere**. Đáp án đúng: **EKS Anywhere**. Vì
sao: control plane của ECS Anywhere ở Region, mất mạng là mất lập lịch; EKS Anywhere chạy
control plane **tại chỗ**, cụm tự đủ.

**10. Transfer Family bị chọn cho lưu lượng nhỏ, không đều.**
Đề: *"one partner sends a 20 MB file weekly, minimize cost"*. Đáp án sai hấp dẫn:
**Transfer Family SFTP endpoint**. Đáp án đúng: **presigned URL của S3**. Vì sao:
Transfer Family tính **$0,30/giờ** cho giao thức đã bật **kể cả khi không ai dùng** —
khoảng $216/tháng cho 80 MB dữ liệu, và `stop-server` không dừng đồng hồ.

---

## Cây quyết định

**Chọn nơi đặt compute ngoài Region** — hỏi "ai sở hữu chỗ đặt" trước:

```
Đề có nhắc ràng buộc vị trí không?
├── Không → dùng Region bình thường
└── Có
    ├── "in our own facility / datacenter / factory" ─► Outposts
    │     rồi kiểm: đường về Region có ổn định không?
    │     Không ổn định → Outposts SAI, cân nhắc Snow hoặc kiến trúc tự chủ
    ├── "low latency to end users in <thành phố>" ────► Local Zone
    ├── "5G", "mobile devices", "carrier network" ────► Wavelength
    └── "hundreds of VMs", "same vSphere tooling" ────► VMware Cloud on AWS
```

**Chọn cách container chạy ở on-premises:**

```
Muốn AWS lo lập lịch, chấp nhận phụ thuộc kết nối ─► ECS Anywhere (launch type EXTERNAL)
Phải tự chủ hoàn toàn, chịu được mất mạng ─────────► EKS Anywhere
Chỉ cần bản phân phối K8s đã kiểm thử ─────────────► EKS Distro (không có support plan)
Chỉ cần NHÌN thấy cụm có sẵn trong console ────────► EKS Connector
```

**Chọn công cụ dữ liệu** — ba câu, theo đúng thứ tự:

```
1. Sau khi xong, on-prem còn phải đọc dữ liệu đó không?
   Còn → Storage Gateway, chọn loại theo GIAO THỨC:
          NFS/SMB → S3 File Gateway
          chỉ SMB, cần file server Windows thật → FSx File Gateway
          iSCSI, muốn giảm đĩa tại chỗ → Volume Gateway cached
          iSCSI, cần toàn bộ dataset nhanh → Volume Gateway stored
          iSCSI VTL → Tape Gateway
   Không → câu 2
2. Ai đẩy dữ liệu?
   Đối tác bên ngoài qua SFTP/FTPS/FTP/AS2 → Transfer Family
   Bạn → câu 3
3. Tính ngày ≈ TB × 132 ÷ Mbps, so với 7–14 ngày của Snow
   Mạng nhanh hơn → DataSync (bật throttle nếu đường dùng chung)
   Mạng chậm hơn đáng kể → Snow Family
```

**Chọn Directory Service:**

```
Cần AD THẬT trên AWS?  (sống sót khi DX đứt · trust · schema · chia sẻ nhiều account
                        · multi-Region · chưa có AD nào)
├── Có ──► AWS Managed Microsoft AD
│           > 30.000 object, hoặc multi-Region, hoặc > ~25 account → Enterprise
│           còn lại → Standard
├── Không, chỉ chuyển tiếp về AD đang có, không cache gì trên AWS ─► AD Connector
└── Không có AD, chỉ cần thứ tương thích, ≤ 5.000 user, không cần FSx/RDS ─► Simple AD
      (đã ngừng nhận khách mới — trong phòng thi vẫn là đáp án hợp lệ)
```

**Chọn cách người dùng vào mạng:**

```
Một mạng nối với VPC ──────────────────► Site-to-Site VPN hoặc Direct Connect
Nhiều người, nhiều nơi, cần cả VPC lẫn on-prem ──► Client VPN
Chỉ cần shell trên một instance ───────► SSM Session Manager (rẻ nhất, không mở port)
```

---

## Nối với thực hành

| Lab | Chạm vào mục nào | Quan sát gì |
|---|---|---|
| [`labs/w11-dr-hybrid/`](../../learn-aws/labs/w11-dr-hybrid/) | Mục 7, 8 | Lab cố ý không có `terraform/` cho DX và TGW. Phần lab được: dựng một **DataSync task** giữa hai bucket S3, đặt `BytesPerSecond` để throttle, chạy rồi đọc `TaskExecution` để thấy verify là một bước riêng |
| [`labs-self/w11-dr-hybrid/`](../../learn-aws/labs-self/w11-dr-hybrid/) | Mục 7 | Tự viết DataSync task có `Includes`/`Excludes` pattern rồi chứng minh chỉ đúng tập file được chuyển |
| [`labs/w02-vpc-networking/`](../../learn-aws/labs/w02-vpc-networking/) | Mục 9, 12 | Chạy `aws ram list-resource-types --profile learn` để thấy danh sách thật, đối chiếu với bảng ở [mục 12](#12-ram--chia-sẻ-tài-nguyên-không-chia-sẻ-quyền) |
| [`labs-self/w02-vpc-networking/`](../../learn-aws/labs-self/w02-vpc-networking/) | Mục 9 | Tự viết một Client VPN endpoint là bài tập tốt nhưng **tốn tiền theo giờ** — `terraform destroy` ngay trong ngày |
| [`labs/w09-security-deep/`](../../learn-aws/labs/w09-security-deep/) | Mục 10 | Đọc phần IAM Identity Center rồi trả lời: đổi identity source sang Managed Microsoft AD thì permission set có đổi gì không |
| [`labs/w04-s3-cloudfront/`](../../learn-aws/labs/w04-s3-cloudfront/) | Mục 6, 7 | Bucket ở lab này chính là đích của một S3 File Gateway trong đời thật. Bật event notification rồi hình dung file ghi qua SMB kích hoạt Lambda |
| [`labs/w12-exam-review/`](../../learn-aws/labs/w12-exam-review/) | Toàn bộ | Thêm một ràng buộc giả — "một phần dữ liệu không được rời khuôn viên" — rồi vẽ lại kiến trúc capstone |

Bài tuần tương ứng: [`docs/aws/w11-dr-hybrid.md`](../aws/w11-dr-hybrid.md).

---

## Nguồn nói khác

| Chỗ | Nguồn `aws-saa-c03/` nói | Thực tế (kiểm tra 2026-08) |
|---|---|---|
| Storage Gateway | Liệt kê **3 loại** | Có **4**: thiếu **Amazon FSx File Gateway** (SMB → FSx for Windows), loại duy nhất cho bạn file server Windows **thật** với ACL và DFS |
| Volume Gateway | Không phân biệt rõ stored với cached, hoặc mô tả ngược | **Stored** = toàn bộ dataset ở on-prem, AWS giữ EBS snapshot. **Cached** = dữ liệu chính ở AWS. Trần khác nhau: 16 TiB với 32 TiB ([Storage Gateway quotas](https://docs.aws.amazon.com/storagegateway/latest/vgw/resource-gateway-limits.html)) |
| Outposts | Mô tả như "AWS trong data center của bạn", không nói gì về phụ thuộc control plane | Outposts **không thiết kế cho vận hành mất kết nối**. Mất service link: instance chạy tiếp nhưng mọi API và mọi thao tác cần IAM đều hỏng; metric/log cache tối đa **7 ngày** ([Outposts maintenance](https://docs.aws.amazon.com/outposts/latest/userguide/outpost-maintenance.html)) |
| Outposts server | Liệt kê 1U/2U như lựa chọn còn dùng được | AWS **đã ngừng bán cả 1U lẫn 2U cho khách hàng mới**, đang chuyển khách hiện có sang rack. Thế hệ 2 của rack có **network rack riêng** ([What is AWS Outposts](https://docs.aws.amazon.com/outposts/latest/network-userguide/what-is-outposts.html)) |
| Simple AD | Trình bày như lựa chọn rẻ còn dùng được | **Không nhận khách hàng mới.** AWS hướng sang Managed Microsoft AD hoặc AD Connector; đề SAA-C03 vẫn hỏi ([Simple AD availability changes](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/simple-ad-availability-change.html)) |
| VMware Cloud on AWS | Liệt kê như dịch vụ AWS bán | **Từ 30/04/2024 AWS và đối tác kênh không còn bán lại**; dịch vụ vẫn tồn tại, do Broadcom bán và hỗ trợ ([VMware Cloud on AWS for SQL Server](https://docs.aws.amazon.com/prescriptive-guidance/latest/migration-sql-server/vmware-sql.html)) |
| Snow Family | Lựa chọn mặc định cho "limited bandwidth + large data"; ghi Snowball Edge **80 TB** | Model 80 TB ngừng từ 11/2024, hiện hành **210 TB**. Từ **07/11/2025** Snowball Edge **không nhận khách hàng mới**. Không có ngưỡng dung lượng cố định nào đúng — phải tính ([Snowball Edge availability change](https://docs.aws.amazon.com/snowball/latest/developer-guide/snowball-edge-availability-change.html)) |
| Directory Service | Nhắc tên ba lựa chọn, không có con số | Standard ~**30.000 object** / 1 GB; Enterprise ~**500.000 object** / 17 GB **và là bản duy nhất có multi-Region**; Simple AD Small 500 user, Large 5.000 user ([Upgrading AWS Managed Microsoft AD](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_upgrade_edition.html)) |
| ECS Anywhere | Mô tả là "chạy ECS ở on-premises", ngang hàng EKS Anywhere | Khác hẳn ở **nơi đặt control plane**. ECS Anywhere: control plane ở Region, **cần kết nối ổn định**. EKS Anywhere: cụm tự đủ tại chỗ. EKS Distro **không có support plan** ([EKS deployment options](https://docs.aws.amazon.com/eks/latest/userguide/eks-deployment-options.html)) |
| RAM | Chỉ liệt kê "subnet, TGW, license" | Rộng hơn nhiều — Route 53 Resolver rule, prefix list, security group, Aurora cluster, Backup vault, Outpost, Private CA, IPAM pool. Nhưng **subnet mặc định không chia sẻ được**, và **IAM role không bao giờ** ([Shareable AWS resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html)) |

---

## Ngoài phạm vi

- **AWS Verified Access** — thay VPN cho ứng dụng web nội bộ theo mô hình zero trust. [Verified Access](https://docs.aws.amazon.com/verified-access/latest/ug/what-is-verified-access.html)
- **AWS Cloud WAN** — quản lý mạng toàn cầu bằng chính sách, mức Professional. [Cloud WAN](https://docs.aws.amazon.com/network-manager/latest/cloudwan/what-is-cloudwan.html)
- **S3 on Outposts** chi tiết (access point, bucket policy riêng) — chỉ cần biết S3 chạy được trên Outpost. [S3 on Outposts](https://docs.aws.amazon.com/AmazonS3/latest/userguide/S3onOutposts.html)
- **AWS Private 5G** — mạng 5G riêng trong khuôn viên, không ra đề SAA. [Private 5G](https://docs.aws.amazon.com/private-networks/latest/userguide/what-is-private-5g.html)
- **Amazon File Cache** — cache tốc độ cao trước dữ liệu phân tán, thuộc họ FSx. [File Cache](https://docs.aws.amazon.com/fsx/latest/FileCacheGuide/what-is.html)
- **AWS Data Transfer Terminal** — cơ sở vật lý thay Snow cho chuyển dữ liệu, chưa vào đề SAA-C03. [Data Transfer Terminal](https://aws.amazon.com/datatransferterminal/)
- **Tinkerbell, CloudStack, Nutanix provider của EKS Anywhere** — chi tiết triển khai. [EKS Anywhere](https://anywhere.eks.amazonaws.com/)

---

## Tự kiểm tra

**1.** Outposts rack trong nhà máy mất service link 4 giờ vì đứt cáp. Nêu chính xác cái
gì còn chạy, cái gì hỏng, và ba thay đổi thiết kế đáng lẽ phải làm từ đầu.

<details><summary>Đáp án</summary>

**Còn chạy:** EC2 đang chạy, EBS volume, local gateway và toàn bộ lưu lượng tới LAN nhà
máy, container ECS đang chạy, RDS trên Outposts đang phục vụ truy vấn. **Data plane sống
nguyên vẹn.**

**Hỏng:** mọi lời gọi API — `RunInstances`, `StartInstances`, `TerminateInstances`. Mọi
thao tác cần **IAM authorization**, kể cả đọc một object S3 ở Region. Systems Manager
không quản được instance. RDS ngừng backup tự động và ngừng tự thay instance hỏng.
Instance dùng service link ra internet thì mất internet. Và **DNS hỏng** nếu Route 53
zone chỉ nằm ở Region. Metric/log được cache tại chỗ tối đa **7 ngày**, nên 4 giờ không
mất gì.

**Ba thay đổi:** (1) **Route 53 Resolver ngay trên Outpost** — điểm hỏng bị quên nhiều
nhất: database chạy cách đó hai mét nhưng ứng dụng không phân giải nổi tên nó. (2) **Bỏ
Auto Scaling khỏi đường sống còn**, cấp đủ capacity thường trực cho tải đỉnh, vì scaling
cần API ở Region. (3) **Hai đường độc lập cho service link** — DX chính, VPN qua internet
dự phòng, BGP tự chuyển.

Điều cần nói được: Outposts tách **data plane** khỏi **control plane** ở hai vị trí địa
lý khác nhau, và thiết kế đúng là thiết kế mà **đường sống còn của ứng dụng không chạm
vào control plane**.
</details>

**2.** Giải thích vì sao "reduce our on-premises storage footprint" dẫn tới cached volumes
còn "we need low-latency access to the entire dataset" dẫn tới stored volumes. Rồi nêu
một hệ quả về DR mà chỉ một trong hai chế độ có.

<details><summary>Đáp án</summary>

**Tên chế độ mô tả cái nằm ở on-premises.**

**Stored** = dữ liệu được **lưu tại chỗ**; toàn bộ dataset nằm trên đĩa của bạn nên mọi
lần đọc ở tốc độ đĩa local — thoả "low-latency access to the **entire** dataset". Cái
giá: vẫn phải mua đủ đĩa cho toàn bộ dữ liệu.

**Cached** = tại chỗ chỉ giữ **cache** phần nóng, dữ liệu chính ở AWS, nên chỉ cần đĩa
bằng kích thước tập nóng — thoả "reduce the on-premises storage footprint". Cái giá: đọc
trúng dữ liệu lạnh phải đi qua mạng.

Hai chế độ trả lời hai câu hỏi ngược nhau, nên chọn sai là chọn đúng cái phá yêu cầu.

**Hệ quả DR chỉ stored có:** stored volume tạo ra **EBS snapshot thật** ở Region, dùng
ngay được để dựng EC2 khi trung tâm dữ liệu mất — kịch bản DR rẻ nhất của Volume Gateway.
Với cached, dữ liệu vốn đã ở AWS nên "khôi phục" là dựng gateway mới trỏ vào cùng volume,
không có snapshot dựng máy được theo cách đó.

Chi tiết đi kèm: snapshot từ **cached volume lớn hơn 16 TiB** khôi phục về Storage
Gateway được nhưng **không** thành EBS volume được, vì trần EBS là 16 TiB — nên kế hoạch
DR "dựng EC2 từ snapshot" đặt ra một trần thiết kế cho kích thước volume.
</details>

**3.** Đề: *"transfer 25 TB of archived files to S3 within 30 days over a dedicated 500
Mbps link. After the migration, an on-premises application must continue reading those
files over NFS."* Chọn công cụ và giải thích vì sao dung lượng và băng thông **không**
tham gia vào quyết định chính.

<details><summary>Đáp án</summary>

Đáp án: **Amazon S3 File Gateway**.

Câu hỏi loại nửa mạnh nhất là *"sau khi xong, on-prem còn phải đọc dữ liệu đó không?"* Ở
đây là **có**, và giao thức là **NFS**. Điều đó chốt Storage Gateway loại file.

**Vì sao phép tính không quyết định:** `25 × 132 ÷ 500` ≈ **6,6 ngày**, thoải mái trong
30 ngày. Nhưng dù kịp hay không thì DataSync **vẫn sai**, vì nó chuyển xong rồi biến mất,
không để lại điểm truy cập NFS nào; Snowball cũng sai vì cùng lý do. Phép tính chỉ dùng
để chọn giữa **DataSync và Snow**, tức chỉ dùng khi đã trả lời "không" cho câu hỏi thứ
nhất.

Điểm cốt lõi: **DataSync và Snow là công cụ di chuyển; Storage Gateway là thành phần
kiến trúc thường trực.** Chúng không cạnh tranh trên cùng một trục. "25 TB" và "500 Mbps"
là **nhiễu có chủ đích**, đặt vào để xem bạn có nhảy thẳng vào phép tính mà bỏ qua câu
*"must continue reading"* hay không.

Bổ sung: File Gateway biến mỗi file thành **một object đọc thẳng được** trong S3, nên dữ
liệu đồng thời sẵn sàng cho Athena, Glue, và lifecycle sang Glacier — thứ một NAS tại chỗ
không cho bạn.
</details>

**4.** Tổ chức 8.000 nhân viên có AD tại chỗ, muốn dùng FSx for Windows và RDS for SQL
Server trên AWS, và yêu cầu tài nguyên AWS vẫn xác thực được khi Direct Connect đứt. Chọn
lựa chọn Directory Service, edition, và nêu vì sao hai lựa chọn kia sai.

<details><summary>Đáp án</summary>

Đáp án: **AWS Managed Microsoft AD, Enterprise Edition**, lập **trust hai chiều** với AD
tại chỗ.

**Vì sao Managed Microsoft AD:** hai domain controller Windows thật chạy trong VPC ở hai
AZ. DX đứt thì FSx và RDS SQL Server vẫn xác thực được với DC ngay bên cạnh chúng — đây
chính là ràng buộc quyết định.

**Vì sao Enterprise:** 8.000 nhân viên cộng máy tính và group vượt xa trần ~30.000 object
của Standard, vì mỗi nhân viên thường kéo theo vài object.

**Vì sao trust hai chiều thay vì di trú user:** giữ AD tại chỗ làm nguồn sự thật duy
nhất, nên chính sách mật khẩu, khoá tài khoản, quy trình on/offboarding không phải làm
hai lần, trong khi việc xác thực hằng ngày diễn ra ngay trong VPC.

**Hai lựa chọn kia sai:** **AD Connector** là **proxy không cache** — mọi lần xác thực
phải đi hết DX về DC ở on-prem, nên DX đứt là không ai đăng nhập được dù FSx vẫn chạy;
nó vi phạm trực tiếp ràng buộc của đề, và cũng **không chia sẻ được cross-account**.
**Simple AD** trần **5.000 user** — không đủ cho 8.000 — và **không hỗ trợ FSx, RDS for
SQL Server, RDS for Oracle, IAM Identity Center, MFA, trust, schema extension**. Nó hỏng
ở hai chỗ độc lập, và đã ngừng nhận khách hàng mới.
</details>

**5.** Giải thích vì sao RAM chia sẻ được **subnet** nhưng không chia sẻ được **VPC**, và
vì sao nó không bao giờ chia sẻ được **IAM role**. Nêu ai trả tiền cho cái gì trong VPC
sharing.

<details><summary>Đáp án</summary>

**Subnet chứ không phải VPC:** đây là quyết định có chủ đích về **quyền kiểm soát**. VPC
mang những thứ mà nếu người nhận sửa được thì phá vỡ mô hình bảo mật của cả tổ chức —
route table, internet gateway, NAT Gateway, peering, VPC endpoint. Chia sẻ ở mức subnet
cho phép **owner giữ toàn bộ quyền định tuyến và lối ra internet**, còn participant chỉ
được làm đúng một việc: đặt tài nguyên vào. Owner định nghĩa mạng, participant tiêu thụ
mạng. Ràng buộc kèm theo: chỉ trong cùng Organization, và **subnet mặc định không chia sẻ
được**.

**IAM role không bao giờ:** danh tính là tài nguyên **nội tại của account** và là ranh
giới bảo mật mạnh nhất của AWS. "Chia sẻ" một role sang account khác nghĩa là xoá bỏ
chính ranh giới đó. Cơ chế đúng đã tồn tại và khác về bản chất: `sts:AssumeRole` với
**trust policy** ghi rõ principal nào được đóng vai, kèm điều kiện như External ID hay
MFA — **cấp quyền có kiểm soát và có kiểm toán**, không phải sao chép danh tính.

Điểm bản chất: **RAM làm tài nguyên xuất hiện trong account của bạn; AssumeRole làm bạn
xuất hiện trong account khác.** Hai hướng ngược nhau.

**Ai trả tiền:** **owner** trả cho **NAT Gateway, VPC endpoint và các gateway**, vì chỉ
owner tạo được chúng. **Participant** trả cho **tài nguyên của chính mình** — EC2, RDS,
ENI của Lambda; tạo được security group riêng nhưng không sửa được gì thuộc về mạng.

Lợi ích thật của VPC sharing không phải tiền mà là **địa chỉ IP và độ phức tạp**: 40 đội
trong 3 VPC chia sẻ là 3 attachment Transit Gateway, thay vì 40.
</details>

**6.** Trạm nghiên cứu ở Nam Cực cần chạy container xử lý ảnh vệ tinh, kết nối vệ tinh
chập chờn, phải hoạt động khi mất mạng nhiều ngày. Chọn giữa ECS Anywhere, EKS Anywhere,
và Outposts, kèm lý do loại hai cái kia.

<details><summary>Đáp án</summary>

Đáp án: **EKS Anywhere**.

**Vì sao:** nó dựng một cụm Kubernetes **hoàn chỉnh và tự đủ** trên hạ tầng của bạn —
control plane, etcd, scheduler đều nằm tại trạm. AWS không tham gia vào vòng đời của pod.
Mất mạng nhiều ngày thì cụm vẫn lập lịch, vẫn thay pod chết, vẫn rolling update từ image
có sẵn cục bộ.

**Vì sao ECS Anywhere sai:** control plane của ECS nằm **ở Region AWS**; máy của bạn chỉ
là capacity đăng ký với launch type `EXTERNAL`. AWS nói rõ ECS Anywhere hỗ trợ kịch bản
có kết nối **ổn định và tin cậy**. Mất mạng thì agent có thể khởi động lại container theo
restart policy, nhưng **không lập lịch được task mới, không triển khai được phiên bản
mới, và agent chết là không có gì cứu**.

**Vì sao Outposts sai:** cùng lý do nhưng sâu hơn — control plane cũng ở Region cha, và
AWS nói thẳng Outposts **không thiết kế cho vận hành mất kết nối**. Thêm nữa nó đòi điện
5–15 kVA, làm mát, không gian 42U và **hợp đồng Enterprise Support**; đưa một giá máy như
vậy xuống Nam Cực là bài toán hậu cần trước khi là bài toán kiến trúc.

Điểm cần nói được: ba dịch vụ này khác nhau ở đúng một trục — **control plane nằm ở đâu**
— và trục đó quyết định hành vi khi mất mạng. Câu hỏi kiểm tra bạn đọc kiến trúc hay đọc
tên dịch vụ.
</details>
