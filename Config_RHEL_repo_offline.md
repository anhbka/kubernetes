###  CentOS Linux 7 End of Life: June 30, 2024

- Fix CentOS 7 Repositories Not Working (2025)

```
sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/*.repo
sed -i 's/^#.*baseurl=http/baseurl=http/g' /etc/yum.repos.d/*.repo
sed -i 's/^mirrorlist=http/#mirrorlist=http/g' /etc/yum.repos.d/*.repo

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```

### 1. Create repo offline with DVD for Centos 7:

- Mount the CD/DVD ROM on any directory of your wish.
```
mount -o loop /dev/sr0 /mnt

* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

mv /etc/yum.repos.d/*.repo /tmp/
cat <<\EOF > /etc/yum.repos.d/local.repo
[LocalRepo]
name=LocalRepository
baseurl=file:///mnt
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF
yum clean all
yum repolist
```

### 2. Create repo offline with DVD for Rhel 7

```
mount -o loop /dev/sr0 /mnt

* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

cp /mnt/media.repo /etc/yum.repos.d/rhel7dvd.repo
chmod 644 /etc/yum.repos.d/rhel7dvd.repo

Edit the new repo file, changing the gpgcheck=0 setting to 1 and adding the following 3 lines:

cat <<\EOF > /etc/yum.repos.d/rhel7dvd.repo
[InstallMedia]
name=DVD for Red Hat Enterprise Linux 7.9 Server
mediaid=1359576196.686790
metadata_expire=-1
gpgcheck=1
cost=500
enabled=1
baseurl=file:///mnt/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

Clear the cache and check whether you can get the packages list from the DVD repo:

yum clean all
yum repolist enabled
yum update
```

### 3. Create repo offline with DVD for Rhel 8

Mount the downloaded RHEL installation ISO to a directory like /mnt:

```
# mount -o loop <downloaded iso name> /mnt

Example:
# mount -o loop rhel-server-8.8-x86_64-dvd.iso /mnt

If you use DVD media, you can mount like below.

mount -o loop /dev/sr0  /mnt

* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

Create new repo file like below. There are two repositories in RHEL 8, named BaseOS and AppStream.

cat << EOF > /etc/yum.repos.d/my.repo 
[dvd-BaseOS]
name=DVD for RHEL - BaseOS
baseurl=file:///mnt/BaseOS/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-AppStream]
name=DVD for RHEL - AppStream/
baseurl=file:///mnt/AppStream/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

Check whether you can get the packages list from the DVD repositories.

yum clean all
yum  --noplugins list

```

### 4. Create repo offline with DVD for Rhel 9:

```
Mount the RHEL Binary DVD ISO to a directory such as /mnt, e.g.:

mkdir -p  /mnt
mount -o loop rhel-baseos-9.0-x86_64-dvd.iso /mnt

mount /dev/sr0  /mnt
* Note: The Warning mount: /mnt/disc: WARNING: source write-protected, mounted read-only. is expected.

cat <<\EOF > /etc/yum.repos.d/rhel9dvd.repo 
[BaseOS]
name=BaseOS Packages Red Hat Enterprise Linux 9
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///mnt/BaseOS/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[AppStream]
name=AppStream Packages Red Hat Enterprise Linux 9
metadata_expire=-1
gpgcheck=1
enabled=1
baseurl=file:///mnt/AppStream/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

- Clear the cache and check whether you are able to get the packages from this DVD repository:

yum clean all
yum repolist

```