# Cấu hình keepalived cho FreeIPA

## Mô hình
- Master01 - ipa01.thaonv.local - 10.10.240.51
- Rep01- ipa02.thaonv.local - 10.10.240.186

## Cài đặt

Cài đặt keepalive trên cả 2 node

`yum install -y keepalived`

Kiểm tra trên cả 2 node

```
keepalived -version
systemctl status keepalived
```

Tạo file tracking trên cả 2 node

```
cd /etc/keepalived
mkdir ipa-monitor
touch vrrp_track_file
```

Tạo file script check trên cả 2 node

```
cat << EOF > /usr/local/bin/ipa-checker.sh
#!/bin/bash
service ipa status > /dev/null 2>&1
if [ $? -eq 0 ]
then
    echo 10 > /etc/keepalived/ipa-monitor/vrrp_track_file
else
    echo 0 > /etc/keepalived/ipa-monitor/vrrp_track_file
fi
EOF
```

Tạo file cấu hình keepalived trên cả 2 node

**Node Master01**

```
vrrp_script ipa_check {
   script "/usr/local/bin/ipa-checker.sh"
   interval 1
   timeout 5
   rise 3
   fall 3
}

vrrp_track_file track_ipa_file {
   file /etc/keepalived/ipa-monitor/vrrp_track_file
}

vrrp_instance ipa-master {
   track_script {
      ipa_check
   }
   track_file {
      track_ipa_file
      weight 1
   }
   state MASTER
   interface ens192
   virtual_router_id 55
   priority 240
   advert_int 1
   unicast_src_ip 10.10.240.51
   unicast_peer {
      10.10.240.186
   }
   authentication {
      auth_type PASS
      auth_pass clusterpass123
   }
   virtual_ipaddress {
      10.10.240.55/24
   }
}
```

**Node Rep01**

```
vrrp_script ipa_check {
   script "/usr/local/bin/ipa-checker.sh"
   interval 1
   timeout 5
   rise 3
   fall 3
}

vrrp_track_file track_ipa_file {
   file /etc/keepalived/ipa-monitor/vrrp_track_file
}
vrrp_instance ipa-rep1 {
   track_script {
      ipa_check
   }
   track_file {
      track_ipa_file
      weight 1
   }
   state MASTER
   interface ens192
   virtual_router_id 55
   priority 235
   advert_int 1
   unicast_src_ip 10.10.240.186
   unicast_peer {
      10.10.240.51
   }
   authentication {
      auth_type PASS
      auth_pass clusterpass123
   }
   virtual_ipaddress {
      10.10.240.55/24
   }

}
```

Start keepalived và kiểm tra ip vip trên cả 2 node

```
systemctl enable keepalived
systemctl restart keepalived
```