# Triển khai Neutron trên Controller và Neutron Node

## Prerequisites

### 1. Chuẩn bị Cơ sở dữ liệu cho Neutron
```shell
# Đăng nhập vào MySQL
mysql
```
```sql
-- Tạo database neutron
CREATE DATABASE neutron;

-- Cấp quyền cho user neutron
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'OpenStack';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'OpenStack';

-- Thoát MySQL
EXIT;
```

### 2. Chuẩn bị Keystone
```shell
# Load biến môi trường admin
. admin-openrc
```
```shell
# Tạo user neutron và các endpoint
openstack user create --domain default --password 'OpenStack' neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public https://neutron.kbuor.io.vn:9696
openstack endpoint create --region RegionOne network internal https://neutron.kbuor.io.vn:9696
openstack endpoint create --region RegionOne network admin https://neutron.kbuor.io.vn:9696
```

---

## Cài đặt và cấu hình Neutron trên Neutron Node

### 3. Cài đặt package Neutron
```shell
apt install -y neutron-server neutron-plugin-ml2 neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

### 4. Cấu hình các file cấu hình Neutron
#### neutron.conf
```shell
cat <<'EOF' > /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:OpenStack@controller01.openstack.local
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[database]
connection = mysql+pymysql://neutron:OpenStack@controller01.openstack.local/neutron
[keystone_authtoken]
www_authenticate_uri = https://keystone.kbuor.io.vn:5000
auth_url = https://keystone.kbuor.io.vn:5000
memcached_servers = controller01.openstack.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = OpenStack
[nova]
auth_url = https://keystone.kbuor.io.vn:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = OpenStack
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
EOF
```

#### ml2_conf.ini
```shell
cat <<'EOF' > /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
EOF
```

#### Cấu hình Open vSwitch
```shell
ovs-vsctl add-br provider-br-01
ovs-vsctl add-port provider-br-01 ens224
```

#### openvswitch_agent.ini
```shell
cat <<'EOF' > /etc/neutron/plugins/ml2/openvswitch_agent.ini
[agent]
tunnel_types = vxlan
l2_population = true
[ovs]
bridge_mappings = provider-network-01:provider-br-01
local_ip = 13.28.4.61
[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
EOF
```

#### l3_agent.ini
```shell
cat <<'EOF' > /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = openvswitch
EOF
```

#### dhcp_agent.ini
```shell
cat <<'EOF' > /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
EOF
```

#### metadata_agent.ini
```shell
cat <<'EOF' > /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = nova.tpcoms.ai
metadata_proxy_shared_secret = OpenStack
EOF
```

---

## Cập nhật Nova để tích hợp Neutron (trên Controller Node)

### 5. Chỉnh sửa file `/etc/nova/nova.conf`
```ini
[neutron]
auth_url = https://keystone.kbuor.io.vn:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = OpenStack
service_metadata_proxy = true
metadata_proxy_shared_secret = OpenStack
```

---

## Đồng bộ database và khởi động dịch vụ Neutron

### 6. Đồng bộ CSDL Neutron
```shell
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

### 7. Khởi động các dịch vụ Neutron
```shell
systemctl enable neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl restart neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
```

---

## Chạy Neutron API bằng Apache HTTPD

### 8. Vô hiệu hóa neutron-server và tạo WSGI script
```shell
systemctl disable neutron-server
systemctl stop neutron-server

apt install apache2 libapache2-mod-wsgi-py3 -y
```

### 9. Tạo VirtualHost chạy Neutron API qua HTTPS
```shell
cat <<'EOF' > /etc/apache2/sites-available/neutron-https.conf
<VirtualHost *:9696>
    ServerName neutron.kbuor.io.vn

    SSLEngine on
    SSLCertificateFile /etc/ssl/openstack/openstack.crt
    SSLCertificateKeyFile /etc/ssl/openstack/openstack.key

    WSGIDaemonProcess neutron user=neutron group=neutron processes=4 threads=10 display-name=%{GROUP}
    WSGIProcessGroup neutron
    WSGIScriptAlias / /usr/bin/neutron-api

    <Directory /usr/bin>
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/neutron_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/neutron_ssl_access.log combined
</VirtualHost>
EOF

echo 'Listen 9696' >> /etc/apache2/ports.conf
```

### 10. Bật HTTPS cho Neutron và khởi động Apache
```shell
a2enmod ssl
a2ensite neutron-https
systemctl restart apache2
systemctl reload apache2
```

---

## Ghi chú
- Đảm bảo các hostname như `controller01.openstack.local`, `neutron.kbuor.io.vn`, và `keystone.kbuor.io.vn` đã được khai báo đúng trong DNS hoặc `/etc/hosts`.
- Địa chỉ `ens224`, `13.28.4.61` cần thay đổi phù hợp với hạ tầng mạng thực tế của bạn.
