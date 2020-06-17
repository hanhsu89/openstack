# **Tuần 1: Tìm hiểu về MySQL, RabbitMQ, Memcache, HAProxy, Keepalive**
## **1. MySQL & RabbitMQ**
### **1.1 Tổng quan:**
- Mọi trạng thái trong OpenStack đều được thực hiện quan message system và database, và tất cả các thành phần khác đều là không trạng thái (ngoại trừ glance).
- Hệ thống hàng đợi cho phép các thành phần giao tiếp với nhau, cơ sở dữ liệu giữ trạng thái cluster. Cả 2 đều tham gia vào mọi yêu cầu của người dùng, cho dù đó là hiển thị  ds các instance hoặc tạo 1 VM mới.
- Default messaging: RabbitMQ
    + Là một message broker ( message-oriented middleware).
    + Sử dụng giao thức AMQP - Advanced Message Queue Protocol (Đây là giao thức phổ biến, thực tế rabbitmq hỗ trợ nhiều giao thức).
    + Ngôn ngữ Erlang.
    + Là một phương tiện trung gian để giao tiếp giữa nhiều thành phần trong một hệ thống lớn (OpenStack, ELK, ...)
- Default database: MySQL (hoặc bất kì CSDL nào support SQLAlchemy đều có thể sử dụng: mariaDB…)
    + MySQL là một hệ thống quản trị cơ sở dữ liệu mã nguồn mở.
    + Hoạt động theo mô hình client-server.

### **1.2 Cách message và database hoạt động trên Openstack:**

*Ví dụ mô tả flow khi người dùng yêu cầu: provisioning an instance:*

- Người dùng gửi yêu cầu của mình tới Openstack bằng cách tương tác với nova-api. Nova-api xử lý yêu cầu tạo instance bằng các gọi hàm creat_instance từ nova-compute api.
Hoạt động của hàm:
    + Xác minh đầu vào của người dùng ( check có yêu cầu vm image, flavor, networks k, nếu không thì set mặc định)
    + Kiểm tra yêu cầu đối với hạn ngạch người dùng.
    + Sau khi xác thực, tạo entry cho instance trong Openstack db (create_db_entry_for_new_instance)
    + Gọi _schedule_run_instance  để chuyển yêu cầu tới nova-scheduler thông qua message queue sử dụng AMQP protocol. Hàm kết thúc bằng việc thực sự gửi message tới AMQP với lời gọi hàm scheduler_rpcapi.run_instance.
- Scheduler nhận được thông báo với các thông số yêu cầu, nó cố gằng tìm một máy chủ phù hợp để sinh ra instance.
- Sau khi tính toán, chọn được máy chủ thích hợp để tạo instance, scheduler gọi hàm cast_to_compute_host:
    + update hot entry cho instance trong nova database ( host = host đã được compute để tạo instance đó)
    + Gửi message qua AMQP cho nova-compute trên host cụ thể này để chạy instance(gồm UUID của instance cần chạy và tiếp đó chạy run_instance.
- Tiếp, nova-compute trên host đã chọn sẽ gọi phương thức _run_instance, lấy thông số instance từ db ( dựa vào UUID) và khởi chạy 1 instance với các tham số thích hợp. Trong quá trình tạo instance, nova-compute cũng giao tiếp qua AMQP với nova-netwok để thiết lập tất cả các mạng. Ở các giai đoạn khác nhau, trạng thái của VM được lưu vào nova db, sử dụng hàm _instance_update.

### **1.3 MySQL cluster**
- MySQL cluster dựa trên một công cụ lưu trữ đặc biệt gọi là NDB (Network DataBase). NDB là một cụm các máy chủ được gọi là các "data nodes", được quản lý bởi các "management nodes". Các dữ liệu được phân vùng và sao chép giữa các data node và có ít nhất 2 bản sao lưu cho cùng 1 phần dữ liệu nhất định. Tất cả các bản sao được đảm bảo nằm trên các data node khác nhau. Ở đầu mỗi data node, một cụm các MySQL server chạy được cấu hình với bộ lưu trữ NDB ở backend. Mỗi mysql processes đều có khả năng read/write và chúng có thể được cân bằng tải để đạt được hiệu quả và tính sẵn sàng cao.
- Trên Master:
    + Binary event được ghi vào binary log, lưu trữ trong data_dir. 
    + Master mở 1 Dump_Thread và gửi binlog tới cho I/O_Thread mỗi khi I/O_Thread từ Slave yêu cầu dữ liệu.
- Trên Slave:
    + Trên mỗi Slave sẽ mở một I/O_Thread kết nối tới Master .
    + Sau khi Dump_Thread gửi binlog tới I/O_Thead, I/O_Thread sẽ có nhiệm vụ đọc binlog này và ghi vào relaylog.
    + Đồng thời trên Slave sẽ mở một SQL_Thread, SQL_Thread có nhiệm vụ đọc các event từ relaylog và apply các event đó vào Slave => quá trình replication hoàn thành.

[![image2.jpg](https://i.postimg.cc/9MbWvH4k/image2.jpg)](https://postimg.cc/9R457SYP)

**NOTE: 1 số config cơ bản cần chú ý:** [Xem](https://docs.google.com/document/d/1MAUM1wKD_5G6WVUCdyxcxc93ABcKCzffKOvl55msZZw/edit?usp=sharing)

### **1.4 RabbitMQ cluster**
- Rabbitm:
    + Producer: thực hiện quá trình gửi bản tin lên RabbitMQ server
    + Exchange: thực hiện nhiệm vụ phân phối bản tin, có 3 kiểu phân phối bản tin direct, topic, fanout
    + Queues: có nhiệm vụ lưu trữ bản tin được gửi lên
    + Consumber: thực hiện việc lấy các bản tin từ queue về
![rb](https://www.cloudamqp.com/img/blog/rabbitmq-beginners-updated.png)

- Trong rabbitmq, một cluster là một nhóm các erlang node làm việc cùng với nhau. Mỗi erlang node có một rabbitmq application hoạt động và cùng chia sẻ tài nguyên: user, vhost, queue, exchange…
- Điều kiện để thiết lập clustering:
    + Tất cả các node phải cùng erlang version và rabbitmq version
    + Các node liên kết qua LAN network
    + Tất cả các node chia sẻ cùng một erlang cookie

*Trong mô hình cluster, các node sử dụng phương thức trao đổi giữa của erlang. Khi sử dụng phương thức này, hai erlang node chỉ nói chuyện được với nhau khi có cùng erlang cookie. Erlang cookie chỉ là một chuỗi ký tự. Khi startup một rabbitmq server lần đầu tiên, mặc định một erlang cookie ngẫu nhiên được sinh ra nằm trong /var/lib/rabbitmq/.erlang.cookie*

## **2. Memcache**
- Memcached cũng là 1 bộ nhớ đệm, nó là 1 service độc lập như mysql.
- Memcached cung cấp khả năng lưu trữ đối tượng bất kỳ vào trong RAM.
- Memcached là một NoSQL được thiết kế với hiệu năng cao, hoạt động theo phương thức distrubuted memory object caching.
- Memcached được tích hợp để giảm tải database cho ứng dụng, website và tăng tốc độ truy cập của website. 

## **3. HAProxy & Keepalived**

### 3.1 HAProxy

- Các thuật toán cân bằng tải:
    + roundrobin: các request sẽ được chuyển đến server theo lượt. Đây là thuật toán mặc định được sử dụng cho HAProxy
    ![HA](https://camo.githubusercontent.com/9cd3913a220f90880e219322f4b8a018738474b9/687474703a2f2f69313336332e70686f746f6275636b65742e636f6d2f616c62756d732f723731342f486f616e674c6f7665397a2f6c756f6e672d686170726f78795f7a7073796f6f37747967612e706e67)
    + leastconn: các request sẽ được chuyển đến server nào có ít kết nối đến nó nhất
    + source: các request được chuyển đến server bằng các hash của IP người dùng. Phương pháp này giúp người dùng đảm bảo luôn kết nối tới một server

- Cấu hình của HAProxy thường được tạo từ 4 thành phần bao gồm global, defaults, frontend, backend. 4 thành phần sẽ định nghĩa cách HAProxy nhận, xử lý các request, điều phối các request tới các Backend phía sau.

```
global
    # Các thiết lập tổng quan

defaults
    # Các thiết lập mặc định

frontend
    # Thiết lập điều phối các request

backend
    # Định nghĩa các server xử lý request
```


- **Global**
    + **maxconn**: Chỉ định giới hạn số kết nối mà HAProxy có thể thiết lập. Sử dụng với mục đích bảo vệ load balancer khởi vấn đề tràn ram sử dụng.
    + **log**: Bảo đảm các cảnh báo phát sinh tại HAProxy trong quá trình khởi động, vận hành sẽ được gửi tới syslog
    + **stats socket**: Định nghĩa runtime api, có thể sử dụng để disable server hoặc health checks, thấy đổi load balancing weights của server. 
    + **user / group**: chỉ định quyền sử dụng để khởi tạo tiến trình HAProxy. Linux yêu cầu xử lý bằng quyền root cho nhưng port nhở hơn 1024. Nếu không định nghĩa user và group, HAProxy sẽ tự động sử dụng quyền root khi thực thi tiến trình.
- **Defaults**
    + **timeout connect**: chỉ định thời gian HAProxy đợi thiết lập kết nối TCP tới backend server. Hậu tố s tại 10s thể hiện khoảng thời gian 10 giây, nếu bạn không có hậu tố s, khoảng thời gian sẽ tính bằng milisecond. 
    + **timeout server**: chỉ định thời gian chờ kết nối tới backend server.
    + Khi thiết lập **mode tcp**t hời gian **timeout server** phải bằng timeout client.
    + **log global:** Chỉ định ‘frontend’ sẽ sử dụng log settings mặc định (trong mục global).
    + **mode:** Thiết lập mode định nghĩa HAProxy sẽ sử dụng TCP proxy hay HTTP proxy. Cấu hình sẽ áp dụng với toàn frontend và backend khi bạn chỉ mong muốn sử dụng 1 mode mặc định trên toàn backend (Có thể thiết lập lại giá trị tại backend)
    + **maxconn:** Thiết lập chỉ định số kết nối tối đa, mặc định bằng 2000.
    + **option httplog:** Bổ sung format log dành riêng cho các request http bao gồm (connection timers, session status, connections numbers, header v.v). Nếu sử dụng cấu hình mặc định các tham số sẽ chỉ bao gồm địa chỉ nguồn và địa chỉ đích.
    + **option http-server-close:** Khi sử dụng kết nối dạng keep-alive, tùy chọn cho phép sử dụng lại các đường ống kết nối tới máy chủ (có thể kết nối đã đóng) nhưng đường ống kết nối vẫn còn tồn tại, thiết lập sẽ giảm độ trễ khi mở lại kết nối từ phía client tới server.
    + **option dontlognull:** Bỏ qua các log format không chứa dữ liệu
    + **option forwardfor:** Sử dụng khi mong muốn backend server nhận được IP thực của người dùng kết nối tới. Mặc định backend server sẽ chỉ nhận được IP của HAProxy khi nhận được request. Header của request sẽ bổ sung thêm trường X-Forwarded-For khi sử dụng tùy chọn.
    + **option redispatch:** Trong mode HTTP, khi sử dụng kỹ thuật stick session, client sẽ luôn kết nối tới 1 backend server duy nhất, tuy nhiên khi backend server xảy ra sự cố, có thể client không thể kết nối tới backend server khác (Trong bài toán load balancer). Sử dụng kỹ thuật cho phép HAProxy phá vỡ kết nối giữa client với backend server đã xảy ra sự cố. Đồng thời, client có thể khôi phục lại kết nối tới backend server ban đầu khi dịch vụ tại backend server đó trở lại hoạt động bình thường.
    + **retries:** Số lần thử kết nối lại backend server trước khi HAProxy đánh giá backend server xảy ra sự cố.
    + **timeout check:** Kiểm tra thời gian đóng kết nối (chỉ khi kết nối đã được thiết lập)
    + **timeout http-request: **Thời gian chờ trước khi đóng kết nối HTTP
    + **timeout queue:** Khi số lượng kết nối giữa client và haproxy đạt tối đã (maxconn), các kết nối tiếp sẽ đưa vào hàng đợi. Tùy chọn sẽ làm sạch hàng chờ kết nối.
- **Frontend** 
    + **bind**: IP và Port HAProxy sẽ lắng nghe để mở kết nối. IP có thể bind tất cả địa chỉ sẵn có hoặc chỉ 1 địa chỉ duy nhất, port có thể là một port hoặc nhiều port (1 khoảng hoặc 1 list).
    + **http-request redirect** Phản hỏi tới client với đường dẫn khác. Ứng dụng khi client sử dụng http và phản hồi từ HAProxy là https, điều hướng người dùng sang giao thức https
    + **use_backend:** Chỉ định backend sẽ xử lý request nếu thỏa mãn điều kiện (Khi sử dụng ACL)
    + **default_backend:** Backend mặc định sẽ xử lý request (Nếu request không thỏa mẵn bất kỳ điều hướng nào)
- **Backend:**
    + **balance:** Kiểm soát cách HAProxy nhận, điều phối request tới các backend server. Đây chính là các thuật toán cân bằng tải.
    + **cookie:** Sử dụng cookie-based. Cấu hình sẽ khiến HAProxy gửi cookie tên SERVERUSED tới client, liên kết backend server với client. Từ đó các request xuất phát từ client sẽ tiến tục nói chuyện với server chỉ định. Cần bổ sung thêm tùy chọn cookie trên server line
    + **option httpchk:** Với tùy chọn, HAProxy sẽ sử dụng health check dạng HTTP (Layer 7) thay vì kiếm trả kết nối dạng TCP (Layer 4). Và khi server không phản hồi request http, HAProxy sẽ thực hiện TCP check tới IP Port. Health check sẽ tự động loại bỏ các backend server lỗi, khi không có backend server sẵn sàng xử lý request, HAProxy sẽ trả lại phản hồi 500 Server Error. Mặc đinh HTTP check sẽ kiểm tra root path / (Có thể thay đổi). Và nếu phản hồi health check là 2xx, 3xx sẽ được coi là thành công.
    + **default-server:** Bổ sung tùy chọn cho bất kỳ backend server thuộc backend section (VD: health checks, max connections, v.v). Điều này kiến cấu hình dễ dàng hơn khi đọc.
    + **server:** Tùy chọn quan trọng nhất trong backend section. Tùy chọn đi kèm bao gồm tên, IP:Port. Có thể dùng domain thay cho IP. Theo ví dụ, tùy chọn maxconn 20 sẽ được bổ sung vào tất cả backend server, tức mỗi server sẽ chỉ phục vụ 20 kết nối đồng thời
- **Listen:** listen là sự kết hợp của cả 2 mục frontend và backend. Vì listen kết hợp cả 2 tính năng backend frontend, nên bạn có thể sử dụng listen thay thế các các mục backend và frontend.
    + **inter:** khoảng thời gian giữa hai lần check liên tiếp.
    + **rise:** Số lần kiểm tra backend server thành công trước khi HAProxy đánh giá nó đang hoạt động bình thường và bắt đầu điều hướng request tới
    + **fall:** Số lần kiểm tra backend server bị tính là thất bại trước khi HAProxy đánh giá nó xảy ra sự cố và không điều hướng request tới.

### 3.2 Keepalived

Hoạt động của keepalived  failover ip:

- Keepalived sẽ gom nhóm các máy chủ dịch vụ  nào tham  gia c ụm HA , khởi tạo 1 virtual server đại diện cho nhóm thiết bị đó với 1 Virtual IP (VIP) + 1 địa chỉ MAC vật lý của máy chỉ dịc vụ nào đang giữ VIP đó.
- Tại mỗi thời điểm chỉ có chỉ có 1 server giữ địa chỉ MAC này tương ứng với VIP.
- Các server chung VIP liên lạc với nhau bằng địa chỉ multicast 224.0.0.18 bằng giao thức VRRP.
- Các server sẽ có độ ưu tiên (priority) trong khoảng từ 1 - 254. Server có độ ưu tiên cao nhất sẽ là MASTER/ACTIVE và các server còn lại sẽ là SLAVE/BACKUP.
- Failover được xử lý bằng giao thức VRRP. Khi bắt đầu dịch vụ, các server tham gia vào 1 nhóm multicast, gửi/nhận các gói tin quảng bá VRRP. Các server quảng bá độ ưu tiên của mình để chọn 1 MASTER. Sau đó, MASTER sẽ chịu trách nhiệm gửi gói tin quảng bá VRRP cho nhóm multicast.
- Nếu MASTER chết (các SLAVE không nhận được gói tin VRRP trong sau 1 khoảng TG nhất định), các server còn lại sẽ bầu ra MASTER mới sử dụng VIP.