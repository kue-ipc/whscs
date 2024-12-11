# glusterのクライアント

EL8はGlusterがない？

sudo dnf install centos-release-storage-common
これで鍵が読み込まれる。
そのあとカスタムにおいてあるファイルをrepoに加えれば行けるはず。
現在、未テスト。

## memo

 centos-release-gluster10

```
[ipcadmin@whs-el8t1 ~]$ rpm -ql centos-release-gluster10
/etc/yum.repos.d/CentOS-Gluster-10.repo
/usr/lib/systemd/system-preset/75-gluster.preset
```

rpm -ql 
```
[ipcadmin@whs-el8t1 ~]$ rpm -ql centos-release-storage-common
/etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
/etc/yum.repos.d/CentOS-Storage-common.repo
```

EL8
```CentOS-Gluster-10.repo
# CentOS-Gluster-10.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/Storage for more
# information

[centos-gluster10]
name=CentOS-$stream - Gluster 10
#mirrorlist=http://mirrorlist.centos.org?arch=$basearch&release=$stream&repo=storage-gluster-10
baseurl=http://mirror.centos.org/$contentdir/$stream/storage/$basearch/gluster-10/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

[centos-gluster10-test]
name=CentOS-$stream - Gluster 10 Testing
baseurl=http://buildlogs.centos.org/centos/$stream/storage/$basearch/gluster-10/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
```

EL9
```CentOS-Gluster-10.repo
# CentOS-Gluster-10.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/Storage for more
# information

[centos-gluster10]
name=CentOS-$stream - Gluster 10
metalink=https://mirrors.centos.org/metalink?repo=centos-storage-sig-gluster-10-$stream&arch=$basearch
#baseurl=http://mirror.stream.centos.org/SIGs/$stream/storage/$basearch/gluster-10/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage

[centos-gluster10-test]
name=CentOS-$stream - Gluster 10 Testing
baseurl=http://buildlogs.centos.org/centos/$stream/storage/$basearch/gluster-10/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
```



https://mirrors.centos.org/metalink?repo=centos-storage-sig-gluster-10-9-stream&arch=x86_64


http://mirrorlist.centos.org?arch=x86_64&release=9-stream&repo=storage-gluster-10


http://mirror.stream.centos.org/SIGs/9-stream/storage/x86_64/gluster-10/


https://mirrors.centos.org/metalink?repo=centos-storage-sig-gluster-10-9&arch=x86_64

https://mirrors.centos.org/metalink?repo=centos-storage-sig-gluster-10-8&arch=x86_64
