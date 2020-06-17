# Tổng quan về Cinder 
## 1 . Cinder là gì?
- Cinder là một Block Storage service trong OpenStack . Nó được thiết kế với khả năng lưu trữ dữ liệu mà người dùng cuối có thể sử dụng bỏi Project Compute (NOVA). Nó có thể được sử dụng thông qua các reference implementation (LVM) hoặc các plugin driver dành cho lưu trữ..
- Có thể hiểu ngắn gọn về Cinder như sau : Cinder là ảo hóa việc quản lý các thiết bị Block Storage và cung cấp cho người dùng một API đáp ứng được như cầu tự phục vụ cũng như yêu cầu tiêu thụ các tài nguyên đó mà không cần có quá nhiều kiến thức về lưu trữ.

## 2. Kiến trúc Cinder 

![Cinder 1](https://github.com/hocchudong/ghichep-OpenStack/blob/master/images/cinder-Architect.png?raw=true)

- Cinder-client : Người dùng sử dụng CLI/UI (Command line interface / User interface) để tạo request.
- Cinder-api : Chấp nhận và chỉ định đường đi cho các request.
- Cinder-scheduler : Lịch trình và định tuyến đường đi cho các request tới những volumes thích hợp.
- Cinder-volume : Quản lý thiết bị Block Storage.
- Driver : Chứa các mã back-end cụ thể để có thể liên lạc với các loại lưu trữ khác nhau. 
- Storage : Các thiết bị lưu trữ từ các nhà cung cấp khác nhau.
- SQL DB : Cung cấp một phương tiện dùng để back up dữ liệu từ Swift/Celp, etc,....

## 3. Volume API 
[Xem](https://docs.openstack.org/api-ref/block-storage/)

## 4. Cinder Driver

- Cinder driver maps các Cinder requests yêu cầu từ dòng lệnh lên external storage platform.
- Có trên 50 loại driver tuy nhiên chúng ta thường sử dụng nhất là LVM (Logical Volume Managerment).
- Để set một volume driver chúng ta dùng tham số `volume_driver` trong file `cinder.conf`

```sh
volume_driver = cinder.volume.drivers.lvm.LVMISCSIDriver
```

- LVM sẽ maps các thiết bị Block storage vật lý trên các thiết bị Block storage ảo cấp cao hơn.
- Cinder Volume được khởi tạo như Logical Volumes bởi LVM.
- Sử dụng giao thức iSCSI để kết nối các volumes tới các compute nodes.
- Không có nhà cung cấp cụ thể.

## 5. Cinder Workflow – Volume Creation
![Cinder 2](http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/WorkFlow.png)

1. Client tạo request thông qua REST API ( sử dụng CLI/UI)
2. Cinder-api process xác nhận yêu cầu, thông tin người dùng. Sau khi xác nhận , đặt tin nhắn vào hàng đợi AMQP để xử lý. 
3. Cinder-volume process lấy message ra khỏi hàng đợi, gửi message đến  cinder-scheduler  để  xác định   backend  cung cấp volume thích hợp
4. Cinder-scheduler process lấy message ra khỏi hàng đợi, tạo danh  sách  dựa trên cơ sở trạng thái hiện tại và tiêu chí  volume được yêu cầu (size, availability zone, volume type(bao gồm cả các thông số kỹ thuật bổ sung...))
5. Cinder-volume đọc message phản hồi từ cinder-scheduler ,  lặp lại  thông qua danh sách  bằng cách gọi phương thức  backend drivercho đến khi thành công.
6. Cinder driver tạo ra volume được yêu cầu thông qua tương tác  với  storage subsystem (phụ thuộc vào cấu hình và giao thức).
7. Cinder-volume process thu thập volume metadata, thông tin kết nối và gửi phản hhồi đến  hàng  đợi AMQP .
8. Cinder-api process đọc phản hồi từ hàng đợi và  phản hồi lại client.
9. Client nhận thông tin bao gồm trạng thái của yêu cầu, volume UUID,... .

## 6. Cinder & Nova Workflow – Volume Attach
![Cinder 3](http://www.sparkmycloud.com/blog/wp-content/uploads/2016/01/atta.png)

1. Client yêu cầu attach volume thông qua việc gọi Nova REST API  (sử dụng CLI/UI)
2. Nova-api process xác nhận yêu cầu, thông tin người dùng. Sau khi xác nhận ,  gọi Cinder API để lấy thông tin kết nối cho volume được chỉ định  .
3. cinder-api process xác nhận yêu cầu, thông tin người dùng. Sau khi các nhận,  gửi mesage cho volume manager  qua AMQP.
4. Cinder-volume đọc messaage từ hàng đợi. gọi Cinder drive tương ứng  với volume được attach .
5. Cinder drive chuẩn bị các yếu tố cần thiệết cho kết nối (Các bước cụ thể phụ thuộc vào giao thức lưu trữ được sử dụng)
6. cinder-volume process gửi thông tin phản hồi cho  cinder-api process thông qua hàng đợi AMOP.
7. cinder-api process đọc thông tin phản hồi từ cinder -volume từ hàng đợi , chuyển thông tin kết nối trong phản hồi RESTful cho người gọi Nova.
8. Nova tạo kết nối đến storage sử dụng thông tin được trả về.
9. Nova chuyển volume device/file tới hypervisor, sau đó gắn  volume device/file vào máy khách VM dưới dạng  block  device thực tế hoặc ảo hóa (tùy thuộc vào giao thức lưu trữ )

***Note:***

**iSCSI**

- Trong hệ thống mạng máy tính, iSCSI (viết tắt của internet Small Computer System Interface) dựa trên giao thức mạng internet (IP) để kết nối các cơ sở dữ liệu.
- Nói một cách đơn giản nhất, iSCSI sẽ giúp tạo 1 ổ cứng Local trong máy tính của bạn với mọi chức năng y như 1 ổ cứng gắn trong máy tính vậy. Chỉ khác ở chỗ dung 
lượng thực tế nằm trên NAS và do NAS quản lý.

**NFS**

- NFS là hệ thống cung cấp dịch vụ chia sẻ file phổ biến trong hệ thống mạng Linux và Unix.
- NFS cho phép các máy tính kết nối tới 1 phân vùng đĩa trên 1 máy từ xa giống như là local disk. Cho phép việc truyền file qua mạng được nhanh và trơn tru hơn.
- NFS sử dụng mô hình Client/Server. Trên server có các disk chứa các file hệ thống được chia sẻ và một số dịnh vụ chạy ngầm (daemon) phục vụ cho việc chia sẻ với Client.
- Cung cấp chức năng bảo mật file và quản lý lưu lượng sử dụng (file system quota).
- Các Client muốn sử dụng các file system được chia sẻ thì sử dụng giao thức NFS để mount các file đó về.

## 7. Cinder Status 
|Status|Mô tả|
|------|-----|
|Creating|Volume được tạo ra|
|Available|Volume ở trạng thái sẵn sàng để attach vào một instane|
|Attaching|Volume đang được gắn vào một instane|
|In-use|Volume đã được gắn thành công vào instane|
|Deleting|Volume đã được xóa thành công|
|Error|Đã xảy ra lỗi khi tạo Volume|
|Error deleting|Xảy ra lỗi khi xóa Volume|
|Backing-up|Volume đang được back up|
|Restore_backup|Trạng thái đang restore lại trạng thái khi back up|
|Error_restoring|Có lỗi xảy ra trong quá trình restore|
|Error_extending|Có lỗi xảy ra khi mở rộng Volume|

## 8. Cinder backup
- Cinder có thể back up một volume.
- Một bản back up là một bản sao lưu được lưu trữ vào ổ đĩa. Bản backup này được lưu trữ vào object storage.
- Backups có thể cho phép phục hồi từ :
    + Dữ liệu volume bị hư.
    + Storage failure.
    + Site failure (Cung cấp giải pháp backup an toàn).

## 9. Advanced Features.
- Snapshot :
  - Một snapshot là một bản sao của một thời điểm của dữ liệu chứa một volume.
  - Một snapshot sẽ cùng tồn tại trên storage backend như một volume đang hoạt động.
- Quota :
  - Admin sẽ đặt giới hạn cho volume, khả năng backup và snapshot tùy thuộc vào chính sách cài đặt.
- Volume transfer :
  - Chuyển một volume từ user này đến một user khác.
- Encryption :
  - Mã hóa được thực hiện bởi NOVA sử dụng dm-crypt , là một hệ thống con minh bạch mã hóa đĩa trong Linux kernel.
- Migration (Admin only):
  - Chuyển dữ liệu từ back-end hiện tại của volume đến một nơi mới.
  - Hai luồng chính phụ thuộc vào việc volume có được gắn vào hay không.
