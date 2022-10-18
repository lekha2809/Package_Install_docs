# ETCD

Etcd is a key-value storage

## Installation
Step1: Download and Install the etcd Binaries.

```bash
curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest \
  | grep browser_download_url \
  | grep linux-amd64 \
  | cut -d '"' -f 4 \
  | wget -i -
```

Unarchive the file and move binaries to /usr/local/bin directory.

```bash
tar xvf etcd-v*.tar.gz
cd etcd-*/
sudo mv etcd* /usr/local/bin/
cd ..
rm -rf etcd*
```

Check etcd and etcdctl version.

```bash
$ etcd --version
etcd Version: 3.5.2
Git SHA: 99018a77b
Go Version: go1.16.3
Go OS/Arch: linux/amd64

$ etcdctl version
etcdctl version: 3.5.2
API version: 3.5

$ etcdutl version
etcdutl version: 3.5.2
API version: 3.5
```

Create the etcd.service systemd unit file
```bash
cat << EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Decription=etcd service
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
Environment=ETCD_DATA_DIR=/var/lib/etcd
Environment=ETCD_NAME=%m
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```
