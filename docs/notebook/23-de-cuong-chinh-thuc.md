# Đề cương chính thức — bốn miền, mười bốn task statement, 118 service

> **Tra nhanh:** đề cương AWS nói gì, và mỗi thứ nó nói nằm ở chương nào của sổ tay này.

`Cả 4 domain`

Nguồn duy nhất của chương này là bản PDF chính thức trong repo:
[`../solutions-architect-associate-03.pdf`](../solutions-architect-associate-03.pdf)
(Exam Guide SAA-C03, bản 2026, 30 trang). Mọi con số ở đây trích thẳng từ đó — không
phải từ blog, không phải từ khoá luyện thi.

Chương này khác mọi chương khác: nó **không dạy kỹ thuật**. Nó là bản đồ giữa *thứ AWS
tuyên bố sẽ hỏi* và *thứ sổ tay này đã viết*. Mở nó khi bạn muốn biết mình còn hổng chỗ nào.

---

## Bản đồ

| Mục | Trả lời câu hỏi |
|---|---|
| [1. Bốn miền và trọng số](#1-bốn-miền-và-trọng-số) | thi cái gì, nặng nhẹ ra sao |
| [2. Mười bốn task statement](#2-mười-bốn-task-statement) | trong mỗi miền, hỏi cụ thể cái gì |
| [3. 118 service in-scope → chương nào](#3-118-service-in-scope--chương-nào) | tra ngược từ tên service |
| [4. Out of scope — và bẫy đi kèm](#4-out-of-scope--và-bẫy-đi-kèm) | học cái gì là phí thời gian |
| [5. Đọc đề cương như một kiến trúc sư](#5-đọc-đề-cương-như-một-kiến-trúc-sư) | vì sao động từ trong task statement quan trọng |

---

## 1. Bốn miền và trọng số

| Miền | Tên | Trọng số | Chương chính |
|---|---|---|---|
| **1** | Design Secure Architectures | **30%** | [`05-security.md`](05-security.md) |
| **2** | Design Resilient Architectures | **26%** | [`11-san-sang-cao.md`](11-san-sang-cao.md), [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) |
| **3** | Design High-Performing Architectures | **24%** | [`12-hieu-nang.md`](12-hieu-nang.md) |
| **4** | Design Cost-Optimized Architectures | **20%** | [`10-chi-phi.md`](10-chi-phi.md) |

Hai điều rút ra ngay từ bảng này, và cả hai đều là quyết định phân bổ thời gian:

**Bảo mật nặng gần gấp rưỡi chi phí.** 30% so với 20%. Nếu bạn học đều tay bốn miền thì
bạn đang học sai tỷ lệ.

**Cả bốn động từ đều là "Design".** Không miền nào tên là "Operate", "Troubleshoot" hay
"Implement". Đề không hỏi bạn gõ lệnh gì; nó hỏi bạn **chọn kiến trúc nào và vì sao**.
Đó là lý do chương [`30-thiet-ke-he-thong.md`](30-thiet-ke-he-thong.md) tồn tại, và là
lý do bộ lab [`labs-self/`](../../learn-aws/labs-self/) bắt bạn tự viết thay vì đọc lời giải.

---

## 2. Mười bốn task statement

Dịch sát nghĩa, kèm chương trả lời.

### Miền 1 — Thiết kế kiến trúc an toàn (30%)

| Task | Nội dung | Đọc ở |
|---|---|---|
| **1.1** | Thiết kế **truy cập an toàn** tới tài nguyên AWS | [`05-security.md`](05-security.md) §1–§7 |
| **1.2** | Thiết kế **workload và ứng dụng** an toàn | [`05-security.md`](05-security.md) §11–§14, [`04-networking.md`](04-networking.md) |
| **1.3** | Xác định **kiểm soát bảo mật dữ liệu** phù hợp | [`05-security.md`](05-security.md) §8–§10, [`02-storage.md`](02-storage.md) |

Chi tiết đáng chú ý trong 1.1: đề cương liệt kê thẳng **"Access controls and management
across multiple accounts"** và **Control Tower, SCP**. Nghĩa là quản trị nhiều account
không phải kiến thức nâng cao tuỳ chọn — nó nằm ngay trong miền nặng nhất. Xem
[`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) §7.

Trong 1.3, đề cương ghi rõ **"Rotating encryption keys and renewing certificates"** —
xoay khoá và gia hạn chứng chỉ là *kỹ năng được nêu tên*, không phải chi tiết phụ.

### Miền 2 — Thiết kế kiến trúc bền bỉ (26%)

| Task | Nội dung | Đọc ở |
|---|---|---|
| **2.1** | Thiết kế kiến trúc **mở rộng được và ghép lỏng** | [`06-tich-hop.md`](06-tich-hop.md), [`01-compute.md`](01-compute.md) |
| **2.2** | Thiết kế kiến trúc **sẵn sàng cao / chịu lỗi** | [`11-san-sang-cao.md`](11-san-sang-cao.md), [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) |

Đề cương gọi tên bốn chiến lược DR trong 2.2 nguyên văn: **backup and restore, pilot
light, warm standby, active-active failover**. Lưu ý chữ cuối — nhiều tài liệu (kể cả
whitepaper DR của chính AWS) gọi nó là **multi-site active/active**. Hai tên, một thứ.
Gặp tên nào cũng phải nhận ra.

2.2 cũng nêu đích danh **RDS Proxy** và **service quota trong môi trường standby** —
hai thứ rất hay bị bỏ qua. Quota của Region dự phòng không tự bằng quota Region chính,
và đó là cách một kế hoạch DR trông hoàn hảo trên giấy chết lúc failover thật.

### Miền 3 — Thiết kế kiến trúc hiệu năng cao (24%)

| Task | Nội dung | Đọc ở |
|---|---|---|
| **3.1** | Lưu trữ hiệu năng cao / mở rộng được | [`02-storage.md`](02-storage.md) |
| **3.2** | Compute hiệu năng cao và co giãn | [`01-compute.md`](01-compute.md) |
| **3.3** | Database hiệu năng cao | [`03-database.md`](03-database.md) |
| **3.4** | Kiến trúc mạng hiệu năng cao / mở rộng được | [`04-networking.md`](04-networking.md) |
| **3.5** | **Nạp và biến đổi dữ liệu** hiệu năng cao | [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md) |

**Task 3.5 là chỗ hổng lớn nhất của hầu hết người tự học.** Nó là một task statement
riêng, ngang hàng với compute và database, và nó nói về ingest + transform: Kinesis,
Firehose, MSK, Glue, EMR, Athena, Redshift. Miền 3 có **năm** task statement — nhiều
nhất trong bốn miền — nên mỗi task ở đây chiếm khoảng 4,8% tổng điểm.

### Miền 4 — Thiết kế kiến trúc tối ưu chi phí (20%)

| Task | Nội dung | Đọc ở |
|---|---|---|
| **4.1** | Lưu trữ tối ưu chi phí | [`10-chi-phi.md`](10-chi-phi.md), [`02-storage.md`](02-storage.md) |
| **4.2** | Compute tối ưu chi phí | [`10-chi-phi.md`](10-chi-phi.md), [`01-compute.md`](01-compute.md) |
| **4.3** | Database tối ưu chi phí | [`10-chi-phi.md`](10-chi-phi.md), [`03-database.md`](03-database.md) |
| **4.4** | **Kiến trúc mạng** tối ưu chi phí | [`10-chi-phi.md`](10-chi-phi.md), [`04-networking.md`](04-networking.md) |

Miền 4 chia đúng theo bốn trụ **storage / compute / database / network**. Task 4.4 tồn
tại vì **data transfer là khoản tiền người mới luôn quên** — NAT Gateway, truyền ra
internet, truyền chéo AZ. Xem [`10-chi-phi.md`](10-chi-phi.md).

---

## 3. 118 service in-scope → chương nào

Danh sách chính thức, giữ nguyên cách AWS phân nhóm. Cột cuối là chương trả lời.

### Analytics

| Service | Đọc ở |
|---|---|
| Athena · Glue · Lake Formation · EMR · Redshift · OpenSearch Service | [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md) |
| Kinesis Data Streams · Data Firehose · MSK | [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md), [`06-tich-hop.md`](06-tich-hop.md) §9 |
| QuickSight · Data Exchange | [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md) |

### Application Integration

| Service | Đọc ở |
|---|---|
| SQS · SNS · EventBridge · Step Functions · Amazon MQ · AppFlow | [`06-tich-hop.md`](06-tich-hop.md) |

### AWS Cost Management

| Service | Đọc ở |
|---|---|
| Budgets · Cost Explorer · Cost and Usage Report · Savings Plans | [`10-chi-phi.md`](10-chi-phi.md) |

### Compute · Containers · Serverless

| Service | Đọc ở |
|---|---|
| EC2 · EC2 Auto Scaling · Batch · Elastic Beanstalk · Lambda · Fargate | [`01-compute.md`](01-compute.md) |
| ECS · ECR · EKS | [`01-compute.md`](01-compute.md) §10 |
| ECS Anywhere · EKS Anywhere · EKS Distro · Outposts · Wavelength · VMware Cloud on AWS | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) |
| Serverless Application Repository | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) — mức "biết nó tồn tại" |

### Database

| Service | Đọc ở |
|---|---|
| RDS · Aurora · Aurora Serverless · DynamoDB · ElastiCache | [`03-database.md`](03-database.md) |
| DocumentDB · Neptune · Keyspaces | [`03-database.md`](03-database.md) — mức "nhận ra bài toán" |
| Redshift | [`08-phan-tich-du-lieu.md`](08-phan-tich-du-lieu.md) |

### Networking and Content Delivery

| Service | Đọc ở |
|---|---|
| VPC · ELB · Route 53 · CloudFront · Global Accelerator · PrivateLink · Transit Gateway | [`04-networking.md`](04-networking.md) |
| Direct Connect · Site-to-Site VPN · Client VPN | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md), [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) §11 |

### Security, Identity, and Compliance

| Service | Đọc ở |
|---|---|
| IAM · IAM Identity Center · Cognito · KMS · CloudHSM · Secrets Manager · ACM | [`05-security.md`](05-security.md) |
| WAF · Shield · Network Firewall · Firewall Manager | [`05-security.md`](05-security.md) §11 |
| GuardDuty · Inspector · Macie · Detective · Security Hub | [`05-security.md`](05-security.md) §12 |
| Directory Service · RAM | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) |
| Artifact | [`05-security.md`](05-security.md) — mức "biết dùng để làm gì" |

### Storage

| Service | Đọc ở |
|---|---|
| S3 · S3 Glacier · EBS · EFS · FSx (mọi loại) | [`02-storage.md`](02-storage.md) |
| AWS Backup | [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) |
| Storage Gateway | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) |

### Management and Governance

| Service | Đọc ở |
|---|---|
| CloudWatch · CloudTrail · Config · Systems Manager · CloudFormation | [`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) |
| Organizations · Control Tower · Service Catalog · License Manager | [`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) §7 |
| Trusted Advisor · Health Dashboard · Well-Architected Tool · Compute Optimizer | [`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) §10, [`10-chi-phi.md`](10-chi-phi.md) |
| Auto Scaling · CLI · Management Console | [`01-compute.md`](01-compute.md) §8, [`00-nen-tang.md`](00-nen-tang.md) §5 |
| Managed Grafana · Managed Service for Prometheus | [`09-ai-ml-media.md`](09-ai-ml-media.md) — mức "biết khi nào chọn thay CloudWatch" |

### Migration and Transfer

| Service | Đọc ở |
|---|---|
| DMS · Application Migration Service · Snow Family | [`13-khoi-phuc-tham-hoa.md`](13-khoi-phuc-tham-hoa.md) §8–§10 |
| DataSync · Transfer Family | [`14-hybrid-va-bien.md`](14-hybrid-va-bien.md) |

### Machine Learning · Media · Front-End

| Service | Đọc ở |
|---|---|
| Comprehend · Lex · Polly · Rekognition · Textract · Transcribe · Translate · SageMaker AI | [`09-ai-ml-media.md`](09-ai-ml-media.md) |
| Elastic Transcoder · Kinesis Video Streams | [`09-ai-ml-media.md`](09-ai-ml-media.md) |
| Amplify · Device Farm | [`09-ai-ml-media.md`](09-ai-ml-media.md) |

### Developer Tools

| Service | Đọc ở |
|---|---|
| X-Ray | [`12-hieu-nang.md`](12-hieu-nang.md), [`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) |

X-Ray là **dịch vụ Developer Tools duy nhất** in-scope. Mọi thứ Code* khác đều ngoài phạm vi.

---

## 4. Out of scope — và bẫy đi kèm

Đề cương có hẳn một danh sách "Out-of-Scope AWS Services". Học chúng là phí thời gian
cho kỳ thi này. Những cái đáng nhớ vì **hay bị nhầm là in-scope**:

| Ngoài phạm vi | Vì sao dễ nhầm |
|---|---|
| **Lightsail** | Trông giống EC2 giá rẻ, hay xuất hiện trong khoá luyện thi cũ. Xem ghi chú ở [`01-compute.md`](01-compute.md) §11 |
| **AWS CDK** | CloudFormation **in-scope**, CDK **không**. Đề hỏi hạ tầng bằng mã ở mức khái niệm |
| **CodeBuild · CodeCommit · CodeDeploy · CodeArtifact** | Cả bộ CI/CD ngoài phạm vi. X-Ray là Developer Tool duy nhất còn lại |
| **Elemental MediaConvert · MediaLive · MediaPackage · MediaTailor · MediaConnect** | Cả họ Elemental ngoài phạm vi, nhưng **Elastic Transcoder thì in-scope**. Đây là bẫy media kinh điển |
| **MWAA (Managed Airflow)** | Step Functions in-scope, MWAA không. Điều phối luồng công việc → nghĩ Step Functions |
| **Fault Injection Simulator** | Chaos engineering là ý tưởng hay nhưng không ra thi |
| **DevOps Guru · Personalize · HealthLake · Location Service** | Là dịch vụ ML/AI nhưng **không** nằm trong 8 dịch vụ AI in-scope |
| **Cloud Map** | Service discovery — ngoài phạm vi, dù ECS có tích hợp |
| **RDS on VMware** | Trong khi **VMware Cloud on AWS** lại in-scope |
| **CloudShell · Console Mobile App · Tools and SDKs** | Công cụ, không phải quyết định kiến trúc |

Bẫy đáng nhớ nhất trong bảng trên là cặp **Elastic Transcoder in / Elemental ngoài** và
cặp **VMware Cloud on AWS in / RDS on VMware ngoài**. Cả hai đều là chỗ trực giác dẫn sai.

---

## 5. Đọc đề cương như một kiến trúc sư

Đề cương không chỉ liệt kê service. Nó nói **mức độ** bạn phải biết, qua hai từ khoá lặp
đi lặp lại trong từng task statement:

- **"Knowledge of"** — bạn phải *nhận ra* và *giải thích được*. Mức đọc hiểu.
- **"Skills in"** — bạn phải *làm được quyết định*. Mức thiết kế.

Ví dụ ở Task 1.1: "Knowledge of: AWS global infrastructure" nhưng "**Skills in**:
Designing a role-based access control strategy". Nghĩa là bạn được phép chỉ *biết* AZ và
Region là gì, nhưng phải *thiết kế được* một chiến lược RBAC. Hai mức khác nhau, và
chúng nói cho bạn biết chỗ nào cần đọc sâu, chỗ nào đọc lướt là đủ.

Đếm nhanh các động từ trong mục "Skills in" của cả 14 task statement, ba động từ chiếm
đa số: **Designing**, **Determining**, **Selecting**. Không có động từ nào là
"Configuring" hay "Troubleshooting". Đó là lý do một người thuộc lòng từng tham số
`terraform` vẫn có thể trượt, còn một người **giải thích được vì sao chọn cái này thay
cái kia** thì qua.

Chỗ luyện đúng năng lực đó:
[`30-thiet-ke-he-thong.md`](30-thiet-ke-he-thong.md) cho phương pháp, và
[`labs-self/`](../../learn-aws/labs-self/) cho phần tay làm.

---

## Bảng số phải nhớ

| Con số | Giá trị | Ghi chú |
|---|---|---|
| Trọng số 4 miền | **30 / 26 / 24 / 20** | Secure / Resilient / Performing / Cost |
| Số task statement | **14** | 3 + 2 + 5 + 4 |
| Miền có nhiều task nhất | **Miền 3** (5 task) | mỗi task ≈ 4,8% tổng điểm |
| Service in-scope | **118** | danh sách "non-exhaustive", có thể đổi |
| Số câu hỏi | **65** = **50** tính điểm + **15** không tính điểm | đề **không đánh dấu** câu nào không tính → không được bỏ câu nào |
| Điểm đạt | **720** trên thang **100–1000** | thang chuẩn hoá, **không phải** phần trăm số câu đúng |
| Điểm liệt theo miền | **không có** | nguyên văn: *"You need to pass only the overall exam"* |
| Kinh nghiệm giả định | **1 năm** thiết kế giải pháp trên AWS | đề viết cho người đã có tay nghề, không phải người mới |
| Câu 1 đáp án | 1 đúng + **3** nhiễu | |
| Câu nhiều đáp án | **≥2** đúng trong **≥5** phương án | |
| Bỏ trống | tính **sai** | không phạt đoán → không bao giờ để trống |
| Thời lượng | **đề cương này không nêu** | tra trang đăng ký, đừng tin con số truyền miệng |

---

## Bẫy đề thi

1. **Nhìn trọng số mà học đều tay.** Miền 1 nặng gấp rưỡi miền 4. Thời gian ôn nên chia
   theo tỷ lệ đó, không chia đều bốn phần.
2. **Bỏ qua task 3.5.** Nạp và biến đổi dữ liệu là một task statement đầy đủ, nhưng hầu
   hết người tự học chỉ ôn EC2/S3/RDS rồi bỏ trống Kinesis–Glue–Athena.
3. **Học service ngoài phạm vi.** Đặc biệt cả họ Elemental và bộ Code*. Danh sách
   out-of-scope dài gần bằng danh sách in-scope, và nó có ở mục 4 bên trên.
4. **Coi "non-exhaustive" là "không cần chính xác".** Danh sách có thể đổi, nhưng nó vẫn
   là cam kết gần nhất bạn có từ phía AWS. Một service không nằm trong đó gần như chắc
   chắn không phải đáp án đúng.
5. **Nhầm "active-active failover" với một chiến lược thứ năm.** Nó chính là multi-site.
6. **Quên rằng cả bốn miền đều bắt đầu bằng "Design".** Đề không hỏi cách làm, nó hỏi
   lựa chọn nào và vì sao.

---

## Cây quyết định

```
Bạn đang muốn gì?
│
├─ "Tôi còn hổng chỗ nào?"
│     -> mục 3, dò cột phải; chương nào bạn chưa mở là chỗ hổng
│
├─ "Nên dành thời gian cho miền nào?"
│     -> mục 1. Chia thời gian theo 30/26/24/20, không chia đều
│
├─ "Service X có ra thi không?"
│     -> mục 3 (có) hoặc mục 4 (không). Không có ở cả hai -> gần như chắc là không
│
├─ "Tôi phải biết service này SÂU tới đâu?"
│     -> mục 5. Tra task statement: "Knowledge of" = nhận ra;
│        "Skills in" = thiết kế được
│
└─ "Tôi biết lý thuyết rồi, giờ luyện thiết kế thế nào?"
      -> 30-thiet-ke-he-thong.md, rồi labs-self/
```

---

## Nối với thực hành

Bản đồ từ task statement sang lab tự viết:

| Task | Lab |
|---|---|
| 1.1, 1.3 | [`w01-iam-foundations`](../../learn-aws/labs-self/w01-iam-foundations/README.md), [`w09-security-deep`](../../learn-aws/labs-self/w09-security-deep/README.md) |
| 1.2 | [`w02-vpc-networking`](../../learn-aws/labs-self/w02-vpc-networking/README.md) |
| 2.1 | [`w06-serverless-api`](../../learn-aws/labs-self/w06-serverless-api/README.md), [`w07-decoupling`](../../learn-aws/labs-self/w07-decoupling/README.md) |
| 2.2 | [`w03-ec2-alb-asg`](../../learn-aws/labs-self/w03-ec2-alb-asg/README.md), [`w11-dr-hybrid`](../../learn-aws/labs-self/w11-dr-hybrid/README.md) |
| 3.1, 3.3 | [`w04-s3-cloudfront`](../../learn-aws/labs-self/w04-s3-cloudfront/README.md), [`w05-databases`](../../learn-aws/labs-self/w05-databases/README.md) |
| 3.4 | [`w08-dns-cdn-edge`](../../learn-aws/labs-self/w08-dns-cdn-edge/README.md) |
| 4.1–4.4 | mọi lab đều có output `chi_phi` và một check phủ định về tiền |
| Tổng ôn | [`w12-exam-review`](../../learn-aws/labs-self/w12-exam-review/README.md) |

Task **3.2** và **3.5** hiện chưa có lab riêng — đó là chỗ hổng đã biết của bộ lab, không
phải chỗ hổng của đề cương.

---

## Nguồn nói khác

**Khoá luyện thi cũ vẫn dạy Lightsail như nội dung chính.** Đề cương 2026 xếp nó vào
out-of-scope. Nguồn cũ không sai vào thời của nó — danh sách đã đổi.

**Nhiều nguồn ghi "65 câu, 720 điểm để đạt" mà không nói 15 câu không tính điểm.** Hệ quả
thực tế: bạn không biết câu nào không tính, nên không được bỏ câu nào.

**Một số nguồn ghi đề có câu kéo-thả hoặc tự luận.** Không có. Chỉ trắc nghiệm một đáp án
và trắc nghiệm nhiều đáp án.

**Rất nhiều nguồn nói bạn phải đạt điểm sàn ở TỪNG miền.** Sai. Đề cương ghi nguyên văn:
*"You need to pass only the overall exam."* Bảng phân tích theo miền trong phiếu điểm chỉ
để bạn biết mình yếu chỗ nào, nó **không** phải điều kiện đạt. Hệ quả chiến thuật: bỏ hẳn
một mảng nhỏ trong miền nhẹ vẫn qua được, miễn tổng đủ 720.

**Bản thân AWS ghi danh sách service là "non-exhaustive and subject to change".** Nên
chương này có mốc: đối chiếu với bản PDF trong repo, tải ngày ghi trong git log. Trước khi
thi, tải lại bản mới nhất và so.

---

## Ngoài phạm vi

- **Đề cương của các kỳ thi khác** (Developer, SysOps, Professional) — cấu trúc khác hẳn.
  [certification](https://aws.amazon.com/certification/)
- **Cách đăng ký, phí thi, chính sách thi lại** — thay đổi theo khu vực, tra trang chính thức.
- **Ngân hàng câu hỏi thật** — không tồn tại hợp pháp. Thứ thay thế đúng đắn là
  [`w12-exam-review`](../../learn-aws/labs-self/w12-exam-review/README.md): tự viết câu
  hỏi buộc bạn hiểu sâu hơn nhiều so với làm câu người khác viết.

---

## Tự kiểm tra

**1.** Bạn có 40 giờ ôn tập. Chia theo miền thế nào, và vì sao không chia đều?

<details><summary>Đáp án</summary>

Chia theo trọng số: **Secure 12 giờ · Resilient 10,5 giờ · Performing 9,5 giờ · Cost 8 giờ**.

Nhưng đó mới là bước một. Bước hai là điều chỉnh theo **chỗ bạn yếu**, vì trọng số nói về
đề chứ không nói về bạn. Cách làm đúng: chia theo trọng số trước, làm một lượt
[`Tự kiểm tra`] cuối mỗi chương, rồi dồn thời gian còn lại vào miền có tỷ lệ sai cao nhất.

Sai lầm phổ biến là dồn hết vào miền mình *thích* — thường là networking, vì nó cụ thể và
dễ thấy kết quả. Networking nằm trong miền 3 (24%) và một phần miền 1.
</details>

**2.** Đề nhắc tới một dịch vụ chuyển mã video. Bạn nghĩ ngay tới MediaConvert. Vì sao đó
có thể là sai, và đáp án đúng nhiều khả năng là gì?

<details><summary>Đáp án</summary>

**Cả họ AWS Elemental — MediaConvert, MediaLive, MediaPackage, MediaTailor, MediaConnect —
đều nằm trong danh sách out-of-scope.** Dịch vụ chuyển mã in-scope duy nhất là
**Amazon Elastic Transcoder**.

Điều cần nói được: MediaConvert là thứ AWS thật sự khuyên dùng ngày nay, và Elastic
Transcoder là dịch vụ cũ hơn. Nên ở đây **đáp án đúng của đề thi khác với lựa chọn đúng
của một kiến trúc thật**. Đó là một trong số ít chỗ hai thứ đó tách nhau, và biết mình
đang ở trong chỗ nào là điều quan trọng.
</details>

**3.** Task statement ghi "Knowledge of: AWS global infrastructure" nhưng "Skills in:
Designing a role-based access control strategy". Hai mức đó khác nhau thế nào trong cách
bạn ôn?

<details><summary>Đáp án</summary>

**Knowledge of** = nhận ra và giải thích được. Với global infrastructure: biết AZ là gì,
Region là gì, ranh giới giữa chúng ra sao. Đọc một lượt, làm vài câu hỏi là đủ.

**Skills in** = ra được quyết định trong tình huống mới. Với RBAC: cho một đề bài có bốn
nhóm người dùng và ba mức dữ liệu, bạn phải **vẽ ra được** mô hình role, trust policy,
và ranh giới quyền — chứ không phải nhớ định nghĩa của "role".

Cách ôn tương ứng cũng khác: mục Knowledge ôn bằng đọc; mục Skills chỉ ôn được bằng
**làm** — tức là [`labs-self/`](../../learn-aws/labs-self/) và các đề thiết kế mở trong
[`30-thiet-ke-he-thong.md`](30-thiet-ke-he-thong.md).
</details>

**4.** Vì sao miền 3 có 5 task statement mà vẫn chỉ chiếm 24%, ít hơn miền 1 chỉ có 3 task?

<details><summary>Đáp án</summary>

Vì **trọng số miền và số task statement không liên quan nhau**. Trọng số là tỷ lệ câu hỏi;
task statement chỉ là cách AWS chia nhỏ nội dung để mô tả.

Hệ quả thực tế đáng nhớ: mỗi task của miền 3 chỉ đáng ≈ **4,8%** tổng điểm, còn mỗi task
của miền 1 đáng ≈ **10%**. Nên nếu bạn phải bỏ một mảng nhỏ, bỏ trong miền 3 rẻ hơn bỏ
trong miền 1 — dù miền 3 trông đồ sộ hơn vì có nhiều đầu mục hơn.
</details>

**5.** Một người ôn ba tháng, thuộc mọi tham số của mọi service, làm đề thử toàn 90%, nhưng
trượt. Đề cương giải thích điều đó thế nào?

<details><summary>Đáp án</summary>

Vì cả bốn miền đều bắt đầu bằng **"Design"**, và mục "Skills in" của cả 14 task statement
dùng ba động từ: **Designing, Determining, Selecting**. Không có "Configuring", không có
"Troubleshooting", không có "Memorizing".

Đề cho bốn phương án **đều chạy được**, rồi hỏi cái nào đúng nhất với **ràng buộc trong
đề** — thường là chi phí, RTO/RPO, hoặc một điều kiện tuân thủ giấu trong một câu văn.
Người thuộc tham số vẫn có thể không rút được ràng buộc ra khỏi văn xuôi.

Đề thử toàn 90% mà trượt thật thường có thêm một lý do nữa: đề thử hay hỏi *"service X làm
gì"*, còn đề thật hỏi *"tình huống này chọn gì"*. Hai kỹ năng khác nhau.
</details>

**6.** Danh sách in-scope ghi rõ "non-exhaustive and subject to change". Bạn nên xử lý câu
đó thế nào cho đúng?

<details><summary>Đáp án</summary>

**Không coi nó là cái cớ để học lan man, cũng không coi nó là danh sách đóng.**

Cách dùng đúng: danh sách in-scope là **ưu tiên**, danh sách out-of-scope là **loại trừ**,
và khoảng trống giữa hai danh sách là vùng xám nhỏ. Một service nằm trong out-of-scope thì
gần như chắc chắn không phải đáp án đúng — đó là thông tin có giá trị khi loại phương án.

Việc cần làm trước khi thi: tải lại bản đề cương mới nhất và `diff` với bản trong repo
([`../solutions-architect-associate-03.pdf`](../solutions-architect-associate-03.pdf)).
AWS đổi danh sách này giữa các bản mà không đổi mã đề.
</details>
