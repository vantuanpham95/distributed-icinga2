# DISTRIBUTED ICINGA2 
 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_roles.png) 
## 1. Roles: master, satellites, clients 

Node master là nốt trên cùng và không có node cha 

Node master là nơi cài đặt icingaweb2. Có thể cài ở node khác nhưng thông thường thì sẽ cài ở master 

Master node có thể bao gồm việc checks từ nodes con và cảnh báo 

Satellites có thể tự check hoặc phân quyền tới các node thấp hơn 

Satellites có thể nhận configurations cho hosts/services từ node cha 

Satellites vẫn tiếp tục chạy kể cả khi master ngừng hoạt động 

Client node là node cuối cùng, chỉ có node cha, không có node con 

Client node tự chạy config check từ cấu hình của nó hoặc nhận lệnh từ node cha 

## 2. Zones  

Hệ thống icinga2 chia làm các zones, các zones có quan hệ parent-chirld với nhau 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_zones.png) 

Ví dụ:  

```
object Zone "master" { 
   //... 
} 
  
object Zone "satellite region 1" { 
  parent = "master" 
  //... 
} 
  
object Zone "satellite region 2" { 
  parent = "master" 
  //... 
}
```

Các node trong vùng cha - master gửi cấu hình đến vùng satellite nhưng các node trong vùng satellite không được phép gửi lệnh tới vùng cha 

## 3. Endpoint 
 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_endpoints.png) 

Các node trong 1 vùng được gọi là các endpoint object 

Ví dụ /etc/icinga2/zones.conf: 

```
object Endpoint "icinga2-master" { 
  host = "192.168.56.101" 
} 
  
object Endpoint "icinga2-satellite1.localdomain" { 
  host = "192.168.56.105" 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master" ] 
} 
  
object Zone "satellite" { 
  endpoints = [ "icinga2-satellite1.localdomain" ] 
  parent = "master" 
}
```

Tất cả các endpoints trong cùng một zone đều được setup HA, ví dụ nếu có 2 nodes trong zone master thì nó sẽ thực hiện cân bằng tải 

## 4. ApiListener 

Apilistener được sử dụng để tải chứng chỉ SSL và chỉ định các restrictions như chấp nhận cấu hình từ master. 

Apilistenner cũng được dùng trong icinga2 REST API, có cùng host và port với giao thức icinga2 cluster 

File cấu hình: /etc/icinga2/features-enabled/api.conf. Các thuộc tính accept_commands và accept_config có thể được cấu hình ở đây 

Kích hoạt: icinga2 feature enable api 

## 5. Security 

icinga2 có cung cấp các cơ chế bảo mật bổ sung với các rules sau 

SSL: bắt buộc dùng SSL với giao tiếp giữa các node 

Các vùng con chỉ nhận được các cập nhật cho các đối tượng đã được cấu hình trước đó 

Các vùng con không được cập nhật cấu hình cho vùng cha 

Các vùng cũng riêng rẽ với nhau và không thể can thiệp vào các vùng khác. Một host hay 1 service chỉ được gán cho một vùng, chỉ có 1 FQDN 

Tất cả các node trong 1 vùng trust nhau 

## 6. Cấu hình Master 

Setup wizard thực hiện các bước sau 

 * Enable the api feature. 
 * Generate a new certificate authority (CA) in /var/lib/icinga2/ca if it doesn’t exist. 
 * Create a certificate signing request (CSR) for the local node. 
 * Sign the CSR with the local CA and copy all files to the /etc/icinga2/pki directory. 
 * Update the zones.conf file with the new zone hierarchy. 
 * Update /etc/icinga2/features-enabled/api.conf and constants.conf 

Ví dụ 

```
[root@icinga2-master /]# icinga2 node wizard 
Welcome to the Icinga 2 Setup Wizard! 
  
We'll guide you through all required configuration details. 
  
Please specify if this is a satellite setup ('n' installs a master setup) [Y/n]: n 
Starting the Master setup routine... 
Please specify the common name (CN) [icinga2-master]: icinga2-master 
Checking for existing certificates for common name 'icinga2-master'... 
Certificates not yet generated. Running 'api setup' now. 
information/cli: Generating new CA. 
information/base: Writing private key to '/var/lib/icinga2/ca/ca.key'. 
information/base: Writing X509 certificate to '/var/lib/icinga2/ca/ca.crt'. 
information/cli: Generating new CSR in '/etc/icinga2/pki/icinga2-master.csr'. 
information/base: Writing private key to '/etc/icinga2/pki/icinga2-master.key'. 
information/base: Writing certificate signing request to '/etc/icinga2/pki/icinga2-master.csr'. 
information/cli: Signing CSR with CA and writing certificate to '/etc/icinga2/pki/icinga2-master.crt'. 
information/cli: Copying CA certificate to '/etc/icinga2/pki/ca.crt'. 
Generating master configuration for Icinga 2. 
information/cli: Adding new ApiUser 'root' in '/etc/icinga2/conf.d/api-users.conf'. 
information/cli: Enabling the 'api' feature. 
Enabling feature api. Make sure to restart Icinga 2 for these changes to take effect. 
information/cli: Dumping config items to file '/etc/icinga2/zones.conf'. 
information/cli: Created backup file '/etc/icinga2/zones.conf.orig'. 
Please specify the API bind host/port (optional): 
Bind Host []: 
Bind Port []: 
information/cli: Created backup file '/etc/icinga2/features-available/api.conf.orig'. 
information/cli: Updating constants.conf. 
information/cli: Created backup file '/etc/icinga2/constants.conf.orig'. 
information/cli: Updating constants file '/etc/icinga2/constants.conf'. 
information/cli: Updating constants file '/etc/icinga2/constants.conf'. 
information/cli: Updating constants file '/etc/icinga2/constants.conf'. 
Done. 
  
Now restart your Icinga 2 daemon to finish the installation! 
  
[root@icinga2-master1 /]# systemctl restart icinga2 
``` 

Khóa CA public và CA private key được lưu trữ trong thư mục /var/lib/icinga2/ca 

Khi bị mất private key CA, phải tạo một CA mới để sign lại các request của client, sau đó lại phải tạo  lại các chứng chỉ đã 
ký cho tất cả các nút hiện có. 

## 7. Client/ Satellite setup 

Icinga2 trên master phải chạy và mở cổng 5655 

### a. CSR auto-signing 

Node Wizard sẽ thiết lập một kết nối satellite / Client bằng cách sử dụng CSR auto-signd. Điều này liên quan đến việc trình hướng dẫn thiết lập sẽ gửi một yêu cầu ký kết chứng chỉ (CSR) đến nút master. Client bắt buộc phải gửi một ticket hợp lệ cho việc CSR auto-signing 

Ticket này phải được tạo trước, thuộc tính ticket_salt cho Apilistener phải được cấu hình để làm việc này. Có 2 cách để nhận ticket: 

 * CLI command trên master 

 * REST API request từ client gửi đến master node 

Ví dụ: Generate một ticket từ master node icinga2-master cho client icinga2-client1 

```
[root@icinga2-master /]# icinga2 pki ticket --cn icinga2-client1 
Trên master, tạo ra một api object có quyền actions/generate-ticket 
[root@icinga2-master /]# vim /etc/icinga2/conf.d/api-users.conf 
  
object ApiUser "client-pki-ticket" { 
  password = "bea11beb7b810ea9ce6ea" //change this 
  permissions = [ "actions/generate-ticket" ] 
} 
  
[root@icinga2-master /]# systemctl restart icinga2 
``` 

Nhận ticket này trên client bằng cách sử dụng curl 

```
[root@icinga2-master1.localdomain /]# curl -k -s -u client-pki-ticket:bea11beb7b810ea9ce6ea -H 'Accept: application/json' \ 
-X POST 'https://icinga2-master1.localdomain:5665/v1/actions/generate-ticket' -d '{ "cn": "icinga2-client1.localdomain" }' 
```
(Nên save mã này lại để sau còn dùng) 

**Client/ Satellite Linux Setup** 

Các bước 

 * Enable the api feature. 
 * Create a certificate signing request (CSR) for the local node. 
 * Request a signed certificate i(optional with the provided ticket number) on the master node. 
 * Allow to verify the parent node’s certificate. 
 * Store the signed client certificate and ca.crt in /var/lib/icinga2/certs. 
 * Update the zones.conf file with the new zone hierarchy. 
 * Update /etc/icinga2/features-enabled/api.conf (accept_config, accept_commands) and constants.conf. 

Update thông tin service trên node master 
```
root@icinga2-master:~# icinga2 node update-config 
```
Liệt kê các node được monitor 
```
root@icinga2-master:~# icinga2 node list 
```
 
# Cấu hình Distributed 

Có nhiều cách khác nhau để khiến các icinga2 cluster node thực hiện việc check và gửi đi các thông điệp cảnh báo. Nhưng thông thường người ta cấu hình các đối tượng monitoring trên master và phân phối các đối tượng này tới satellites và clients 
 
## 1. TOP DOWN 

Có 2 hành vi khác nhau trong việc thực hiện check: 

Gửi lệnh thực hiện từ nút cha: Trình lập lịch trên nút cha 

Đồng bộ các đối tượng hosts/services tới các nút con 

## 2. TOP DOWN COMMAND ENDPOINT 

Chế độ này buộc nút icinga2 thực hiện các lệnh trên một endpoint được chỉ định. Cấu hình hosts/services nằm trên master/satellites và clients chỉ cần các định nghĩa của đối tượng checkcommand được sử dụng ở đó 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_top_down_command_endpoint.png) 

**Ưu điểm:** 

 * Không cần định nghĩa check trên nút con (client) 
 * Thực hiện light-weight remote check (các sự kiện không đồng bộ) 
 * Không cần relay log cho các nút con 
 * Ghim check tới các endpoint cụ thể  
**Nhược điểm:** 

 * Nếu nút con bị mất hoặc không có kết nối thì sẽ không thực hiện check được 
 * Yêu cầu các thuộc tính cấu hình bổ sung cho các đối tượng hosts/services 
 * Yêu cầu cấu hình đối tượng checkcommand cục bộ. Tốt nhất là nên sử dụng global config zone 

Để đảm bảo tất cả các nút có liên quan chấp nhận cấu hình hay các mệnh lệnh, ta cần cấu hình zone và endpoint phân cấp trên tất cả các nút đó 

Cấu hình zone và endpoint trên cả 2 node trong file /etc/icinga2/zones.conf 

```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  host = "192.168.56.101" 
} 
  
object Endpoint "icinga2-client1.localdomain" { 
  host = "192.168.56.111" 
} 
Sau đó, ta cần định nghĩa 2 zones. Zone "master" là cha của zone "icinga2-client1.localdomain" 
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain" ] //array with endpoint names 
} 
  
object Zone "icinga2-client1.localdomain" { 
  endpoints = [ "icinga2-client1.localdomain" ] 
  
  parent = "master" //establish zone hierarchy 
} 
``` 
Ngoài ra, thêm một global zone để đồng bộ các check commands sau này 
```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Zone "global-templates" { 
  global = true 
} 
```

Ta không cần bất cứ một cấu hình nội tại nào trên máy client ngoại trừ các định nghĩa checkcommand có thể được đồng bộ hóa sử dụng global zone nói trên. Vì vậy ta sẽ disable thư mục conf.d trong file /etc/icinga2/icinga2.conf trên máy client 

```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/icinga2.conf 
  
// Commented out, not required on a client as command endpoint 
//include_recursive "conf.d" 
```

Cấu hình các thuộc tính accept_commands và accept_configs trong api feature trên máy client (sửa file /etc/icinga2/features-enabled/api.conf) 

```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/features-enabled/api.conf 
  
object ApiListener "api" { 
   //... 
   accept_commands = true 
   accept_config = true 
}
```

Restart lại dịch vụ trên cả 2 nodes 

Một khi clients được kết nối thành công, ta sẽ thực hiện remote check trên client sử dụng command endpoint 

Ta nên cấu hình các đối tượng hosts/services vào trong zone master. Việc này sẽ giúp sau này thêm một master thứ 2 phục vụ cho High Availability sau này 
```
[root@icinga2-master1.localdomain /]# mkdir -p /etc/icinga2/zones.d/master 
```

Thêm các đối tượng hosts/services muốn monitor. Không có giới hạn nào về số file và thư mục, nhưng tốt nhất ta nên sắp xếp và phân loại chúng. Ta quy ước rằng một ddối tượng master/satellite/client nên sử dụng cùng tên với đối tượng endpoint. 

```
[root@icinga2-master1.localdomain /]# cd /etc/icinga2/zones.d/master 
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim hosts.conf 
  
object Host "icinga2-client1.localdomain" { 
  check_command = "hostalive" //check is executed on the master 
  address = "192.168.56.111" 
  
  vars.client_endpoint = name //follows the convention that host name == endpoint name 
} 
```

Ví dụ: thêm một remote check về disk monitoring trên một máy Linux client 
```
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim services.conf 
  
apply Service "disk" { 
  check_command = "disk" 
  
  //specify where the check is executed 
  command_endpoint = host.vars.client_endpoint 
  
  assign where host.vars.client_endpoint 
} 
```

Thêm một check command tùy biến vào global zone: 
```
[root@icinga2-master1.localdomain /]# mkdir -p /etc/icinga2/zones.d/global-templates 
[root@icinga2-master1.localdomain /]# vim /etc/icinga2/zones.d/global-templates/commands.conf 
  
object CheckCommand "my-cmd" { 
  //... 
} 
```
Lưu cấu hình và khởi động lại dịch vụ trên node master 

Các bước tiếp theo sẽ xảy ra như sau: 

Icinga2 xác nhận cấu hình trên icinga2-master và khởi động lại 

Node Icinga2-client1 nhận và thực hiện các lệnh với các thông số lệnh bổ sung 

icinga2-client1 node ánh xạ các thông số lệnh tới các check command nội tại, thực hiện lệnh check tại nó và gửi lại các thông điệp chứa kết quả của việc check 

--> Không có tương tác nào thực hiện trực tiếp lên máy client và cũng không cần restart lại dịch vụ icinga2 trên máy client 

## 3. TOP DOWN CONFIG SYNC 

Chế độ này sẽ đồng bộ hóa các tập tin cấu hình trong các vùng được chỉ định, hữu ích nếu ta muốn cấu hình tất cả mọi thứ trên node master và đồng bộ nó xuống các satellite. Satellite sẽ thực hiện các lệnh này và gửi trả lại kết quả 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_top_down_config_sync.png)

**Ưu điểm:**

 * Đồng bộ các file cấu hình từ zone cha tới zone con 
 * Không yêu cầu khởi động lại thủ công trên các node con, cũng như việc đồng bộ, việc xác nhận và khởi động lại diễn ra tự động 
 * Thực thi lệnh check trưc tiếp tới các node con được chỉ định 
 * Sử dụng global zone để đồng bộ templates, groups,… 

**Nhược điểm**

 * Yêu cầu một thư mục config trên node master 
 * Yêu cầu các thông số cấu hình bổ sung về zone và endpoint 
 * Để đảm bảo tất cả các node liên quan chấp nhận cấu hình và lệnh, ta cần cấu hình zone và endpoint trên tất cả các nút 

Trong kịch bản này: 

 * icinga2-master1.localdomain là node master 
 * icinga2-client2.localdomain là client và nhận các cấu hình từ master 

Thêm các cấu hình zone và endpoint vào cả 2 nodes trong file /etc/icinga2/zones.conf 
```
[root@icinga2-client2.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  host = "192.168.56.101" 
} 
  
object Endpoint "icinga2-client2.localdomain" { 
  host = "192.168.56.112" 
} 
Tiếp theo, ta cần định nghĩa 2 zone. Zone master là cha của zone icinga2-client2.localdomain 
[root@icinga2-client2.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain" ] //array with endpoint names 
} 
  
object Zone "icinga2-client2.localdomain" { 
  endpoints = [ "icinga2-client2.localdomain" ] 
  
  parent = "master" //establish zone hierarchy 
} 
```

Cấu hình các thuộc tính accept_commands và accept_configs trong api feature trên máy client (sửa file /etc/icinga2/features-enabled/api.conf) 

```
[root@icinga2-client2.localdomain /]# vim /etc/icinga2/features-enabled/api.conf 
  
object ApiListener "api" { 
   //... 
   accept_config = true 
}
```

Restart lại dịch vụ trên cả 2 nodes 

Một khi clients được kết nối thành công, ta sẽ thực hiện remote check trên client sử dụng đồng bộ hóa cấu hình. 

Quay trở lại /etc/icinga2/zones.d trên master node icinga2-master1.localdomain và tạo một thư mục mới với tên giống tên của satellite/client 

```
[root@icinga2-master1.localdomain /]# mkdir -p /etc/icinga2/zones.d/icinga2-client2.localdomain 
```

Thêm các đối tượng hosts/services muốn monitor. Không có giới hạn nào về số file và thư mục, nhưng tốt nhất ta nên sắp xếp và phân loại chúng. Ta quy ước rằng một ddối tượng master/satellite/client nên sử dụng cùng tên với đối tượng endpoint. 

```
[root@icinga2-master1.localdomain /]# cd /etc/icinga2/zones.d/icinga2-client2.localdomain 
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/icinga2-client2.localdomain]# vim hosts.conf 
  
object Host "icinga2-client2.localdomain" { 
  check_command = "hostalive" 
  address = "192.168.56.112" 
  zone = "master" //optional trick: sync the required host object to the client, but enforce the "master" zone to execute the check 
} 
```

Ví dụ: thêm một remote check về disk monitoring trên một máy Linux client 
```
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/icinga2-client2.localdomain]# vim services.conf 
  
object Service "disk" { 
  host_name = "icinga2-client2.localdomain" 
  
  check_command = "disk" 
}
```
Lưu cấu hình và khởi động lại dịch vụ trên node master 

Các bước tiếp theo sẽ xảy ra như sau: 

 * Icinga2 xác nhận cấu hình trên icinga2-master và khởi động lại 
 * icinga2 copy cấu hình tới zone mà nó được chỉ định trong /var/lib/icinga2/api/zones 
 * icinga2-master1.localdomain gửi cập nhật cấu hình tới tất cả các endpoint trong cùng zone hoặc gửi trực tiếp đến một zone con 
 * icinga2-client2.localdomain chấp nhận cấu hình 
 * icinga2-client2.localdomain xác nhận cấu hình và tự động restart dịch vụ 

Cũng như trước, client không yêu cầu ta tương tác gì với nó cả 

Ta cũng có thể sử dụng đồng bộ hóa cấu hình bên cạnh một high availability zone để đảm bảo rằng tất cả các đối tượng config được sync đến từng zone members 

**Chú ý:** ta chỉ có thể có một "config-master" trong một zone được cấu hình trong thư mục zones.d Không hỗ trợ các file cấu hình nhiều nodes trong thư mục zones.d 

Bây giờ chúng ta sẽ đi thực hiện một kịch bản cụ thể để hiểu rõ cách thiết lập cấu hình distributed icinga2 
 
 
**KỊCH BẢN** 
 
 * Một master và các clients 
 * High Availability master và clients as command endpoint 
 * 3 level cluster với cấu hình HA master, satellites nhận các đồng bộ hóa cầu hình và clients checks sử dụng command endpoint 
 
#### a. Một master và các clients 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_scenarios_master_clients.png)

 * icinga2-master1.localdomain is the primary master node. 
 * icinga2-client1.localdomain and icinga2-client2.localdomain are two child nodes as clients. 

Edit file zones.conf trên master 
```
[root@icinga2-master1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
} 
  
object Endpoint "icinga2-client1.localdomain" { 
  host = "192.168.56.111" //the master actively tries to connect to the client 
} 
  
object Endpoint "icinga2-client2.localdomain" { 
  host = "192.168.56.112" //the master actively tries to connect to the client 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain" ] 
} 
  
object Zone "icinga2-client1.localdomain" { 
  endpoints = [ "icinga2-client1.localdomain" ] 
  
  parent = "master" 
} 
  
object Zone "icinga2-client2.localdomain" { 
  endpoints = [ "icinga2-client2.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
} 
``` 
Về cơ bản, 2 node client không cần biết về nhau. Thứ duy nhất mà nó cần là biết về zone cha và các endpoint member của nó (cùng thông tin về global zone) 

Nếu ta chỉ định thuộc tính host trong endpoint object icinga2-master1.localdomain, client sẽ chủ độn thực hiện kết nối tới node master. Cho tới khi thuộc tính của endpoint client cuối cùng được chỉ định, chúng ta không muốn client thực hiện kết nối tới master.
```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-client1.localdomain" { 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain" ] 
} 
  
object Zone "icinga2-client1.localdomain" { 
  endpoints = [ "icinga2-client1.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
} 
  
[root@icinga2-client2.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-client2.localdomain" { 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain" ] 
} 
  
object Zone "icinga2-client2.localdomain" { 
  endpoints = [ "icinga2-client2.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
}
```
Bây giờ ta sẽ định nghĩa 2 máy client hosts và apply service check sử dụng command endpoint thực thi lệnh trên chúng. 

Tạo một thư mục configuration mới trên master node 
```
[root@icinga2-master1.localdomain /]# mkdir -p /etc/icinga2/zones.d/master 
```

Thêm 2 client nodes như là 2 đối tượng hosts: 
```
[root@icinga2-master1.localdomain /]# cd /etc/icinga2/zones.d/master 
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim hosts.conf 
  
object Host "icinga2-client1.localdomain" { 
  check_command = "hostalive" 
  address = "192.168.56.111" 
  vars.client_endpoint = name //follows the convention that host name == endpoint name 
} 
  
object Host "icinga2-client2.localdomain" { 
  check_command = "hostalive" 
  address = "192.168.56.112" 
  vars.client_endpoint = name //follows the convention that host name == endpoint name 
} 
```

Thêm services sử dụng command endpoint check: 

```
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim services.conf 
  
apply Service "ping4" { 
  check_command = "ping4" 
  //check is executed on the master node 
  assign where host.address 
} 
  
apply Service "disk" { 
  check_command = "disk" 
  
  //specify where the check is executed 
  command_endpoint = host.vars.client_endpoint 
  
  assign where host.vars.client_endpoint 
} 
``` 
Xác nhận thiết lập cấu hình và khởi động lại dịch vụ icinga2 trên master node icinga2-master1.localdomain 
Mở icingaweb2 và kiểm tra 2 client hosts với 2 dịch vụ mới được tạo: một dịch vụ thực thi locally (ping 4) và một dịch vụ sử dụng command endpoint (disk) 
 
#### c. High-Availability Master with Clients 
 
![](https://www.icinga.com/docs/icinga2/latest/doc/images/distributed-monitoring/icinga2_distributed_scenarios_ha_master_clients.png) 

Kịch bản này tương tự như phần trên, chỉ khác là vùng master được thiết lập High Availability. Các nút này phải được cấu hình như các đối tượng zone và endpoint 

Thiết lập sử dụng các tính năng của icinga2 cluster. Tất cả các zone member sao chép các sự kiện replicate cluster với những zone member còn lại. Thêm vào đó, một vài icinga2 features có thể bật HA functionality 

Chú ý: tất cả các node trong cùng một zone yêu cầu bật tính năng HA giống nhau 

Trong kịch bản này: 

 * icinga2-master1.localdomain là config master master node 
 * icinga2-master2.localdomain là master node thứ 2 và không có cấu hình trong zones.d 
 * icinga2-client1.localdomain và icinga2-client2.localdomain là 2 node con 

Bây giờ đã có 2 nodes có chung zone, ta cần phải xem xét các tính năng high availability 

check và notifications được cân bằng giữa 2 master nodes, điều này đòi hỏi check plugins và các scripts notifications tồn tại trên cả 2 máy 

Mặc định, IDO features sẽ chỉ được kích hoạt trên một node. Cho đến khi tất cả các events được replicate giữa cả 2 node thì sẽ dễ dàng hơn nếu chỉ có 1 database trung tâm duy nhất. 

Ví dụ cấu hình zones.conf trên icinga2-master1.localdomain 
```
[root@icinga2-master1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  host = "192.168.56.101" 
} 
  
object Endpoint "icinga2-master2.localdomain" { 
  host = "192.168.56.102" 
} 
  
object Endpoint "icinga2-client1.localdomain" { 
  host = "192.168.56.111" //the master actively tries to connect to the client 
} 
  
object Endpoint "icinga2-client2.localdomain" { 
  host = "192.168.56.112" //the master actively tries to connect to the client 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain", "icinga2-master2.localdomain" ] 
} 
  
object Zone "icinga2-client1.localdomain" { 
  endpoints = [ "icinga2-client1.localdomain" ] 
  
  parent = "master" 
} 
  
object Zone "icinga2-client2.localdomain" { 
  endpoints = [ "icinga2-client2.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
} 
``` 

2 node client không cần phải biết về nhau. Điều quan trọng mà chúng cần phải biết là về parent zone và các endpoint member của nó (cùng với cả thông tin global zone) 

Nếu ta gán các thuộc tính host vào trong icinga2-master1.localdomain và icinga2-master2.localdomain endpoint objects thì client sẽ cố gắng để kết nối tới master node. Trước khi chúng ta hoàn thiện các thuộc tính của endpoint client trên master node, ta không muốn client thực hiện kết nối. 
```
[root@icinga2-client1.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-master2.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-client1.localdomain" { 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain", "icinga2-master2.localdomain" ] 
} 
  
object Zone "icinga2-client1.localdomain" { 
  endpoints = [ "icinga2-client1.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
} 
  
[root@icinga2-client2.localdomain /]# vim /etc/icinga2/zones.conf 
  
object Endpoint "icinga2-master1.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-master2.localdomain" { 
  //do not actively connect to the master by leaving out the 'host' attribute 
} 
  
object Endpoint "icinga2-client2.localdomain" { 
} 
  
object Zone "master" { 
  endpoints = [ "icinga2-master1.localdomain", "icinga2-master2.localdomain" ] 
} 
  
object Zone "icinga2-client2.localdomain" { 
  endpoints = [ "icinga2-client2.localdomain" ] 
  
  parent = "master" 
} 
  
/* sync global commands */ 
object Zone "global-templates" { 
  global = true 
}
```
Bây giờ ta sẽ định nghĩa 2 client host và apply service check sử dụng command endpoint 

Tạo ra một folder configuration mới trên master node icinga2-master1.localdomain --> node master thứ 2 sẽ tự sync 
```
[root@icinga2-master1.localdomain /]# mkdir -p /etc/icinga2/zones.d/master 
```

Thêm 2 clients node như là host objects 
```
[root@icinga2-master1.localdomain /]# cd /etc/icinga2/zones.d/master 
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim hosts.conf 
  
object Host "icinga2-client1.localdomain" { 
  check_command = "hostalive" 
  address = "192.168.56.111" 
  vars.client_endpoint = name //follows the convention that host name == endpoint name 
} 
  
object Host "icinga2-client2.localdomain" { 
  check_command = "hostalive" 
  address = "192.168.56.112" 
  vars.client_endpoint = name //follows the convention that host name == endpoint name 
} 
Thêm service sử dụng command endpoint check 
[root@icinga2-master1.localdomain /etc/icinga2/zones.d/master]# vim services.conf 
  
apply Service "ping4" { 
  check_command = "ping4" 
  //check is executed on the master node 
  assign where host.address 
} 
  
apply Service "disk" { 
  check_command = "disk" 
  
  //specify where the check is executed 
  command_endpoint = host.vars.client_endpoint 
  
  assign where host.vars.client_endpoint 
}
```
Xác nhận cấu hình và khởi động lại icinga2 trên icinga2-master1.localdomain 
 
