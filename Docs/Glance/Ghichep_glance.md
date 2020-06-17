# Tổng quan Image service (glance) project
## 1. Glance là gì?
- Glane (Image Service) là image service cung cấp khả năng discovering, registering (đăng ký), retrieving (thu thập) các image cho virtual machine. OpenStack Glance là central repository cho virtual image.
- Glance cung cấp RESTful API cho phép querying VM image metadata cũng như thu thập các actual image.
- VM image có sẵn thông qua Glance có thể stored trong nhiều vị trí khác nhau, từ file system đến object storage system như OpenStack Swift OpenStack
- Trong Glance, images được lưu trữ như template, được sử dụng để launching new instances. Glance được thiết kế để trở thành một service độc lập đối với các user cần tổ chức large virtual disk images. Glance cung cấp giải pháp end-to-end cho cloud disk image management. Nó cũng có thể lấy các bản snapshots từ running instance cho việc backing up VM và các states của VM.

## 2. Glance Components

Glance có các components:

- glance-api: chấp nhận API calls cho việc tìm kiếm, lấy và lưu trữ image.
- glance-registry: thực hiện lưu trữ, xử lý và lấy thông tin metadata của image.
- database: lưu trữ metadata của image.
- storage repository: tích hợp với nhiều thanh phần của OpenStack như file systems, Amazon S3 và HTTP cho image storages.

<img src="http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/Untitled-drawing2.png" />

Glance chấp nhận API request cho image từ end-users hoặc Nova components và có thể stores nó trong object storage service,swift hoặc storage repository khác.

Image server hỗ trợ các back-end stores:

- File system  
OpenStack Image server lưu trữ virtual machine images trong file system back end là default.Đây là back end đơn giản lưu trữ image files trong local file system.    
- Object Storage  
Là hệ thống lưu trữ do OpenStack Swift cung cấp - dịch vụ lưu trữ có tính sẵn sàng cao , lưu trữ các image dưới dạng các object.   
- Block Storage  
Hệ thống lưu trữ có tính sẵn sàng cao do OpenStack Cinder cung cấp, lưu trữ các image dưới dạng block.  
- VMware  
ESX/ESXi or vCenter Server target system.  
- S3  
The Amazon S3 service.  
- HTTP  
OpenStack Image service có thể đọc virtual machine images mà có sẵn trên Internet sử dụng HTTP. Đây là store chỉ đọc.
- RADOS Block Device (RBD)  
Stores images trong Ceph storage cluster sử dụng Ceph’s RBD interface.  
- Sheepdog  
A distributed storage system dành cho QEMU/KVM.  
- GridFS  
Stores images sử dụng MongoDB. 

## 3. Glance Architecture

<img src="https://docs.openstack.org/glance/pike/_images/architecture.png" />

<img src="http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/Untitled-drawing11.png" />

- Glance có cấu trúc theo mô hình client-server và cung cấp RESTful API mà thông qua đó các yêu cầu được gửi đến server để thực hiện. Yêu cầu từ các client được chấp nhận thông qua RESTful API và chờ keystone xác thực.
- Glance Domain controller thực hiện quản lý tất cả các hoạt động bên trong. Các hoạt động được chia ra thành các tầng khác nhau. Mỗi tầng thực hiện một chức năng riêng biệt.
- Glane store là lớp giao tiếp giữa glane và storage back end ở ngoài glane hoặc local filesystem và nó cung cấp giao diện thống nhất để truy cập. Glane sử dụng SQL central Database để truy cập cho tất cả các thành phần trong hệ thống.
- Glance bao gồm một vài thành phần sau:
    - Client: Bất kỳ ứng dụng nào sử dụng Glance server đều được gọi là client.
    - REST API: dùng để gọi đến các chức năng của Glance thông qua REST.
    - Database Abstraction Layer (DAL): một API để thống nhất giao tiếp giữa Glance và database.
    - Glance Domain Controller: là middleware thực hiện các chức năng chính của Glance là: authorization, notifications, policies, database connections.
    - Glance Store: tổ chức các tác động giữa Glance và lưu trữ dữ liệu khác.
    - Registry Layer: Tùy chọn tổ chức một lớp trao đổi thông tin an toàn giữa các miền và các DAL bằng cách sử dụng một dịch vụ riêng biệt.

## 4. Glance Formats
- Khi chúng ta upload image cho glance, chúng ta cần chỉ định format của Virtual machine images. Glance hỗ trợ nhiều format khác nhau như Disk Formats và Container Formats.
- Virtual disk là tương tự physical server’s boot driver.
- Các loại virtualization khác nhau hỗ trợ disk formats khác nhau.

### 4.1 Disk formats 

Các định dạng trên đĩa (Disk Formats) của một image máy ảo là định dạng của hình ảnh đĩa cơ bản. Sau đây là các định dạng đĩa được hỗ trợ bởi OpenStack Glance.

| Định dạng | Mô tả |
|:----------------:|:--------:|
| raw | Định dạng có cấu trúc |
| vhd | disk của máy ảo VMware, Xen... |
| vmdk | Hỗ trợ bởi nhiều máy ảo phổ biến |
| vdi | Hỗ trợ bởi VirtualBox,QEMU |
| iso | Định dạng lưu trữ dữ liệu của đĩa quang học |
| qcow2 | Hỗ trợ bởi QEMU  (QEMU image format, native format for KVM and QEMU . Support advanced functions )|
| aki | Amazon kernel image |
| ari | Amazon ramdisk image |
| ami | Amazon machine image |

### 4.2 .Container Formats

OpenStack Glance cũng hỗ trợ các khái niệm về container format. Mô tả các định dạng tệp tin và chứa thêm các metadata về máy ảo.


| Định dạng | Mô tả |
|:----------------:|:--------:|
| bare | Không lưu trữ hoặc đóng gói siêu dữ liệu |
| ovf | Định dạng OVF |
| aki | Amazon kernel image |
| ari | Amazon ramdisk image |
| ami | Amazon machine image |
| ova | tập lưu trữ OVA tar |
| docker | Docker tar archive of the container filesystem |

## 5. Glance Status Flow
- Glane Status Flow cho chúng ta thấy tình trạng của Image trong khi chúng ta tải lên. Khi chúng ta khởi tại một image, bước đầu tiên là queuing. Image sẽ được sắp xếp vào một hàng đợi trong một thời gian ngắn để định danh (hàng đợi này dành cho image) và sẵn sàng được upload. Sau khi kết thúc thời gian queuing thì image sẽ được upload đến "Saving" , tuy nhiên ở đây không phải image nào cũng được tải lên hoàn toàn. Những Image nào được tải lên hoàn toàn sẽ trong trạng thái "Active". Khi upload không thành công nó sẽ đến trạng thái "killed" hoặc "deleted" . Chúng ta có thể tắt và tái kích hoạt một Image đang "Active" hoàn toàn bằng một lệnh.
- Diagram bên dưới show status flow của glance:

<img src="http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/Untitled-drawing1.jpg" />

- Các trạng thái:
  - **queued**: Bộ nhận diện image đã được dành riêng cho một image trong registry Glance. Không có dữ liệu nào trong image được tải lên Glance và kích thước image không rõ ràng sẽ được đặt thành 0 khi tạo.
  - **saving**: Biểu thị rằng dữ liệu của image đang được upload lên glance. Khi một image đăng ký với một call đến POST/image và có một x-image-meta-location vị trí tiêu đề hiện tại, image đó sẽ không bao giờ được trong tình trạng saving (dữ liệu Image đã có sẵn ở vị trí khác).
  - **active**:  Biểu thị một image đó là hoàn toàn có sẵn trong Glane. Điều này xảy ra khi các dữ liệu image được tải lên hoặc kích thước image được rõ ràng để thiết lập được tạo.
  - **deactivated**: Biểu thị rằng quyền truy cập vào Image không được phép truy cập từ bất kỳ ai cả admin-user.
  - **killed**: Biểu thị một lỗi xảy ra trong quá trình truyền tải dữ liệu của một image, và image là không thể đọc được.
  - **deleted**: Trong Glane đã giữ lại các thông tin về image, nhưng không còn có sẵn để sử dụng. Một image trong trạng thái này sẽ được gỡ bỏ tự động vào một ngày sau đó.
- **Deactivating and Reactivating an image**: Có thể hủy kích hoạt (tạm thời)  1 image, sau đó  kích hoạt lại nó hoặc xóa đi nếu nó gây hại đến hệ thống. Trong khi thực hiện cập nhật image, ta có thể muốn ẩn nó khỏi hệ thống sau đó kích hoạt lại để người dùng có thể khởi tạo  máy ảo.

## 6. Glance Configuration Files

Các file cấu hình glance nằm trong thư mục /etc/glance. Các file quan trọng:
- `Glance-api.conf` : Configuration file for image service API.
- `Glance-registry.conf` : Configuration file for glance image registry which stores metadata about images.
- `glance-api-paste.ini`: Cấu hình cho các API middleware pipeline của Image service
- `glance-manage.conf`: Là tệp cấu hình ghi chép tùy chỉnh. Các tùy chọn thiết lập trong tệp `glance-manage.conf` sẽ ghi đè lên các section cùng tên thiết lập trong các tệp glance-registry.conf và glance-api.conf. Tương tự như vậy, các tùy chọn thiết lập trong tệp glance-api.conf sẽ ghi đè lên các tùy chọn thiết lập trong tệp `glance-registry.conf`
- `glance-registry-paste.ini`: Tệp cấu hình middle pipeline cho các registry của Image service.
- `glance-scrubber.conf` : Tiện ích được xử dụng cho quá trình dọn sạch các image trong status "deleted". Multiple glance-scrubber có thể chạy trong deployment đơn giản, nhưng chỉ có 1 scrubber được thiết lập để dọn dẹp trong `scrubber.conf` file. Clean-up scrubber phối hợp với glance scrubbers khác bởi việc duy trì một queue chính của image mà cần xóa. `glance-scrubber.conf` file chỉ định cấu hình các giá trị quan trọng như time giữa các lần runs, thời gian chờ của các image trước khi bị xóa. Glance-scrubber có thể run theo định kì hoặc như một daemon chạy trong một khoảng thời gian dài.  
- `policy.json` : File tùy chọn được thêm vào để điều khiển truy cập áp dụng với image service. Trong file này ta có thể định nghĩa các roles và policies. Nó là tính năng bảo mật trong OpenStack Glance.  

## 7. Image and Instance
- Khi image được lưu trữ như các mẫu. Image service điều khiểu lưu trữ và quản lý image. Instance là những máy ảo độc lập chạy trên các compute node, compute node quản lý các instance. Người dùng có thể khởi động với số lượng bất kỳ các máy ảo cùng một image. Mỗi lần chạy một máy ảo thì được thực hiện bằng cách sao chép từ base image, bất kỳ sửa đổi nào trên instance không ảnh hưởng đển các base image. Chúng ta có thể snaphost một instance đang chạy và có thể chạy chúng như một instance khác.
- Khi chạy một instance chúng ta cần xác định các flavor. Đó là đại diện cho tài nguyên ảo. Flavor định xác định bao nhiêu CPU ảo cho một Instance cần có và số lượng RAM sẵn có cho nó, và kích thước của nó trong bộ nhớ tạm của mình. OpenStack cung cấp một thiết lập flavor được xác định từ trước, chúng ta có thể chỉnh sửa các flavor riêng của chúng ta. Sơ đồ dưới đây cho biết tình trạng của hệ thống trước khi lauching an instance. Các image store có số lượng image được xác định trước, compute node chứa CPU có sẵn, bộ nhớ và tài nguyên local disk và cinder-volume chứa số lượng đã được xác định từ trước .

<img src="http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/Untitled-drawing2.jpg" />

- Trước khi chạy một instance chọn một image, flavor và bất kỳ thuôc tính tùy chọn nào . Chọn flavor cung cấp một root volume, nhãn (lable) là "vda" và một bổ sung vào bộ nhớ tạm thời dán nhãn là "vdb" và cinder-volume được ánh xạ tới ổ đĩa thứ 3 gọi là "vdc".

<img src="http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/new.jpg" />