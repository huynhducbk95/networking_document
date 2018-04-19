
etcd -proxy on \
     -listen-client-urls http://127.0.0.1:2379,http://127.0.0.1:4001 \
     -initial-cluster database=http://192.168.122.55:2380


# Docs install Calico trên Openstack All-in-one Openvswitch hoặc linuxbridge
Tài liệu này sẽ hướng dẫn cách cài đặt calico là core plugin cho neutron service của openstack và network plugin cho docker container.
## 1. Mô hình triển khai
Cụm openstack gồm có 2 node: phiên bản `Pike`
- Controller node: cài all in one
 - Hostname: controller
 - IP: 192.168.122.55
- Compute node:
 - Hostname: compute
 - IP: 192.168.122.211

Và 2 node docker container:
- docker01 node:
 - Hostname: docker01
 - IP: 192.168.122.45
- docker02 node:
 - Hostname: docker02
 - IP: 192.168.122.13

Yêu cầu bắt buộc:
- Openstack đã triển khai là bản pike với đầy đủ cả core service
- Các node cùng dải mạng, có thể ping được với nhau

NOTE: Chú ý những câu lệnh dùng quyền root
## 2. Cài đặt ETCD cluster
ETCD là một cơ sở dữ liệu phân tán, lưu trữ theo dạng key/value. Trong môi trường calico, ETCD được sử dụng để lưu trữ thông tin của các calico node, thông tin của các workload(container,virtual machine,....)

NOTE: Chúng ta sẽ tạo ra một ETCD cluster sử dụng chung cho cả openstack node và docker node, để chúng có thể liên lạc được với nhau. Điều này là bắt buộc trong yêu cầu tạo ra kết nối giữa VM và container.

Hiện nay, mình sẽ tạo một ETCD cluster trên một node duy nhất và nó cùng nằm trên node controller. Các node khác sẽ sử dụng ETCD proxy để proxy đến ETCD cluster trên node controller.

NOTE: Trong môi trường tốt hơn, ETCD cluster nên được cài đặt trên nhiều node tách biệt với các node trong openstack deployment và các docker node. Các openstack node và docker node sẽ cài đặt etcd proxy đến ETCD cluster đó.

#### Cài đặt ETCD cluster trên `controller` node
ssh vào `controller` node và thực hiện các bước cài đặt sau:

Tạo etcd user:
```
# groupadd --system etcd
# useradd --home-dir "/var/lib/etcd" \
      --system \
      --shell /bin/false \
      -g etcd \
      etcd

```

Tạo các thư mục cần thiết:

```
mkdir -p /etc/etcd
chown etcd:etcd /etc/etcd
mkdir -p /var/lib/etcd
chown etcd:etcd /var/lib/etcd
```

Mount một disk RAM tại `/var/lib/etcd`:

```
mount -t tmpfs -o size=512m tmpfs /var/lib/etcd
```

Thêm dòng sau vào cuối file `/etc/fstab` để disk RAM sẽ lấy lại trạng thái dữ liệu trong ETCD cluster sau mỗi lần reboot:

```
tmpfs /var/lib/etcd tmpfs nodev,nosuid,noexec,nodiratime,size=512M 0 0
```

Download và cài đặt etcd:

```
ETCD_VER=v3.2.18
rm -rf /tmp/etcd && mkdir -p /tmp/etcd
curl -L https://github.com/coreos/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1
cp /tmp/etcd/etcd /usr/bin/etcd
cp /tmp/etcd/etcdctl /usr/bin/etcdctl
```

Tạo file `/etc/etcd/etcd.conf.yml`, thêm và thay giá trị địa chỉ ip của controller node phù hợp:

```
name: controller
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-01'
initial-cluster: controller=http://192.168.122.55:2380
initial-advertise-peer-urls: http://192.168.122.55:2380
advertise-client-urls: http://192.168.122.55:2379,http://192.168.122.55:4001
listen-peer-urls: http://192.168.122.55:2380
listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
```
Thông tin về các tham số trong file này nằm ngoài phạm vi của tài liệu này. Cụ thể xin tham khảo [tại đây](https://coreos.com/etcd/docs/latest/op-guide/configuration.html).

Tiếp theo, tạo và thêm nội dung cho file `/lib/systemd/system/etcd.service`:

```
[Unit]
After=network.target
Description=etcd - highly-available key value store

[Service]
LimitNOFILE=65536
Restart=on-failure
Type=notify
ExecStart=/usr/bin/etcd --config-file /etc/etcd/etcd.conf.yml
User=etcd

[Install]
WantedBy=multi-user.target
```

Cuối cùng, enable và restart etcd:

```
# systemctl enable etcd
# systemctl start etcd
```
Kiểm tra hoạt động của ETCD bằng các câu lệnh sau:
```
# etcdctl member list
2ec67378394c9ef8: name=controller peerURLs=http://192.168.122.55:2380 clientURLs=http://192.168.122.55:2379,http://192.168.122.55:4001 isLeader=true

# etcdctl cluster-health
member 2ec67378394c9ef8 is healthy: got healthy result from http://192.168.122.55:2379
cluster is healthy

# etcdctl --version
etcdctl version: 3.2.18
API version: 2

```

NOTE: Calico được sử dụng trong tài liệu này có version là v2.6 và yêu cầu ETCD APIv2. Do đó, chúng ta cần sử dụng ETCD API version 2. Các version >3 của ETCD có hỗ trợ cả APIv2 và APIv3. Sử dụng biến môi trường `ETCDCTL_API=2` để thiết lập APIv2 cho cluster nếu cluster đang sử dụng APIv3.

Kết thúc cài đặt ở `controller` node. Tiếp theo sẽ cài đặt ETCD proxy trên các node còn lại. Và các bước cài đặt này là tương tự trên các node. 

NOTE: Thay đổi hostname và địa chỉ IP phù hợp với từng node
#### Cài đặt ETCD cluster trên `compute` node
ssh vào `compute` node và thực hiện các bước cài đặt sau:

Tạo etcd user:
```
# groupadd --system etcd
# useradd --home-dir "/var/lib/etcd" \
      --system \
      --shell /bin/false \
      -g etcd \
      etcd

```

Tạo các thư mục cần thiết:

```
mkdir -p /etc/etcd
chown etcd:etcd /etc/etcd
mkdir -p /var/lib/etcd
chown etcd:etcd /var/lib/etcd
```

Download và cài đặt etcd:

```
ETCD_VER=v3.2.18
rm -rf /tmp/etcd && mkdir -p /tmp/etcd
curl -L https://github.com/coreos/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1
cp /tmp/etcd/etcd /usr/bin/etcd
cp /tmp/etcd/etcdctl /usr/bin/etcdctl
```

Tạo file `/etc/etcd/etcd.conf.yml`, thêm và thay giá trị địa chỉ ip và hostname của `compute` node cho phù hợp:

```
name: compute
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-02'
initial-cluster: compute=http://192.168.122.211:2380
initial-advertise-peer-urls: http://192.168.122.211:2380
advertise-client-urls: http://192.168.122.211:2379,http://192.168.122.211:4001
listen-peer-urls: http://192.168.122.211:2380
listen-client-urls: http://192.168.122.211:2379,http://192.168.122.211:4001
```

Tiếp theo, tạo và thêm nội dung cho file `/lib/systemd/system/etcd.service`:

```
[Unit]
After=network.target
Description=etcd - highly-available key value store

[Service]
LimitNOFILE=65536
Restart=on-failure
Type=notify
ExecStart=/usr/bin/etcd --config-file /etc/etcd/etcd.conf.yml
User=etcd

[Install]
WantedBy=multi-user.target
```

Cuối cùng, enable và restart etcd:

```
# systemctl enable etcd
# systemctl start etcd
```
Tiếp theo, để tạo proxy đến ETCD cluster trên `controller` node chúng ta mở một tab mới (tab này không được tắt, nên sử dụng `byobu` để chạy câu lệnh này để tạo  một tiến trình trên `compute` node - tránh trường hợp đóng tab tiến trình sẽ không bị ngắt):
```
# etcd -proxy on \
     -listen-client-urls http://127.0.0.1:2379,http://127.0.0.1:4001 \
     -initial-cluster database=http://192.168.122.55:2380
2018-03-30 11:02:02.488159 I | etcdmain: etcd Version: 3.2.18
2018-03-30 11:02:02.488304 I | etcdmain: Git SHA: eddf599c6
2018-03-30 11:02:02.488325 I | etcdmain: Go Version: go1.8.7
2018-03-30 11:02:02.488343 I | etcdmain: Go OS/Arch: linux/amd64
2018-03-30 11:02:02.488361 I | etcdmain: setting maximum number of CPUs to 2, total number of available CPUs is 2
2018-03-30 11:02:02.489085 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
2018-03-30 11:02:02.489473 N | etcdmain: proxy: this proxy supports v2 API only!
2018-03-30 11:02:02.512866 I | etcdmain: proxy: using peer urls [http://192.168.122.55:2380] 
2018-03-30 11:02:02.573039 I | proxy/httpproxy: endpoints found ["http://192.168.122.55:4001" "http://192.168.122.55:2379"]
2018-03-30 11:02:02.573106 E | etcdmain: forgot to set Type=notify in systemd service file?
2018-03-30 11:02:02.573164 I | etcdmain: proxy: listening for client requests on http://127.0.0.1:4001
2018-03-30 11:02:02.573692 I | etcdmain: proxy: listening for client requests on http://127.0.0.1:2379

```

Đóng `byobu` tab lại và mở 1 tab mới để kiểm tra hoạt động của ETCD bằng các câu lệnh sau:
```
# etcdctl member list
2ec67378394c9ef8: name=controller peerURLs=http://192.168.122.55:2380 clientURLs=http://192.168.122.55:2379,http://192.168.122.55:4001 isLeader=true

# etcdctl cluster-health
member 2ec67378394c9ef8 is healthy: got healthy result from http://192.168.122.55:2379
cluster is healthy

# etcdctl --version
etcdctl version: 3.2.18
API version: 2

```

NOTE: Các câu lệnh `etcdctl` sẽ sử dụng các tham số mặc định nếu chúng không được khai báo. Do đó tham số `ETCD_ENDPOINT` sẽ có giá trị mặc định là `http://127.0.0.1:2379`. Để ý câu lệnh chạy ETCD proxy, tham số `listen-client-urls` được sử dụng để lắng nghe các request từ client từ địa chỉ `http://127.0.0.1:2379`và `http://127.0.0.1:4001`. Tiếp theo, request sẽ được gửi đến ETCD cluster thông qua địa chỉ `http://192.168.122.55:2380`.

Như vậy là chúng ta đã tạo ra được ETCD proxy trên `compute` node. Các bước thực hiện tương tự trên các node còn lại.

## 3. Cài đặt Calico trên các node của Openstack deployment
Đối với các node trong openstack deployment, chúng ta sẽ thực hiện trong 3 phần được nêu chi tiết dưới đây.
NOTE: Các phần được thực hiện theo thứ tự trên xuống. Và nếu một node cài đặt cả thành phần controller và compute thì sẽ thực hiện cả 3 phần.

## 3.1. Cài đặt các repository
Phần này được thực hiện trên tất cả các node trong openstack deployment.

Thêm các repository sau để có thể sử dụng được Calico PPA:

```
 $ sudo add-apt-repository ppa:project-calico/calico-2.6
```

và official BIRD PPA:

```
$ sudo add-apt-repository ppa:cz.nic-labs/bird
```

Sau khi add thành công các repository. Thực hiện cập nhật các repository:

```
$ sudo apt-get update
```

## 3.2. Cài đặt trên `controller` node

Thực hiện upgrade và dist-upgrade để cập nhật Calico với các package của Openstack và namespace `dnsmasq`.

```
# apt-get upgrade
# apt-get dist-upgrade
```

Sau khi cập nhật xong, cài đặt `calico-control` package. Package này sẽ thực hiện cài đặt thành phần `networking-calico` hoạt động như core plugin cho neutron serivce:

```
$ sudo apt-get install -y calico-control
```

Tiếp theo, thực hiện thay đổi file `/etc/neutron/neutron.conf` để sử dụng calico là core plugin cho neutron service.
- Thay đổi giá trị của `core_plugin` trong phần `[DEFAULT]` đến `calico`.
- Calico có plugin router của nó nên thay đổi giá trị của `service_plugin` trong phần `[DEFAULT]` là: `service_plugin = `
- Nếu trong cấu hình neutron có tùy chọn `dhcp_agents_per_network` thì cài đặt giá trị là số lượng compute node. Trong trường hợp tài liệu này, có 2 node compute nên giá trị của tùy chọn này là 1.

Sau đó, khởi động lại `neutron-server`:

```
$ sudo service neutron-server restart
```

Sau khi restart `neutron-server`, neutron-server sẽ kiểm tra định kỳ trong ETCD cluster xem đã có Calico Felix agent nào đăng ký chưa (tài liệu về Felix xem [tại đây](https://docs.projectcalico.org/v2.6/reference/architecture/#felix)). Một đoạn log của `neutron-server` sẽ hiển thị như sau, log này sẽ lặp lại cho đến khi có 1 `Felix agent` đăng ký tới cùng ETCD cluster mà `networking-calico` đang trỏ tới:
```
root@controller:/home/controller# tail -f /var/log/neutron/neutron-server.log
2018-03-30 11:15:52.907 26908 INFO networking_calico.plugins.ml2.drivers.calico.election [-] Refreshing master role
2018-03-30 11:15:52.912 26908 INFO networking_calico.plugins.ml2.drivers.calico.election [-] Refreshed master role
2018-03-30 11:15:53.380 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:54.385 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:55.389 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:56.393 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:57.397 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:58.401 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:15:59.406 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...
2018-03-30 11:16:00.410 26908 INFO networking_calico.etcdutils [-] Waiting for etcd directory '/calico/felix/v1/host' to exist...

```

Tiếp theo là các bước cài đặt trên `compute` node
## 3.3 Các cài đặt trên `compute` node

Disable SElinux nếu nó đang chạy. Thực hiện kiểm tra bằng việc chạy câu lệnh:

```
# sestatus
SELinux status:                 disabled

```

Thay đổi cấu hình libvirt phù hợp với calico. Trong file `/etc/libvirt/qemu.conf`, thêm hoặc thay đổi giá trị của các tùy chọn sau:

```
clear_emulator_capabilities = 0
user = "root"
group = "root"
cgroup_device_acl = [
     "/dev/null", "/dev/full", "/dev/zero",
     "/dev/random", "/dev/urandom",
     "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
     "/dev/rtc", "/dev/hpet", "/dev/net/tun",
]
```

Khởi động lại libvirt để thực hiện những thay đổi:


```
$ sudo service libvirt-bin restart
```

Tiếp theo, trong file cấu hình nova `/etc/nova/nova.conf`, nếu đang xét thì xóa các dòng sau:
- Trong phần [DEFAULT] nếu có thì xóa dòng: `linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver`
- Trong phần [neutron] nếu có thì xóa dòng: `service_neutron_metadata_proxy = true` và dòng `service_metadata_proxy = true`.

Khởi động lại nova-compute để cập nhật những thay đổi:
```
$ sudo service nova-compute restart
```

### Đối với các openstack deployment sử dụng OpenVswitch
Thực hiện các câu lệnh sau để dừng các service đang chạy.

```
 $ sudo service openvswitch-switch stop
 $ sudo service neutron-openvswitch-agent stop
```

Để tránh các service này chạy lại khi reboot, thực hiện các lệnh sau:

```
$ sudo sh -c "echo 'manual' > /etc/init/openvswitch-switch.override"
$ sudo sh -c "echo 'manual' > /etc/init/openvswitch-force-reload-kmod.override"
$ sudo sh -c "echo 'manual' > /etc/init/neutron-openvswitch-agent.override"
```

Sau đó, kiểm tra các agent của openstack networking đã bị dừng lại bằng câu lệnh sau:

```
neutron agent-list
```

Xóa các agent đó bằng câu lệnh sau. Trong trường hợp này thì service `neutron-openvswitch-agent' bị dừng lại:

```
neutron agent-delete <agent-id>
```

### Đối với các openstack deployment sử dụng Linuxbridge

```
$ sudo service neutron-linuxbridge-agent stop
```

Để tránh service này tự động chạy khi reboot, thực hiện câu lệnh sau:
```
$ sudo sh -c "echo 'manual' > /etc/init/neutron-linuxbridge-agent.override"
```
Sau đó, xóa các agent của openstack networking bị dừng lại. Trong trường hợp này thì service `neutron-linuxbridge-agent` đã bị dừng lại:

```
neutron agent-delete <agent-id>
```

### Tiếp tục với tất cả các openstack deplyoment
Cài đặt các package bổ sung.
NOTE: Nếu trên node `controller` có triển khai all in one, bước này không cần thực hiện.

```
# apt-get install neutron-common neutron-dhcp-agent nova-api-metadata
```

Tiếp theo, đối với các phiên bản Openstack từ Liberty về sau, chúng ta sử dụng `calico-dhcp-agent` để thay cho việc sử dụng `neutron-dhcp-agent'. Thực hiện bằng các câu lệnh sau:

```
$ sudo service neutron-dhcp-agent stop
$ echo manual | sudo tee /etc/init/neutron-dhcp-agent.override
$ sudo apt-get install calico-dhcp-agent
```

Xóa agent `neutron-dhcp-agent` với câu lệnh `neutron agent-delete <agent-id>`.


Tiếp theo, cài đặt `calico-compute` package bằng câu lệnh sau:

```
$ sudo apt-get install calico-compute
```

Chọn `yes` ở các tùy chọn được cung cấp.

Sau khi cài đặt xong, tạo một file `/etc/calico/felix.cfg` với nội dung như sau:

```
[global]
DatastoreType = etcdv2
EtcdAddr = <controller_ip>:4001
FelixHostname = <hostname>
ReportingIntervalSecs = 30
```

Thay `<controller_ip>` và `<hostname>` phù hợp với ETCD cluster trên host đang triển khai.

Cuối cùng, restart `calico-felix`:

```
$ sudo service calico-felix restart
```

Như vậy là chúng ta đã cài đặt thành công calico cho triển khai Openstack. Để kiểm tra cài đặt đã hoạt động đúng. Thực hiện các bước sau:
- Kiểm tra các agent của openstack networking:
```
$ openstack network agent list
```