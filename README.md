# k3s cluster

## k3s

### ansible

#### deploy

```yaml
cat << EOF > k3s-config.yml
k3s_cluster_secret: Yourk3sClusterSecret
consul_encrypt_securepass: YourConsulSecurePasswd==
net_interface: eth1   # network interface to use
datacenter: mydc # consul datacenter
domain: consul # consul domain
EOF
```

```bash
ansible-playbook \
    --inventory inventory \
    --extra-vars "@k3s-config.yml" \
    ./k3s-consul.yml
```

#### dns round robin kubernetes api server

##### test

```bash
dig @192.168.100.201 -p 8600 k3s.service.mydc.consul A +short
dig @192.168.100.202 -p 8600 k3s.service.mydc.consul A +short
dig @192.168.100.203 -p 8600 k3s.service.mydc.consul A +short
```

##### dns server

###### dnsmasq

declare **consul** domain in `/etc/dnsmasq.conf`

Example:

```
server=/consul/192.168.100.201#8600
server=/consul/192.168.100.202#8600
server=/consul/192.168.100.203#8600
```

### troubleshooting

#### dqlite

##### symptoms

k3s server issues (showned in `journalctl -u k3s`):

```bash
Nov 23 13:34:36 <node> k3s[11896]: 00:38:04.914 [DEBUG]: segment 207068-207649time="2019-11-23T13:34:36.603215399Z" level=fatal msg="starting kubernetes: failed to start task"
```

stop k3s service: `systemctl restart k3s`

try to connect to a consistent node:

```bash
/usr/local/bin/k3s --debug server --server https://<node>:6443`
...
00:52:23.825 [WARN ]: discarding non contiguous segment 208485-208724
00:52:23.825 [ERROR]: found closed segment past last snapshot: 208485-208731
```

explain: **it's a corruption of segments which represents dqlite database**

##### fix

###### 1 - remove some dqlite files

```bash
rm /var/lib/rancher/k3s/server/db/state.dqlite/*
rm /var/lib/rancher/k3s/server/db/joined-*
```

###### 2 - (optionnal) required if impacted node has dqlite leader => `--cluster-init` in systemd k3s service

define a consistent node as new dqlite leader in order to prevent `no available dqlite leader server found` error

###### 2.a - update k3s systemd service on new dqlite leader node

edit /etc/systemd/system/k3s.service & change `--server https://<node>:6443` with `--cluster-init`
reload systemd daemon `systemctl daemon-reload`
restart k3s `systemctl restart k3s`

```bash
perl -i -p0e "s#--server\W.*?https://[^:]+:\d+#--cluster-init#se" /etc/systemd/system/k3s.service
systemctl daemon-reload
systemctl restart k3s
```

###### 2.b - update k3s systemd service on old dqlite leader node

```bash
perl -i -p0e "s#--cluster-init#--server https://<node>:6443#" /etc/systemd/system/k3s.service
systemctl daemon-reload
```

###### 3 - start k3s service

`systemctl restart k3s`
