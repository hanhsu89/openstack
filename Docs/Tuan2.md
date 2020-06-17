# **Tuần 2: Tìm hiểu tổng quan Openstack, object storage vs block storage**
## 1. Tổng quan OpenStack.
### 1.1 OpenStack là gì?
- Hệ điều hành cloud (cloud OS).
- Miễn phí và mã nguồn mở hoàn toàn.
- Hỗ trợ public cloud và private cloud.
- Do NASA và rackspace khởi xướng, và được phát triển bởi cộng đồng
- Tương thích với EC2 của Amazon.
- Đối tượng sử dụng: 
    + Là những nhà cung cấp dịch vụ, các trung tâm dữ liệu, chính phủ, công ty đa quốc gia ... cần triển khai điện toán đám mây với quy mô lớn.
    + Các công ty cần tinh giản hạ tầng CNTT.

### 1.2 Thành phần
| service | project name  |
| ------ | ------ |
| Compute  | nova |
| Networking | Neutron|
| Object Storage | Swiff |
| Block Storage | Cinder |
| Dashboard | Horizon |
| Image service | Glance |
| Identity service | Keystone |
| Telemetry | cilometer |
| Orchestration | Heat |

#### **Mô hình điện toán**
![OP](https://dzone.com/storage/temp/1839453-openstack.png)
![OP1](https://camo.githubusercontent.com/bc18e1635965370feadce3e8dd327868a0ad7055/68747470733a2f2f64726976652e676f6f676c652e636f6d2f75633f69643d30427739366652767139494c506546706655336c61566d5934596b6b)

- Compute Infrastructure (Nova): chạy máy ảo, cấu hình mạng…
    + Là thành phần quản lý các máy ảo (Virtual Compute Instance) 
    + Tương tự dịch vụ EC2 của Aws
    + Được gọi bằng Openstack API hoặc EC2 API
    + Hỗ trợ nhiều công nghệ ảo hóa: Xen, KVM, QEMU, hyper-V …
- Storage Infrastructure (Swift): lưu dữ liệu, có thể mở rộng và chịu lỗi bằng cách sao lưu dữ liệu
    + Cung cấp dịch vụ lưu trưc file
    + Tương tự với dịch vụ S3 của Aws
    + Cung cấp khả năng mở rộng, dự phòng, sao lưu, phân tán
    + Tương thích với S3 API

- Storage Infrastructure (Cinder): 
    + Cung cấp thiết bị lưu trữ ảo cho các máy ảo của Openstack.
    + Tương tự như dịch vụ EBS của Amazon.
    + Có khả năng mở rộng, phân tán.

- Imaging Service (Glance): xử lí các file image của máy ảo
    + Dịch vụ lưu trữ và truy xuất ổ đĩa ảo (VDI)
    +  Hỗ trợ nhiều định dạnh (VHD, VMDK, OVF, ...)
    + Tính năng chính:
    
        -> Người quản trị tạo sẵn template để user có thể tạo máy ảo nhanh chóng.
    
        -> Người dùng có thể tạo máy ảo từ ổ đĩa ảo có sẵn.
        
        -> Sao lưu máy ảo có sẵn bằng tính năng Snapshot. 

- Networking (Neutron):  Cung cấp dịch vụ mạng (network as a Service) cho các dịch vụ khác của OP
    + Sử dụng kiến trúc “plug-in”: các plug-in được implement trên nhiều kiến trúc khác nhau, như Nicira NVP, Open vSwitch, linux bridge, Cisco...
    + Cho phép tùy biến, mở rộng.
    + Cho phép tạo private network.
    + Switch ảo, firewall, DHCP, VPN, load balancing…

- Dashboard (Horizon)
    + Ứng dụng web chạy trên nền apache.
    + Cung cấp giao diện tương tác cho administrator để quản lý các dịch vụ khác của Openstack.
    + Tương thích với EC2 API của amazon.

#### **[Share service]**
- Identity Service (Keystone)
    + Dịch vụ xác thực người dùng.
    + Hỗ trợ nhiều kiểu xác thực.
    + Phân quyền dựa trên tính năng role-base access control (RBAC).

- Telemetry service (Ceilometer)
    + Dịch vụ giám sát và thống kê.
    + Ví dụ: Thu thập thông tin về quá trình sử dụng để tính hóa đơn, xác định mức độ sử dụng hệ thống ...
- Orchestration Service (Heat)
    + Cung cấp các template cho những ứng dụng phổ biến.
    + Template sẽ mô tả cấu hình các thành phần compute, storage và networking để đáp ứng yêu  cầu của ứng dụng.
    + Kết hợp với Ceilometer để có thể “tự co dãn” tài nguyên.
    + Tương thích với AWS CloudFormation APIs.

### **1.2 Ưu điểm.**
- Tiết kiệm chi phí.
- Hiệu suất cao.
- Nền tảng mở.
- Mềm dẻo trong việc tương tác.
- Khả năng phát triển, mở rộng cao.

### **1.3 Nhược điểm**
- Độ ổn định chưa cao.
- Hỗ trợ đa ngôn ngữ chưa tốt.
- Chỉ có hỗ trợ kĩ thuật qua chat và email.

## **2. object storage vs block storage**
### **2.1 Storage type**
- Emphemaral storage
    + Nếu trong hệ thống Cloud không triển khai bất kỳ hình thức nào của Persistent Storage cho end-user sử dụng, các disk của VMs được tạo ra sẽ tồn tại dưới dạng Ephemeral storage, khi tiến hành xóa bỏ VMs (terminate VMs), các ephemeral disk này sẽ bị xóa theo.

- Presitent storage
    + Persistent storage được hiểu như đúng nghĩa đen của nó, là tài nguyên lưu trữ tồn tại độc lập, luôn luôn có sẵn mặc dù các instance có thể thay đổi, xóa bỏ,… Hiện nay, Cloud OpenStack tồn tại 2 loại persistent storage là: object storage và block storage.

### **2.2 Object Storage**
- Một ví dụ rõ ràng nhất của Object Storage là Amazon S3. Trong OpenStack, triển khai Object Storage sử dụng Swift, 1 trong 3 project cores đầu tiên của OpenStack (bên cạnh Nova và Glance). Người dùng có thể truy cập Object Storage thông qua RESR API. Trong trường hợp cần lưu trữ và quản lý một lượng dữ liệu lớn, Object storage là một lựa chọn hiệu quả. Ví dụ, trong OpenStack thay vì có thể lưu trữ các images (ví dụ file ảnh Ubuntu12.04, Windows7, …) trên File System, có thể sử dụng Object Storage – Swift để lưu trữ. OpenStack Object Storage cung cấp một hệ thống lưu trữ với độ sẵn sàng cao và dễ mở rộng.
- **1 trong những ưu điểm chính của object là khả năng phân phát yêu cầu cho các đối tượng trên 1 lượng lớn các máy chủ lưu trữ. Điều này cung cấp tính chất đáng tin cây, khả năng mở rộng lưu trữ cho 1 lượng lớn dữ kiệu với chi phí thấp.
Khi hệ thống đủ lớn, nó có thể chỉ ra 1 không gian tên duy nhất. Điều này nghĩa là 1 ứng dụng hoặc người dùng không cần thiết và không nên biết những hệ thống lưu trữ sẽ được sử dụng. Điều này sẽ giảm gánh nặng lên nhà điều hành, không giống như các file system, mà các nhà khai thác phải quản lý nhiều về dung lương lưu trữ, bời vì 1 hệ thống lưu trữ object cung cấp 1 không gian tên, không cần cắt nhỏ dữ liệu để lưu trữ ở các điểm khác nhau giúp cho việc quản lý dữ liệu trở nên dễ dàng hơn.**

### **2.3 Block Storage**
- Block storage (còn được gọi là volume storage) được gán vào các VMs dưới dạng các volumes. Trong OpenStack, Cinder là tên mã phần mềm triển khai Block storage.
Các Volume này là "persistent", nghĩa là các storage volume này có thể gán cho 1 instance, rồi gỡ bỏ (detached) và gán cho 1 instance khác mà vẫn giữ nguyên dữ liệu. Các block storage drivers cho phép instance truy cập trực tiếp đến phần cứng storage của thiết bị thật, việc này giúp tăng hiệu suất đọc/ghi IO.
- Lưu trữ dữ liệu có cấu trúc, được lưu vào các khối( mỗi khối 2^12 bit) có kích thước bằng nhau. Thông thường loại lưu trữ này phù hợp với các ứng dụng quản lý chặt chẽ lưu trữ dữ liệu có cấu trúc.
- Tương tự như Amazon Elastic Block Store (EBS).
- Các volume có vòng đời phụ thuộc vào thời gian sống của VM.

![OP3](https://i2.wp.com/blogit.edu.vn//wp-content/uploads/2014/06/12.jpg)

- OpenStack sử dụng Object Storage (Swift) để lưu trữ các hình ảnh và sử dụng Block storage (Cinder) để cấp volumes cho các instances.

### **2.4 So sánh Object storage vs Block Storage**
![ss](https://camo.githubusercontent.com/f31f904de7f0b4a960b2139ff943e4841c40ff00/687474703a2f2f692e696d6775722e636f6d2f674173364e526b2e706e67)

|             |Block Storage|Object Storage|
|-------------|----------------|-------------|
|Hình thức sử dụng |Thêm một persistent storage vào VM|lưu trữ các VM iamge , disk volume , snapshot VM,....|
|Hình thức truy cập|Một Block device có thể là một partition, formated, mounted (giống như là : /dev/vdc)|Thông qua RESTAPI|
|Có thể truy cập từ|Trong một VM|Bất kỳ đâu|
|Quản lý bởi| Cinder|Nova|
|Vấn đề còn tồn tại | Có thể được xóa bởi người dùng |   |
|Kích cỡ được xác định  bởi | Dựa theo các đặc điểm yêu cầu của người dùng | Số lượng luwau trữ và máy vật lý hiện có |

- Điểm khác biệc nữa với block storage, là block storage được truy cập trực tiếp bởi hệ điều hành trong khi Object storage sẽ bị ảnh hưởng lớn tới hiệu năng khi làm điều đó.

- Object storage là ý tưởng để giải quyết vấn đề về việc dữ liệu tăng. Dữ liệu ngày càng được tạo ra nhiều, hệ thống lưu trữ cũng cùng phát triển với tốc độ tương ứng. Điều j sẽ xảy ra nếu bạn cố gắng phát triển hệ thống lưu trữ khối lên vượt quá cả trăm tetabytes hoặc xa hơn là petabyte? Bạn có thể gặp vấn đề về độ bền, hạn chế khó khăn với cơ sở hạ tầng mà bạn đang có hoặc chi phí quản lý của bạn có thể lên tận nóc.

- Giải quyết vấn đề về quản lý dự phòng do việc mở rộng dung lượng lưu trữ ở quy mô này là điểm mạnh của Object storage. Đối với các Web tĩnh, dữ liệu dự phòng, hay các văn bản lưu trữ là trường hợp sử dụng tuyệt vời của object storage.

- Kiến trúc lưu trữ dựa trên đối tượng có thể được thu nhỏ và quản lý chỉ đơn giản bằng cách thêm vào các nodes. Tổ chức thành các không gian tên của dữ liệu, kết hợp với các chức năng mở rộng siêu dữ liệu (metadata) khiến cho việc này được dễ dàng sử dụng. 1 ưu điểm khác là nó đảm bảo được sự phục hồi dữ liệu và giảm thiểu chi phí thấp. Đối tượng dữ liệu được bảo về bằng cách lưu trữ thành nhiều bản sao của dữ liệu trên hệ thống phân phối rộng. Nếu 1 hoặc nhiều node lưu trữ bị gặp sự cố, dữ liệu vẫn có thể được sử dụng. Hầu hết các trường hợp người dùng cuối đều không bị ảnh hưởng về downtime. Hầu hết các trường hợp thì có ít nhất 3 bản sao được lưu trữ.

- Object storage tiết kiệm tiền với giá thành hệ thống với phần cứng rẻ hơn, giảm thời gian quản lý thông qua việc dễ dàng mở rông, cung cấp linh hoạt cho 1 số loại lưu trữ. Tuy nhiên thì nó không phải là câu trả lời cho mọi vấn đề về lưu trữ. Đôi khi sử dụng block storage là hợp lý hơn, object storage sẽ không đáp ứng được trong 1 vài trường hợp. Object cung cấp khả năng mở rộng không giới hạn đảm bảo tính sẵn sàng cao tuy nhiên là đối với các kiểu dữ liệu lâu bền, không đòi hỏi nhiều sự thay đổi về nội dung, và nhược điểm là nó sẽ không đảm bảo việc trả về các phiên bản mới nhất của dữ liệu

- Khối lượng công việc object vs block: Object làm việc rất tốt với các kiểu dữ liệu không cấu trúc, nơi mà các dữ liệu chỉ được đọc chứ không được ghi, web tĩnh, dữ liệu backup, lưu trữ ảnh, video, music... Tất cả được lưu trữ 1 cách tốt nhất với object storage. Cơ sở dữ liệu trong môi trường object storage lý tưởng có bộ dữ liệu không được tổ chức cấu trúc, nơi mà các dữ liệu sẽ không đòi hỏi 1 số lượng lớn các bài viết chỉnh sửa cập nhật. Phân phối các back-end storage với địa lý khác nhau cũng là 1 trường hợp sửu dụng tuyệt vời của việc lưu trữ object. Các ứng dụng lưu trữ object hiện nay như lưu trữ mạng và hỗ trợ metadata mở rộng cho phân phối hiệu quả và truy cập song song tới đối tượng, Nó khiến cho việc di chuyển các cụm backend storage quan các trung tâm dữ liệu khác nhau được thực hiện. Tuy nhiên đối với các dữ liệu được giao dịch thì không nên sử dụng vì nó có thể ảnh hưởng đến bản cập nhật dữ liệu cuối cùng.

- Các thiết bị lưu trữ block được truy cập như 1 volumes và trực tiếp đối với hệ điều hành (không qua metadata) nên nó có thể thực hiện việc lưu trữ tốt hơn trong 1 số trường hợp, ví dụ là đối với các dữ liệu có cấu trúc, có thể đọc ghi dữ liệu, tuy nhiên do việc không sử dụng metadata nên sẽ giảm hiệu suất đáng kể đối với hệ thống phân tán về mặt vật lý.

- Lưu trữ theo block được coi là phát triển hơn, nhưng filesystem thì dễ quản lý và thực thi. Nó cũng ít đắt hơn so với block-storage được sử dụng ở các máy nhà hoặc công ty nhỏ, còn block storage được sử dụng ở doanh nghiệp lớn hơn. Mỗi block thì được kiểm soát bởi 1 drive cứng của nó và quản lý máy chủ dựa vào hệ điều hành riêng.


Note:  Metadata - Là dạng dữ liệu miêu tả về dữ liệu. Bao quát tất cả các phương diện của dữ liệu như những thông tin về người khởi tạo, trường khởi tạo...
