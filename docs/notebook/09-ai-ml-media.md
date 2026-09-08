# AI/ML dựng sẵn, SageMaker và media

> **Tra nhanh:** dịch vụ AI nào nhận đầu vào gì, trả ra gì, đồng bộ hay bất đồng bộ —
> và bạn ráp nó vào luồng thật ở chỗ nào.

`Domain 3 · Design High-Performing Architectures (24% đề)` · chạm `Domain 2 · Resilient (26%)` và `Domain 4 · Cost-Optimized (20%)`

Đề SAA hỏi nhóm này rất nông: nhìn thấy "trích chữ từ ảnh chụp" thì chọn Textract, hết
câu. Chương này **không dừng ở đó**. Biết tên dịch vụ không giúp bạn dựng được hệ thống;
thứ giúp bạn dựng được là biết **Textract có hai API khác nhau hoàn toàn về hình dạng
kiến trúc** — một cái trả lời ngay, một cái trả `JobId` rồi bắn SNS sau vài phút. Chọn
nhầm cái nào là sai từ dòng đầu tiên của thiết kế, và không có cách sửa nào rẻ.

Với mỗi dịch vụ trong chương, bạn phải trả lời được ba câu: **nhận gì, trả gì, đồng bộ
hay không**. Ba câu đó quyết định kiến trúc. Tên dịch vụ chỉ quyết định một đáp án trắc
nghiệm.

## Bản đồ

| Mục | Đọc khi bạn cần |
|---|---|
| [1. Ba câu hỏi trước khi vẽ](#1-ba-câu-hỏi-phải-trả-lời-trước-khi-vẽ-bất-cứ-thứ-gì) | Cái khung chung cho cả chương — đọc trước tiên |
| [2. Bài toán tới dịch vụ](#2-bài-toán--dịch-vụ-và-chỗ-đề-gài-nhầm) | Bảng tra một dòng, kèm cột "đề hay gài nhầm sang cái nào" |
| [3. Textract](#3-textract--lấy-chữ-ra-khỏi-tài-liệu) | Hoá đơn, form, PDF nhiều trang, CMND |
| [4. Rekognition](#4-rekognition--nhìn-ảnh-và-video) | Nhận diện vật thể, khuôn mặt, kiểm duyệt nội dung |
| [5. Transcribe](#5-transcribe--tiếng-nói-thành-chữ) | Ghi âm cuộc gọi, phụ đề, streaming realtime |
| [6. Polly](#6-polly--chữ-thành-tiếng-nói) | Đọc bài báo, IVR, giọng nói cho ứng dụng |
| [7. Translate](#7-translate--dịch) | Dịch nội dung, dịch cả thư mục S3 |
| [8. Comprehend](#8-comprehend--hiểu-văn-bản) | Cảm xúc, thực thể, PII trong văn bản |
| [9. Lex](#9-lex--chatbot-có-ý-định-và-slot) | Chatbot, IVR có hội thoại |
| [10. SageMaker và khi nào KHÔNG dùng](#10-sagemaker-ai--và-khi-nào-không-dùng-nó) | Ranh giới giữa API dựng sẵn và tự huấn luyện |
| [11. Bốn chế độ suy luận](#11-bốn-chế-độ-suy-luận-của-sagemaker) | Câu hỏi thiết kế thật nhất của SageMaker |
| [12. Media](#12-media-elastic-transcoder-đã-chết-kinesis-video-streams-thì-không) | Transcode video, luồng video từ camera |
| [13. Amplify và Device Farm](#13-amplify-và-device-farm) | Front-end host, kiểm thử trên máy thật |
| [14. Mẫu kiến trúc bất đồng bộ](#14-mẫu-kiến-trúc-bất-đồng-bộ--cái-khung-dùng-lại-được) | Vì sao gần như mọi luồng AI phải qua hàng đợi |
| [15. Chi phí và dữ liệu](#15-chi-phí-và-dữ-liệu--hai-thứ-dễ-mất-kiểm-soát) | Tính tiền theo đơn vị nào, dữ liệu khách hàng đi đâu |
| [16. Bài toán thiết kế](#16-ba-bài-toán-thiết-kế-có-lời-giải) | Ba đề có lời giải và lý do loại phương án khác |

Liên quan: [tích hợp](06-tich-hop.md) cho SNS/SQS/Step Functions/API Gateway — chương này
dùng lại toàn bộ; [compute](01-compute.md) cho Lambda; [storage](02-storage.md) cho S3
event notification; [chi phí](10-chi-phi.md) cho bức tranh tiền tổng thể.

---

## 1. Ba câu hỏi phải trả lời trước khi vẽ bất cứ thứ gì

### Câu 1 — nó nhận gì

Chia làm hai nhóm, và ranh giới này quan trọng hơn tên dịch vụ:

- **Nhận bytes trong request**: bạn nhét dữ liệu thẳng vào lời gọi API. Luôn có giới hạn
  nhỏ (5–10 MB) vì đó là một HTTP request.
- **Nhận con trỏ S3**: bạn đưa `{Bucket, Name}`, dịch vụ tự đọc từ S3 bằng IAM role. Giới
  hạn lớn hơn nhiều (500 MB tới 10 GB), và **luôn đi kèm mô hình bất đồng bộ**.

Nếu bạn thấy một API bắt buộc nhận đường dẫn S3, gần như chắc chắn nó là job bất đồng bộ.
Ngược lại cũng đúng.

### Câu 2 — nó trả gì

- **Trả kết quả ngay trong response**: JSON, hoặc audio stream. Bạn xử lý tiếp trong cùng
  một lời gọi.
- **Trả `JobId`**: kết quả nằm ở nơi khác, bạn phải đi lấy. Ba cách lấy, và mỗi dịch vụ
  chọn một cách khác nhau — đây là chỗ hay bị nhầm nhất:

| Cách báo xong | Dịch vụ dùng cách này | Bạn phải dựng gì |
|---|---|---|
| **SNS topic** khai báo trong `NotificationChannel` | **Textract** (async), **Rekognition Video** (stored) | Một SNS topic + IAM role cho dịch vụ publish, rồi Lambda/SQS subscribe |
| **Sự kiện EventBridge** khi job đổi trạng thái | **Transcribe** (`Transcribe Job State Change`), **Translate** (`Translate TextTranslationJob State Change`) | Một EventBridge rule khớp `source` và `detail.status` |
| **Ghi thẳng file kết quả vào S3** | **Polly** async, **Translate** batch, **Transcribe** batch, **SageMaker** async/batch | S3 event notification trên bucket đích, hoặc kết hợp hai cách trên |
| **Tự poll `Describe*Job`** | **Comprehend** async | Step Functions `Wait` + `Choice`, hoặc Lambda có lịch |

Ba dịch vụ ghi ra S3 *và* bắn thông báo — bạn chọn bắt cái nào. Bắt cả hai là nhân đôi
việc xử lý.

### Câu 3 — đồng bộ hay bất đồng bộ

Bảng này là thứ đáng thuộc nhất trong cả chương:

| Dịch vụ | API đồng bộ | API bất đồng bộ | Ranh giới quyết định |
|---|---|---|---|
| **Textract** | `DetectDocumentText`, `AnalyzeDocument` | `StartDocumentTextDetection`, `StartDocumentAnalysis` | **PDF/TIFF nhiều hơn 1 trang → bắt buộc async** |
| **Rekognition** | mọi API ảnh (`DetectLabels`, `CompareFaces`…) | `StartLabelDetection`, `StartContentModeration`… cho video | **Ảnh → sync. Video → async, không có lựa chọn** |
| **Transcribe** | streaming (HTTP/2, WebSocket) | `StartTranscriptionJob` | Cần chữ hiện ngay khi đang nói → streaming; có file sẵn → job |
| **Polly** | `SynthesizeSpeech` (≤ 3.000 ký tự tính tiền) | `StartSpeechSynthesisTask` (≤ 100.000) | Độ dài văn bản, không phải yêu cầu độ trễ |
| **Translate** | `TranslateText` (≤ 10.000 byte) | `StartTextTranslationJob` (S3 → S3) | Một đoạn text → sync; cả thư mục file → batch |
| **Comprehend** | `Detect*`, `BatchDetect*` (25 doc) | `Start*DetectionJob` | Trên 25 tài liệu hoặc tài liệu > 100 KB → async |
| **Lex** | `RecognizeText`, `RecognizeUtterance` | không có | Luôn đồng bộ — nó là hội thoại |
| **SageMaker** | real-time, serverless | asynchronous, batch transform | Xem [mục 11](#11-bốn-chế-độ-suy-luận-của-sagemaker) |

### Cái khung mà 80% luồng AI thật đều dùng

```
                                            ┌──────────────────────┐
   người dùng                               │  SNS / EventBridge   │
       │ PUT                                │  (job xong)          │
       ▼                                    └──────────┬───────────┘
  ┌─────────┐   s3:ObjectCreated   ┌────────┐          │
  │ S3 vào  │────────────────────► │ Lambda │──Start*──┤
  └─────────┘   (event / EventBr.) │  khởi  │  JobId   │
                                   │  động  │          ▼
                                   └────────┘     ┌────────┐   Get*   ┌──────────┐
                                                  │ Lambda │◄─────────│ dịch vụ  │
                                                  │  thu   │  kết quả │    AI    │
                                                  └───┬────┘          └──────────┘
                                                      │ ghi
                                            ┌─────────▼─────────┐
                                            │ DynamoDB / S3 ra  │
                                            └───────────────────┘
```

Hai Lambda, không phải một. Lambda thứ nhất **chỉ khởi động job rồi chết** — nó chạy vài
trăm mili giây. Lambda thứ hai được đánh thức bởi thông báo, đi lấy kết quả và lưu lại.
Không có Lambda nào ngồi chờ.

Người mới hay viết một Lambda duy nhất: gọi `Start*` rồi `while` cho tới khi job xong.
Sai ở ba chỗ: bạn trả tiền Lambda cho thời gian **ngủ**; job dài quá 15 phút thì Lambda
chết giữa chừng và bạn mất `JobId`; và mỗi lần retry là một job mới, tính tiền lại từ đầu.

### Bức tường 29 giây của API Gateway

Nếu client gọi qua API Gateway, bạn có **29 giây** cho toàn bộ integration (mặc định; nâng
được nhưng chỉ với REST API kiểu Regional/Private, và đánh đổi bằng throttle quota mức
account — xem [mục 8 của chương tích hợp](06-tich-hop.md#8-api-gateway)).

29 giây đủ cho: một ảnh Rekognition, một trang Textract sync, một đoạn `TranslateText`,
một câu Polly ngắn, một lần `DetectSentiment`.

29 giây **không** đủ cho: PDF 40 trang, video 10 phút, file audio 1 giờ, SageMaker
asynchronous endpoint. Với những thứ đó, API Gateway phải trả `202 Accepted` kèm một
`jobId` ngay lập tức, và client tự hỏi lại — hoặc bạn đẩy kết quả về bằng WebSocket API.

Đây không phải chi tiết vận hành. Đây là **lý do luồng media/AI gần như luôn bất đồng bộ**.

---

## 2. Bài toán → dịch vụ, và chỗ đề gài nhầm

| Bài toán trong đề | Dịch vụ đúng | Đề hay gài nhầm sang | Vì sao cái kia sai |
|---|---|---|---|
| Trích **chữ và bảng biểu** từ ảnh chụp hoá đơn / form scan | **Textract** | Rekognition (`DetectText`) | `DetectText` chỉ đọc **tối đa 100 từ** rời rạc trong ảnh (biển số, chữ trên áo). Nó không hiểu **cấu trúc**: không có key-value, không có ô bảng, không có PDF |
| Đọc **biển số xe / chữ trên biển hiệu** trong ảnh đường phố | **Rekognition** `DetectText` | Textract | Textract chỉ nhận **JPEG, PNG, PDF, TIFF** và tối ưu cho tài liệu phẳng, không phải cảnh thật |
| Nhận diện **vật thể, khuôn mặt, nội dung nhạy cảm** trong ảnh | **Rekognition** Image (sync) | Comprehend | Comprehend chỉ nhận **văn bản**, không nhận ảnh |
| Việc trên nhưng với **video** | **Rekognition Video** (async, S3) | Rekognition Image + tự cắt frame | Cắt frame thủ công thì mất `Person Pathing`, mất timestamp, và **tính tiền theo ảnh** — đắt hơn nhiều lần |
| Camera an ninh, phát hiện người **trong lúc đang quay** | **Kinesis Video Streams** + **Rekognition streaming video events** | Rekognition Video stored | Stored video analysis chỉ đọc **file trong S3**, không đọc luồng đang chạy |
| Ghi âm cuộc gọi → **chữ** | **Transcribe** | Comprehend | Comprehend nhận **chữ**, không nhận audio. Nó là bước **sau** Transcribe |
| **Chữ → giọng nói** | **Polly** | Transcribe | Ngược chiều. Nhớ: Trans**cribe** = *chép lại*, Polly = *nói ra* |
| Dịch nội dung sang ngôn ngữ khác | **Translate** | Comprehend | Comprehend **nhận ra** ngôn ngữ (`DetectDominantLanguage`) nhưng không dịch |
| Rút **thực thể, cụm từ khoá, cảm xúc, PII** từ văn bản | **Comprehend** | Kendra / Textract | Kendra là **tìm kiếm** doanh nghiệp, không phân tích. Textract lấy chữ ra, không hiểu nghĩa chữ |
| **Chatbot** có ý định và slot, hỏi lại người dùng khi thiếu thông tin | **Lex** | Comprehend | Comprehend không có **trạng thái hội thoại**: không nhớ slot đã điền, không biết hỏi tiếp |
| Dự đoán / phân loại theo dữ liệu **riêng của bạn** mà không API nào làm được | **SageMaker AI** | Comprehend Custom | Comprehend Custom chỉ làm **phân loại văn bản** và **thực thể tuỳ chỉnh**. Ngoài văn bản thì hết đường |
| Chuyển mã video VOD sang nhiều bitrate | **AWS Elemental MediaConvert** | **Elastic Transcoder** | Elastic Transcoder **đã ngừng hoạt động 13/11/2025** — xem [mục 12](#12-media-elastic-transcoder-đã-chết-kinesis-video-streams-thì-không) |
| Host web app React/Next.js kèm CI/CD theo nhánh Git | **Amplify** | S3 + CloudFront tự dựng | Không sai về kỹ thuật, nhưng đề nói "ít công vận hành nhất, deploy theo nhánh, có preview cho pull request" thì đó là Amplify |
| Kiểm thử app di động trên **máy thật** | **Device Farm** | EC2 với emulator | Emulator không phát hiện lỗi phần cứng, lỗi driver, lỗi màn hình notch. Đề nói "real devices" là chỉ thẳng tên |

Bốn cặp đáng thuộc, vì đề gài đúng bốn cặp này nhiều nhất:

**Textract ≠ Rekognition.** Cả hai "đọc chữ trong ảnh". Textract hiểu **tài liệu** (bảng,
key-value, chữ ký, form). Rekognition hiểu **cảnh** (vật thể, mặt người, chữ rời rạc).
Từ khoá `invoice`, `form`, `receipt`, `PDF`, `key-value pair`, `table` → Textract, luôn.

**Transcribe ≠ Comprehend.** Transcribe biến âm thanh thành chữ. Comprehend biến chữ
thành *ý nghĩa*. Đề contact center thường cần **cả hai, nối tiếp**.

**Comprehend ≠ Kendra.** Comprehend phân tích **một tài liệu bạn đưa cho nó**. Kendra
**lục tìm** trong hàng triệu tài liệu để trả lời câu hỏi bằng ngôn ngữ tự nhiên.

**Lex ≠ Comprehend.** Lex có **trạng thái**. Nó nhớ đang ở intent nào, còn thiếu slot nào,
và biết hỏi lại. Comprehend là một hàm thuần: vào chữ, ra JSON, không nhớ gì.

---

## 3. Textract — lấy chữ ra khỏi tài liệu

### Hai API, hai kiến trúc hoàn toàn khác

```
ẢNH MỘT TRANG (đồng bộ)                   PDF NHIỀU TRANG (bất đồng bộ)

 client                                    Lambda
   │ AnalyzeDocument(bytes ≤ 10 MB)          │ StartDocumentAnalysis({S3Object},
   │                                         │                      NotificationChannel)
   ▼                                         ▼
 Textract ──► JSON ngay trong response     Textract ──► JobId (ngay lập tức)
   (vài trăm ms tới vài giây)                 │
                                              │ xử lý nền, phút tới hàng chục phút
                                              ▼
                                          SNS topic ──► Lambda ──► GetDocumentAnalysis(JobId)
                                                                       │ phân trang NextToken
                                                                       ▼
                                                                  DynamoDB / S3
```

Bên trái không cần hạ tầng gì. Bên phải cần **một SNS topic, một IAM role cho Textract
publish vào topic đó, và một Lambda thứ hai**. Đề mô tả "PDF hồ sơ vay 300 trang" mà bạn
vẽ luồng bên trái là hỏng ngay từ ô đầu tiên.

### Con số ép bạn chọn nhánh nào (tính đến 2026-08)

| | Đồng bộ | Bất đồng bộ |
|---|---|---|
| Định dạng | JPEG, PNG, PDF, TIFF | JPEG, PNG, PDF, TIFF |
| Kích thước | **10 MB** | JPEG/PNG **10 MB**; PDF/TIFF **500 MB** |
| Số trang | PDF/TIFF **1 trang** | **3.000 trang** |
| Queries mỗi trang | **15** | **30** |
| Nguồn dữ liệu | bytes trong request, hoặc S3 | **chỉ S3** |
| Báo xong | response | **SNS** `NotificationChannel` |

Giới hạn khác cắt ngang cả hai nhánh:

- **Ngôn ngữ: chỉ English, French, German, Italian, Portuguese, Spanish.** Không có tiếng
  Việt, không có tiếng Nhật/Trung. Queries chỉ tiếng Anh. Chữ viết tay chỉ tiếng Anh.
  Đây là câu hỏi đầu tiên phải hỏi khi khách hàng Việt Nam nói "số hoá hoá đơn".
- Không đọc được **văn bản dọc** (kiểu tiếng Nhật truyền thống).
- Chữ tối thiểu **15 pixel** chiều cao (≈ font 8pt ở 150 DPI). Scan 72 DPI là hỏng.
- PDF **không được đặt mật khẩu**, không hỗ trợ PDF dạng XFA.
- Ảnh tối đa **10.000 pixel** mỗi chiều.

### Năm API, và giá chênh nhau 33 lần

| API / feature | Trả về gì | Giá mỗi trang (Oregon, 1M trang đầu) |
|---|---|---|
| `DetectDocumentText` | dòng và từ, kèm toạ độ | **$0,0015** |
| `AnalyzeDocument` — Tables | ô bảng, hàng, cột | $0,015 |
| `AnalyzeDocument` — Queries | trả lời câu hỏi bạn đặt ("Tên khách hàng là gì?") | $0,015 |
| `AnalyzeDocument` — Forms | cặp key-value | **$0,05** |
| `AnalyzeDocument` — Forms + Tables + Queries | cả ba | **$0,070** |
| `AnalyzeExpense` | hoá đơn: tổng tiền, thuế, dòng hàng | riêng |
| `AnalyzeID` | hộ chiếu, bằng lái | riêng |

Forms đắt gấp **33 lần** Detect. Bài học thiết kế: nếu bạn chỉ cần toàn bộ chữ để đẩy vào
tìm kiếm, đừng bật Forms. Chỉ bật Forms cho **đúng loại tài liệu cần nó**, và định tuyến
bằng một bước phân loại rẻ trước đó.

**Queries là thứ hay bị bỏ qua.** Thay vì tự viết regex mò trên output của Forms, bạn hỏi
thẳng bằng tiếng Anh và nhận đúng giá trị. Giá bằng Tables, rẻ hơn Forms ba lần rưỡi. Đề
mô tả "trích 10 trường cố định từ nhiều mẫu form khác nhau" → Queries, không phải Forms.

---

## 4. Rekognition — nhìn ảnh và video

### Ảnh: đồng bộ, và tính tiền theo **lời gọi API**, không theo ảnh

| | Giá trị |
|---|---|
| Định dạng | **chỉ PNG và JPEG** — không TIFF, không PDF, không GIF |
| Ảnh gửi dạng bytes trong request | **5 MB** (4 MB với `DetectProtectiveEquipment`) |
| Ảnh là object trong S3 | **15 MB** |
| Kích thước tối thiểu | **80 × 80 pixel** |
| Kích thước tối đa | 10.000 pixel mỗi chiều (`DetectLabels`, `DetectModerationLabels`) |
| Mặt nhỏ nhất nhận ra được | **40 × 40 px** trong ảnh 1920 × 1080 |
| `DetectText` | tối đa **100 từ** mỗi ảnh |
| Face collection | tối đa **20 triệu** face vector |

Giá chia hai nhóm (tính đến 2026-08, us-east-1):

- **Group 2** — `DetectLabels`, `DetectFaces`, `DetectModerationLabels`, `DetectText`,
  `RecognizeCelebrities`, `DetectProtectiveEquipment`: **$0,0010/ảnh** cho 1 triệu ảnh
  đầu, $0,0008 tiếp theo.
- **Group 1** — `IndexFaces`, `SearchFacesByImage`, `CompareFaces`, `SearchUsers`…: giá
  riêng, cộng thêm **$0,00001/face metadata/tháng** cho phần lưu trữ vector.

Bẫy tiền nằm ở một dòng trong tài liệu giá: *"Running multiple APIs against a single image
counts as processing multiple images."* Bạn gọi `DetectLabels` + `DetectFaces` +
`DetectModerationLabels` trên **cùng một ảnh** thì bị tính **ba ảnh**. Một pipeline
"phân tích toàn diện mỗi ảnh upload" vô tình đắt gấp ba so với dự toán.

### Video: bất đồng bộ, luôn luôn

```
S3 (video)                                Kinesis Video Streams (camera)
   │ StartLabelDetection({S3Object},         │
   │   NotificationChannel: SNS)             │  stream processor
   ▼                                         ▼
 JobId ──► (xử lý) ──► SNS ──► Lambda      Rekognition streaming video events
                         │ GetLabelDetection  │  (person / pet / package)
                         ▼                    ▼
                    kết quả có timestamp   Kinesis Data Streams / SNS
```

| | Giá trị |
|---|---|
| Kích thước file | **10 GB** |
| Độ dài | **6 giờ** |
| Codec | **H.264**, đóng gói MP4 hoặc MOV; audio phải là **AAC** |
| Job đồng thời | **20** mỗi account |
| TTL của `NextToken` khi phân trang kết quả | 24 giờ |
| Giá — Label Detection | **$0,10/phút** |
| Giá — Shot Detection | $0,05/phút |
| Giá — Content Moderation | **$0,10/phút** |
| Giá — streaming video events | **$0,00817/phút**, tối đa **120 giây mỗi sự kiện** |

Cùng bẫy như ảnh: chạy nhiều API trên cùng một đoạn video thì trả tiền nhiều lần.

Hai con số đáng để ý cạnh nhau: video stored **$0,10/phút**, streaming events
**$0,00817/phút** — chênh **12 lần**. Lý do: streaming chỉ phân tích đoạn ngắn sau khi có
chuyển động, không quét toàn bộ. Với camera an ninh 24/7, đây là khác biệt giữa hoá đơn
$4.300/tháng và $350/tháng cho một camera.

`Person Pathing` (theo dấu người qua các frame), `Face Search` trong video, và
`Content Moderation` là ba thứ chỉ Rekognition Video làm được — cắt frame rồi gọi API ảnh
không thay thế được, vì mất chiều thời gian.

---

## 5. Transcribe — tiếng nói thành chữ

### Batch: một job đọc S3, ghi S3

```
S3 (mp3/wav/flac/mp4) ──StartTranscriptionJob──► Transcribe
                                                    │ vài phút
                                                    ▼
                     EventBridge "Transcribe Job State Change"
                        detail.TranscriptionJobStatus = COMPLETED
                                                    │
                                                    ▼
                                        Lambda ──► đọc transcript JSON từ S3
```

Đây là job bất đồng bộ thuần: bạn đưa URI của file trong S3, Transcribe ghi một file JSON
kết quả vào S3 (bucket của bạn hoặc bucket do dịch vụ quản lý).

**Cách bắt sự kiện xong là EventBridge, không phải SNS.** Textract dùng `NotificationChannel`
với SNS; Transcribe không có tham số đó. Rule:

```json
{
  "source": ["aws.transcribe"],
  "detail-type": ["Transcribe Job State Change"],
  "detail": { "TranscriptionJobStatus": ["COMPLETED", "FAILED"] }
}
```

Nhớ bắt cả `FAILED`. Job hỏng mà không ai biết là lỗi vận hành phổ biến nhất của luồng này.

### Con số (tính đến 2026-08)

| | Giá trị |
|---|---|
| Kích thước file tối đa | **2 GB** |
| Độ dài audio tối đa | **28.800 giây = 8 giờ** (Medical: 4 giờ) |
| Job batch đồng thời | **250** mỗi region (nâng được) |
| Luồng streaming đồng thời (HTTP/2 + WebSocket) | **25** mỗi region (nâng được) |
| Bản ghi job được giữ | **90 ngày** — sau đó `GetTranscriptionJob` không còn thấy |
| Custom vocabulary | **51.200 byte**, tối đa 100 vocabulary mỗi region |
| Kênh audio nhận diện được | **2** |
| Đơn vị tính tiền | **giây audio**, không có mức tối thiểu |

Hai kênh được tính là **một** thời lượng, không phải hai — nên ghi âm cuộc gọi hai kênh
(mỗi người một kênh) không đắt gấp đôi, và bạn nên bật `ChannelIdentification` vì nó cho
kết quả tách người nói chính xác hơn hẳn `SpeakerDiarization` đoán mò trên một kênh.

Bản ghi job giữ **90 ngày** là con số dễ bỏ sót: transcript nằm trong S3 vĩnh viễn, nhưng
**metadata của job** thì không. Nếu quy trình của bạn dựa vào việc gọi lại
`GetTranscriptionJob` để tra cứu, nó sẽ vỡ sau ba tháng.

### Streaming khác batch ở đâu

Streaming mở một kết nối HTTP/2 hoặc WebSocket, bạn đẩy từng chunk audio (khuyến nghị
≤ 1 giây mỗi chunk) và nhận về kết quả **từng phần** rồi được sửa dần khi có thêm ngữ
cảnh. Đây là thứ duy nhất cho phụ đề trực tiếp và trợ lý cho tổng đài viên đang nghe máy.

Đừng dùng streaming cho file đã có sẵn: bạn phải tự phát lại file theo thời gian thực, tức
là một file 1 giờ mất 1 giờ để xử lý, trong khi batch xong nhanh hơn nhiều và rẻ hơn để
vận hành. Đề nói "đã có sẵn hàng nghìn file ghi âm" → batch, không bàn cãi.

---

## 6. Polly — chữ thành tiếng nói

Hai API, và ranh giới **là độ dài văn bản**, không phải yêu cầu độ trễ:

| | `SynthesizeSpeech` | `StartSpeechSynthesisTask` |
|---|---|---|
| Kiểu | đồng bộ, trả **audio stream** | bất đồng bộ, **ghi file vào S3** |
| Ký tự tính tiền tối đa | **3.000** | **100.000** |
| Tổng ký tự (gồm thẻ SSML) | **6.000** | **200.000** |
| Lexicon mỗi lời gọi | 5 | 5 |
| Báo xong | ngay | file xuất hiện trong S3 (bắt bằng S3 event) |

**Thẻ SSML không bị tính tiền.** Đây là chi tiết dễ chịu và có ích: bạn thêm `<break>`,
`<prosody>`, `<phoneme>` thoải mái mà không tốn thêm đồng nào. Chỉ nội dung đọc ra mới tính.

### Bốn engine, giá chênh 25 lần (tính đến 2026-08)

| Engine | Giá / 1 triệu ký tự | Dùng khi |
|---|---|---|
| **Standard** | **$4** | IVR, thông báo, khối lượng lớn, chất lượng đủ dùng |
| **Neural** | **$16** | giọng tự nhiên cho sản phẩm hướng người dùng |
| **Generative** | **$30** | hội thoại có cảm xúc |
| **Long-form** | **$100** | sách nói, podcast, đoạn văn dài |

1 triệu ký tự ≈ **23 giờ audio**. Chọn Long-form cho toàn bộ IVR của một tổng đài là cách
tiêu $100 thay vì $4 cho cùng một việc.

**Mẫu tiết kiệm quan trọng nhất: cache.** Câu chào của IVR không đổi mỗi ngày. Sinh một
lần, lưu MP3 trong S3, phát qua CloudFront. Gọi Polly mỗi cuộc gọi cho một câu tĩnh là
trả tiền lặp lại cho cùng một kết quả — và cộng thêm độ trễ mạng vào mỗi cuộc gọi.

**Speech marks** trả về JSON đánh dấu thời điểm từng từ/câu/âm vị trong audio. Đây là thứ
bạn cần cho karaoke, cho highlight chữ theo giọng đọc, cho lip-sync avatar. Nó không tính
tiền thêm nhưng là một lời gọi riêng.

---

## 7. Translate — dịch

| | `TranslateText` (sync) | `StartTextTranslationJob` (batch) |
|---|---|---|
| Đầu vào | chuỗi trong request, **10.000 byte UTF-8** | **thư mục trong S3** |
| Đầu ra | chuỗi trong response | file trong S3 |
| Định dạng | text thuần, HTML | txt, **html, docx, pptx, xlsx, xliff** |
| Mỗi tài liệu | tối đa 100.000 ký tự | **20 MB**, 1 triệu ký tự; phần chữ dịch được ≤ 1 MB |
| Cả lô | — | **5 GB**, tối đa 1 triệu tài liệu |
| Ngôn ngữ đích mỗi job | 1 | **10** |
| Job đồng thời | — | **10** |
| Báo xong | response | EventBridge `Translate TextTranslationJob State Change` |

Giá **$15 / 1 triệu ký tự**, tính cả khoảng trắng, **giống nhau cho sync và batch**. Nên
chọn batch không phải để rẻ hơn, mà vì ba lý do khác:

1. Nó **giữ nguyên định dạng** file Office và HTML. Tự cắt text ra khỏi `.docx` rồi ghép
   lại là một dự án riêng, và bạn sẽ làm hỏng bảng biểu.
2. Một lời gọi cho cả thư mục, thay vì tự viết vòng lặp có retry và throttling.
3. Nó nhận **10 ngôn ngữ đích** trong một job.

**Custom terminology** là file CSV/TMX (≤ 10 MB) ép Translate dịch một số thuật ngữ theo
đúng cách bạn muốn — tên sản phẩm, tên thương hiệu, từ nội bộ. Đây là câu trả lời cho đề
"bản dịch phải giữ nguyên tên sản phẩm của công ty". Không phải huấn luyện mô hình, không
tốn thêm tiền.

**Active Custom Translation** đi xa hơn: bạn đưa *parallel data* (cặp câu nguồn–đích mẫu)
và Translate điều chỉnh output ngay lúc chạy, không cần huấn luyện mô hình riêng. Đây là
bậc thang giữa "dùng mặc định" và "SageMaker tự huấn luyện".

---

## 8. Comprehend — hiểu văn bản

### Ba mức API, và giới hạn kích thước ép bạn chọn

| Mức | API | Giới hạn |
|---|---|---|
| **Một tài liệu, đồng bộ** | `DetectEntities`, `DetectKeyPhrases`, `DetectDominantLanguage` | **100 KB** |
| | `DetectSentiment`, `DetectTargetedSentiment`, `DetectSyntax` | **5 KB** |
| **Nhiều tài liệu, đồng bộ** | `BatchDetect*` | **25 tài liệu**, mỗi cái **5 KB** |
| **Bất đồng bộ** | `StartEntitiesDetectionJob`, `StartPiiEntitiesDetectionJob`… | mỗi tài liệu **1 MB**, cả job **5 GB**, tối đa **1 triệu file** |
| | `StartSentimentDetectionJob` | mỗi tài liệu **5 KB** |
| | `StartTopicsDetectionJob` | một file tối đa **100 MB** |

Job bất đồng bộ: tối đa **10 job đang chạy** mỗi loại API. Comprehend **không có
`NotificationChannel`** — bạn tự poll `Describe*Job`. Cách sạch là Step Functions với
`Wait` + `Choice`, không phải Lambda gọi `sleep`.

### Đơn vị tính tiền là chỗ dễ đốt tiền nhất trong cả chương

Comprehend tính theo **unit = 100 ký tự**, và **tối thiểu 3 unit (300 ký tự) mỗi request**.

Hệ quả trực tiếp: bạn phân tích cảm xúc 10 triệu tweet, mỗi tweet 60 ký tự. Bạn *nghĩ*
mình trả tiền cho 600 triệu ký tự. Thực tế bạn trả cho **3 tỉ ký tự** — gấp **5 lần** —
vì mỗi request bị làm tròn lên 300.

Cách sửa: gom bằng `BatchDetectSentiment` (25 tài liệu một lời gọi) hoặc dùng job bất đồng
bộ với định dạng "một tài liệu mỗi dòng". Đây là ví dụ điển hình của việc **hiểu đơn vị
tính tiền thay đổi thiết kế**, không chỉ thay đổi dự toán.

### PII detection — thứ hay ra đề nhất

`DetectPiiEntities` tìm vị trí PII trong văn bản; `ContainsPiiEntities` chỉ trả lời có/không
(rẻ hơn khi bạn chỉ cần lọc). Job bất đồng bộ `StartPiiEntitiesDetectionJob` chạy được ở
chế độ **redaction**: ghi ra bản đã che.

Ranh giới với [Macie](05-security.md): **Macie quét S3 để tìm bucket nào chứa dữ liệu nhạy
cảm** — nó là công cụ quản trị dữ liệu, có dashboard, có finding gửi vào Security Hub.
**Comprehend PII xử lý một tài liệu bạn đưa cho nó** — nó là một hàm trong pipeline. Đề
nói "phát hiện dữ liệu nhạy cảm nằm ở đâu trong data lake" → Macie. Đề nói "che số thẻ
tín dụng trong transcript trước khi lưu" → Comprehend PII (hoặc bật thẳng redaction trong
Transcribe).

### Custom model, và cái bẫy endpoint

Comprehend Custom làm được hai việc: **custom classification** và **custom entity
recognition**. Bạn đưa dữ liệu gán nhãn, nó huấn luyện — **$3/giờ huấn luyện**, **$0,50/tháng**
quản lý mô hình.

Suy luận có hai đường:

- **Bất đồng bộ**: tính theo unit như bình thường. Không có gì thường trực.
- **Real-time endpoint**: bạn cấp phát **Inference Unit (IU)**, mỗi IU = **100 ký tự/giây**
  throughput, tối đa 10 IU. Giá **$0,0005 mỗi IU-giây**, tính **từ lúc tạo tới lúc xoá**,
  bất kể có gọi hay không.

Một endpoint 5 IU chạy 24/7 là 5 × $0,0005 × 86.400 × 30 ≈ **$6.480/tháng** cho một mô
hình có thể không ai gọi lúc 3 giờ sáng. Đây đúng là mẫu bẫy tiền của SageMaker real-time
endpoint, thu nhỏ lại — và nó xuất hiện ở mọi dịch vụ AI có khái niệm "endpoint".

**Comprehend không có ở mọi region.** Tính đến 2026-08 nó chạy ở 13 region (không có
Jakarta, không có Paris, không có Hong Kong). Đây là ràng buộc kiến trúc thật: nếu dữ liệu
buộc phải nằm trong một region không có Comprehend, bạn phải đổi thiết kế, không phải đổi
tham số.

---

## 9. Lex — chatbot có ý định và slot

Lex là dịch vụ **có trạng thái** duy nhất trong nhóm AI dựng sẵn. Bốn khái niệm phải phân
biệt được:

- **Intent** — việc người dùng muốn làm: `DatPhong`, `KiemTraDonHang`.
- **Sample utterance** — các cách người ta nói ra ý định đó. Đây là dữ liệu huấn luyện
  NLU, không phải regex.
- **Slot** — thông tin cần thu thập cho intent: ngày, số người, mã đơn. Mỗi slot có một
  **slot type** (dựng sẵn như `AMAZON.Date`, `AMAZON.Number`, hoặc do bạn định nghĩa) và
  một **prompt** để hỏi khi thiếu.
- **Fulfillment** — việc thực sự làm sau khi đủ slot. Thường là một Lambda.

Vòng đời hội thoại và hai điểm móc Lambda:

```
người dùng nói ──► Lex: nhận intent + rút slot
                     │
                     ├── thiếu slot? ──► phát prompt, chờ trả lời  ◄──┐
                     │                                                │
                     ├── dialog code hook (Lambda) ────────────────────┘
                     │    kiểm tra hợp lệ, gợi ý lại, điền slot suy ra
                     │
                     └── đủ slot ──► fulfillment code hook (Lambda) ──► DynamoDB / API
                                          │
                                          ▼
                                    câu trả lời (text, hoặc Polly đọc ra)
```

Con số cấu hình (Lex V2, tính đến 2026-08):

| | Giá trị |
|---|---|
| Session timeout | mặc định **5 phút**, đặt được **0 – 1.440 phút (24 giờ)** |
| Timeout của code hook Lambda | mặc định **30 giây**, tối đa **120 giây** qua `x-amz-lex:codehook-timeout-ms` |

Session timeout là tham số thiết kế thật, không phải chi tiết vụn: nó quyết định người
dùng bỏ đi 10 phút rồi quay lại có phải khai lại từ đầu hay không. Đặt 24 giờ cho một luồng
điền form dài; đặt ngắn cho luồng có thông tin nhạy cảm.

**Lex kết hợp với ai:** với **Amazon Connect** để làm IVR (Connect gọi Lex, Lex hiểu ý
định, Lambda tra dữ liệu, Polly đọc câu trả lời). Với web/mobile qua SDK. Bản thân Lex đã
dùng ASR và TTS bên trong cho kênh thoại — bạn **không** phải tự ghép Transcribe và Polly
vào trước/sau Lex. Đề nào vẽ `Transcribe → Lex → Polly` cho một IVR là vẽ thừa hai dịch vụ.

---

## 10. SageMaker AI — và khi nào KHÔNG dùng nó

### Thang ba bậc

Đây là mục quan trọng nhất của nửa sau chương. Trước khi mở SageMaker, đi hết ba bậc theo
đúng thứ tự:

| Bậc | Cái gì | Thời gian tới production | Bạn phải có gì |
|---|---|---|---|
| **1. API dựng sẵn** | Rekognition, Textract, Comprehend, Translate… | **hàng giờ** | không gì cả |
| **2. SageMaker built-in algorithm / JumpStart** | XGBoost, Linear Learner, DeepAR, BlazingText; mô hình dựng sẵn để fine-tune | **ngày tới tuần** | dữ liệu gán nhãn, biết chọn hyperparameter |
| **3. Container tự viết** | mã training của bạn trong Docker image riêng | **tuần tới tháng** | đội ML, MLOps, ngân sách GPU |

**Quy tắc: nếu một API dựng sẵn giải được bài toán, tự huấn luyện là sai — sai về chi phí,
sai về thời gian, và sai về rủi ro vận hành.** Không phải "kém tối ưu", mà là sai.

Con số so sánh cho dễ hình dung: nhận diện vật thể trong 1 triệu ảnh bằng Rekognition tốn
**$1.000** và bạn viết xong trong một buổi chiều. Huấn luyện một mô hình detection tương
đương trên SageMaker: vài nghìn đô GPU-giờ, vài tuần công của một người biết việc, cộng
với một endpoint chạy 24/7 và trách nhiệm giám sát model drift **vĩnh viễn**.

### Khi nào bậc 1 không đủ, và bạn phải leo lên

Bốn dấu hiệu, và chỉ bốn:

1. **Nhãn không nằm trong bộ có sẵn.** Rekognition biết "xe", "người", "chó". Nó không
   biết "mối hàn bị nứt" hay "lá lúa bị đạo ôn". Bậc 2 bắt đầu ở đây — và với riêng ảnh,
   thử **Rekognition Custom Labels** trước SageMaker.
2. **Dữ liệu không phải văn bản/ảnh/audio.** Dự đoán churn từ 200 cột số trong data
   warehouse: không API nào làm. XGBoost built-in là bậc 2 chuẩn.
3. **Phải giải thích được quyết định**, hoặc mô hình phải chạy trong VPC không ra Internet
   với ràng buộc tuân thủ riêng.
4. **Ngôn ngữ hoặc miền dữ liệu không được hỗ trợ.** Textract không có tiếng Việt. Đó là
   một lý do hợp lệ để leo bậc — nhưng hãy kiểm tra bậc 2 (fine-tune một mô hình OCR có
   sẵn từ JumpStart) trước khi nghĩ tới bậc 3.

Nếu không rơi vào bốn cái trên mà vẫn muốn SageMaker, lý do thật thường là "đội muốn làm
ML", không phải yêu cầu kỹ thuật. Trong đề thi, đó luôn là đáp án sai.

### Ở mức SAA, bạn cần biết gì về training

Ít hơn bạn tưởng. Ba điều:

- Training job đọc dữ liệu từ **S3**, chạy trên **instance ML riêng** (`ml.*`), ghi
  artifact mô hình trở lại S3. Instance chỉ sống trong lúc train.
- **Managed Spot Training** giảm tới ~90% chi phí training, đổi lại job có thể bị ngắt —
  nên bật checkpoint. Đây là đáp án Domain 4 cho "giảm chi phí huấn luyện".
- Notebook instance là một EC2 **chạy liên tục cho tới khi bạn dừng nó**. Notebook quên
  tắt là khoản chi âm thầm kinh điển, cùng họ với endpoint quên xoá.

---

## 11. Bốn chế độ suy luận của SageMaker

Đây là câu hỏi thiết kế thật của SageMaker, và là chỗ đề SAA hỏi sâu nhất trong cả chương.

| | Real-time | Serverless | Asynchronous | Batch transform |
|---|---|---|---|---|
| Có endpoint thường trực | **có** | có (nhưng vô hình) | có | **không** |
| Payload tối đa | **25 MB** | **4 MB** | **1 GB** | **100 MB** mỗi record |
| Thời gian xử lý tối đa | **60 giây** (8 phút nếu streaming response) | **60 giây** | **60 phút** | hàng **ngày** |
| Trả kết quả kiểu gì | trong response | trong response | **ghi S3** + SNS báo | **ghi S3** |
| Có hàng đợi bên trong | không | không | **có** (TTL 6 giờ) | không |
| Scale về 0 khi rảnh | **không** | **có** | **có** | không có gì để scale |
| Trả tiền lúc rảnh | **có** — theo instance-giờ | **không** | **không** (nếu cấu hình scale-to-zero) | **không** |
| Cold start | không | **có** | có khi vừa scale từ 0 | không áp dụng |
| Cấu hình cỡ máy | chọn instance type | chọn **RAM 1024–6144 MB** | chọn instance type | chọn instance type |

Tham số riêng đáng nhớ:

- **Serverless**: RAM chỉ nhận 6 giá trị — 1024, 2048, 3072, 4096, 5120, 6144 MB — theo
  bước 1 GB, kèm **5 GB disk tạm** bất kể chọn cỡ nào. `MaxConcurrency` **1–200**. Có
  `ProvisionedConcurrency` để giết cold start, nhưng bật cái đó là bạn quay lại trả tiền
  cho lúc rảnh.
- **Asynchronous**: hàng đợi nội bộ, message TTL **6 giờ**. Bạn khai báo SNS topic cho
  thành công và cho lỗi. Đây là chế độ duy nhất vừa **scale về 0** vừa nhận payload lớn.
- **Batch transform**: `MaxPayloadInMB` mặc định **6**, tối đa **100**; và ràng buộc
  `MaxConcurrentTransforms × MaxPayloadInMB ≤ 100 MB`. `InvocationsTimeoutInSeconds` mặc
  định 600 giây, tối đa **3.600**.

### Cây chọn trong bốn dòng

1. Không cần trả lời ngay, có sẵn cả tập dữ liệu, chạy theo lịch → **batch transform**.
   Không có endpoint thì không có gì để quên tắt.
2. Cần trả lời cho từng request, nhưng payload lớn (ảnh y tế, video) hoặc xử lý lâu hơn
   60 giây → **asynchronous**.
3. Cần trả lời trong response, tải **thất thường hoặc thưa**, chịu được cold start →
   **serverless**.
4. Cần trả lời trong response, tải **ổn định**, cần độ trễ thấp nhất và ổn định →
   **real-time**. Đây là chế độ duy nhất bạn trả tiền 24/7.

### Bẫy tiền lớn nhất của cả chương

Real-time endpoint tính tiền **theo instance-giờ, từ lúc `CreateEndpoint` tới lúc
`DeleteEndpoint`**, không liên quan gì tới số lần gọi. Một `ml.m5.xlarge` chạy quên trong
ba tháng là hoá đơn bốn chữ số cho một mô hình không ai dùng.

Đề Domain 4 dựng bẫy này bằng câu: *"mô hình chỉ được gọi vài trăm lần mỗi ngày, phân bố
không đều"*. Đáp án sai hấp dẫn là "giảm cỡ instance". Đáp án đúng là **serverless
inference** (hoặc asynchronous với scale-to-zero) — vì vấn đề không phải cỡ máy, mà là
**bạn đang trả tiền cho thời gian không có request nào**.

---

## 12. Media: Elastic Transcoder đã chết, Kinesis Video Streams thì không

### Elastic Transcoder — biết để nhận ra nó sai

**AWS ngừng hỗ trợ Amazon Elastic Transcoder từ 13/11/2025.** Sau ngày đó console và tài
nguyên Elastic Transcoder không truy cập được nữa.

Nó vẫn nằm trong đề cương SAA-C03 và vẫn xuất hiện trong tài liệu ôn thi cũ, nên bạn cần
biết ba điều:

- Nó là dịch vụ **transcode VOD**: đọc file video từ S3, chuyển sang định dạng/bitrate
  khác, ghi trở lại S3. Mô hình: **pipeline** + **job** + **preset**. Job bất đồng bộ, báo
  xong qua SNS.
- Người thay thế là **AWS Elemental MediaConvert**: rẻ hơn (khởi điểm **$0,0075/phút** so
  với **$0,015/phút**), nhiều codec hiện đại hơn, có accelerated transcoding.
- Trong một hệ thống thật ngày hôm nay, **mọi câu trả lời "Elastic Transcoder" đều sai**.
  Trong phòng thi, nếu nó vẫn xuất hiện làm đáp án cho "chuyển mã video theo lô", nó là
  đáp án mà đề coi là đúng — hãy biết cả hai sự thật.

### Kinesis Video Streams — không phải Kinesis Data Streams

KVS nhận luồng media **có đánh chỉ mục thời gian** từ thiết bị: camera an ninh, dashcam,
drone, thiết bị y tế. Producer SDK trên thiết bị chia media thành **fragment** và đẩy lên;
KVS gắn timestamp và số thứ tự cho từng fragment.

```
camera ──Producer SDK──► Kinesis Video Stream ──┬──► GetMedia / GetMediaForFragmentList
 (fragment)                  (time-indexed)      │        (ứng dụng tự xử lý)
                                                 ├──► HLS / DASH session URL
                                                 │        (phát lại trên trình duyệt)
                                                 └──► Rekognition streaming video events
                                                          (person / pet / package)
```

Tham số quyết định kiến trúc là **`DataRetentionInHours`**:

- Mặc định **0** — stream **không lưu gì**. Consumer chỉ đọc được phần còn nằm trong buffer
  của host: **5 phút hoặc 200 MB**, cái nào đến trước.
- Tối thiểu khi bật là **1 giờ**.

Đây là bẫy thiết kế thật: bạn dựng stream với tham số mặc định, mọi thứ chạy tốt trong
demo (consumer bám sát realtime), rồi consumer chết 10 phút và **dữ liệu 10 phút đó biến
mất vĩnh viễn**. Nếu bạn cần phát lại, cần điều tra sau sự cố, cần chạy lại phân tích —
phải đặt retention từ đầu.

Mã hoá at-rest bằng KMS, mặc định khoá `aws/kinesisvideo`.

**WebRTC là một sản phẩm khác trong cùng dịch vụ**: peer-to-peer độ trễ **dưới một giây**
cho hai chiều (nói chuyện với camera cửa), dùng signaling channel + STUN/TURN. Giới hạn
của phần ingest/lưu qua WebRTC: bitrate **1 Mbps**, phiên **1 giờ**, idle timeout 3 phút,
tối đa **3 viewer** đồng thời một phiên. Đề nói "xem trực tiếp và nói chuyện hai chiều,
độ trễ dưới một giây" → WebRTC. Đề nói "ghi lại để phân tích sau" → stream thường có
retention.

### KVS và Kinesis Data Streams: cùng họ, khác việc

| | Kinesis Video Streams | Kinesis Data Streams |
|---|---|---|
| Dữ liệu | media có timestamp (fragment) | record dữ liệu (≤ 1 MB) |
| Đơn vị dung lượng | không có shard — tự scale | **shard** bạn tự đếm (hoặc on-demand) |
| Lưu trữ | `DataRetentionInHours`, mặc định **0** | mặc định **24 giờ**, tới 365 ngày |
| Consumer điển hình | Rekognition, HLS player, ứng dụng CV | Lambda, Flink, Firehose |
| Bài toán | camera, video từ thiết bị | telemetry, clickstream, log |

Chi tiết của Kinesis Data Streams nằm ở [mục 9 chương tích hợp](06-tich-hop.md#9-kinesis).

---

## 13. Amplify và Device Farm

Hai dịch vụ front-end trong đề cương, và ở mức SAA bạn cần biết đúng "khi nào chọn".

**AWS Amplify** là CI/CD + hosting cho ứng dụng web, gắn với Git. Đẩy commit lên nhánh →
Amplify build → deploy ra CDN, kèm domain tuỳ chỉnh và TLS được quản lý. **Mỗi nhánh Git
là một môi trường**, và pull request được cấp một URL preview riêng. Phần backend (Gen 2)
khai báo bằng TypeScript và sinh ra hạ tầng qua CDK.

Ranh giới với S3 + CloudFront tự dựng: về mặt hạ tầng cuối cùng thì giống nhau. Amplify
mua cho bạn **cái pipeline và cái quy ước nhánh**. Đề có chữ *"deploy tự động khi push",
"preview cho mỗi pull request", "ít công vận hành nhất cho đội front-end"* → Amplify. Đề
mô tả static site đơn giản và nhấn mạnh **chi phí thấp nhất** → S3 + CloudFront, xem
[networking](04-networking.md) và [storage](02-storage.md).

**AWS Device Farm** cho bạn chạy test trên **điện thoại và tablet thật** đặt trong hạ tầng
AWS, thay vì emulator. Hai chế độ: **automated test run** (chạy suite test song song trên
nhiều thiết bị, trả về log, video, screenshot, số đo hiệu năng) và **remote access** (bạn
bấm trực tiếp vào một máy thật qua trình duyệt). Nó cũng chạy được test Selenium trên
**desktop browser**.

Từ khoá nhận diện: **"real devices"**, **"different OS versions"**, **"physical device
fragmentation"**. Không có dịch vụ AWS nào khác làm việc này, nên khi từ khoá xuất hiện,
câu hỏi coi như đã có đáp án.

---

## 14. Mẫu kiến trúc bất đồng bộ — cái khung dùng lại được

### Vì sao gần như mọi luồng AI phải qua hàng đợi hoặc thông báo

Bốn lý do, xếp theo mức độ hay bị bỏ qua:

1. **Thời gian xử lý vượt mọi giới hạn đồng bộ.** 29 giây của API Gateway. 15 phút của
   Lambda. 30 giây mặc định của một client HTTP. Một PDF 500 trang không quan tâm tới cái
   nào trong ba con số đó.
2. **Quota TPS của dịch vụ AI là một trần cứng.** Textract, Rekognition, Comprehend đều
   có quota transaction/giây theo account theo region. Gọi thẳng từ một Lambda scale tự do
   là cách chắc chắn nhất để tự đâm vào `ProvisionedThroughputExceededException`. Hàng đợi
   + `maximum_concurrency` của event source mapping là cái van điều tiết.
3. **Retry phải giữ được ngữ cảnh.** Gọi đồng bộ mà lỗi thì bạn mất luôn yêu cầu, trừ khi
   client tự thử lại. Message trong SQS thì vẫn nằm đó, và sau `maxReceiveCount` lần thì
   rơi vào DLQ để bạn xem sau.
4. **Chi phí đỉnh tải.** Người dùng upload 5.000 file lúc 9 giờ sáng thứ Hai. Không có
   hàng đợi thì bạn phải chịu được đỉnh đó ngay lập tức; có hàng đợi thì bạn xử lý đều
   trong hai giờ và trả cùng số tiền.

### Điều gì xảy ra nếu bạn gọi đồng bộ từ API Gateway

```
client ──► API Gateway ──► Lambda ──► StartDocumentAnalysis + poll
             │ 29 giây          │ 15 phút
             │                  └── vẫn đang chờ Textract...
             ▼
      504 Gateway Timeout  ← client nhận cái này ở giây thứ 29
                             Lambda VẪN CHẠY và VẪN TÍNH TIỀN tới hết timeout
                             Textract VẪN XỬ LÝ và VẪN TÍNH TIỀN
                             Client thử lại → job thứ hai → tính tiền hai lần
```

Ba hậu quả, và cái thứ ba là cái đau nhất: **client thử lại tạo ra công việc trùng**. Bạn
trả tiền hai lần, ba lần cho cùng một tài liệu, và không hề có lỗi nào trong log nói cho
bạn biết.

### Ba mẫu đúng

**Mẫu A — trả `202` rồi cho client hỏi lại.** Đơn giản nhất, đủ cho phần lớn trường hợp.

```
POST /documents ──► API Gateway ──► Lambda ──► S3 (lưu file)
                         │                 └─► DynamoDB: {jobId, status: PENDING}
                         └──► 202 Accepted { "jobId": "abc" }

GET /documents/abc ──► API Gateway ──► Lambda ──► DynamoDB ──► { status, result }
```

Client poll `GET`. Rẻ, không có gì phải giữ kết nối. Nhược: client phải viết vòng poll, và
độ trễ nhận kết quả bằng chu kỳ poll.

**Mẫu B — SNS/EventBridge đánh thức bước sau.** Đây là mẫu chuẩn của Textract, Rekognition
Video, Transcribe, SageMaker async. Không có gì phải poll, và mỗi bước là một Lambda ngắn.
Xem sơ đồ ở [mục 1](#1-ba-câu-hỏi-phải-trả-lời-trước-khi-vẽ-bất-cứ-thứ-gì).

**Mẫu C — Step Functions điều phối cả chuỗi.** Khi luồng có nhiều bước AI nối tiếp nhau và
cần rẽ nhánh:

```
Start ─► StartTranscriptionJob ─► Wait(30s) ─► GetJob ─► Choice
                                     ▲                     │ IN_PROGRESS
                                     └─────────────────────┘
                                                           │ COMPLETED
                                                           ▼
                                                   DetectSentiment (Comprehend)
                                                           │
                                                     Choice: điểm < -0.5?
                                                      │ có          │ không
                                                      ▼             ▼
                                                 SNS cảnh báo    ghi DynamoDB
```

Ba lý do chọn Step Functions ở đây thay vì chuỗi Lambda: retry và backoff là **cấu hình**
chứ không phải code; bạn **nhìn thấy** workflow đang đứng ở bước nào khi có sự cố; và với
Comprehend — dịch vụ không có `NotificationChannel` — vòng `Wait` + `Choice` là cách poll
duy nhất không đốt thời gian chạy Lambda.

Chọn Standard hay Express: luồng media/AI gần như luôn dài hơn 5 phút → **Standard**. Chi
tiết ở [mục 7 chương tích hợp](06-tich-hop.md#7-step-functions--khi-nào-chuỗi-lambda-là-sai).

### Van điều tiết — đừng bỏ qua

Đặt SQS giữa S3 event và Lambda gọi dịch vụ AI, rồi đặt `maximum_concurrency` trên event
source mapping. Không có nó, 5.000 file upload cùng lúc sẽ sinh 5.000 Lambda gọi Textract
đồng thời, đụng TPS quota, và bạn nhận về một đống lỗi throttle mà retry chỉ làm tệ hơn.

---

## 15. Chi phí và dữ liệu — hai thứ dễ mất kiểm soát

### Mỗi dịch vụ tính tiền theo một đơn vị khác nhau

| Dịch vụ | Đơn vị tính tiền | Giá tham khảo (2026-08, us-east-1/Oregon) |
|---|---|---|
| Textract | **trang**, khác nhau theo feature | $0,0015 (Detect) → $0,070 (Forms+Tables+Queries) |
| Rekognition Image | **lời gọi API trên một ảnh** | $0,0010/ảnh (Group 2, 1M đầu) |
| Rekognition Video stored | **phút video**, mỗi API tính riêng | $0,10/phút (Label, Moderation) |
| Rekognition streaming events | **phút video được xử lý** | $0,00817/phút |
| Transcribe | **giây audio** | tính theo bậc, không có mức tối thiểu |
| Polly | **ký tự tính tiền** (SSML miễn phí) | $4 → $100 / 1 triệu ký tự tuỳ engine |
| Translate | **ký tự** (kể cả khoảng trắng) | $15 / 1 triệu ký tự |
| Comprehend | **unit 100 ký tự, tối thiểu 3 unit/request** | theo bậc |
| Comprehend endpoint | **IU-giây**, từ lúc tạo tới lúc xoá | $0,0005 / IU-giây |
| SageMaker real-time | **instance-giờ**, từ lúc tạo tới lúc xoá | theo instance |
| SageMaker serverless | **GB-giây** + số request | không tính lúc rảnh |
| SageMaker batch/async | **instance-giờ** trong lúc chạy | không tính lúc rảnh |
| Kinesis Video Streams | **GB nạp vào + GB lưu + GB đọc ra** | theo bậc |

Bốn cái bẫy tiền của cả chương, gom lại:

1. **Nhiều API trên cùng một ảnh/video = tính tiền nhiều lần** (Rekognition).
2. **Tối thiểu 300 ký tự mỗi request** (Comprehend) — biến 60 ký tự thành 300.
3. **Endpoint tính tiền theo thời gian tồn tại, không theo lượt gọi** (SageMaker real-time,
   Comprehend custom endpoint).
4. **Bật feature đắt cho toàn bộ luồng** thay vì chỉ cho tài liệu cần nó (Textract Forms,
   Polly Long-form).

Cả bốn đều là bẫy **thiết kế**, không phải bẫy giá. Bạn không sửa được bằng cách xin giảm
giá; bạn sửa bằng cách vẽ lại luồng.

### Dữ liệu bạn gửi đi là dữ liệu khách hàng

Ba ràng buộc phải nói ra khi thiết kế, không phải khi bị hỏi:

**Ranh giới Region.** Lời gọi tới `textract.ap-southeast-1.amazonaws.com` xử lý dữ liệu
trong Singapore. Nếu ứng dụng ở Singapore mà bạn gọi endpoint `us-east-1` — vì đó là ví dụ
trong tài liệu — bạn vừa chuyển dữ liệu khách hàng qua biên giới, và trả thêm tiền
data transfer. Kiểm tra region trong client config, không giả định.

**Không phải dịch vụ nào cũng có ở mọi region.** Comprehend có mặt ở 13 region. Nếu dữ liệu
buộc phải ở lại một region không có dịch vụ, đó là ràng buộc kiến trúc — bạn phải đổi
phương án, không phải đổi tham số.

**Đường đi của gói tin.** Mặc định lời gọi tới các dịch vụ AI đi qua Internet công cộng
(vẫn mã hoá TLS). Với workload trong VPC không có đường ra Internet, hoặc có yêu cầu tuân
thủ, dùng **interface VPC endpoint (AWS PrivateLink)** để lưu lượng ở lại mạng AWS. Xem
[networking](04-networking.md).

Thêm hai điều thuộc [bảo mật](05-security.md) nhưng liên quan trực tiếp:

- Dữ liệu vào/ra hầu hết luồng này nằm trong **S3**. Mã hoá bằng SSE-KMS, và **IAM role
  của dịch vụ AI phải có quyền dùng khoá KMS đó** — đây là nguyên nhân số một khiến job
  bất đồng bộ báo `AccessDeniedException` dù bucket policy trông đã đúng.
- Bật **CloudTrail** cho các lời gọi này. Nó ghi lại ai gọi, gọi cái gì, lúc nào — và với
  dữ liệu khách hàng, đó là yêu cầu kiểm toán chứ không phải tuỳ chọn.

---

## 16. Ba bài toán thiết kế có lời giải

### Bài 1 — số hoá hồ sơ vay

*"Ngân hàng nhận hồ sơ vay dạng PDF, mỗi hồ sơ 50–300 trang, 2.000 hồ sơ mỗi ngày. Cần
trích 12 trường cố định (tên, thu nhập, số tài khoản…) và phát hiện chữ ký. Kết quả vào
hệ thống thẩm định. Người dùng upload qua web."*

**Thiết kế:**

```
Web ──presigned URL──► S3 (raw/) ──EventBridge──► Lambda "khởi động"
                                                     │ StartDocumentAnalysis(
                                                     │   FeatureTypes=[QUERIES, SIGNATURES],
                                                     │   NotificationChannel=SNS)
                                                     ▼
                                                  JobId → DynamoDB {jobId, status}
                                                     │
        SNS "textract-done" ◄────────────────────────┘ (sau vài phút)
              │
              ▼
         SQS ──► Lambda "thu kết quả" ──GetDocumentAnalysis(phân trang)──► DynamoDB
                                                                              │
                                                                              ▼
                                                                     EventBridge → thẩm định
```

**Vì sao từng lựa chọn:**

- **Presigned URL** thay vì upload qua API Gateway: payload API Gateway tối đa 10 MB, PDF
  300 trang vượt xa. Và bạn không muốn trả tiền compute để làm đường ống chuyển byte.
- **`StartDocumentAnalysis` bất đồng bộ**: bắt buộc. API đồng bộ chỉ nhận PDF **1 trang**.
  Không có cách nào lách.
- **Queries thay vì Forms**: 12 trường cố định là đúng bài toán của Queries. Giá **$0,015**
  so với **$0,05** mỗi trang. Với 2.000 hồ sơ × 150 trang = 300.000 trang/ngày, chênh lệch
  là **$10.500/ngày**. Đây không phải tối ưu vặt.
- **SQS giữa SNS và Lambda thu kết quả**: `GetDocumentAnalysis` phải phân trang, có thể
  chậm và có thể lỗi. SQS cho bạn retry và DLQ mà không mất `JobId`.

**Loại các phương án khác:**

- *Rekognition `DetectText`*: chỉ đọc tối đa 100 từ rời rạc, không đọc PDF, không có
  key-value. Sai về bản chất.
- *Textract đồng bộ, tự tách PDF thành từng trang bằng Lambda*: bạn phải viết và bảo trì
  code tách PDF, trả tiền compute cho việc đó, mất ngữ cảnh giữa các trang (bảng vắt qua
  hai trang là hỏng), và tăng số lời gọi API lên 150 lần — đụng TPS quota.
- *Lambda gọi `Start*` rồi poll trong cùng một invocation*: job có thể chạy lâu hơn 15
  phút; bạn trả tiền Lambda cho thời gian ngủ.
- *Gọi đồng bộ qua API Gateway*: 504 ở giây thứ 29, người dùng bấm lại, mỗi lần bấm là một
  job tính tiền mới.

### Bài 2 — kiểm duyệt video người dùng đăng

*"Nền tảng chia sẻ video. 10.000 video/ngày, trung bình 3 phút. Cần chặn nội dung khiêu
dâm/bạo lực trước khi công khai, và tạo phụ đề tự động. Video chỉ hiện sau khi duyệt xong."*

**Thiết kế:**

```
upload ──► S3 (quarantine/) ──EventBridge──► Step Functions (Standard)
                                                │
                    ┌───────────────────────────┴──────────────────────────┐
                    ▼ Parallel                                             ▼
   StartContentModeration (Rekognition)              StartTranscriptionJob (Transcribe)
        │ SNS → callback                                    │ EventBridge → callback
        ▼                                                   ▼
   nhãn + độ tin cậy                                   transcript JSON
                    └───────────────────────────┬──────────────────────────┘
                                                ▼
                                        Choice: có nhãn cấm > 80%?
                                    ┌───────────┴────────────┐
                          có ────►  SQS "review người"   không ────► S3 (public/)
                                    (SNS báo đội duyệt)              + DynamoDB: APPROVED
```

**Vì sao từng lựa chọn:**

- **`StartContentModeration` chứ không phải cắt frame rồi gọi `DetectModerationLabels`**:
  Rekognition Video trả về timestamp của từng vi phạm; cắt frame thì mất chiều thời gian
  và tính tiền theo ảnh. 3 phút video ở 1 frame/giây = 180 ảnh × $0,0010 = **$0,18**, so
  với 3 phút × $0,10 = **$0,30** — tưởng rẻ hơn, nhưng 1 frame/giây bỏ sót nội dung và bạn
  phải tự viết code cắt frame, tự lưu frame, tự dọn frame.
- **Bucket `quarantine/` riêng, không phải một cột `status` trong DB**: video chưa duyệt
  không được nằm chung chỗ với video công khai. Đây là ranh giới quyền, không phải ranh
  giới dữ liệu — bucket policy khác nhau, IAM khác nhau.
- **Step Functions với `Parallel`**: hai job chạy song song, tổng thời gian bằng job dài
  hơn chứ không phải tổng hai job. Standard vì luồng dài hơn 5 phút.
- **Ngưỡng độ tin cậy đưa vào `Choice`, không chôn trong code Lambda**: chỉnh ngưỡng là
  chỉnh state machine, không phải deploy lại function.

**Loại các phương án khác:**

- *Lambda gọi trực tiếp không qua Step Functions*: hai job bất đồng bộ với hai cơ chế
  callback khác nhau (SNS và EventBridge) phải được join lại. Tự viết cái join đó là tự
  viết một orchestrator tệ hơn.
- *SageMaker với mô hình moderation tự huấn luyện*: đây đúng là bậc 3 khi bậc 1 đã giải
  được. Chi phí và thời gian gấp hàng chục lần, và bạn phải chịu trách nhiệm về chất lượng
  mô hình vĩnh viễn.
- *Cho video public ngay rồi gỡ sau*: đề nói rõ "chỉ hiện sau khi duyệt xong".

### Bài 3 — phân tích cuộc gọi tổng đài

*"5.000 cuộc gọi/ngày, trung bình 6 phút, ghi âm stereo (mỗi người một kênh). Cần: chữ
transcript, cảm xúc khách hàng, che số thẻ tín dụng, và cảnh báo trong 15 phút nếu cuộc
gọi rất tiêu cực."*

**Thiết kế:** `S3 → EventBridge → Step Functions`, trong đó:

1. `StartTranscriptionJob` với `ChannelIdentification: true` (không phải speaker
   diarization — đề đã cho stereo, mỗi kênh một người) và `ContentRedaction` bật sẵn để
   **Transcribe tự che PII ngay trong transcript**.
2. `Wait` + `GetTranscriptionJob` + `Choice` cho tới `COMPLETED`.
3. `BatchDetectSentiment` (Comprehend) trên transcript đã cắt thành đoạn.
4. `Choice`: điểm `NEGATIVE` vượt ngưỡng → SNS tới đội quản lý.

**Vì sao từng lựa chọn:**

- **Redaction làm ở Transcribe, không phải Comprehend PII sau đó**: nếu bạn transcribe rồi
  mới che, số thẻ **đã từng tồn tại** trong file transcript gốc nằm trong S3. Che ở bước
  sinh ra dữ liệu là đúng nguyên tắc; che ở bước sau là vá.
- **`BatchDetectSentiment` chứ không phải `DetectSentiment` từng câu**: mỗi request tính
  tối thiểu 300 ký tự. 5.000 cuộc gọi × ~120 câu = 600.000 request/ngày, phần lớn là câu
  ngắn bị làm tròn lên 300 ký tự. Gom 25 câu một lời gọi cắt chi phí đi nhiều lần.
- **`ChannelIdentification` chứ không phải `SpeakerDiarization`**: đề đã cho stereo. Hai
  kênh vẫn tính là một thời lượng audio, nên không đắt thêm, và kết quả tách người nói
  chính xác hơn hẳn.
- **15 phút là ràng buộc lỏng**: cuộc gọi 6 phút transcribe trong vài phút, Comprehend
  vài giây. Không cần streaming. Nếu đề đổi thành *"cảnh báo cho tổng đài viên trong lúc
  đang nói chuyện"* thì toàn bộ thiết kế đổi sang **Transcribe streaming** — và bạn phải
  nhớ trần **25 luồng đồng thời** mỗi region.

---

## Bảng số phải nhớ

| Thứ | Con số |
|---|---|
| **API Gateway integration timeout** | **29 giây** — bức tường của mọi luồng đồng bộ |
| Textract sync | **10 MB**, PDF/TIFF **1 trang**, 15 query/trang |
| Textract async | PDF/TIFF **500 MB**, **3.000 trang**, 30 query/trang |
| Textract ngôn ngữ | **EN, FR, DE, IT, PT, ES** — không có tiếng Việt |
| Textract giá | Detect **$0,0015**/trang · Tables/Queries $0,015 · **Forms $0,05** |
| Rekognition ảnh | bytes **5 MB**, S3 **15 MB**, **chỉ PNG/JPEG**, tối thiểu 80×80 px |
| Rekognition `DetectText` | tối đa **100 từ** mỗi ảnh |
| Rekognition video | **10 GB**, **6 giờ**, H.264 trong MP4/MOV, **20 job** đồng thời |
| Rekognition giá | ảnh **$0,0010** · video **$0,10/phút** · streaming **$0,00817/phút** |
| Transcribe batch | **2 GB**, **8 giờ** (28.800 giây), **250 job** đồng thời |
| Transcribe streaming | **25 luồng** đồng thời mỗi region |
| Transcribe bản ghi job | giữ **90 ngày** |
| Polly sync | **3.000 ký tự tính tiền** / 6.000 tổng |
| Polly async | **100.000 ký tự tính tiền** / 200.000 tổng |
| Polly giá | Standard **$4** · Neural **$16** · Generative $30 · Long-form **$100** / 1M ký tự |
| Translate sync | **10.000 byte** UTF-8 |
| Translate batch | doc **20 MB**, lô **5 GB**, **10 ngôn ngữ đích**, 10 job đồng thời |
| Translate giá | **$15 / 1 triệu ký tự** (sync và batch như nhau) |
| Comprehend sync | entities/key phrase **100 KB**; sentiment/syntax **5 KB** |
| Comprehend batch | **25 tài liệu** × 5 KB |
| Comprehend async | doc **1 MB**, job **5 GB**, **10 job** đang chạy mỗi API |
| Comprehend giá | unit = **100 ký tự**, **tối thiểu 3 unit (300 ký tự)** mỗi request |
| Comprehend endpoint | 1 IU = **100 ký tự/giây**, tối đa 10 IU, **$0,0005/IU-giây** |
| Lex session timeout | mặc định **5 phút**, đặt được **0 – 1.440 phút** |
| Lex code hook timeout | mặc định **30 giây**, tối đa **120 giây** |
| SageMaker real-time | payload **25 MB**, **60 giây** (8 phút nếu streaming response) |
| SageMaker serverless | payload **4 MB**, 60 giây, RAM **1024–6144 MB**, MaxConcurrency **200** |
| SageMaker async | payload **1 GB**, **60 phút**, TTL hàng đợi **6 giờ**, **scale về 0** |
| SageMaker batch transform | `MaxPayloadInMB` mặc định **6**, tối đa **100**; timeout tối đa 3.600 giây |
| Kinesis Video Streams | `DataRetentionInHours` mặc định **0**; buffer host **5 phút / 200 MB** |
| KVS WebRTC ingest | **1 Mbps**, phiên **1 giờ**, tối đa **3 viewer** |
| **Elastic Transcoder** | **ngừng hoạt động 13/11/2025** → MediaConvert |

---

## Bẫy đề thi

**1. "Trích dữ liệu từ ảnh chụp hoá đơn."**
Sai hấp dẫn: **Rekognition** — vì nó "phân tích ảnh". Đúng: **Textract**. Rekognition
`DetectText` đọc tối đa **100 từ rời rạc** và không hiểu cấu trúc: không key-value, không
ô bảng, không PDF. Từ khoá phân biệt: `invoice`, `form`, `receipt`, `table`, `key-value`,
`PDF` → Textract, luôn luôn.

**2. "Xử lý PDF 200 trang, trả kết quả cho người dùng qua API."**
Sai hấp dẫn: `AnalyzeDocument` sau API Gateway. Đúng: **`StartDocumentAnalysis` bất đồng
bộ**, API trả `202` kèm `jobId`. Hai lý do độc lập, mỗi lý do đủ để loại phương án kia:
API đồng bộ **chỉ nhận PDF 1 trang**, và API Gateway timeout ở **29 giây**.

**3. "Phân tích video, dùng Rekognition Image cho từng frame."**
Đúng: **Rekognition Video** với API `Start*`. Cắt frame làm mất chiều thời gian (`Person
Pathing`, timestamp vi phạm), tính tiền theo ảnh, và bắt bạn viết + bảo trì code cắt frame.
Video → async, không có API đồng bộ, không có lựa chọn.

**4. "Chuyển audio thành chữ rồi phân tích cảm xúc — dùng Comprehend cho cả hai."**
Đúng: **Transcribe rồi Comprehend**, nối tiếp. Comprehend **không nhận audio**. Đề contact
center gần như luôn cần cả hai, và thứ tự là cố định.

**5. "Cần chatbot hiểu ý định người dùng → Comprehend."**
Đúng: **Lex**. Comprehend là hàm thuần — vào chữ, ra JSON, không nhớ gì. Lex có **trạng
thái hội thoại**: nhớ intent đang mở, slot nào còn thiếu, và biết hỏi lại. Từ khoá:
`intent`, `slot`, `chatbot`, `conversational`.

**6. "Mô hình chỉ được gọi vài trăm lần mỗi ngày, giảm chi phí SageMaker."**
Sai hấp dẫn: "chọn instance nhỏ hơn cho real-time endpoint". Đúng: **serverless inference**
(hoặc **asynchronous** với scale-to-zero). Vấn đề không phải cỡ máy — real-time endpoint
tính tiền **theo thời gian tồn tại**, nên máy nhỏ chạy 24/7 vẫn là trả tiền cho lúc rảnh.

**7. "Payload 200 MB cho suy luận."**
Sai hấp dẫn: real-time endpoint với instance to hơn. Đúng: real-time tối đa **25 MB**,
serverless **4 MB**. Trên ngưỡng đó chỉ còn **asynchronous** (1 GB) hoặc **batch
transform** (100 MB mỗi record). Kích thước payload là ràng buộc cứng, không mua được
bằng tiền.

**8. "Có sẵn API dựng sẵn nhưng đề vẫn đưa SageMaker làm đáp án."**
Đúng: nếu Rekognition/Textract/Comprehend giải được bài toán, **SageMaker luôn sai** —
sai về chi phí, sai về thời gian tới production, sai về rủi ro vận hành. SageMaker chỉ
đúng khi nhãn không có sẵn, dữ liệu không phải văn bản/ảnh/audio, hoặc có ràng buộc tuân
thủ đặc biệt.

**9. "Elastic Transcoder để chuyển mã video."**
Đúng ở phòng thi (đề cương chưa cập nhật), **sai trong thực tế**: dịch vụ **đã ngừng hoạt
động từ 13/11/2025**. Người thay là **AWS Elemental MediaConvert**, rẻ hơn một nửa ở mức
khởi điểm. Biết cả hai sự thật.

**10. "Kinesis Video Streams giữ dữ liệu như Kinesis Data Streams."**
Đúng: KDS mặc định giữ **24 giờ**. KVS mặc định **`DataRetentionInHours = 0`** — không lưu
gì, consumer chỉ đọc được buffer **5 phút / 200 MB**. Không đặt retention từ đầu là mất
dữ liệu vĩnh viễn khi consumer chết.

**11. "Phân tích cảm xúc 10 triệu tweet ngắn bằng `DetectSentiment` từng cái."**
Đúng: Comprehend tính **tối thiểu 3 unit (300 ký tự) mỗi request**. Tweet 60 ký tự bị tính
như 300 — đắt gấp **5 lần**. Dùng `BatchDetectSentiment` (25 tài liệu) hoặc job bất đồng bộ.

**12. "Chạy `DetectLabels` + `DetectFaces` + `DetectModerationLabels` trên mỗi ảnh."**
Đúng: **mỗi lời gọi API trên cùng một ảnh tính là một ảnh riêng**. Ba API = trả tiền ba
ảnh. Chỉ gọi API bạn thực sự cần cho từng loại ảnh.

**13. "Ghép Transcribe → Lex → Polly để làm IVR."**
Đúng: **Lex đã có ASR và TTS bên trong** cho kênh thoại. Ghép thêm Transcribe và Polly là
vẽ thừa hai dịch vụ, thêm độ trễ, thêm tiền. Với tổng đài, đường đúng là **Amazon Connect
gọi thẳng Lex**.

**14. "Textract xử lý tài liệu tiếng Việt."**
Đúng: Textract chỉ hỗ trợ **English, French, German, Italian, Portuguese, Spanish**. Đây
không phải chi tiết vụn — nó loại thẳng Textract khỏi bài toán, và bạn phải leo lên bậc 2
hoặc 3 của thang.

---

## Cây quyết định

**Đầu vào là ảnh.** Cần chữ có cấu trúc (form, bảng, key-value, PDF) → **Textract**. Cần
biết trong ảnh có gì (vật thể, mặt người, nội dung nhạy cảm, chữ rời rạc) → **Rekognition
Image**, đồng bộ. Nhãn không nằm trong bộ có sẵn → **Rekognition Custom Labels** trước,
SageMaker sau.

**Đầu vào là video.** File trong S3 → **Rekognition Video**, bất đồng bộ, kết quả qua SNS.
Luồng đang chạy từ camera → **Kinesis Video Streams** + **Rekognition streaming video
events**. Cần chuyển mã sang nhiều bitrate → **MediaConvert** (không phải Elastic
Transcoder — đã chết). Cần xem trực tiếp hai chiều dưới một giây → **KVS WebRTC**.

**Đầu vào là audio.** Có file sẵn → **Transcribe batch**. Cần chữ hiện ngay khi đang nói →
**Transcribe streaming**, nhớ trần 25 luồng. Cần che PII → bật `ContentRedaction` ngay
trong Transcribe, đừng che ở bước sau.

**Đầu vào là văn bản.** Cần cảm xúc/thực thể/PII → **Comprehend**; trên 25 tài liệu hoặc
tài liệu > 100 KB thì dùng job bất đồng bộ. Cần dịch → **Translate**; cả thư mục file
Office thì dùng batch. Cần đọc thành tiếng → **Polly**; trên 3.000 ký tự tính tiền thì
dùng `StartSpeechSynthesisTask`.

**Cần hội thoại có trạng thái.** → **Lex**. Gắn với tổng đài → Amazon Connect gọi Lex trực
tiếp, không ghép thêm Transcribe/Polly.

**Không API nào giải được.** Đi thang ba bậc, theo thứ tự: dựng sẵn → **built-in algorithm
/ JumpStart** → container tự viết. Chỉ leo bậc khi có một trong bốn lý do ở
[mục 10](#10-sagemaker-ai--và-khi-nào-không-dùng-nó).

**Đã chọn SageMaker, giờ chọn chế độ suy luận.** Có sẵn cả tập dữ liệu, chạy theo lịch →
**batch transform**. Payload lớn hoặc xử lý quá 60 giây → **asynchronous**. Tải thưa và
thất thường → **serverless**. Tải ổn định, cần độ trễ thấp nhất → **real-time** (chế độ
duy nhất trả tiền 24/7).

**Ghép vào luồng.** Xử lý xong dưới 29 giây và client cần kết quả ngay → gọi đồng bộ qua
API Gateway. Mọi trường hợp còn lại → **bất đồng bộ**: API trả `202` + `jobId`, và dùng
SNS/EventBridge/Step Functions để nối các bước. Nhiều bước AI nối tiếp có rẽ nhánh →
**Step Functions Standard**.

**Front-end.** Web app cần CI/CD theo nhánh Git và preview cho pull request → **Amplify**.
Static site tối ưu chi phí → S3 + CloudFront. Kiểm thử trên máy thật → **Device Farm**.

---

## Nối với thực hành

| Lab | Chạm vào mục nào |
|---|---|
| [`labs/w06-serverless-api/`](../../learn-aws/labs/w06-serverless-api/) | Mục 1 và 14: API Gateway + Lambda + DynamoDB là đúng bộ khung của mẫu "trả `202` rồi cho client hỏi lại". Thêm một Lambda ngủ 35 giây để **tự tạo lỗi 504 ở giây thứ 29** — bức tường quan trọng nhất của chương này |
| [`labs-self/w06-serverless-api/`](../../learn-aws/labs-self/w06-serverless-api/) | Bản tự viết. Đây là chỗ đáng dựng thêm bảng `jobs` trong DynamoDB và một route `GET /jobs/{id}` để hiểu vì sao trạng thái job phải nằm ngoài Lambda |
| [`labs/w07-decoupling/`](../../learn-aws/labs/w07-decoupling/) | Mục 14: SNS topic + SQS + DLQ + Step Functions chính là bộ khung callback của Textract/Rekognition Video. Thay message giả bằng một payload hình dạng `NotificationChannel` thật để thấy Lambda thu kết quả cần đọc field nào |
| [`labs-self/w07-decoupling/`](../../learn-aws/labs-self/w07-decoupling/) | Bản tự viết. Đặt `maximum_concurrency` trên event source mapping rồi đẩy 1.000 message để thấy cái van điều tiết ở mục 14 hoạt động ra sao |
| [`labs/w04-s3-cloudfront/`](../../learn-aws/labs/w04-s3-cloudfront/) | Mục 15: S3 event notification và presigned URL — hai thứ mọi luồng AI đều bắt đầu bằng |

Ba quan sát đáng làm nhất, không cần gọi API AI nào:

1. Trong lab w06, đặt một Lambda `time.sleep(35)` rồi gọi qua API Gateway. Bạn nhận **504
   ở giây 29**, nhưng xem CloudWatch Logs sẽ thấy Lambda **vẫn chạy tới giây 35 và vẫn
   tính tiền**. Đây là toàn bộ lý do luồng AI phải bất đồng bộ, nhìn thấy bằng mắt.
2. Trong lab w07, dựng chuỗi `SNS → SQS → Lambda` rồi tự publish một message có hình dạng
   giống thông báo Textract (`{"JobId": "...", "Status": "SUCCEEDED"}`). Bạn vừa dựng xong
   nửa sau của mẫu B mà không tốn một đồng Textract nào.
3. Vẽ lại bằng tay sơ đồ ở [mục 1](#1-ba-câu-hỏi-phải-trả-lời-trước-khi-vẽ-bất-cứ-thứ-gì)
   cho ba dịch vụ khác nhau — Textract (SNS), Transcribe (EventBridge), Comprehend (tự
   poll). Ba cách báo xong khác nhau là thứ dễ nhầm nhất khi làm thật.

---

## Nguồn nói khác

Chỗ `aws-saa-c03/12-ml-ai.md` sai, cũ hoặc thiếu (kiểm chứng ngày 2026-08):

| Nguồn nói | Thực tế | Docs |
|---|---|---|
| **Không nhắc Textract một lần nào** trong cả submodule | Textract là dịch vụ AI **ra đề nhiều nhất** trong nhóm này, và là cặp gài nhầm số một với Rekognition. Toàn bộ mục 3 là phần viết mới | [Textract limits](https://docs.aws.amazon.com/textract/latest/dg/limits-document.html) |
| **Không nhắc Elastic Transcoder, Kinesis Video Streams, Amplify, Device Farm** | Cả bốn nằm trong đề cương chính thức SAA-C03. Elastic Transcoder còn **đã ngừng hoạt động 13/11/2025** — một sự thật mà không tài liệu ôn thi nào cập nhật | [thông báo ngừng hỗ trợ](https://aws.amazon.com/blogs/media/support-for-amazon-elastic-transcoder-ending-soon/) |
| "SageMaker = Build, train, deploy ML models. Use case: Custom ML models" | Ba dòng cho dịch vụ có **bốn chế độ suy luận khác nhau về payload, timeout, chi phí lúc rảnh** — và đó mới là thứ đề hỏi. Câu hỏi SAA không hỏi "SageMaker là gì", nó hỏi "chọn chế độ nào" | [Inference options](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model-options.html) |
| Bảng "Quick reference" chỉ ánh xạ **tên → một dòng use case** | Thiếu hoàn toàn trục **đồng bộ / bất đồng bộ**, mà đó chính là thứ quyết định kiến trúc. Biết "Rekognition → images/videos" không nói cho bạn biết video **bắt buộc** đi đường async qua SNS | [Rekognition video](https://docs.aws.amazon.com/rekognition/latest/dg/video.html) |
| Không nhắc giới hạn kích thước nào của dịch vụ nào | Mọi ranh giới thiết kế trong chương này đều là một con số: 1 trang vs 3.000 trang, 5 MB vs 15 MB, 25 MB vs 1 GB. Không có con số thì không thiết kế được | [Textract](https://docs.aws.amazon.com/textract/latest/dg/limits-document.html), [Rekognition](https://docs.aws.amazon.com/rekognition/latest/dg/limits.html) |
| Không nhắc **ngôn ngữ được hỗ trợ** | Textract không có tiếng Việt; Comprehend chỉ có ở 13 region. Với người học ở Việt Nam, đây là ràng buộc đầu tiên phải kiểm tra, không phải chi tiết cuối cùng | [Comprehend regions](https://docs.aws.amazon.com/comprehend/latest/dg/guidelines-and-limits.html) |
| Không nhắc **đơn vị tính tiền** của dịch vụ nào | Comprehend tính tối thiểu 300 ký tự mỗi request; Rekognition tính mỗi API trên cùng một ảnh là một ảnh riêng; SageMaker real-time tính theo thời gian tồn tại endpoint. Cả ba đều đổi thiết kế, không chỉ đổi dự toán | [Comprehend pricing](https://aws.amazon.com/comprehend/pricing/), [Rekognition pricing](https://aws.amazon.com/rekognition/pricing/) |
| Liệt kê **Forecast** và **Personalize** ngang hàng với bảy dịch vụ kia | Hai dịch vụ này **không nằm trong phạm vi chương này** theo đề cương. Chúng ra đề rất ít; xem mục Ngoài phạm vi | — |

Một điểm nguồn nói đúng nhưng gây hiểu nhầm: *"ML/AI chiếm 3-5% câu hỏi. Biết cơ bản là
đủ."* Đúng cho phòng thi. **Sai cho việc dựng hệ thống** — và sổ tay này viết cho việc
thứ hai.

---

## Ngoài phạm vi

- **Amazon Bedrock** — nền tảng mô hình sinh (foundation model). Không nằm trong đề cương
  SAA-C03 hiện hành. [docs](https://docs.aws.amazon.com/bedrock/)
- **Amazon Kendra** — tìm kiếm doanh nghiệp bằng ngôn ngữ tự nhiên. Chỉ cần phân biệt được
  với Comprehend. [docs](https://docs.aws.amazon.com/kendra/)
- **Amazon Forecast, Amazon Personalize** — dự báo chuỗi thời gian và gợi ý. Nhận diện một
  dòng: "demand forecasting" → Forecast, "recommendation" → Personalize. [docs](https://docs.aws.amazon.com/personalize/)
- **AWS Elemental MediaConvert / MediaLive / MediaPackage** — bộ media đầy đủ. Chỉ cần
  biết MediaConvert thay Elastic Transcoder. [docs](https://docs.aws.amazon.com/mediaconvert/)
- **SageMaker Ground Truth, Feature Store, Pipelines, Model Monitor, Clarify** — vòng đời
  MLOps. Ngoài phạm vi SAA hoàn toàn. [docs](https://docs.aws.amazon.com/sagemaker/)
- **Amazon Transcribe Call Analytics / Medical, Comprehend Medical** — biến thể theo ngành.
  [docs](https://docs.aws.amazon.com/transcribe/)
- **Amazon Connect** — tổng đài đám mây. Ra đề ít; chỉ cần biết nó là nơi Lex được gắn vào.
  [docs](https://docs.aws.amazon.com/connect/)
- **Amazon Rekognition Face Liveness, Custom Moderation** — tính năng chuyên sâu.
  [docs](https://docs.aws.amazon.com/rekognition/)

---

## Tự kiểm tra

**1.** Bạn cần trích 8 trường cố định từ PDF hợp đồng 120 trang, 500 hợp đồng mỗi ngày.
Mô tả luồng, nói rõ vì sao không dùng API đồng bộ, và vì sao chọn Queries thay vì Forms.

<details><summary>Đáp án</summary>

`S3 → EventBridge → Lambda gọi StartDocumentAnalysis(FeatureTypes=[QUERIES], NotificationChannel=SNS)`
→ SNS → SQS → Lambda gọi `GetDocumentAnalysis` (có phân trang bằng `NextToken`) → DynamoDB.

**Không dùng API đồng bộ vì hai lý do độc lập, mỗi lý do đã đủ để loại:** (1)
`AnalyzeDocument` đồng bộ chỉ nhận PDF/TIFF **1 trang** — 120 trang là bất khả thi về mặt
API, không phải về hiệu năng; (2) kể cả nếu được, một job như vậy không xong trong 29 giây
của API Gateway hay 15 phút của Lambda.

**Queries thay vì Forms:** 8 trường **cố định** đúng là bài toán của Queries — bạn hỏi
thẳng bằng tiếng Anh và nhận đúng giá trị, không phải mò trong output key-value. Giá
**$0,015** so với **$0,05** mỗi trang. Với 500 × 120 = 60.000 trang/ngày, chênh lệch là
**$2.100/ngày**, tức khoảng **$63.000/tháng**. Async cho phép 30 query mỗi trang, thừa cho
8 trường.

Điều phải kiểm tra trước tiên nhưng dễ quên: **ngôn ngữ của hợp đồng**. Textract chỉ hỗ trợ
EN, FR, DE, IT, PT, ES. Hợp đồng tiếng Việt thì cả thiết kế này vô nghĩa.

</details>

**2.** Giải thích vì sao gọi `DetectSentiment` cho từng bình luận 50 ký tự trên 20 triệu
bình luận là một sai lầm chi phí, và tính nhẩm mức độ.

<details><summary>Đáp án</summary>

Comprehend tính theo **unit = 100 ký tự**, với **mức tối thiểu 3 unit (300 ký tự) mỗi
request**. Bình luận 50 ký tự đáng lẽ là 1 unit, nhưng bị tính **3 unit**.

20 triệu request × 3 unit = **60 triệu unit** = tương đương **6 tỉ ký tự**. Nếu tính đúng
theo nội dung thật (20 triệu × 50 = 1 tỉ ký tự = 10 triệu unit) thì bạn đang trả **gấp 6
lần**.

Cách sửa: `BatchDetectSentiment` gộp **25 tài liệu** mỗi lời gọi — số request giảm 25 lần
và mức tối thiểu 3 unit chỉ áp một lần cho cả lô. Hoặc dùng
`StartSentimentDetectionJob` bất đồng bộ với file định dạng "một tài liệu mỗi dòng", cho
phép tới 1 triệu dòng trong một job.

Điểm chung của mọi bài như thế này: **đơn vị tính tiền là một quyết định thiết kế**. Bạn
không sửa nó bằng cách xin giảm giá.

</details>

**3.** Đội bạn muốn huấn luyện một mô hình SageMaker để phát hiện ảnh khoả thân trong nội
dung người dùng đăng. Phản biện, và nêu điều kiện nào thì họ đúng.

<details><summary>Đáp án</summary>

**Phản biện:** `Rekognition DetectModerationLabels` đã làm đúng việc đó, có sẵn phân cấp
nhãn (Explicit Nudity, Suggestive, Violence, Drugs…) kèm điểm tin cậy, giá **$0,0010/ảnh**,
và bạn tích hợp xong trong một buổi. Đây là bậc 1 của thang, và bậc 1 giải được thì leo
bậc là sai — sai về chi phí (GPU-giờ + endpoint 24/7), sai về thời gian tới production
(tuần thay vì giờ), và sai về rủi ro vận hành (bạn phải chịu trách nhiệm về chất lượng mô
hình và model drift vĩnh viễn, thay vì AWS).

**Họ đúng khi:** (a) nhãn cần phát hiện **không nằm trong bộ có sẵn** — ví dụ logo đối thủ,
lỗi sản phẩm đặc thù, biểu tượng cấm theo quy định địa phương; (b) có ràng buộc tuân thủ
buộc mô hình phải chạy trong VPC riêng, không được gửi ảnh ra dịch vụ ngoài; (c) đã đo
được rằng độ chính xác của Rekognition trên **đúng phân phối dữ liệu của họ** không đạt
ngưỡng nghiệp vụ — và (c) chỉ có giá trị khi đã đo, không phải khi đang phỏng đoán.

Kể cả khi họ đúng, bậc tiếp theo là **Rekognition Custom Labels** (huấn luyện nhãn riêng
trên nền Rekognition), không nhảy thẳng lên container tự viết.

</details>

**4.** Một mô hình SageMaker nhận ảnh y tế 300 MB, mất 12 phút để chạy, và được gọi khoảng
40 lần mỗi ngày, không đều. Chọn chế độ suy luận và loại từng chế độ còn lại.

<details><summary>Đáp án</summary>

**Asynchronous inference.**

- **Real-time**: loại vì **hai** ràng buộc cứng — payload tối đa **25 MB** (300 MB không
  vừa) và thời gian xử lý tối đa **60 giây** (12 phút không vừa). Ngoài ra nó tính tiền
  24/7 cho 40 lần gọi mỗi ngày.
- **Serverless**: loại vì payload tối đa **4 MB** và cũng giới hạn 60 giây. Còn xa hơn
  real-time.
- **Batch transform**: loại vì đây là **yêu cầu theo từng request rời rạc, đến bất chợt**,
  không phải một tập dữ liệu có sẵn chạy theo lịch. Dùng batch thì bạn phải tự gom request
  lại và chờ tới giờ chạy, tức là tự dựng một hàng đợi tệ hơn cái async đã có sẵn.
- **Asynchronous**: payload tới **1 GB** (300 MB vừa), thời gian tới **60 phút** (12 phút
  vừa), có hàng đợi nội bộ, ghi kết quả vào S3 và bắn SNS khi xong, và **scale về 0** khi
  không có request — nên 40 lần gọi mỗi ngày không phải trả tiền cho 24 giờ.

Điều phải nói thêm cho đủ: hàng đợi có **TTL 6 giờ**, nên nếu endpoint đang ở 0 instance
và burst request đến, phải đảm bảo autoscaling kịp gọi lên trước khi message hết hạn.

</details>

**5.** So sánh ba cơ chế báo "job xong" của Textract, Transcribe và Comprehend. Vì sao
chúng khác nhau lại là vấn đề khi bạn dựng một pipeline dùng cả ba?

<details><summary>Đáp án</summary>

- **Textract**: tham số `NotificationChannel = {SNSTopicArn, RoleArn}` ngay trong lời gọi
  `Start*`. Textract **tự publish** vào topic của bạn. Bạn phải tạo topic và cấp role cho
  Textract publish.
- **Transcribe**: **không có** tham số đó. Nó phát sự kiện lên **EventBridge** với
  `source: aws.transcribe`, `detail-type: Transcribe Job State Change`. Bạn viết rule khớp
  `detail.TranscriptionJobStatus`. (Translate cũng theo kiểu này.)
- **Comprehend**: **không có cả hai**. Bạn phải tự gọi `Describe*Job` cho tới khi thấy
  `COMPLETED`.

**Vì sao là vấn đề:** ba cơ chế nghĩa là **ba đoạn code khác nhau, ba loại quyền IAM khác
nhau, ba cách xử lý lỗi khác nhau** trong cùng một pipeline. Và nguy hiểm hơn: người thiết
kế dễ giả định cả ba giống nhau, viết xong luồng Textract chạy tốt, rồi đem đúng khuôn đó
áp cho Comprehend và ngồi chờ một thông báo **vĩnh viễn không tới**.

Cách xử lý sạch: dùng **Step Functions** làm lớp đồng nhất. Textract và Transcribe dùng
callback (`.waitForTaskToken` hoặc EventBridge → `SendTaskSuccess`), Comprehend dùng
`Wait` + `GetJob` + `Choice`. Khác biệt bị nhốt trong định nghĩa state machine thay vì
rải ra khắp code Lambda.

Đừng quên bắt trạng thái **thất bại** ở cả ba. Job hỏng mà im lặng là lỗi vận hành phổ
biến nhất của luồng bất đồng bộ.

</details>

**6.** Camera an ninh gửi video liên tục lên Kinesis Video Streams với cấu hình mặc định.
Consumer chết 20 phút vì lỗi deploy. Chuyện gì xảy ra với dữ liệu, và bạn đã nên làm gì
khác từ đầu?

<details><summary>Đáp án</summary>

**Dữ liệu 20 phút đó mất vĩnh viễn.** `DataRetentionInHours` mặc định là **0**, nghĩa là
stream **không lưu trữ gì**. Consumer chỉ đọc được phần fragment còn nằm trong buffer của
host, giới hạn ở **5 phút hoặc 200 MB**, cái nào đến trước. Sau 5 phút, dữ liệu bị đẩy ra
khỏi buffer và không có bản sao nào.

Đáng lẽ phải đặt `DataRetentionInHours` ngay lúc `CreateStream` — tối thiểu 1 giờ, thực tế
nên đặt theo thời gian bạn cần để phát hiện và sửa một sự cố (vài ngày là hợp lý cho
camera an ninh). Retention là thứ **phải quyết định trước**, không thêm vào sau khi mất dữ
liệu.

Đây đúng là cùng một bài học với EventBridge archive ở chương tích hợp: **khả năng phát
lại phải được thiết kế trước, không thêm vào sau sự cố**.

Nếu yêu cầu là "xem trực tiếp và nói chuyện hai chiều dưới một giây" chứ không phải lưu
lại, thì đây là bài toán của **KVS WebRTC** — nhưng WebRTC lại có giới hạn riêng: 1 Mbps,
phiên 1 giờ, tối đa 3 viewer. Hai sản phẩm, hai bài toán, đừng trộn.

</details>

**7.** Đề: *"Xử lý 3.000 ảnh sản phẩm mỗi giờ, mỗi ảnh cần nhãn vật thể, kiểm duyệt nội
dung và trích chữ trên bao bì. Tối ưu chi phí."* Vẽ thiết kế và chỉ ra bẫy tiền.

<details><summary>Đáp án</summary>

`S3 → SQS (có maximum_concurrency) → Lambda → Rekognition + Textract → DynamoDB`.

**Bẫy tiền thứ nhất:** Rekognition tính **mỗi lời gọi API trên cùng một ảnh là một ảnh
riêng**. `DetectLabels` + `DetectModerationLabels` trên cùng ảnh = **hai** ảnh tính tiền.
3.000 ảnh/giờ × 2 API × 24 giờ = 144.000 lượt/ngày, không phải 72.000. Không tránh được
nếu thật sự cần cả hai — nhưng phải đưa vào dự toán, và phải hỏi lại xem có cần kiểm duyệt
**mọi** ảnh hay chỉ ảnh do người dùng đăng.

**Bẫy tiền thứ hai:** "trích chữ trên bao bì" nghe giống Textract nhưng **không phải**.
Chữ trên bao bì là chữ rời rạc trong một cảnh, không phải tài liệu có cấu trúc — đó là
`Rekognition DetectText` ($0,0010/ảnh, tối đa 100 từ), không phải Textract. Chọn nhầm sang
`AnalyzeDocument` Forms là **$0,05/trang**, đắt gấp 50 lần cho một việc nó làm không tốt
bằng.

**Vì sao có SQS:** 3.000 ảnh/giờ không đều — ảnh đến theo lô khi nhà cung cấp đẩy hàng.
Không có hàng đợi thì burst sẽ sinh hàng nghìn Lambda gọi Rekognition đồng thời và đụng
**TPS quota** của account, trả về lỗi throttle mà retry chỉ làm nặng thêm.
`maximum_concurrency` trên event source mapping là cái van.

**Không cần** Step Functions ở đây: cả hai lời gọi đều đồng bộ, xong trong vài giây, không
có bước chờ nào. Thêm Step Functions là thêm **$25/triệu state transition** cho một luồng
không có gì để điều phối.

</details>

**8.** Vì sao gọi một dịch vụ AI đồng bộ từ sau API Gateway lại nguy hiểm hơn "chỉ là chậm"?
Nêu ba hậu quả cụ thể.

<details><summary>Đáp án</summary>

**Hậu quả 1 — client nhận 504 nhưng công việc vẫn chạy.** API Gateway cắt kết nối ở giây
**29**. Lambda phía sau **không** bị dừng: nó chạy tiếp tới hết timeout của chính nó (tối
đa 15 phút) và tính tiền đủ. Job Textract/Transcribe đã khởi động cũng chạy tiếp và tính
tiền đủ. Bạn trả tiền cho một kết quả không ai nhận.

**Hậu quả 2 — retry của client nhân đôi chi phí.** Client thấy 504 thì thử lại. Lần thử
thứ hai tạo một job **mới**, hoàn toàn độc lập với job thứ nhất vẫn đang chạy. Ba lần thử
= ba lần tính tiền cho cùng một tài liệu. Và không có dòng log nào nói "bạn vừa xử lý cùng
một file ba lần" — bạn chỉ thấy hoá đơn cao bất thường.

**Hậu quả 3 — không có chỗ nào giữ trạng thái.** Kết quả nằm trong response của một lời
gọi đã chết. Không có `jobId`, không có bản ghi trong DynamoDB, không có message trong
queue. Công việc đã làm xong nhưng **không thể lấy lại được** — bạn buộc phải làm lại từ
đầu.

Cách sửa là mẫu A ở [mục 14](#14-mẫu-kiến-trúc-bất-đồng-bộ--cái-khung-dùng-lại-được): API
trả `202 Accepted` kèm `jobId` **ngay lập tức**, ghi trạng thái vào DynamoDB, và cho client
hỏi lại bằng `GET /jobs/{id}`. Bức tường 29 giây không còn liên quan, retry trở thành
idempotent, và trạng thái nằm ở nơi sống lâu hơn một HTTP request.

</details>
