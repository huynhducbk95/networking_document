# Triển khai Calico là plugin cho docker network trong một docker node

Trong phần này, chúng ta sẽ cài đặt Calico sử dụng như network plugin trên 2 node docker. Sau đó, kết nối các docker node và các openstack node để tạo ra một môi trường hoàn chỉnh.

Docker node:
- hostname: docker01
- ip: 192.168.122.45

## 1. Cài đặt ETCD cluster trên `docker01` node
ssh vào `docker01` node và thực hiện các bước cài đặt sau:

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
name: docker01
data-dir: /var/lib/etcd
initial-cluster-state: 'new'
initial-cluster-token: 'etcd-cluster-02'
initial-cluster: docker01=http://192.168.122.45:2380
initial-advertise-peer-urls: http://192.168.122.45:2380
advertise-client-urls: http://192.168.122.45:2379,http://192.168.122.45:4001
listen-peer-urls: http://192.168.122.45:2380
listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
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

Kiểm tra hoạt động của ETCD bằng các câu lệnh sau:

```
# etcdctl member list
2ec67378394c9ef8: name=docker01 peerURLs=http://192.168.122.45:2380 clientURLs=http://192.168.122.45:2379,http://192.168.122.45:4001 isLeader=true

# etcdctl cluster-health
member 2ec67378394c9ef8 is healthy: got healthy result from http://192.168.122.45:2379
cluster is healthy

# etcdctl --version
etcdctl version: 3.2.18
API version: 2

```

## 2. Cài đặt và cấu hình docker
Tài liệu này không nêu ra chi tiết cài đặt docker, thông tin cụ thể về các bước cài đặt có thể tham khoản tại [docs.docker.com](https://docs.docker.com/).

Dưới đây là các câu lệnh cài đặt docker:
```
# apt-get remove docker docker-engine docker.io

# apt-get update

# apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# apt-key fingerprint 0EBFCD88

# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# apt-get update

# apt-get install docker-ce
```

Kiểm tra cài đặt docker bằng cách run container `hello-world`:

```
# docker run hello-world
```

## 3. Cấu hình Docker
Để sử dụng calico như là plugin cho docker networking, docker daemon cần phải được cấu hình để gọi đến một cluster store.

Cấu hình ETCD như là một cluster store cho docker bằng cách thêm giá trị cho tham số `cluster-store` với cấu trúc là `etcd://<ETCD_IP>:<ETCD_PORT>`. Với ETCD_IP và ETCD_PORT là địa chỉ của ETCD cluster node hoặc của ETCD proxy.

Trong ubuntu, để thực hiện việc này, mở file `/lib/systemd/system/docker.service`. Tìm đến dòng bắt đầu bằng `ExecStart=/usr/bin/dockerd -H fd://`. Sau đó, thêm vào cuối dòng này `--cluster-store=etcd://127.0.0.1:2379`

NOTE: Nguyên nhân sử dụng địa chỉ `127.0.0.1:2379` là do ETCD proxy lắng nghe các request của client trên địa chỉ này.

Cuối cùng, restart docker:

```
systemctl daemon-reload
systemctl restart docker
```

## 4. Cài đặt calico node

Trước khi thực hiện các hoạt động với calico, chúng ta cần thực hiện câu lệnh sau để định nghĩa tiền tố cho các interface của các container. Có nghĩa là các container sau này sẽ được tạo ra với tiền tố là `cali`:

```
# etcdctl set /calico/v1/host/<docker_host>/config/InterfacePrefix cali
```

NOTE: thay đổi giá trị `<docker_host>` phù hợp với hostname của node.

Tiếp theo, download **calicoctl**:

```
# wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.6.3/calicoctl
```

Thêm quyền cho `/usr/local/bin/calicoctl`:

```
# chmod +x /usr/local/bin/calicoctl
```

Cuối cùng, chạy câu lệnh sau để tạo ra một container calico node. Calico node này hoạt động như plugin cho docker networking:

```
# calicoctl node run --no-default-ippools --ip=NODE_IP --node-image=quay.io/calico/node:v2.6.9
```

NOTE: Thay `NODE_IP` bằng địa chỉ IP của node đang chạy câu lệnh này.

## 5. Demo: Tạo network và các container

Khi sử dụng calico là plugin cho docker network, calico sử dụng một khái niệm là **profile** để đại diện cho một network trong docker. Mỗi network được tạo ra, calico tự động tạo ra một profile tương ứng với network đó. Profile này sẽ quy định khả năng truy cập của các container trong network đó. Các container cùng một profile thì có thể truy cập được với nhau.

Calico cũng hỗ trợ tạo ra các profile do người dùng định nghĩa, các profile có thể được thêm đến các network để tạo ra một môi trường network đa dạng hơn. Tính năng này sẽ được sử dụng để tạo kết nối giữa VM trong openstack và container trong docker.

Bây giờ, chúng ta sẽ thực hiện tạo một network. Sau đó, tạo ra 2 container gán đến network đó. Và kiểm tra kết nối giứa 2 container này:

Tạo một ipPool:

```
# cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 192.0.2.0/24
EOF
```

Tạo network với ipPool vừa được tạo ra ở trên với calico là plugin:

```
# docker network create --driver calico --ipam-driver calico-ipam --subnet=192.0.2.0/24 my_net
```

Tạo 2 container với IP xác định từ pool trên:

```
# docker run --net my_net --name my_workload2 --ip 192.0.2.2 -tid busybox
# docker run --net my_net --name my_workload3 --ip 192.0.2.3 -tid busybox
```

Kiểm tra IP của các container vừa được tạo ra bằng câu lệnh sau:

```
# docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my_workload2
192.0.2.2
```

Tiếp theo, kiểm tra kết nối giữa hai container này:
```
# docker attach my_workload2
/ #  ping 192.0.2.3
PING 192.0.2.3 (192.0.2.3): 56 data bytes
^C
--- 192.0.2.3 ping statistics ---
12 packets transmitted, 0 packets received, 100% packet loss
```

:((

Em dùng tcpdump 2 thằng interface của 2 container này ko thấy thằng reply packet về.
Nếu em ping đến docker host thì vẫn thấy reply packet, nhưng ko show lên trong ouput

```
# ping 192.168.122.45
PING 192.168.122.45 (192.168.122.45): 56 data bytes
^C
--- 192.168.122.45 ping statistics ---
12 packets transmitted, 0 packets received, 100% packet loss
```
```
root@docker01:~# tcpdump -i calif73b54c4622
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on calif73b54c4622, link-type EN10MB (Ethernet), capture size 262144 bytes
15:36:17.823026 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 2, length 64
15:36:17.823070 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 2, length 64
15:36:18.823253 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 3, length 64
15:36:18.823316 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 3, length 64
15:36:19.823580 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 4, length 64
15:36:19.823664 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 4, length 64
15:36:20.823828 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 5, length 64
15:36:20.823901 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 5, length 64
15:36:20.827264 ARP, Request who-has 169.254.1.1 tell 192.0.2.102, length 28
15:36:20.827279 ARP, Reply 169.254.1.1 is-at 32:82:ad:c5:c4:0d (oui Unknown), length 28
15:36:21.824100 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 6, length 64
15:36:21.824166 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 6, length 64
15:36:22.824399 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 7, length 64
15:36:22.824458 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 7, length 64
15:36:23.824561 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 8, length 64
15:36:23.824603 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 8, length 64
15:36:24.824816 IP 192.0.2.102 > docker01: ICMP echo request, id 3584, seq 9, length 64
15:36:24.824886 IP docker01 > 192.0.2.102: ICMP echo reply, id 3584, seq 9, length 64

```
