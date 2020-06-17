# 1. Keystone là gì?

Keystone là OpenStack project cung cấp các dịch vụ Identity, Token, Catalog, Policy cho các project khác trong OpenStack. Nó triển khai Identity API của OpenStack.  Hai tính năng chính của keystone:
- User Management: keystone xác thực tài khoản người dùng và chỉ định xem người dùng có quyền được làm gì.
- Service Catalog: Cung cấp một danh mục các dịch vụ sẵn sàng cùng với các API endpoints để truy cập các dịch vụ đó.

# 2. Kiến trúc keystone

[![687474703a2f2f692e696d6775722e636f6d2f7742344b7943692e706e67.png](https://i.postimg.cc/FR5x89GV/687474703a2f2f692e696d6775722e636f6d2f7742344b7943692e706e67.png)](https://postimg.cc/NyDTrv3K)

- Identity: các Identity service cung cấp dịch vụ xác thực các thông tin chứng thực người dùng gửi tới, cung cấp dữ liệu về Users, Projects, Roles cũng như các metadata khác.
    + Groups là một nhóm người dùng. Có thể được gán trên domain của group hoặc trên project của group đấy.
    + Roles chỉ ra vai trò của người dùng trong project hoặc trong domain,... Mỗi user có vao trò khác nhau với từng project. 
- Token: xác nhận và quản lý các Tokens sử dụng cho việc xác thực các yêu cầu sau khi thông tin của các user/project đã được xác thực.
- Catalog: cung cấp endpoints của các dịch vụ sử dụng cho việc tìm kiếm và truy cập các dịch vụ.
- Policy: cung cấp cơ chế ủy quyền rule-based
- Resource
    + Trong keystone, projects là khái niệm trừu tượng, sử dụng bởi các dịch vụ khác trong OpenStack.
    + Projects có chứa các tài nguyên.
    + Dùng để cô lập tầm nhìn, tập hợp các project, user cho 1 tổ chức cụ thể.
    + 1 domain có thể bao gồm user, group, project....
    + Domain cho phép bạn phân chia các nguồn tài nguyên trong cloud vào các tổ chức cụ thể.
- Assignment
    + Thể hiện sự kết nối giữa một actor(user và user group) với một actor(domain, project) và một role.
    + Role assignment được cấp phát và thu hồi, và có thể được kế thừa giữa các user và group trên project của domains

Mỗi dịch vụ lại được cấu hình để sử dụng một backend cho phép keystone lưu trữ thông tin Identity như thông tin credentials, token, etc. Việc quy định mỗi dịch vụ sử dụng hệ thống backend nào được cấu hình trong file keystone.conf. 

- SQL Backend: cung cấp hệ thống backend bền vững để lưu trữ thông tin.
- LDAP Backend: LDAP là hệ thống lưu trữ các user và project trong các subtree tách biệt nhau.
- Multiple Backend: sử dụng kết hợp nhiều hệ thống Backend, trong đó SQL lưu trữ các service account (tài khoản của các dịch vụ như: nova glance, etc.), còn LDAP sử dụng lưu trữ thông tin người dùng, etc.

![arc](https://media.springernature.com/full/springer-static/image/art%3A10.1186%2Fs13174-018-0090-7/MediaObjects/13174_2018_90_Fig3_HTML.png)

# 3. Keystone workflow

[![687474703a2f2f692e696d6775722e636f6d2f75447a504c6e612e706e67.png](https://i.postimg.cc/jqvzQQVX/687474703a2f2f692e696d6775722e636f6d2f75447a504c6e612e706e67.png)](https://postimg.cc/6T22BZp2)

1. User gửi thông tin đến Keystone (Username và Password)
2. Keystone kiểm tra thông tin. Nếu đúng, nó sẽ gửi về user 1 token.
3. User gửi token và yêu cầu đến Nova.
4. Nova gửi token đến Keystone để kiểm tra token này có đúng không? có những quyền hạn gì. Keystone sẽ trả lời lại cho Nova.
5. Nếu token có quyền, Nova gửi token và yêu cầu image đến Glance.
6. Glance gửi token về Keystone để xác thực và kiểm tra xem user này có quyền với file image này không. Keystone sẽ trả lời đến Glance.
7. Kiểm tra ok, glance sẽ gửi image cho nova.
8. Nova gửi token, và yêu cầu về mạng đến Neutron.
9. Neutron gửi token đến Keystone. Keystone sẽ trả lời cho Neutron là user này có được phép hay không.
10. Neutron trả lời cho Nova.
11. Nova trả lời cho người dùng.

# 4. Token format 

- UUID
- PKI - PKIZ
- Fernet:
    + Sử dụng mã hóa đối xưng (Sử dụng chung key để mã hóa và giải mã).
    + Có kích thước khoảng 255 byte, không nén, lớn hơn UUID và nhỏ hơn PKI.
    + Chứa các thông tin cần thiết như userid, projectid, domainid, methods, expiresat,....Không chứa serivce catalog.
    + Không lưu token vào database.
    + Cần phải gửi lại keystone để xác nhận.
    + Cần phải phân phối khóa cho các khu vực khác nhau trong OpenStack.
    + Sử dụng cơ chế xoay khóa để tăng tính bảo mật.
    + Nhanh hơn 85% so với UUID và 89% so với PKI.