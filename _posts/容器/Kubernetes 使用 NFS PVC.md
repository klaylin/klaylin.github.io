## 存储端配置
多个存储节点使用 DRBD & Pacemaker 配置高可用 nfs 服务
### 软件安装

```
apt install -y nfs-kernel-server pacemaker crmsh corosync ntpdate
```
[DRBD 相关安装](https://www.jianshu.com/p/ff45245d4db0)
### 服务配置
#### 同步时间

```
ntpdate -u ntp.api.bz
```
#### 创建 DRBD 卷
略
#### 创建 Corosync 集群
配置 /etc/corosync/corosync.conf 文件，内容如下
_*注意 bindnetaddr 与 nodelist 的地址*_
```
root@km99:~# cat /etc/corosync/corosync.conf
# Please read the corosync.conf.5 manual page
totem {
        version: 2

        # Corosync itself works without a cluster name, but DLM needs one.
        # The cluster name is also written into the VG metadata of newly
        # created shared LVM volume groups, if lvmlockd uses DLM locking.
        # It is also used for computing mcastaddr, unless overridden below.
        cluster_name: k8snfsserver

        # How long before declaring a token lost (ms)
        token: 3000

        # How many token retransmits before forming a new configuration
        token_retransmits_before_loss_const: 10

        # Limit generated nodeids to 31-bits (positive signed integers)
        clear_node_high_bit: yes

        # crypto_cipher and crypto_hash: Used for mutual node authentication.
        # If you choose to enable this, then do remember to create a shared
        # secret with "corosync-keygen".
        # enabling crypto_cipher, requires also enabling of crypto_hash.
        # crypto_cipher and crypto_hash should be used instead of deprecated
        # secauth parameter.

        # Valid values for crypto_cipher are none (no encryption), aes256, aes192,
        # aes128 and  3des. Enabling crypto_cipher, requires also enabling of
        # crypto_hash.
        crypto_cipher: none

        # Valid values for crypto_hash are  none  (no  authentication),  md5,  sha1,
        # sha256, sha384 and sha512.
        crypto_hash: none

        # Optionally assign a fixed node id (integer)
        # nodeid: 1234

        # interface: define at least one interface to communicate
        # over. If you define more than one interface stanza, you must
        # also set rrp_mode.
        interface {
                # Rings must be consecutively numbered, starting at 0.
                ringnumber: 0
                # This is normally the *network* address of the
                # interface to bind to. This ensures that you can use
                # identical instances of this configuration file
                # across all your cluster nodes, without having to
                # modify this option.
                bindnetaddr: 10.203.1.0
                # However, if you have multiple physical network
                # interfaces configured for the same subnet, then the
                # network address alone is not sufficient to identify
                # the interface Corosync should bind to. In that case,
                # configure the *host* address of the interface
                # instead:
                # bindnetaddr: 192.168.1.1
                # When selecting a multicast address, consider RFC
                # 2365 (which, among other things, specifies that
                # 239.255.x.x addresses are left to the discretion of
                # the network administrator). Do not reuse multicast
                # addresses across multiple Corosync clusters sharing
                # the same network.
                # mcastaddr: 239.255.1.1
                # Corosync uses the port you specify here for UDP
                # messaging, and also the immediately preceding
                # port. Thus if you set this to 5405, Corosync sends
                # messages over UDP ports 5405 and 5404.
                mcastport: 5405
                # Time-to-live for cluster communication packets. The
                # number of hops (routers) that this ring will allow
                # itself to pass. Note that multicast routing must be
                # specifically enabled on most network routers.
                ttl: 1
        }
}
nodelist { 
   node {
      ring0_addr: 10.203.1.99
      name: km99
   } 
   node {
      ring0_addr: 10.203.1.101
      name: ubuntu
   }  
}
logging {
        # Log the source file and line where messages are being
        # generated. When in doubt, leave off. Potentially useful for
        # debugging.
        fileline: off
        # Log to standard error. When in doubt, set to no. Useful when
        # running in the foreground (when invoking "corosync -f")
        to_stderr: no
        # Log to a log file. When set to "no", the "logfile" option
        # must not be set.
        to_logfile: no
        #logfile: /var/log/corosync/corosync.log
        # Log to the system log daemon. When in doubt, set to yes.
        to_syslog: yes
        # Log with syslog facility daemon.
        syslog_facility: daemon
        # Log debug messages (very verbose). When in doubt, leave off.
        debug: off
        # Log messages with time stamps. When in doubt, set to on
        # (unless you are only logging to syslog, where double
        # timestamps can be annoying).
        timestamp: on
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}

quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
        expected_votes: 2
}
```
重启服务

```
systemctl restart corosync
```
查看心跳线状态

```
corosync-cfgtool -s
```
#### NFS service 配置
使用 crm cof edit 命令打开编辑，配置信息如下
```
primitive nfs IPaddr \
        params ip=10.203.1.87
primitive nfs_start systemd:nfs-server \
        op start timeout=100 interval=0 \
        op stop timeout=100 interval=0
primitive nfsserver Filesystem \
        params device="/dev/drbd1002" directory="/home/share/minionfs" fstype=ext4 \
        op start timeout=60 interval=0 \
        op stop timeout=60 interval=0
location cli-prefer-nfs nfs role=Started inf: km99
colocation nfs_start_with_nfsserver inf: nfs_start nfsserver
order server_befor_start Mandatory: nfsserver nfs_start
colocation vip_with_nfs inf: nfs nfs_start
``` 
## 应用端配置
### 软件安装

```
apt install nfs-common
```
### pv & pvc

```
root@km99:~/k8syaml/nfs# cat minionfspv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minionfspv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /home/share/minionfs
    server: 10.203.1.87
```

```
root@km99:~/k8syaml/nfs# cat minionfspvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minionfspvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs
```
### Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minionfs-ha
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: minionfsha
  template:
    metadata:
      labels:
        app: minionfsha
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: lst
                operator: In
                values:
                - yyyyy
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - minionfsha
            topologyKey: kubernetes.io/hostname
      containers:
      - name: minionfsha
        image: minio/minio:RELEASE.2020-12-03T00-03-10Z
        #args:
        #- server
        #- /data
        command:
          - /bin/sh
          - '-ce'
          - /usr/bin/docker-entrypoint.sh minio -C /root/.minio/ server /data 
        ports:
        - containerPort: 9000
          protocol: TCP
        volumeMounts:
        - name: minio-volume
          mountPath: /data
      volumes:
      - name: minio-volume
        persistentVolumeClaim:
          claimName: minionfspvc
```
