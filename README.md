# Hướng dẫn cấu hình Keepalived thực hiện IP Failover cho OpenVPN Server
### Keepalived là gì ?

Chương trình keepalived cho phép nhiều máy tính cùng chia sẻ một địa chỉ IP ảo với nhau theo mô hình Active – Passive (ta có thể cấu hình thêm một chút để chuyển thành mô hình Active – Active nâng cao). Khi người dùng cần truy cập vào dịch vụ, người dùng chỉ cần truy cập vào địa chỉ IP ảo dùng chung này thay vì phải truy cập vào những địa chỉ IP thật của các thiết bị kia.
Cơ chế VRRP trong mô hình HA Keepalived
Như đã nói ở trên, các router/server vật lý dùng chung VIP sẽ có 2 trạng thái là MASTER/ACTIVE và BACKUP/SLAVE. Cơ chế failover được xử lý bởi giao thức VRRP, khi khởi động dịch vụ, toàn bộ các server dùng chung VIP sẽ gia nhập vào một nhóm multicast. Nhóm multicast này dùng để gởi /nhận các gói tin quảng bá VRRP. Các router/server sẽ quảng bá độ ưu tiên (priority) của mình, server với độ ưu tiên cao nhất sẽ được chọn làm MASTER. Một khi nhóm đã có 1 MASTER thì MASTER này sẽ chịu trách nhiệm gửi các gói tin quảng bá VRRP định kỳ cho nhóm multicast.
Nếu vì một sự cố gì đó mà các server BACKUP không nhận được các gói tin quảng bá từ MASTER trong một khoảng thời gian nhất định thì cả nhóm sẽ bầu ra một MASTER mới. MASTER mới này sẽ tiếp quản địa chỉ VIP của nhóm và gởi các gói tin ARP báo là nó đang giữ địa chỉ VIP này. Khi MASTER cũ hoạt động bình thường trở lại thì router này có thể lại trở thành MASTER hoặc trở thành BACKUP tùy theo cấu hình độ ưu tiên của các router.
 
 
 
 
 
Mô hình Lab Keepalived IP Failover OpenVPN
```
– Master server
Hostname: Master-server
OS: Ubuntu server 16.04
Service : Keepalived + OpenVPN
Private IP: 10.3.52.55
 
– slave server
Hostname: slave-server
OS: Ubuntu server 16.04
Service : Keepalived + OpenVPN
Private IP: 10.3.52.56
```
Virtual ip address sử dụng cho cả 2 là  10.3.52.100
Hướng dẫn cầu hình.
Để OpenVPN failover cần setup 2 server chạy openvpn có cấu hình tương tự nhau. Để khi master server chết hoặc service chết thì VIP sẽ chuyển sang server slave thì client sẽ khởi tạo lại tunnel mà không bị ảnh hưởng đến kết nối.

#### Bước 1: Cấu hình cài đặt openvpn client to site trên master server
#### Bước 2: Cấu hình cài đặt trên slave server
Cài đặt openvpn

```
$ yum install openvpn -y
$ yum install rsync -y
$ rsync -avzh root@10.3.52.55:/etc/openvpn/* /etc/openvpn
```
Khởi động dịch vụ openvpn
```
$ service openvpn@server restart
$ systemctl enable openvpn@server
```
Bước 3. Cấu hình Keepalive trên cả 2 server
```
$ yum install keepalived -y
$ vim /etc/keepalived/keepalived.conf
```
Thêm vào trong file cấu hình nội dung như sau:
```
vrrp_script chk_openvpn {
    script "/usr/sbin/pidof openvpn"
    interval 2
    weight 3
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 167
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass SDAWE434
    }
    virtual_ipaddress {
        10.3.52.100 dev eth0
    }
    track_script {
    chk_openvpn
    }
}
```
Trong file cấu hình dev eth0 có nghĩa sẽ pair VIP với ip trên interface eth0:
Track_script: khai báo script để track.
Chk_openvpn: đường dẫn nội dung script, weight trọng số khi script check thấy service hoạt động + vào metric. interval là tần suất script chạy để check service tình bằng second.
Priority: giá trị để bình bầu giữa nhưng thành viên tham gia để nhận được VIP.
Bước 4: Khởi động keepalived trên cả 2 server.
  ```
  $ service keepalived restart
  ```
Cho keepalived khởi động cùng hệ thống
  ```
 	$ systemctl enable keepalived.service
  ```
