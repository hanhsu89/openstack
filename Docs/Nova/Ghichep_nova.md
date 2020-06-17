# Ghi chép về NOVA (Compute service)

## 1. Chức năng
- Thực hiện quản lý vòng đời các máy ảo, cung cấp abstract layer tương tác với hypervisors được hỗ trợ: Hyper-V, VMware, XenServer, Xen via libvirt, KVM (libvirt/QEMU)
- Nova phân lại hypervisors thành 3 nhóm dựa trên số lượng các bài kiểm thử thành công với các driver tương tác với hypervisors: 
    - GROUP A: libvirt (qemu/KVM on x86). Các drivers này được hỗ trợ hoàn toàn. Các bài kiểm tra bao gồm: unit test và functional testing 
    - GROUP B: Hyper-V, VMware, XenServer. Các drivers này được hỗ trợ ở mức trung bình. Các bài test bao gồm: unit test và functional testing cung cấp bởi một hệ thống bên ngoài. 
    - GROUP C: baremetal, docker, Xen via libvirt, LXC via libvirt. Các driver này được thực hiện một số bài kiểm thử nhỏ và có thể hoạt động không ổn định. Việc sử dụng chúng là mạo hiểm. Các bài test bao gồm: unit tests và các bài test chức năng không public

## 2. Nova System Architecture
![compute service ](https://docs.openstack.org/nova/queens/_images/architecture.svg)

- DB: Sql database để lưu trữ dữ liệu
- API: Thành phần nhận HTTP request, chuyển đổi lệnh và giao tiếp với các thành phần khác thông qua hàng đợi  oslo.messaging hoặc HTTP.
- scheduler: Lấy các yêu cầu tạo máy ảo từ hàng đợi và xác định xem server compute nào sẽ được chọn để vận hành máy ảo.
- Network: Tiếp nhận yêu cầu về network từ hàng đợi và điều khiển mạng, thực hiện các tác vụ như thiết lập các giao diện bridging và thay đổi các luật của IPtables. 
- Compute: Thực hiện tác vụ quản lý vòng đời các máy ảo như: tạo và hủy các instance thông qua các hypervisor APIs. Ví dụ: 
    - XenAPI đối với XenServer/XCP
    - libvirt đối với KVM hoặc QEMU
    - VMwareAPI đối với VMware
- Conductor: Là module trung gian tương tác giữa nova-compute và cơ sở dữ liệu. Nó hủy tất cả các truy cập trự tiếp vào cơ sở dữ liệu tạo ra bởi nova-compute nhằm mục đích bảo mật, tránh trường hợp máy ảo bị xóa mà không có chủ ý của người dùng.

## 3. Mối quan hệ giữa các thành phần 
[![a.png](https://i.postimg.cc/1zKT5tNc/a.png)](https://postimg.cc/nsCT3n2C)

[![b.gif](https://i.postimg.cc/vBN0c689/b.gif)](https://postimg.cc/DJ1rN0Xy)

- Mô tả ngắn gọn về Nova:
    - End users (DevOps, Dev, hoặc có thể là các thành phần khác trong OpenStack) sẽ "nói chuyện" với nova-api để tương tác với OpenStack Nova
    - Các OpenStack Nova daemons trao đổi thông tin qua queue (lưu trữ các actions) và database (infomation) để thực hiện các API requests
    - OpenStack Glance giao tiếp với OpenStack Nova interfaces thông qua Glance API
    - Nova-networking vẫn được sử dụng trong một số use cases. Người dùng có thể lựa chọn nova-networking hoặc Neutron.

## 4. Reference

- [https://www.oreilly.com/library/view/deploying-openstack/9781449311223/ch04.html](https://www.oreilly.com/library/view/deploying-openstack/9781449311223/ch04.html)