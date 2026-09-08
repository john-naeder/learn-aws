# Dữ liệu và phân tích

> **Tra nhanh:** dữ liệu đi từ nguồn tới biểu đồ qua sáu chặng — chặng nào dùng
> dịch vụ nào, cơ chế bên dưới là gì, cái gì làm nó gãy ở quy mô lớn, và quyết
> định nào đổi hoá đơn hàng trăm lần.

`Domain 3 · Kiến trúc hiệu năng cao (24% đề)` · `Domain 4 · Kiến trúc tối ưu chi phí (20% đề)`

Không có tuần riêng cho chủ đề này: phần streaming nằm ở
[Tuần 7](../aws/w07-decoupling.md), phần lưu trữ ở [Tuần 4](../aws/w04-s3-cloudfront.md).
File này viết để bạn **thiết kế được một hệ dữ liệu**, không phải để nhận diện từ
khoá — mục [Bẫy đề thi](#bẫy-đề-thi) là phần phụ, không phải trục chính.

Ba câu hỏi được trả lời cho mọi dịch vụ trong file: **nó chạy thế nào bên dưới**,
**nó ngồi ở chặng nào trong một thiết kế thật**, và **con số nào chứng minh lựa
chọn đó đúng** (mốc kiểm chứng: **2026-08**).

## Bản đồ

| Mục | Đọc khi bạn cần |
|---|---|
| [1. Sáu chặng và sơ đồ luồng dữ liệu](#1-sáu-chặng-và-sơ-đồ-luồng-dữ-liệu) | chưa biết đặt dịch vụ nào ở đâu; cần cái khung trước khi đọc chi tiết |
| [2.1 Bốn cách đưa dữ liệu vào](#21-bốn-cách-đưa-dữ-liệu-vào) | đề trộn Kinesis Data Streams, Firehose, MSK và SQS — bảng quan trọng nhất file |
| [2.2 Kinesis Data Streams](#22-kinesis-data-streams-ở-góc-phân-tích) | phải tính số shard, gặp hot shard, hoặc cần replay |
| [2.3 Amazon Data Firehose](#23-amazon-data-firehose-và-cơ-chế-buffer) | nạp vào S3/OpenSearch, và mọi câu hỏi về độ trễ hoặc kích thước file |
| [2.4 Amazon MSK](#24-amazon-msk) | đề nói "Kafka", hoặc phải chọn giữa MSK và Kinesis |
| [3. Định dạng, nén, phân vùng](#3-chặng-lưu-định-dạng-nén-phân-vùng) | hoá đơn Athena cao, query chậm, "sao vẫn quét cả bảng" |
| [4. AWS Glue](#4-aws-glue) | catalog schema, crawler, ETL job — và khi nào **không** cần crawler |
| [5. Lake Formation](#5-lake-formation) | quyền mức cột/hàng, chia sẻ chéo account, "cấp quyền rồi mà ai cũng đọc được" |
| [6. Athena](#6-athena) | truy vấn ad-hoc trên S3, kiểm soát chi phí, giới hạn thật |
| [7. Redshift](#7-redshift) | kho dữ liệu, Serverless, Spectrum, concurrency |
| [8. Data lake hay data warehouse](#8-data-lake-hay-data-warehouse) | phải chọn giữa S3+Athena và Redshift, hoặc chứng minh "cả hai" |
| [9. Amazon EMR](#9-amazon-emr) | Spark/Hive/HBase, lift-and-shift Hadoop, Spot cho task node |
| [10. OpenSearch Service](#10-opensearch-service) | full-text, log analytics, phân tầng hot/UltraWarm/cold |
| [11. QuickSight](#11-quicksight) | dashboard, SPICE, và mô hình giá là câu hỏi thi thật |
| [12. Ba bài toán thiết kế](#12-ba-bài-toán-thiết-kế) | muốn thấy toàn bộ file được dùng một lần, kèm lý do loại phương án |
| [Bảng số phải nhớ](#bảng-số-phải-nhớ) | ôn 10 phút trước khi thi |
| [Nguồn nói khác](#nguồn-nói-khác) | `aws-saa-c03/09-analytics-bigdata.md` đã cũ ở nhiều chỗ |

Liên quan: [`02-storage.md`](02-storage.md) cho S3 và storage class,
[`03-database.md`](03-database.md) cho DynamoDB/Aurora làm nguồn,
[`06-tich-hop.md`](06-tich-hop.md) cho Kinesis ở góc *tích hợp ứng dụng*,
[`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md) cho CloudWatch Logs,
[`10-chi-phi.md`](10-chi-phi.md) cho mô hình giá.

---

## 1. Sáu chặng và sơ đồ luồng dữ liệu

Mọi hệ dữ liệu trên AWS đều là sáu chặng nối nhau. Bạn thiết kế bằng cách chọn
**đúng một dịch vụ cho mỗi chặng**, rồi kiểm tra ranh giới giữa hai chặng kề
nhau — chỗ gãy luôn nằm ở ranh giới, không nằm trong một dịch vụ.

```
 NGUỒN            INGEST              LƯU              XỬ LÝ           TRUY VẤN        TRỰC QUAN
 -----            ------              ---              -----           --------        ---------

 app / web  --+
 clickstream  |
              +--> Kinesis Data --+
 IoT          |    Streams        |
 telemetry  --+                   +--> Data Firehose --> S3 raw --+
                                  |    (buffer, Parquet)          |
 log ứng dụng --> CloudWatch Logs |                               |
                  (subscription   |                               v
                   filter) -------+                          Glue ETL job
                                                             EMR (Spark)
 Kafka có sẵn --> Amazon MSK -----------------------------+       |
                                                          |       v
 RDS / Aurora --> zero-ETL / DMS -----------------------+ |  S3 curated  --+--> Athena -----+
 DynamoDB                                               | |  (Parquet,     |                |
                                                        v v   phân vùng)   +--> Redshift ---+--> QuickSight
 SaaS         --> AppFlow ---------------------------> S3 lake             |    Spectrum    |    (SPICE)
 (Salesforce)                                            |                 |                |
                                                         |                 +--> Redshift ---+
 bên thứ ba   --> AWS Data Exchange ------------------>  |                      (COPY)      |
                                                         |                                  |
                                                         +--> OpenSearch -------------------+
                                                                                  (Dashboards)

           QUẢN TRỊ NGANG QUA MỌI CHẶNG:
           Glue Data Catalog  = schema dùng chung cho Athena, EMR, Redshift Spectrum, Glue
           Lake Formation     = quyền mức database / bảng / cột / hàng trên chính catalog đó
```

Ba điều đọc ra được từ sơ đồ, và cả ba đều ra thi:

**Firehose ngồi *sau* Data Streams chứ không thay nó.** Một stream có thể có
Firehose làm consumer để đổ vào S3, đồng thời có consumer khác đọc real-time. Đề
mô tả "vừa cần dashboard dưới một giây, vừa cần lưu vào data lake" — đó là **một**
stream với **hai** consumer, không phải hai đường ống.

**Glue Data Catalog là điểm hội tụ.** Athena, EMR, Redshift Spectrum và Glue job
đều đọc cùng một catalog. Đây là lý do "chuyển từ Athena sang Redshift Spectrum"
không phải là một dự án di chuyển dữ liệu — bảng đã ở đó rồi.

**Đơn vị tính tiền đổi ở mỗi chặng.** Ingest tính theo GB hoặc shard-giờ; lưu tính
theo GB-tháng; truy vấn tính theo **byte quét** (Athena, Spectrum) hoặc theo
**thời gian máy chạy** (Redshift, EMR); trực quan hoá tính theo **người dùng**.
Tối ưu một chặng mà không nhìn chặng kế bên là cách phổ biến nhất để làm hoá đơn
tăng: giảm buffer Firehose xuống 60 giây cho "gần real-time hơn" sẽ sinh ra file
nhỏ và làm mọi query Athena phía sau chậm và đắt hơn.

| Chặng | Dịch vụ chính | Đơn vị mở rộng | Đơn vị tính tiền |
|---|---|---|---|
| Ingest streaming | Kinesis Data Streams, Firehose, MSK | shard · (không) · broker+partition | shard-giờ · **GB nạp** · broker-giờ |
| Ingest batch | AppFlow, Data Exchange, DMS, zero-ETL | flow · (không) | theo flow run · theo bộ dữ liệu |
| Lưu | S3 | không giới hạn | GB-tháng + request |
| Xử lý | Glue ETL, EMR, Managed Service for Apache Flink | DPU · node · KPU | DPU-giờ · instance-giờ |
| Truy vấn | Athena, Redshift, Redshift Spectrum | (không) · node/RPU | **byte quét** · node-giờ / RPU-giờ |
| Trực quan hoá | QuickSight, OpenSearch Dashboards | (không) · node | **người dùng** · instance-giờ |

---

## 2. Chặng ingest

### 2.1 Bốn cách đưa dữ liệu vào

Đây là câu hỏi thiết kế kinh điển của cả mảng dữ liệu. Bốn dịch vụ đều "nhận dữ
liệu và chuyển đi tiếp", nhưng khác nhau ở một trục quyết định mọi thứ còn lại:
**ai giữ vị trí đọc**.

| Trục | Kinesis Data Streams | Amazon Data Firehose | Amazon MSK | SQS |
|---|---|---|---|---|
| Bản chất | log phân tán có thứ tự | đường ống nạp, không lưu | Apache Kafka do AWS vận hành | hàng đợi công việc |
| **Ai giữ vị trí đọc** | **consumer** — sequence number, KCL lưu trong bảng DynamoDB riêng | **không ai** — không có khái niệm consumer | **consumer group** — offset lưu trong topic `__consumer_offsets` | **không ai** — message bị xoá sau khi xử lý xong |
| **Replay** | **có**, trong retention | **không**, dữ liệu đi là mất | **có**, trong retention | **không** (chỉ redrive từ DLQ) |
| **Thứ tự** | trong một **shard**, quyết bởi partition key | **không** — buffer gom lại rồi ghi | trong một **partition**, quyết bởi message key | chỉ FIFO queue, trong một `MessageGroupId` |
| Nhiều consumer đọc **cùng** dữ liệu | có, mỗi consumer một vị trí | không | có, mỗi consumer group một vị trí | không |
| **Độ trễ điển hình** | ~200 ms (poll) · **~70 ms** với enhanced fan-out | **60 giây – 15 phút**; buffer 0 giây thì **vài giây** | **~10 ms** (tuỳ `linger.ms`) | ~10 ms; long polling tới 20 giây |
| **Đơn vị mở rộng** | **shard**: 1 MB/s hoặc 1.000 record/s ghi | **không có** — tự scale | **broker + partition** | không có — tự scale, vô hình |
| **Ai vận hành** | AWS lo hạ tầng, **bạn đếm shard** và làm resharding | AWS lo hết | **bạn** lo phiên bản Kafka, số partition, rebalance, storage | AWS lo hết |
| Giữ dữ liệu tối đa | **365 ngày** | không lưu | tuỳ dung lượng broker; **không giới hạn** với Serverless | **14 ngày** |
| Kích thước bản ghi | **1 MiB** | **1.000 KiB** | 8 MiB (Serverless), cấu hình được | **256 KB** |
| Tính tiền | shard-giờ + PUT payload unit 25 KB | **theo GB nạp vào** | broker-giờ + storage; Serverless: cluster-giờ + partition-giờ + GB | theo request, mỗi 64 KB = 1 request |

Cách dùng bảng này trong một thiết kế thật:

**Đề nói "nhiều nhóm cần dữ liệu, một nhóm phải chạy lại N ngày"** → chỉ Data
Streams và MSK làm được, vì chỉ hai cái đó để consumer tự giữ vị trí. SQS xoá
message; Firehose không có consumer nào để mà tua.

**Đề nói "không viết consumer, chỉ cần dữ liệu vào S3/OpenSearch"** → Firehose.
Đây là dịch vụ duy nhất trong bốn cái **không cần bạn viết một dòng code đọc**.

**Đề nói "Kafka" hoặc liệt kê Kafka Connect / Kafka Streams / log compaction** →
MSK. Nếu đề chỉ mô tả *hành vi* (log, thứ tự, replay) mà không nhắc Kafka thì
Kinesis là đáp án rẻ hơn và ít việc hơn.

**Đề nói "một message một worker, xử lý xong thì bỏ"** → SQS. Dùng Kinesis cho
task queue là sai kiến trúc: bạn sẽ phải tự viết cơ chế báo "đã xử lý xong item
này", trong khi SQS có sẵn visibility timeout và DLQ.

Chi tiết SQS ở [`06-tich-hop.md`](06-tich-hop.md#2-sqs--hàng-đợi-kéo), phần
Kinesis ở góc *tích hợp ứng dụng* ở
[`06-tich-hop.md` §9](06-tich-hop.md#9-kinesis). Bên dưới là phần đi sâu hơn ở
góc **phân tích dữ liệu**: tính toán capacity, hot shard, và ảnh hưởng của
Firehose lên chi phí truy vấn.

### 2.2 Kinesis Data Streams ở góc phân tích

**Cơ chế.** Stream là một tập **shard**; mỗi shard là một log có thứ tự, bất
biến, chỉ ghi thêm. Producer đưa vào một `PartitionKey`, Kinesis băm MD5 khoá đó
thành một số 128 bit rồi ánh xạ vào khoảng hash mà shard đang giữ. Ba hệ quả trực
tiếp: thứ tự chỉ đảm bảo **trong một shard**; mọi bản ghi cùng khoá luôn vào cùng
shard; và **phân bố khoá lệch = shard nóng**, y hệt hot partition của DynamoDB.

**Tính số shard.** Hai trục, lấy cái lớn hơn rồi cộng biên độ:

```
shard_theo_MB     = ceil( throughput_MB_per_giây / 1 )
shard_theo_record = ceil( record_per_giây / 1000 )
shard             = max(hai cái trên) * 1,25   # biên độ cho đỉnh và khoá lệch
```

50.000 sự kiện/giây, mỗi sự kiện 1 KB: trục record cho 50 shard, trục MB cho 50
MB/s → 50 shard. Cộng biên độ 25% → **64 shard**. Chi phí shard-giờ ở us-east-1
là $0,015 → 64 shard ≈ **$690/tháng** trước tiền PUT payload unit.

**Chế độ dung lượng.** Provisioned rẻ hơn khi tải ổn định và bạn chịu đếm shard.
On-demand tự lo, mặc định **200 MB/s ghi và 400 MB/s đọc mỗi stream**, xin tăng
được tới **10 GB/s ghi và 20 GB/s đọc** (mốc 11/2024). On-demand tính theo GB nạp
cộng phí stream-giờ, và tự tách shard khi lưu lượng tăng gấp đôi trong 15 phút.

**Resharding là chỗ đau của provisioned.** Bạn không "đặt lại số shard"; bạn gọi
`UpdateShardCount` và Kinesis thực hiện chuỗi thao tác `SplitShard` /
`MergeShards`. Trong lúc đó shard cũ chuyển sang trạng thái `CLOSED` nhưng **vẫn
còn dữ liệu tới hết retention**, và consumer phải đọc hết shard cha trước khi
sang shard con để giữ thứ tự — KCL làm việc này giúp bạn, code tự viết bằng
`GetRecords` thì không. `UpdateShardCount` giới hạn nhân đôi hoặc chia đôi trong
một lần gọi, và không quá 10 lần mỗi 24 giờ.

**Chỉ số duy nhất phải cảnh báo: `IteratorAgeMilliseconds`.** Nó là khoảng cách
giữa bản ghi consumer vừa đọc và bản ghi mới nhất. Tăng đều nghĩa là consumer
chậm hơn producer; chạm mốc retention nghĩa là **bạn đang mất dữ liệu ngay lúc
này**. `WriteProvisionedThroughputExceeded` chỉ nói phía ghi bị chặn; nó không
cho biết consumer có kịp không.

**Enhanced fan-out (EFO).** Không bật thì mọi consumer chia chung **2 MB/s mỗi
shard** và phải poll. Bật thì mỗi consumer đăng ký được **2 MB/s riêng mỗi shard**
và Kinesis **đẩy** dữ liệu qua HTTP/2 — độ trễ xuống ~70 ms. Trần **20 consumer
EFO mỗi stream** (50 với On-demand Advantage). Tính phí riêng theo GB truy xuất,
nên EFO đắt hơn rõ khi có nhiều consumer; chỉ bật cho consumer thật sự cần độ trễ
thấp hoặc bị consumer khác bóp nghẹt.

### 2.3 Amazon Data Firehose và cơ chế buffer

**Cơ chế phải thuộc:** Firehose gom bản ghi vào bộ đệm và ghi ra đích khi
**buffer size** đầy **HOẶC** **buffer interval** hết — **cái nào tới trước**. Đây
là câu duy nhất giải thích được mọi hành vi còn lại của dịch vụ, kể cả hoá đơn
Athena ở chặng sau.

| Tham số | Dải giá trị | Ghi chú quyết định thiết kế |
|---|---|---|
| Buffer size (S3) | **1–128 MB** | mặc định 5 MiB — quá nhỏ cho data lake |
| Buffer interval (S3) | **0–900 giây** | **0 giây** = zero buffering, giao trong vài giây |
| Buffer size khi bật **chuyển Parquet/ORC** hoặc **dynamic partitioning** | **64–128 MB**, mặc định **128 MB** | Firehose **ép** sàn 64 MB — không thể vừa Parquet vừa dưới một phút |
| Throughput mỗi stream (direct PUT) | **5.000 record/giây · 2.000 giao dịch/giây · 5 MB/giây** | quota mềm, xin tăng được; không áp dụng khi nguồn là Data Streams hoặc MSK |
| Kích thước bản ghi | **1.000 KiB** | `PutRecordBatch` tối đa 500 bản ghi hoặc 4 MiB |

Ba điều rút ra:

**Firehose là nơi bạn quyết định kích thước file trong data lake.** Buffer 128 MB
với dữ liệu 2 TB/ngày cho ~16.000 file/ngày. Buffer 5 MB cho ~400.000 file/ngày,
và mọi query Athena phía sau trả giá — xem [3.5](#35-bài-toán-file-nhỏ). Đây là
ví dụ rõ nhất của "chỗ gãy nằm ở ranh giới giữa hai chặng".

**"Gần real-time" không còn là 60 giây.** Zero buffering (buffer interval = 0)
giao dữ liệu trong vài giây; với đích S3, Firehose chuyển sang **multipart upload**
khi interval dưới 60 giây. Nhưng zero buffering **không dùng được cùng dynamic
partitioning**, và không áp cho S3 backup. Đề vẫn ra theo lối cũ ("cần dưới một
giây → Data Streams"), nhưng con số "Firehose tối thiểu 60 giây" đã sai — xem
[Nguồn nói khác](#nguồn-nói-khác).

**Firehose tự tăng buffer khi tụt hậu.** Nếu đích nhận chậm hơn nguồn đẩy vào,
Firehose tự nâng buffer size để đuổi kịp. Hệ quả vận hành: kích thước file bạn
thấy ở S3 có thể lớn hơn cấu hình, và độ trễ tăng mà không có lỗi nào được ghi.

**Bốn việc Firehose làm trên đường đi** — mỗi việc là một đáp án thi:

- **Transform bằng Lambda**: lọc, làm giàu, đổi định dạng bản ghi. Lambda phải
  trả về từng bản ghi kèm `result` là `Ok`/`Dropped`/`ProcessingFailed`. Bản ghi
  lỗi đi vào prefix `processing-failed/` chứ không chặn stream.
- **Chuyển sang Parquet/ORC**: cần một bảng trong **Glue Data Catalog** để biết
  schema. Đây là cách rẻ nhất để có data lake dạng cột mà không viết ETL job nào.
- **Dynamic partitioning**: lấy giá trị từ chính bản ghi (qua JQ hoặc Lambda) để
  dựng prefix `s3://bucket/service=api/dt=2026-08-21/`. Không có nó thì Firehose
  chỉ phân vùng theo thời gian **giao hàng** dạng `YYYY/MM/DD/HH` — không phải
  Hive-style, nên Athena không tự nhận ra là partition.
- **Nén và mã hoá**: GZIP, Snappy, ZIP, Hadoop-Snappy; SSE-KMS ở đích.

**Chỗ Firehose gãy ở quy mô lớn:** không có replay. Đích hỏng quá thời gian retry
(S3: tới 24 giờ) thì bản ghi đi vào S3 backup nếu bạn bật, còn không thì mất. Với
data lake nghiêm túc, mẫu an toàn là **Data Streams làm nguồn cho Firehose** —
stream giữ 7 ngày, Firehose chỉ là một consumer thay thế được.

### 2.4 Amazon MSK

**Cơ chế.** MSK chạy Apache Kafka thật trên broker EC2 trong VPC của bạn (ENI
trong subnet của bạn). AWS lo dựng cụm, patch, thay broker hỏng và backup metadata
— **không** lo số partition, không lo rebalance, không lo phiên bản client. Đó là
ranh giới trách nhiệm khác hẳn Kinesis.

| | MSK Provisioned | MSK Serverless |
|---|---|---|
| Bạn chọn | loại broker, số broker, dung lượng EBS | không chọn gì |
| Throughput | theo loại broker và mạng | **200 MB/s ghi, 400 MB/s đọc** mỗi cụm |
| Partition | tuỳ cấu hình, hàng nghìn | **2.400** (topic thường), **120** (compacted) |
| Mỗi partition | tuỳ broker | **5 MB/s ghi, 10 MB/s đọc** |
| Retention | tuỳ dung lượng EBS (hoặc tiered storage) | **không giới hạn** |
| Kết nối client | tuỳ broker | **3.000** mỗi cụm, 100 kết nối mới/giây |
| Message tối đa | cấu hình được | **8 MiB** |
| Giá | broker-giờ + EBS | cluster-giờ + partition-giờ + GB vào/ra |

**Partition là đơn vị của cả thứ tự lẫn song song**, đúng như shard của Kinesis và
`MessageGroupId` của SQS FIFO. Một consumer group có tối đa **một consumer mỗi
partition** — thêm consumer quá số partition thì consumer thừa nằm không. Đây là
lý do "tăng số consumer mà throughput không tăng" là câu hỏi vận hành phổ biến
nhất của Kafka, và câu trả lời là **tăng partition**, không phải tăng máy.

**Chọn MSK khi và chỉ khi**: đã có producer/consumer viết bằng API Kafka và không
muốn sửa; cần hệ sinh thái Kafka (Kafka Connect, Kafka Streams, log compaction,
exactly-once transaction của Kafka); hoặc cần retention rất dài với chi phí thấp
qua tiered storage. Mọi trường hợp khác, **Kinesis Data Streams rẻ hơn về tiền và
rẻ hơn nhiều về công vận hành** — MSK Provisioned nhỏ nhất cũng cỡ $150/tháng, và
bạn vẫn phải tự quyết số partition.

### 2.5 AppFlow và AWS Data Exchange

Hai dịch vụ cùng giải một bài toán: **dữ liệu không nằm trong tài khoản của bạn**.

**Amazon AppFlow** nối SaaS với AWS mà không viết code: Salesforce, SAP,
ServiceNow, Zendesk, Slack, Google Analytics, Marketo... Một *flow* chạy theo lịch,
theo sự kiện, hoặc chạy tay; làm được lọc, ánh xạ trường, ghép và validate ngay
trên đường đi. Đích là S3, Redshift, hoặc chính một SaaS khác. Đi qua **AWS
PrivateLink** nên dữ liệu không ra Internet công cộng. Đề nói *"lấy dữ liệu
Salesforce về data lake, không muốn viết và vận hành code tích hợp"* → AppFlow;
đáp án sai hấp dẫn là Lambda gọi API SaaS theo lịch, vốn đúng về kỹ thuật nhưng
bắt bạn tự lo phân trang, rate limit, retry và ánh xạ schema.

**AWS Data Exchange** là chợ dữ liệu của bên thứ ba (dữ liệu tài chính, thời tiết,
dân số, y tế). Bạn đăng ký một bộ dữ liệu, nó xuất hiện trong tài khoản của bạn
dưới dạng file trong S3, bảng qua Redshift data sharing, hoặc API. Điểm thiết kế:
nó thay thế việc **tự tải và tự đồng bộ** dữ liệu của nhà cung cấp — bản mới về là
có sự kiện EventBridge, không phải cron kiểm tra FTP. Đề nói *"cần dữ liệu thị
trường của bên thứ ba, cập nhật hàng ngày, không muốn dựng pipeline nhập"* → Data
Exchange.

---

## 3. Chặng lưu: định dạng, nén, phân vùng

Đây là mục có ảnh hưởng tiền lớn nhất trong cả file. Athena và Redshift Spectrum
tính **$5 cho mỗi TB quét**. Ba đòn bẩy — định dạng cột, nén, phân vùng — nhân với
nhau, và tích của chúng thường là **hai tới ba bậc độ lớn**.

### 3.1 Cột hay hàng, và con số thật

CSV và JSON lưu theo **hàng**: mọi cột của một bản ghi nằm cạnh nhau. Muốn đọc một
cột, engine phải đọc qua tất cả các cột khác. Parquet và ORC lưu theo **cột**: giá
trị của cùng một cột nằm liền nhau thành *column chunk*, kèm thống kê min/max cho
mỗi *row group*. Engine đọc đúng column chunk cần và **bỏ qua cả row group** khi
min/max cho thấy không có dòng nào khớp điều kiện.

Ví dụ chính thức của AWS, bảng **100 cột, tổng 4 TB, truy vấn lấy 1 cột**:

| Cách lưu | Byte thật sự bị quét | Tiền một lần chạy |
|---|---|---|
| Text thuần, không nén | **4 TB** | **$20,00** |
| Text nén GZIP (tỉ lệ 4:1) | **1 TB** | **$5,00** |
| Parquet nén (tỉ lệ 4:1) | **10 GB** | **$0,05** |

**400 lần.** Cùng một truy vấn, cùng một dữ liệu, khác mỗi cách ghi file. Và con
số này áp cho **cả Athena lẫn Redshift Spectrum** vì hai dịch vụ dùng chung đơn
giá và chung cách tính byte quét.

Nhận ra hai hiệu ứng riêng biệt trong bảng: nén cắt 4 lần **cho mọi truy vấn**;
định dạng cột cắt thêm 100 lần **chỉ khi truy vấn không lấy hết cột**. `SELECT *`
trên Parquet không được hưởng hiệu ứng thứ hai — đó là lý do "đừng `SELECT *`"
trong data lake không phải lời khuyên phong cách mà là dòng chi phí.

Parquet hay ORC: ở mức SAA coi như tương đương. Parquet phổ biến hơn trong hệ sinh
thái Spark/Athena/Firehose; ORC mạnh hơn trong hệ Hive và có ACID với Hive. Chọn
**Parquet** nếu không có ràng buộc gì khác.

### 3.2 Nén và tính chia tách được

Không phải kiểu nén nào cũng bằng nhau, và trục phân biệt là **splittable** — file
có chia được thành nhiều phần để nhiều worker đọc song song hay không.

| Kiểu nén | Tỉ lệ điển hình | Splittable | Dùng khi |
|---|---|---|---|
| **GZIP** | ~4:1 | **không** | file text đã nhỏ (< 128 MB); tuyệt đối tránh cho file lớn |
| **Snappy** | ~2,5:1 | có, trong Parquet/ORC | mặc định tốt cho Parquet |
| **ZSTD** | ~4:1 | có, trong Parquet/ORC | tốt hơn Snappy cả về tỉ lệ lẫn tốc độ giải nén |
| **BZIP2** | ~5:1 | có | nén rất chậm, hiếm khi đáng |
| **LZO** | ~2:1 | có (cần index) | di sản Hadoop |

Bẫy thật, và nó không phải bẫy đề thi mà là bẫy sản xuất: một file **CSV nén GZIP
2 GB** chỉ được **một** worker xử lý, vì không có cách nào bắt đầu giải nén từ
giữa luồng. Cùng dữ liệu đó ở dạng 16 file GZIP 128 MB chạy nhanh gấp 16 lần với
đúng số byte quét. Bên trong Parquet thì chuyện này không xảy ra: nén áp cho từng
column chunk, nên file Parquet luôn chia tách được bất kể dùng codec nào.

### 3.3 Phân vùng: chọn cột nào, chia nhỏ tới đâu

Phân vùng là cách bạn ánh xạ giá trị của một cột vào **đường dẫn thư mục** trên
S3, theo quy ước Hive `khoá=giá_trị`:

```
s3://lake/logs/service=api/dt=2026-08-21/part-0000.snappy.parquet
s3://lake/logs/service=api/dt=2026-08-22/part-0000.snappy.parquet
s3://lake/logs/service=worker/dt=2026-08-21/part-0000.snappy.parquet
```

Khi truy vấn có `WHERE dt='2026-08-21' AND service='api'`, engine chỉ liệt kê và
đọc đúng một prefix — gọi là **partition pruning**. Không có mệnh đề đó thì nó đọc
tất cả, và số tiền là số tiền của cả bảng.

Ba quy tắc chọn cột phân vùng:

**Phân vùng theo cột luôn xuất hiện trong `WHERE`.** Gần như luôn là thời gian.
Cột không ai lọc theo thì phân vùng chỉ tạo thêm thư mục chứ không cắt được gì.

**Cardinality phải thấp.** `user_id` là cột phân vùng tệ nhất có thể: hàng triệu
thư mục, mỗi thư mục vài KB. Cột phân vùng tốt có hàng chục tới hàng nghìn giá
trị, không phải hàng triệu.

**Mỗi partition nên chứa ít nhất ~128 MB.** Đây là quy tắc thực dụng nối 3.3 với
3.5. Với 2 TB/ngày, phân vùng theo **giờ** cho ~83 GB mỗi partition — rất tốt. Với
2 GB/ngày, phân vùng theo giờ cho 83 MB — đã ở ngưỡng dưới, và phân vùng theo
**phút** thì thảm hoạ. Cùng một sơ đồ phân vùng đúng cho hệ này và sai cho hệ kia;
con số quyết định là **lưu lượng**, không phải sở thích.

Phân vùng nhiều tầng nên đi từ **hẹp tới rộng** theo thứ tự lọc thường dùng:
`service=` rồi `dt=` nếu bạn luôn lọc theo service, ngược lại thì `dt=` trước.

### 3.4 Ba cách nạp partition vào catalog

Dữ liệu ở đúng chỗ trên S3 vẫn chưa đủ — catalog phải **biết** partition đó tồn
tại. Ba cách, và đề phân biệt chúng bằng từ khoá về tần suất:

| Cách | Cơ chế | Chi phí | Dùng khi |
|---|---|---|---|
| `MSCK REPAIR TABLE` | Athena **liệt kê toàn bộ** cây thư mục của bảng rồi thêm mọi partition Hive-style tìm được | DDL **miễn phí**, nhưng chạy lâu và có thể chạm **DDL query timeout** | tạo bảng lần đầu, hoặc khi nghi ngờ catalog lệch với S3 |
| `ALTER TABLE ADD PARTITION` | thêm đúng partition bạn chỉ định | miễn phí, chạy tức thì | thêm partition định kỳ (mỗi giờ, mỗi ngày) từ một job |
| **Partition projection** | Athena **tự tính** danh sách partition từ cấu hình (dải ngày, danh sách giá trị, số nguyên) — **không hỏi catalog** | miễn phí, **không có bước lấy metadata** | sơ đồ phân vùng đoán trước được, và số partition lớn |

Ba chỗ `MSCK REPAIR TABLE` phá bạn, đều có trong docs: nó **quét cả thư mục con**
nên hai bảng lồng nhau sẽ nuốt partition của nhau; nó **thất bại im lặng** khi giá
trị partition chứa dấu hai chấm (rất hay gặp với timestamp); và nó **bỏ qua** cột
partition bắt đầu bằng dấu gạch dưới. Nó cũng liệt kê lại **toàn bộ** lịch sử mỗi
lần chạy — bảng có dữ liệu từ 2020 thì thêm partition của hôm nay vẫn phải đọc lại
mọi thư mục từ 2020.

**Partition projection là câu trả lời đúng cho log.** Với bảng phân vùng theo ngày
trong 5 năm, catalog phải giữ 1.825 partition và Athena phải gọi Glue lấy chúng
trước mỗi truy vấn; với projection, Athena sinh danh sách trong bộ nhớ từ cấu hình
`dt.type=date`, `dt.range=2021-01-01,NOW`, `dt.format=yyyy-MM-dd`. Không crawler,
không `MSCK`, không lệch giữa S3 và catalog. Đánh đổi: partition **không tồn tại**
vẫn được sinh ra và Athena sẽ đi tìm thư mục rỗng — không sai kết quả, chỉ tốn
thời gian liệt kê.

### 3.5 Bài toán file nhỏ

Byte quét không phải chi phí duy nhất. Athena/Trino mở **mỗi object bằng một hoặc
nhiều request S3**, và lập kế hoạch bằng cách liệt kê prefix. Với hàng triệu file
nhỏ, thời gian truy vấn bị chi phối bởi độ trễ request chứ không bởi lượng dữ liệu.

Con số cho 2 TB dữ liệu một ngày:

| Kích thước file | Số object | Tiền request S3 GET một lần quét | Hệ quả |
|---|---|---|---|
| 100 KB | **20.000.000** | ~**$8,00** | query mất hàng chục phút, phần lớn là chờ S3 |
| 8 MB | 250.000 | ~$0,10 | vẫn chậm hơn cần thiết |
| **128 MB** | **16.384** | ~**$0,0066** | mục tiêu |

(GET ở us-east-1 là $0,0004 mỗi 1.000 request.) Cộng thêm: Athena tính **tối
thiểu 10 MB mỗi truy vấn**, nên một truy vấn quét 500 file 1 KB vẫn bị tính 10 MB.

Ba cách chữa, theo thứ tự nên thử:

1. **Tăng buffer Firehose lên 128 MB.** Sửa ở nguồn, không tốn gì thêm.
2. **Job compaction định kỳ**: Athena `CREATE TABLE AS SELECT` (CTAS) đọc partition
   hôm qua và ghi lại thành ít file lớn, hoặc Glue ETL job với
   `coalesce`/`repartition`. Chạy mỗi ngày một lần cho partition đã đóng.
3. **Dùng bảng Iceberg** nếu bạn cần vừa ghi liên tục vừa nén file — nhưng đó là
   ngoài phạm vi SAA.

Tổng kết ba đòn bẩy, áp lần lượt cho cùng một truy vấn "1 cột, 1 ngày trong 1 năm
dữ liệu":

| Áp thêm đòn bẩy | Byte quét | Tiền |
|---|---|---|
| CSV thuần, không phân vùng | 1.460 TB | $7.300 |
| + nén GZIP | 365 TB | $1.825 |
| + Parquet (100 cột) | 3,65 TB | $18,25 |
| + phân vùng theo ngày | **10 GB** | **$0,05** |

Đây là lý do mục này nằm trước mục Athena trong file: **thiết kế lưu trữ quyết
định chi phí truy vấn, không phải ngược lại.**

---

## 4. AWS Glue

### 4.1 Data Catalog

Data Catalog là một **Hive Metastore do AWS vận hành**: nó lưu database, bảng,
cột, kiểu dữ liệu, vị trí S3, định dạng file và danh sách partition. Nó **không
lưu dữ liệu**. Giá trị của nó nằm ở chỗ Athena, Redshift Spectrum, EMR, Glue ETL,
Lake Formation và QuickSight đều đọc **cùng một** catalog — một bảng định nghĩa
một lần, mọi engine thấy.

Giá: **1 triệu object đầu tiên (database + bảng + partition) miễn phí lưu, 1 triệu
request đầu tiên mỗi tháng miễn phí**. Với đa số hệ ở mức SAA, catalog thực tế
miễn phí — trừ khi bạn có bảng hàng triệu partition, và lúc đó vấn đề thật là sơ
đồ phân vùng chứ không phải tiền catalog.

Một catalog cho mỗi **account mỗi Region**. Chia sẻ chéo account phải qua Lake
Formation ([mục 5](#5-lake-formation)), không có cách nào khác ở mức managed.

### 4.2 Crawler và chỗ nó phá schema

**Cơ chế**: crawler liệt kê một đường dẫn S3, **lấy mẫu** một phần file, chạy chuỗi
classifier (built-in cho JSON/CSV/Parquet/ORC/Avro, hoặc grok tự viết) để suy ra
schema, rồi gộp các thư mục có schema **tương thích** thành một bảng có partition.
Giá $0,44/DPU-giờ, **tối thiểu 10 phút**, thường 2 DPU → mỗi lần chạy tối thiểu
khoảng $0,15.

Ba chỗ nó gây hại, và đây là phần cheat sheet không bao giờ nói:

- **Nó suy schema từ mẫu.** Cột toàn số trong 1.000 dòng mẫu thành `bigint`; dòng
  thứ 5.000 có chữ thì Athena trả `NULL` chứ không báo lỗi.
- **Thư mục không nhất quán sinh bảng mới.** Nếu vài prefix có schema lệch, crawler
  tạo `logs_1`, `logs_2` thay vì thêm partition. Đặt
  `TableGroupingPolicy = CombineCompatibleSchemas` để ép gộp.
- **`DELETE_FROM_DATABASE`** khiến crawler **xoá bảng** khi dữ liệu tạm biến mất
  khỏi S3. Mặc định an toàn hơn là `LOG` hoặc `DEPRECATE_IN_DATABASE`.

**Khi nào không cần crawler:** khi bạn *biết* schema. `CREATE EXTERNAL TABLE` viết
tay cộng **partition projection** cho kết quả tất định, chạy tức thì, không tốn
tiền và không bao giờ tự đổi kiểu cột sau lưng bạn. Crawler đúng cho **khám phá**
dữ liệu lạ, không đúng cho một pipeline bạn tự sinh ra dữ liệu.

### 4.3 ETL job: DPU, worker type, bookmark, Flex

Glue ETL là **Spark do AWS vận hành**. Đơn vị tính là **DPU** (Data Processing
Unit) = 4 vCPU + 16 GB RAM, giá **$0,44/DPU-giờ**, tính theo giây, **tối thiểu 1
phút** cho job Spark.

| Worker type | DPU | vCPU | RAM | Dùng cho |
|---|---|---|---|---|
| `G.025X` | 0,25 | 2 | 4 GB | **chỉ streaming job**, luồng nhỏ |
| `G.1X` | 1 | 4 | 16 GB | mặc định cho batch |
| `G.2X` | 2 | 8 | 32 GB | shuffle nặng, join lớn |
| `G.4X` / `G.8X` | 4 / 8 | 16 / 32 | 64 / 128 GB | job đói bộ nhớ, tránh `Container killed by YARN` |

Ba tính năng quyết định chi phí và tính đúng đắn:

- **Job bookmark** — Glue nhớ vị trí đã xử lý (theo timestamp object hoặc khoá
  chính) để lần chạy sau chỉ đọc dữ liệu mới. **Mặc định tắt.** Không bật thì job
  hàng ngày xử lý lại toàn bộ lịch sử, và hoá đơn tăng tuyến tính theo tuổi của
  data lake. Đây là lỗi vận hành phổ biến nhất của Glue.
- **Flex execution class** — chạy trên capacity dư của AWS, rẻ hơn đáng kể, đổi
  lại không cam kết thời điểm bắt đầu và có thể bị ngắt. Đúng cho job đêm không
  gấp; sai cho job có SLA.
- **Auto scaling** — Glue thêm/bớt worker theo từng stage thay vì bắt bạn đoán
  trước số worker.

**Glue Streaming** đọc từ Kinesis Data Streams hoặc MSK bằng Spark Structured
Streaming theo micro-batch. Nó là lựa chọn giữa Firehose (không lập trình được)
và Managed Service for Apache Flink (mạnh nhất, phức tạp nhất).

### 4.4 Chọn giữa Glue, EMR và Lambda

**Glue Studio** là giao diện kéo thả sinh ra code PySpark/Scala thật — dùng để
dựng nhanh và để đọc, không phải một runtime riêng.

| | Lambda | Glue ETL | EMR |
|---|---|---|---|
| Giới hạn thời gian | **15 phút** | không | không |
| Bộ nhớ | tới 10 GB, một máy | cụm Spark | cụm bạn định nghĩa |
| Bạn kiểm soát phiên bản engine | không | hạn chế (Glue version) | **đầy đủ** |
| Spot | không | không | **có, cho task node** |
| Hợp với | biến đổi từng object nhỏ, phản ứng sự kiện S3 | ETL định kỳ, không muốn quản cụm | job dài, tuỳ biến sâu, tối ưu chi phí bằng Spot |

Đề nói *"ETL không cần quản máy chủ"* → Glue. Đề nói *"đã có job Spark/Hive, muốn
chuyển lên cloud giữ nguyên"* hoặc *"cần giảm chi phí bằng Spot"* → EMR.

---

## 5. Lake Formation

**Cơ chế thật, và nó khác hẳn cách mọi người mô tả.** Bạn *đăng ký* một vị trí S3
với Lake Formation và giao cho nó một service role có quyền đọc S3. Từ đó, khi
Athena hay Redshift Spectrum chạy truy vấn, chúng **không** dùng quyền S3 của
người dùng: chúng hỏi Lake Formation, Lake Formation kiểm tra quyền rồi phát
**credential tạm có phạm vi hẹp**. Chính vì tầng này chèn vào giữa mà Lake
Formation làm được thứ IAM không làm được: **lọc cột và lọc hàng** — nó gắn thêm
điều kiện vào kế hoạch truy vấn trước khi dữ liệu rời khỏi tầng lưu trữ.

Ba khả năng đáng nhớ:

- **Named resource** — cấp quyền trực tiếp lên database/bảng/cột cho một
  principal. Đơn giản, không scale.
- **LF-Tags (tag-based access control)** — gắn nhãn kiểu `sensitivity=PII` lên
  bảng/cột, rồi cấp quyền theo **biểu thức nhãn**. Bảng mới mang nhãn đúng là tự
  có quyền đúng; đây là cách duy nhất quản lý được hàng nghìn bảng.
- **Cross-account** — chia sẻ qua **AWS RAM**; account nhận tạo *resource link* và
  truy vấn như bảng của mình, dữ liệu không hề được copy.

**Bẫy số một, và nó làm hỏng cả cấu hình:** khi bạn bật Lake Formation trên một
catalog đã có, AWS giữ nguyên nhóm ảo **`IAMAllowedPrincipals`** với quyền `Super`
để không phá hệ thống đang chạy. Chừng nào nhóm đó còn quyền trên một bảng, **mọi
quyền Lake Formation bạn cấp cho bảng đó đều vô nghĩa** — ai có IAM/S3 policy phù
hợp vẫn đọc được. Người ta cấu hình xong, thử truy vấn, thấy chạy được, và kết
luận là đã bảo vệ dữ liệu. Cách chuyển dần đúng đắn là **hybrid access mode**
(2024): bật Lake Formation cho từng principal một, phần còn lại vẫn đi đường IAM.

**Khi nào không dùng Lake Formation:** một team, một account, phân quyền ở mức
bucket hoặc prefix là đủ. Lúc đó IAM policy cộng bucket policy đơn giản hơn, ít
tầng phải gỡ hơn khi có sự cố. Lake Formation trả công khi có **nhiều account
tiêu thụ** hoặc **yêu cầu quyền mức cột/hàng**.

---

## 6. Athena

### 6.1 Bên dưới là Trino

Athena là **Trino** (engine v3; v2 là Presto 0.217) chạy trên đội máy dùng chung
của AWS. Không có cụm nào của bạn: mỗi truy vấn được cấp một tập worker tạm thời,
đọc schema từ Glue Data Catalog và đọc dữ liệu **thẳng từ S3**. Hệ quả trực tiếp:
không có index, không có cache dữ liệu giữa các truy vấn, không có thống kê bảng
tốt như một kho dữ liệu — **mỗi lần chạy là đọc lại từ S3**.

### 6.2 Hoá đơn và bốn đòn bẩy

**$5 mỗi TB quét**, làm tròn lên MB, **tối thiểu 10 MB mỗi truy vấn**. Câu lệnh
DDL (`CREATE`/`ALTER`/`DROP`, quản lý partition) và truy vấn **thất bại** không
tính tiền; truy vấn bị **huỷ giữa chừng vẫn tính** phần đã quét.

Bốn đòn bẩy, theo thứ tự tác động: **phân vùng** (mục 3.3) → **định dạng cột**
(mục 3.1) → **nén** (3.2) → **chỉ chọn cột cần**. Một cái nữa hay bị hiểu sai:
`LIMIT 10` **không** giảm byte quét trong phần lớn trường hợp — Trino vẫn phải đọc
các split; chỉ điều kiện trên **cột phân vùng** mới cắt được file khỏi kế hoạch.

**Query result reuse** cho phép tái dùng kết quả cũ tới 7 ngày cho cùng một câu
lệnh — chạy lại một dashboard trong cửa sổ đó tốn **0 byte**. Bật ở mức workgroup
hoặc mức truy vấn; đây là cách rẻ nhất để chịu được nhiều người xem cùng báo cáo.

### 6.3 Workgroup và hàng rào chi phí

**Workgroup** tách người dùng thành nhóm có: vị trí lưu kết quả riêng, cấu hình mã
hoá riêng, tag chi phí riêng, và **data usage control** — hàng rào chi phí duy
nhất của Athena.

- **Per-query limit**: đặt ngưỡng byte quét từ **10 MB tới 7 EB**; truy vấn vượt bị
  **huỷ**, và hành vi này *không đổi được*. Đây là thứ chặn một câu `SELECT *` viết
  ẩu quét cả data lake.
- **Per-workgroup limit**: ngưỡng byte theo khoảng thời gian, hành động là **báo
  qua SNS**, không huỷ.

Đề Domain 4 hỏi thẳng cấu trúc này: *"ngăn một truy vấn lẻ tạo hoá đơn lớn"* →
per-query limit; *"cảnh báo khi một phòng ban vượt ngân sách tháng"* →
per-workgroup limit cộng SNS.

### 6.4 CTAS, UNLOAD, federated query, capacity reservation

**Athena ghi được, không chỉ đọc.** `CREATE TABLE AS SELECT` đọc dữ liệu CSV/JSON
và ghi ra Parquet có phân vùng ngay trong Athena — nghĩa là bạn chuyển đổi được
định dạng cả data lake mà **không cần Glue job nào**. `INSERT INTO` thêm dữ liệu
vào bảng có sẵn; `UNLOAD` xuất kết quả truy vấn ra định dạng bạn chọn.

**Federated query** dùng một Lambda connector để đọc nguồn ngoài S3 — RDS,
DynamoDB, Redshift, CloudWatch Logs, HBase. Bạn trả thêm tiền Lambda và độ trễ tăng
rõ; đây là công cụ cho truy vấn **thăm dò** xuyên nguồn, không phải cho pipeline
sản xuất.

**Capacity reservation** đổi mô hình tính tiền từ byte quét sang **DPU-giờ**: tối
thiểu **4 DPU**, đặt chỗ tối thiểu **1 phút** (từ 02/2026). Chọn nó khi tải đều,
truy vấn quét nhiều, và bạn cần **không bị xếp hàng** — nó cũng là cách duy nhất
kiểm soát concurrency trực tiếp trong Athena.

### 6.5 Cái gì làm Athena gãy

**Concurrency.** Quota *Active DML queries* (gồm cả đang chạy và đang xếp hàng)
tuỳ Region — docs lấy ví dụ **25**; vượt là `TooManyRequestsException`. Một
dashboard 300 người dùng bấm refresh sẽ đụng trần này, và câu trả lời không phải
"xin tăng quota" mà là **SPICE** hoặc **Redshift**.

**Thời gian.** DML query timeout mặc định **30 phút**, xin tăng tối đa **240
phút**. Job cần lâu hơn là job của EMR hoặc Glue, không phải của Athena.

**Chi phí tuyến tính theo lượt xem.** Athena không có cụm nên chi phí ở 0 truy vấn
là 0 — nhưng cũng vì thế chi phí ở 10.000 truy vấn là 10.000 lần chi phí một truy
vấn. Đây là ranh giới thật với Redshift, và là nội dung [mục 8](#8-data-lake-hay-data-warehouse).

---

## 7. Redshift

### 7.1 Kiến trúc MPP và RA3

Một **leader node** nhận SQL, lập kế hoạch, sinh code C++ rồi phân phát cho các
**compute node**; mỗi compute node chia thành **slice**, mỗi slice giữ một phần dữ
liệu và chạy song song. Leader gộp kết quả. Dữ liệu lưu **theo cột**, nén theo cột
(encoding tự chọn với `ANALYZE COMPRESSION` hoặc `AUTO`).

**RA3 tách compute khỏi storage**: dữ liệu nằm ở **Redshift Managed Storage (RMS)**
trên S3, node giữ cache SSD local cho phần nóng. Giá RMS **$0,024/GB-tháng**. Hệ
quả thiết kế: bạn đổi số node vì **CPU**, không vì dung lượng — điều ngược lại
hoàn toàn với DC2 đời cũ, nơi thêm dung lượng nghĩa là thêm node.

### 7.2 Distribution style và sort key

Hai quyết định vật lý quyết định hiệu năng, và ở mức SAA bạn cần hiểu **vì sao**
chứ không cần tinh chỉnh:

| Distribution style | Cơ chế | Chọn khi |
|---|---|---|
| `KEY` | băm một cột, dòng cùng giá trị vào **cùng slice** | bảng lớn hay join với nhau theo cột đó — join thành *local*, không phải chuyển dữ liệu qua mạng |
| `ALL` | **bản sao đầy đủ** trên mọi node | bảng dimension nhỏ, ít đổi |
| `EVEN` | round-robin | không có khoá join rõ ràng |
| `AUTO` | Redshift tự chọn và đổi theo kích thước bảng | mặc định, và là lựa chọn đúng cho hầu hết trường hợp |

**Sort key** sắp dữ liệu trên đĩa và Redshift giữ **zone map** (min/max cho mỗi
khối 1 MB). Predicate trên sort key cho phép bỏ qua nguyên khối mà không đọc — đây
là "index" của Redshift, và là lý do lọc theo cột thời gian nhanh hơn hẳn.

Chọn sai distribution key gây **data skew**: một slice giữ phần lớn dữ liệu, chạy
lâu hơn mọi slice khác, và cả truy vấn chờ nó. Triệu chứng giống hot partition của
DynamoDB và hot shard của Kinesis — cùng một bệnh, ba dịch vụ.

### 7.3 Concurrency scaling và WLM

Tải đỉnh thì Redshift bật **cụm tạm thời** phục vụ thêm truy vấn, tự tắt khi hết.
Con số ra thi: **1 giờ credit miễn phí mỗi 24 giờ** cho mỗi cluster đang chạy, tích
luỹ tối đa **30 giờ**; quá đó tính theo giây với **tối thiểu 1 phút** mỗi lần kích
hoạt. Bật theo từng **WLM queue**, nên bạn cho phép hàng đợi BI dùng và cấm hàng
đợi ETL dùng.

### 7.4 Redshift Serverless

| Thứ | Con số |
|---|---|
| Đơn vị | **RPU**, 1 RPU = **16 GB RAM** |
| Base capacity | **4 → 1.024 RPU**, mặc định **128** |
| Tính tiền | theo giây, **tối thiểu 60 giây**, từ **$1,50/giờ** (4 RPU) |
| Rảnh | **không tính tiền** |
| Bao gồm sẵn | **concurrency scaling và Spectrum**, không tính riêng |
| Recovery point | tự tạo mỗi **30 phút**, giữ **24 giờ** |

Đây là bước ngoặt trong cách ra đề: câu "Redshift đắt vì phải nuôi cụm 24/7" chỉ
còn đúng với provisioned. Serverless đưa Redshift vào cùng nhóm với Athena cho tải
gián đoạn — khác nhau còn lại là **Athena tính theo byte quét, Serverless tính theo
thời gian tính toán**.

### 7.5 Redshift Spectrum

Truy vấn thẳng dữ liệu S3 từ trong Redshift, qua một *external schema* trỏ tới
**Glue Data Catalog** — đúng cái catalog Athena đang dùng. Truy vấn chạy trên đội
máy Spectrum riêng của AWS, **không tiêu slot của cluster**, và tính **$5/TB quét**
— **cùng đơn giá Athena, cùng cách tính, cùng ba đòn bẩy ở mục 3**.

Hai điều quyết định:

- Với **Redshift Serverless**, truy vấn dữ liệu S3 **không tính riêng** — nó nằm
  trong RPU-giờ. Đây là lợi thế chi phí thật của Serverless khi bạn đọc data lake
  nhiều.
- Spectrum đúng cho **dữ liệu lịch sử join với dữ liệu nóng**. Nếu cùng một tập dữ
  liệu S3 bị join lại hàng chục lần mỗi ngày, `COPY` nó vào cluster rẻ hơn — bạn
  trả tiền quét mỗi lần thay vì trả tiền lưu một lần.

### 7.6 Data sharing và zero-ETL

**Data sharing** (RA3 trở lên) cho cluster/workgroup khác đọc dữ liệu **live**,
không copy, kể cả khác account và khác Region. Đây là đáp án cho *"tách tải BI khỏi
tải ETL mà không nhân đôi dữ liệu"* ở tầng kho dữ liệu, tương ứng với custom
endpoint của Aurora ở tầng OLTP.

**Zero-ETL integration** đẩy thay đổi từ Aurora, RDS MySQL/PostgreSQL và DynamoDB
sang Redshift gần thời gian thực mà không cần pipeline CDC. Đề nói *"phân tích trên
dữ liệu giao dịch mà không làm chậm database sản xuất và không tự viết pipeline"*
→ zero-ETL, không phải DMS, không phải Glue job đọc read replica.

---

## 8. Data lake hay data warehouse

Đây là câu hỏi thiết kế lớn nhất của cả chương, và bảng dưới là toàn bộ nội dung
của nó. Bên trái là **S3 + Glue Data Catalog + Athena**; bên phải là **Redshift**.

| Trục | Data lake (S3 + Glue + Athena) | Data warehouse (Redshift) |
|---|---|---|
| Đơn vị tính tiền | **byte quét** mỗi truy vấn | **node-giờ / RPU-giờ** |
| Chi phí khi không ai truy vấn | ~0 (chỉ tiền S3) | provisioned: đủ cả · Serverless: 0 |
| Chi phí khi 10.000 truy vấn/ngày | **tuyến tính**, không có trần | **phẳng** |
| Độ trễ | giây tới phút, biến động theo tải chung của AWS | dưới giây tới giây, **ổn định**, có result cache |
| Concurrency | quota Active DML (docs ví dụ **25**) | WLM + concurrency scaling → hàng nghìn |
| Schema | **schema-on-read** — sửa catalog là xong | **schema-on-write** — phải `COPY` |
| Join nhiều bảng lớn | được, chậm, không có statistics tốt | **mạnh nhất**: distribution key, sort key, zone map |
| Sửa/xoá từng dòng | phải ghi lại file | `UPDATE` / `DELETE` bình thường |
| Dữ liệu bán cấu trúc, schema hay đổi | **tự nhiên** | phải mô hình hoá trước |

**Điểm hoà vốn tính được.** Redshift Serverless 8 RPU chạy liên tục ≈ $3/giờ ≈
**$2.190/tháng**. Cùng ngân sách đó mua **438 TB quét** trên Athena. Nhưng
Serverless chỉ tính lúc chạy, nên con số thực dụng là: dưới khoảng **10 truy vấn
nặng mỗi ngày** thì Athena; dashboard nhiều người và truy vấn lặp lại thì Redshift.
Trục quyết định không phải dung lượng dữ liệu — nó là **số truy vấn và mức độ lặp**.

**Vì sao "cả hai" thường là đáp án thật.** Redshift Spectrum đọc thẳng S3 qua đúng
Glue Data Catalog mà Athena dùng, và với Serverless thì không tính tiền riêng. Nên
kiến trúc chuẩn là: **mọi dữ liệu đổ vào S3** ở dạng Parquet có phân vùng; **phần
nóng** (vài tháng gần nhất, thứ bị join liên tục) được `COPY` vào Redshift; **phần
lạnh** ở nguyên S3 và được join qua Spectrum khi cần; **một** catalog và **một** bộ
quyền Lake Formation cho cả hai. Đề mô tả đúng kiến trúc này bằng câu *"keep
historical data in S3 and join it with recent data in the warehouse"* — và đáp án
là Spectrum, không phải "nạp toàn bộ lịch sử vào Redshift".

---

## 9. Amazon EMR

Cụm Hadoop/Spark/Hive/HBase/Presto/Flink do AWS dựng. Ba vai trò node, và phân
biệt được ba vai này là giải được mọi câu EMR trong đề:

| Vai trò | Chạy gì | Spot được không | Mất node thì sao |
|---|---|---|---|
| **Primary** | YARN ResourceManager, HDFS NameNode | **không** | cụm chết (trừ khi bật multi-master 3 node) |
| **Core** | DataNode — **giữ HDFS** | **không nên** | mất dữ liệu HDFS, cụm phải nhân bản lại |
| **Task** | chỉ tính toán, không giữ dữ liệu | **có, lý tưởng** | job chậm lại, không mất gì |

**Instance fleet** khai báo nhiều loại instance cho mỗi vai trò để tăng khả năng
lấy được Spot: tối đa **5 loại** khi không bật allocation strategy, **30 loại** khi
bật. EMR chọn một AZ trong số subnet bạn đưa.

**EMRFS** cho Spark/Hive đọc ghi thẳng S3 thay HDFS. Đây là điều làm EMR hiện đại
khác EMR sách vở: **storage tách khỏi compute**, nên cụm *transient* (chạy xong tự
tắt) là mẫu chuẩn và cũng là đáp án chi phí. "EMRFS consistent view" không còn cần
từ khi S3 có strong read-after-write consistency — xem
[`02-storage.md`](02-storage.md#1-consistency-model--mục-bị-dạy-sai-nhiều-nhất).

**EMR Serverless** bỏ hẳn cụm: khai báo một application, gửi job, trả theo vCPU-giờ
và GB-giờ lúc job chạy. **EMR on EKS** chạy Spark trên cụm EKS sẵn có, dùng chung
node với workload khác.

Chọn EMR khi đề nói tên framework (**Spark, Hive, HBase, Presto, Flink**), khi cần
kiểm soát phiên bản engine, khi lift-and-shift Hadoop on-premises, hoặc khi cần
**Spot** để cắt chi phí xử lý. Đề chỉ nói "ETL serverless" thì đó là Glue; đề chỉ
nói "SQL ad-hoc trên S3" thì đó là Athena.

---

## 10. OpenSearch Service

**Cơ chế.** Dữ liệu vào **index**, index chia thành **shard**, mỗi shard là một
Lucene index độc lập có inverted index cho tìm kiếm toàn văn. Truy vấn được phát
tán tới mọi shard rồi gộp kết quả. **Số shard cố định lúc tạo index** — đổi phải
reindex; đây là quyết định khó sửa nhất của OpenSearch. Hướng dẫn thực dụng: shard
**10–50 GB**, và không quá **25 shard cho mỗi GB heap JVM** của node.

**Ba tầng lưu trữ** là phần quyết định chi phí:

| Tầng | Dữ liệu nằm ở đâu | Ghi được | Con số |
|---|---|---|---|
| **Hot** | EBS/NVMe của data node | có | nhanh nhất, đắt nhất; replica **tính tiền** |
| **UltraWarm** | **S3**, node chỉ là cache | **không** (read-only) | chỉ tính **primary shard** — index 20 GB tốn 40 GB ở hot chỉ tốn **20 GB** ở warm; `ultrawarm1.large` tới **20 TiB** mỗi node |
| **Cold** | S3, **không gắn vào cụm** | không | rẻ nhất; phải *attach* trước khi truy vấn |

**ISM policy** (Index State Management) tự chuyển index theo tuổi: hot 7 ngày →
UltraWarm 30 ngày → cold 90 ngày → xoá. Đây là cách duy nhất giữ OpenSearch trong
ngân sách cho log analytics dài hạn.

**Dedicated master node**: 3 node (số lẻ để tránh split-brain), không giữ dữ liệu,
chỉ giữ trạng thái cụm. Bắt buộc cho cụm sản xuất.

**OpenSearch Serverless** tính theo **OCU** (1 OCU = 6 GB RAM), tách OCU indexing
và OCU search. Bẫy chi phí: collection kiểu *Classic* có **sàn 2 OCU** cho
collection đầu tiên trong account — nghĩa là "serverless" ở đây **không** về 0 và
có chi phí sàn hàng trăm đô mỗi tháng. Collection *NextGen* mới scale về 0 sau 10
phút rảnh.

**Chọn giữa ba công cụ truy vấn log** — câu đề hỏi nhiều nhất: full-text, tổng hợp
gần thời gian thực, dashboard nhiều người xem → **OpenSearch**; log lịch sử ở S3,
truy vấn thưa thớt, ngân sách thấp → **Athena**; log đã ở CloudWatch, cần truy vấn
nhanh trong một account → **CloudWatch Logs Insights** (xem
[`07-quan-tri-giam-sat.md`](07-quan-tri-giam-sat.md)).

---

## 11. QuickSight

**SPICE** (Super-fast, Parallel, In-memory Calculation Engine) là kho cột trong bộ
nhớ của QuickSight. Bạn **nạp dữ liệu vào SPICE** theo lịch (Enterprise: tới mỗi 15
phút, hỗ trợ incremental refresh); dashboard đọc từ SPICE. Chế độ còn lại là
**direct query**: mỗi lần mở dashboard là một truy vấn xuống nguồn.

Đây là quyết định thiết kế quan trọng nhất của chặng cuối, và nó là quyết định về
**tiền**: dashboard chạy direct query trên Athena nghĩa là **trả $5/TB cho mỗi lượt
người dùng bấm mở hoặc lọc**. Cùng dashboard đó trên SPICE trả tiền quét **một
lần mỗi lần refresh**, bất kể bao nhiêu người xem.

Mô hình giá là câu hỏi thi thật, không phải chi tiết vụn:

| Thành phần | Giá (Enterprise, us-east-1, 2026-08) |
|---|---|
| Author | **$24/tháng** trả theo tháng, **$18/tháng** trả năm |
| Reader | **$3/tháng**, tính theo phiên **$0,30** mỗi phiên 30 phút, trần $3 |
| SPICE | **10 GB kèm mỗi author**, thêm **$0,38/GB-tháng** |
| Session capacity | từ **$250/tháng** cho 500 phiên — dùng khi nhúng dashboard công khai |

Đề nói *"hàng nghìn người xem báo cáo, không muốn mua license cho từng người"* →
**reader tính theo phiên** hoặc session capacity, không phải author. Enterprise
edition thêm **row-level và column-level security** (ánh xạ người dùng tới tập dòng
được thấy) và **VPC connection** để đọc nguồn trong subnet private.

---

## 12. Hai bài toán thiết kế

### 12.1 Log 2 TB mỗi ngày, ad-hoc dưới 30 giây, ngân sách thấp

> 400 dịch vụ trên ECS sinh **2 TB log JSON mỗi ngày**. Phải giữ **1 năm**. Kỹ sư
> truy vấn ad-hoc vài lần mỗi ngày, luôn lọc theo **một ngày và một service**, cần
> kết quả trong khoảng **30 giây**. Ngân sách là ràng buộc cứng.

**Thiết kế.** Fluent Bit trong task ECS ghi thẳng vào **Amazon Data Firehose** →
S3. Firehose bật **chuyển đổi Parquet** (buffer **128 MB**) và **dynamic
partitioning** theo `service` và `dt`. Bảng khai bằng `CREATE EXTERNAL TABLE` với
**partition projection** (`dt.type=date`, `dt.range=2025-01-01,NOW`) — không
crawler, không `MSCK REPAIR`. **Athena** truy vấn, dùng workgroup có **per-query
limit 100 GB** làm hàng rào. Lifecycle S3 chuyển sang Glacier Instant Retrieval sau
90 ngày.

**Con số chứng minh.** 2 TB JSON thô co còn khoảng **250 GB/ngày** ở Parquet+ZSTD →
~90 TB cho cả năm, tiền S3 Standard ≈ **$2.100/năm** (rẻ hơn nữa sau lifecycle).
Một truy vấn lọc 1 ngày, 1 service, 10 cột trong 60 quét vài GB → **vài cent**,
chạy trong 10–30 giây vì partition pruning cắt xuống một prefix. Không phân vùng
thì đúng truy vấn đó quét ~90 TB = **$450 một lần bấm**.

**Vì sao loại các phương án khác:**

- **CloudWatch Logs giữ 1 năm.** Riêng phí nạp ~$0,50/GB × 2 TB/ngày ≈
  **$1.000 mỗi ngày**. Loại ngay ở dòng đầu tiên của phép tính, trước cả khi bàn
  tới truy vấn.
- **OpenSearch.** Đúng về trải nghiệm truy vấn nhưng sai về hình dạng tải: 2 TB/ngày
  vào hot tier cần hàng chục data node chạy 24/7 để phục vụ **vài truy vấn mỗi
  ngày**. Kể cả ISM đẩy sang UltraWarm, bạn vẫn nuôi cụm liên tục. OpenSearch thắng
  khi truy vấn **liên tục** và cần full-text, không phải khi thưa.
- **Redshift.** Phải `COPY` toàn bộ vào kho và nuôi compute cho vài truy vấn ngày.
  Serverless đỡ hơn nhưng vẫn tính theo thời gian tính toán, trong khi Athena tính
  theo byte và ở đây byte đã bị cắt xuống mức vài GB.
- **EMR.** Một cụm và một người vận hành cho việc mà một câu SQL làm được.

### 12.2 Clickstream 50.000 sự kiện/giây, ba nhóm tiêu thụ

> Ba nhóm cần cùng luồng: (a) dashboard vận hành trễ **dưới 5 giây**; (b) data lake
> cho phân tích lịch sử; (c) mô hình chống gian lận cần **đọc lại 7 ngày** mỗi lần
> huấn luyện lại.

**Thiết kế.** **Kinesis Data Streams**, retention đặt **7 ngày**. Provisioned:
50.000 ÷ 1.000 = 50 shard theo trục record, 50 MB/s theo trục dung lượng → cộng
biên độ thành **64 shard**; hoặc on-demand nếu tải dao động mạnh. `PartitionKey` là
**`user_id`** (cardinality cao), không phải `event_type`. Ba consumer trên cùng
stream đó:

- (a) **Managed Service for Apache Flink** tổng hợp theo cửa sổ → OpenSearch
- (b) **Firehose** đọc chính stream đó → S3 Parquet, dynamic partitioning
- (c) ứng dụng KCL với **enhanced fan-out**, khi retrain thì đọc lại từ sequence
  number cũ trong 7 ngày

**Vì sao loại các phương án khác:**

- **SQS.** Message biến mất khi một consumer xử lý xong; không có ba consumer độc
  lập, và **không tua lại được**. SNS fanout ra ba queue giải quyết được vế "ba
  nhóm" nhưng vẫn không giải được vế "đọc lại 7 ngày".
- **Chỉ Firehose.** Không có consumer nào khác đọc được luồng, không replay, và độ
  trễ buffer không phục vụ được yêu cầu dưới 5 giây khi đã bật Parquet (sàn 64 MB).
- **MSK.** Chạy được về kỹ thuật, nhưng bạn nhận thêm trách nhiệm chọn số partition,
  vá phiên bản Kafka và xử lý rebalance mà bài toán không đòi. Chọn MSK khi đã có
  code Kafka hoặc cần Kafka Connect/Streams.
- **Kinesis với `PartitionKey = event_type`.** Vài chục giá trị nghĩa là vài chục
  shard nhận toàn bộ tải trong khi 64 shard đã cấp — hot shard, throttle dù tổng
  throughput còn thừa. Cùng bệnh với hot partition của DynamoDB.

---

## Bảng số phải nhớ

Mốc kiểm chứng **2026-08**. Giá theo `us-east-1`.

### Ingest

| Con số | Giá trị |
|---|---|
| Kinesis shard — ghi | **1 MB/giây** hoặc **1.000 record/giây** |
| Kinesis shard — đọc | **2 MB/giây** chia cho mọi consumer · **2 MB/giây riêng** mỗi consumer với enhanced fan-out |
| Kinesis — độ trễ | ~200 ms poll · **~70 ms** enhanced fan-out |
| Kinesis — giữ dữ liệu | mặc định **24 giờ**, tối đa **365 ngày** |
| Kinesis — bản ghi tối đa | **1 MiB** · PUT payload unit tính theo **25 KB** |
| Firehose — buffer S3 | **1–128 MB** · **0–900 giây**, cái nào tới trước |
| Firehose — buffer khi bật Parquet/ORC | sàn **64 MB**, mặc định **128 MB** |
| Firehose — throughput direct PUT | **5.000 record/giây · 2.000 giao dịch/giây · 5 MB/giây** |
| SQS — giữ tin | tối đa **14 ngày** · bản tin **256 KB** · tính theo **64 KB** = 1 request |

### Lưu và truy vấn

| Con số | Giá trị |
|---|---|
| Athena | **$5 / TB quét**, làm tròn lên MB, **tối thiểu 10 MB** mỗi truy vấn |
| Athena — DDL và truy vấn lỗi | **$0** · truy vấn **huỷ giữa chừng vẫn tính** phần đã quét |
| Athena — query result reuse | tái dùng tới **7 ngày**, tốn **0 byte** |
| Parquet nén vs text thuần | **4 TB → 10 GB** = **400 lần**, $20,00 → **$0,05** |
| Nén riêng lẻ | **~4 lần**, áp cho **mọi** truy vấn |
| Định dạng cột riêng lẻ | tới **100 lần**, **chỉ khi** không lấy hết cột |
| Kích thước file mục tiêu trong data lake | **128 MB – 1 GB** |
| Redshift Spectrum | dùng **chung đơn giá $5/TB** và chung cách đếm byte với Athena |

### Quy đổi nhanh khi thiết kế

| Từ | Sang |
|---|---|
| 2 TB/ngày, buffer **128 MB** | ~**16.000** file/ngày |
| 2 TB/ngày, buffer **5 MB** | ~**400.000** file/ngày |
| 50.000 sự kiện/giây, mỗi sự kiện 1 KB | **50 MB/giây** → tối thiểu **50 shard** |

---

## Bẫy đề thi

1. **"Firehose tối thiểu 60 giây".** Sai từ khi có zero buffering. Nhưng bẫy thật
   nằm chỗ khác: **không dùng được zero buffering cùng dynamic partitioning**, và
   bật Parquet thì buffer bị ép sàn **64 MB**. Đề hỏi "vừa Parquet vừa dưới một
   phút" → không tồn tại.
2. **Chọn Kinesis cho task queue.** Đề mô tả "mỗi việc một worker, xong thì bỏ" là
   SQS. Kinesis không có visibility timeout, không có DLQ — bạn phải tự viết.
3. **Chọn MSK vì thấy chữ "streaming".** Chỉ chọn MSK khi đề **nhắc Kafka** hoặc
   nhắc Kafka Connect / Kafka Streams / log compaction. Mô tả *hành vi* thuần
   (thứ tự, replay, nhiều consumer) → Kinesis rẻ hơn và ít việc hơn.
4. **`LIMIT 10` để giảm tiền Athena.** Không giảm. Chỉ điều kiện trên **cột phân
   vùng** mới loại được file khỏi kế hoạch đọc.
5. **Nghĩ Parquet luôn cứu hoá đơn.** `SELECT *` trên Parquet mất toàn bộ lợi ích
   cột, chỉ còn phần nén.
6. **Redshift cho truy vấn ad-hoc thi thoảng.** Cluster tính tiền theo giờ kể cả
   khi không ai truy vấn. Ad-hoc thưa → **Athena**. Xem [mục 8](#8-data-lake-hay-data-warehouse).
7. **Crawler cho mọi thứ.** Schema ổn định thì `CREATE EXTERNAL TABLE` một lần rồi
   dùng **partition projection** — không crawler, không `MSCK REPAIR`, không tiền.
8. **Nhầm Lake Formation với IAM.** Lake Formation cấp quyền mức **cột, hàng, ô**;
   IAM chỉ cấp tới mức **object S3**. Đề nói "che cột PII cho một nhóm" → Lake Formation.
9. **OpenSearch cho phân tích lịch sử dài hạn.** Nó là công cụ **tìm kiếm và quan
   sát gần thời gian thực**. Lưu lâu, quét sâu, chi phí thấp → S3 + Athena.
10. **Quên rằng Athena và Spectrum tính tiền giống hệt nhau.** Chuyển truy vấn từ
    Athena sang Spectrum **không** làm rẻ đi.

---

## Cây quyết định

```
Dữ liệu vào bằng đường nào?
│
├─ Cần replay, hoặc nhiều nhóm đọc CÙNG dữ liệu ở vị trí riêng?
│   ├─ Đề nhắc "Kafka" / Kafka Connect / Streams  -> MSK
│   └─ Không nhắc Kafka                            -> Kinesis Data Streams
│
├─ Chỉ cần đổ vào S3 / OpenSearch / Redshift, không viết consumer?
│                                                   -> Data Firehose
│
└─ Mỗi bản tin một worker, xử lý xong thì bỏ?       -> SQS  (06-tich-hop.md)


Truy vấn thế nào?
│
├─ Ad-hoc, thưa, dữ liệu đã ở S3?                   -> Athena
│     └─ Hoá đơn cao?  phân vùng -> cột -> nén -> chọn đúng cột
│
├─ BI nhiều người, truy vấn phức tạp, chạy liên tục? -> Redshift
│     ├─ Tải lên xuống thất thường / không muốn quản -> Redshift Serverless
│     └─ Có bảng nóng ở Redshift, bảng nguội ở S3    -> Redshift Spectrum
│
├─ Tìm toàn văn, log gần thời gian thực, dashboard?  -> OpenSearch Service
│
└─ Spark / Hive / Presto tự quản, job lớn, dùng Spot? -> EMR


Lưu thế nào?  (quyết định này đổi hoá đơn nhiều nhất)
│
├─ Định dạng   -> Parquet, trừ khi có ràng buộc khác
├─ Nén         -> Snappy nếu cần chia tách; GZIP nếu ưu tiên dung lượng
├─ Phân vùng   -> theo cột LUÔN có trong WHERE; đừng chia quá nhỏ
└─ Kích thước  -> gộp về 128 MB – 1 GB mỗi file
```

---

## Nối với thực hành

| Bạn vừa đọc | Làm ở |
|---|---|
| Kinesis vs SQS vs SNS, chọn đúng cơ chế ghép lỏng | [`labs-self/w07-decoupling`](../../learn-aws/labs-self/w07-decoupling/README.md) |
| Query vs Scan, `ScannedCount` — cùng bài học "đọc ít đi" ở tầng DynamoDB | [`labs-self/w05-databases`](../../learn-aws/labs-self/w05-databases/README.md) |
| Lớp lưu trữ S3, lifecycle, versioning — nền của data lake | [`labs-self/w04-s3-cloudfront`](../../learn-aws/labs-self/w04-s3-cloudfront/README.md) |
| Số đo sinh từ log, Logs Insights — họ hàng gần của Athena | [`labs-self/w10-observability-iac`](../../learn-aws/labs-self/w10-observability-iac/README.md) |
| Tự viết bảng so sánh bốn đường ingest thành dữ liệu máy chấm được | [`labs-self/w12-exam-review`](../../learn-aws/labs-self/w12-exam-review/README.md) |

**Bài tự luyện không cần AWS, làm trên giấy 20 phút:** lấy bảng ở
[2.1](#21-bốn-cách-đưa-dữ-liệu-vào), che cột tên dịch vụ đi, chỉ đọc dòng "ai giữ
vị trí đọc" và "replay" — rồi tự khôi phục lại tên. Nếu làm được, bạn đã nắm phần
lõi của cả mảng này; ba mục còn lại chỉ là chi tiết.

Chưa có lab riêng cho phân tích dữ liệu — đó là chỗ hổng đã biết của bộ lab. Lý do
thực tế: Kinesis, Glue và Redshift đều tính tiền theo giờ hoặc theo shard, không
nằm trong hàng rào **$0/giờ** của [`labs-self/`](../../learn-aws/labs-self/README.md).
Athena là ngoại lệ rẻ nhất nếu bạn muốn tự thử: vài MB dữ liệu, mỗi truy vấn tính
tối thiểu 10 MB, tức khoảng **$0,00005**.

---

## Nguồn nói khác

**"Firehose tối thiểu 60 giây".** Đúng cho tới khi có **zero buffering**: đặt
buffer interval = 0 thì dữ liệu tới đích trong **vài giây**, và Firehose chuyển
sang multipart upload khi interval dưới 60 giây. Ràng buộc thật không phải thời
gian mà là **tính năng loại trừ nhau**: zero buffering không đi cùng dynamic
partitioning. [Firehose buffering](https://docs.aws.amazon.com/firehose/latest/dev/buffering-hints.html)

**"Kinesis giữ tối đa 7 ngày".** Cũ. **365 ngày** từ 2020 (long-term retention).
Nhiều tài liệu luyện thi vẫn ghi 7 ngày.

**"Glue crawler là bước bắt buộc".** Không. Crawler tiện khi schema thay đổi hoặc
chưa biết; với schema ổn định thì `CREATE EXTERNAL TABLE` cộng **partition
projection** rẻ hơn, nhanh hơn, và không có rủi ro crawler tự đoán sai kiểu dữ liệu.

**"Athena không cấp phát hạ tầng nên không có giới hạn".** Có. Quota số truy vấn
DDL và DML chạy đồng thời là quota mềm nhưng có thật, và một truy vấn đơn lẻ có
giới hạn thời gian chạy. Xem [mục 6.5](#65-cái-gì-làm-athena-gãy).

**"Amazon Quick" trong đề cương chính thức.** Đó là lỗi trích xuất tên
**Amazon QuickSight** trong bản PDF — xem
[`23-de-cuong-chinh-thuc.md`](23-de-cuong-chinh-thuc.md).

---

## Ngoài phạm vi

- **Tinh chỉnh Spark** (partition, shuffle, broadcast join, quản lý bộ nhớ
  executor) — là kỹ năng kỹ sư dữ liệu, không phải kiến trúc sư.
  [EMR best practices](https://aws.amazon.com/emr/)
- **Cú pháp SQL của Athena, Redshift, OpenSearch DSL** — đề không hỏi bạn viết
  truy vấn, chỉ hỏi bạn **chọn công cụ nào**.
- **Kafka mức vận hành** (chọn số partition, chiến lược rebalance, ISR, tuning
  producer) — thuộc kỳ thi Data Engineer.
- **MWAA (Managed Airflow)** — điều phối luồng công việc nhưng **ngoài phạm vi**
  SAA. Điều phối trong đề SAA là **Step Functions**, xem
  [`06-tich-hop.md` §7](06-tich-hop.md#7-step-functions--khi-nào-chuỗi-lambda-là-sai).
- **Mô hình hoá kho dữ liệu** (star schema, slowly changing dimension) — kiến thức
  nền tảng tốt nhưng không ra trong SAA.

---

## Tự kiểm tra

**1.** Đề: 50.000 sự kiện/giây, mỗi sự kiện ~1 KB. Ba nhóm cần dữ liệu: một nhóm
tính số liệu thời gian thực, một nhóm đổ vào data lake, một nhóm phải **chạy lại
7 ngày gần nhất** sau khi sửa lỗi. Thiết kế đường ingest, và tính số shard.

<details><summary>Đáp án</summary>

**Kinesis Data Streams**, retention đặt **≥ 7 ngày**.

Tính shard: 50.000 × 1 KB = **50 MB/giây**. Mỗi shard ghi 1 MB/giây → **50 shard**
là sàn lý thuyết. Thực tế cấp **60–64 shard** để có biên cho lệch phân bố và đỉnh tải.
Kiểm chéo bằng trục thứ hai: 50.000 record/giây ÷ 1.000 record/giây mỗi shard = 50 —
hai trục cho cùng con số, nên đây là bài toán cân bằng; nhiều bài toán thật lệch một
trục và bạn phải lấy **max** của hai.

Ba nhóm tiêu thụ:
- Nhóm thời gian thực → **enhanced fan-out** (2 MB/giây riêng, ~70 ms), vì nó nhạy độ trễ.
- Nhóm data lake → **Firehose** đọc từ stream, buffer **128 MB**, ghi **Parquet**.
- Nhóm chạy lại → consumer riêng, tua sequence number về mốc cần.

Điều cần nói được: **chỉ có Data Streams và MSK** cho phép ba nhóm đọc cùng dữ liệu
ở ba vị trí độc lập. Firehose không có consumer để tua; SQS xoá bản tin sau khi xử lý.
Yêu cầu "chạy lại 7 ngày" một mình đã loại hai phương án.
</details>

**2.** Hoá đơn Athena tháng này $2.000. Dữ liệu là JSON nén GZIP trong S3, phân vùng
theo `region`. Truy vấn hay dùng lọc theo `ngày` và lấy 3 trong 80 cột. Bạn sửa gì
trước, và ước tính cắt được bao nhiêu?

<details><summary>Đáp án</summary>

Theo thứ tự tác động:

1. **Đổi phân vùng sang `ngày`** (hoặc thêm `ngày` làm cấp phân vùng). Phân vùng
   theo `region` vô dụng khi mọi truy vấn lọc theo ngày — engine vẫn đọc mọi ngày.
   Đây là đòn bẩy mạnh nhất và cũng là đòn bẩy duy nhất **loại được file khỏi kế
   hoạch đọc**.
2. **Chuyển JSON → Parquet.** Lấy 3/80 cột nghĩa là bỏ được ~96% dữ liệu. Riêng
   bước này theo bảng ở [3.1](#31-cột-hay-hàng-và-con-số-thật) là cỡ **hàng chục
   tới 100 lần**.
3. **Gộp file nhỏ** về 128 MB – 1 GB nếu đang phân mảnh.
4. **Bật query result reuse** nếu là dashboard chạy lặp.

Ước tính: phân vùng đúng cắt theo tỉ lệ số ngày bị loại; Parquet cắt thêm ~25 lần
cho tỉ lệ cột 3/80. Hai thứ nhân nhau, nên **$2.000 xuống vài chục đô** là kết quả
bình thường, không phải lạc quan.

Bẫy cần tránh: **đừng bắt đầu bằng `LIMIT`** hay bằng việc đổi engine sang Spectrum
— Spectrum tính tiền y hệt. Và nhớ **GZIP không chia tách được**: file GZIP lớn
buộc một worker đọc tuần tự, nên nếu giữ định dạng hàng thì nên đổi sang định dạng
chia tách được.
</details>

**3.** Vì sao "vừa xuất Parquet vừa độ trễ dưới một phút" là bất khả thi với Firehose,
và bạn làm gì khi đề đòi cả hai?

<details><summary>Đáp án</summary>

Vì bật chuyển đổi Parquet/ORC làm Firehose **ép buffer size sàn 64 MB** (mặc định
128 MB). Chuyển sang định dạng cột cần gom đủ một khối dữ liệu mới xây được row
group và thống kê min/max — bản chất của định dạng cột là **theo lô**.

Khi đề đòi cả hai, đó là dấu hiệu bạn cần **hai đường**, không phải một:
- **Đường nóng**: Data Streams → consumer thời gian thực (hoặc Firehose zero
  buffering ghi JSON) cho phần cần dưới một phút.
- **Đường nguội**: cùng stream → Firehose buffer 128 MB → Parquet cho phần phân tích.

Đây chính là **kiến trúc lambda**, và lý do nó tồn tại là ràng buộc vật lý vừa nêu,
không phải sở thích kiến trúc.
</details>

**4.** Đội bạn có sẵn ứng dụng dùng Kafka producer/consumer, muốn lên AWS mà không
sửa code. Chọn MSK hay Kinesis? Nếu chọn MSK, bạn nhận thêm trách nhiệm gì?

<details><summary>Đáp án</summary>

**MSK** — vì tương thích giao thức Kafka, code không phải sửa. Kinesis dùng API
khác hẳn, chọn nó nghĩa là viết lại tầng ingest.

Trách nhiệm nhận thêm, và đây mới là phần đề hay hỏi:
- **Chọn số partition** và sống với nó (tăng được, giảm thì không).
- **Vá và nâng phiên bản Kafka.**
- **Xử lý rebalance** của consumer group.
- **Quản dung lượng broker** — hết đĩa là cluster dừng nhận.
- **Trả tiền broker theo giờ** kể cả lúc không có dữ liệu.

MSK Serverless bỏ được phần chọn broker và dung lượng, nhưng tính tiền theo
cluster-giờ + partition-giờ + GB, và vẫn không bỏ được phần Kafka-hoá tư duy.

Điều cần nói được: chọn MSK là quyết định **tương thích**, không phải quyết định
kỹ thuật thuần. Nếu không có ràng buộc code sẵn thì Kinesis luôn ít việc hơn.
</details>

**5.** Một người nói: "Đổi hết sang Redshift đi, Athena chậm quá." Bạn hỏi lại ba
câu gì trước khi đồng ý?

<details><summary>Đáp án</summary>

1. **Truy vấn chạy bao nhiêu lần một ngày, và có bao nhiêu người dùng đồng thời?**
   Redshift tính tiền theo **giờ cluster**, không theo truy vấn. Ad-hoc vài lần
   mỗi ngày thì cluster ngồi không 23 giờ — đắt hơn Athena rất nhiều dù mỗi truy
   vấn nhanh hơn.
2. **Athena chậm vì cái gì?** Nếu chậm vì dữ liệu là JSON không phân vùng thì
   Redshift đọc cùng đống dữ liệu đó cũng chậm. Sửa cách **lưu** (phân vùng +
   Parquet) thường giải quyết xong vấn đề mà không đổi engine.
3. **Có cần join phức tạp nhiều bảng lớn lặp đi lặp lại không?** Đây là chỗ
   Redshift thật sự thắng: MPP với distribution key và sort key đã sắp xếp sẵn.
   Nếu chỉ quét-lọc-tổng hợp một bảng thì lợi thế đó không dùng tới.

Đáp án thật thường là **cả hai**: bảng nóng trong Redshift, dữ liệu lịch sử ở S3,
nối bằng **Spectrum**. Xem [mục 8](#8-data-lake-hay-data-warehouse).
</details>

**6.** Đề nói: "che cột số căn cước với nhóm phân tích, nhưng họ vẫn phải truy vấn
được các cột còn lại trong cùng bảng." IAM giải được không?

<details><summary>Đáp án</summary>

**Không.** IAM cấp quyền tới mức **object S3** — cho đọc file hay không cho đọc
file. Một cột nằm bên trong file, nên IAM không có chỗ để diễn đạt điều kiện này.

**Lake Formation** giải được: nó cấp quyền mức **cột, hàng, và ô**, và cơ chế là
Lake Formation cấp **credential tạm** cho engine (Athena, Redshift Spectrum, EMR,
Glue) sau khi đã lọc theo chính sách — engine không bao giờ thấy dữ liệu bị che.

Bẫy đi kèm: khi đã dùng Lake Formation, quyền S3 thô cấp trực tiếp bằng IAM có thể
**đi vòng qua** nó. Đó là lý do triển khai đúng phải thu hồi quyền S3 trực tiếp và
để Lake Formation làm cửa duy nhất. Đây cũng là câu trả lời cho "cấp quyền rồi mà
ai cũng đọc được".
</details>

**7.** Bạn thấy S3 có 400.000 file mỗi ngày, mỗi file ~5 MB, và Athena chậm bất
thường dù đã dùng Parquet và phân vùng đúng. Chuyện gì đang xảy ra?

<details><summary>Đáp án</summary>

**Bài toán file nhỏ.** Mỗi file là một đơn vị công việc: Trino phải liệt kê object,
mở file, đọc footer Parquet, lập kế hoạch. Với hàng trăm nghìn file, thời gian
**lập kế hoạch và mở file** vượt xa thời gian đọc dữ liệu thật. Phân vùng và định
dạng cột không cứu được, vì chúng giảm **byte quét** chứ không giảm **số lần mở file**.

Nguyên nhân gốc gần như luôn nằm ở chặng trước: **buffer Firehose quá nhỏ**. 5 MB
là mặc định, và mặc định đó sai cho data lake. Sửa: nâng buffer lên **128 MB**
(2 TB/ngày → ~16.000 file thay vì 400.000), và chạy một job gộp file cho dữ liệu cũ.

Đây là ví dụ rõ nhất của nguyên tắc xuyên suốt chương này: **chỗ gãy nằm ở ranh
giới giữa hai chặng**, không nằm trong chặng nào. Người vận hành Athena đi tối ưu
Athena sẽ không bao giờ tìm ra, vì lỗi nằm ở cấu hình Firehose.
</details>
