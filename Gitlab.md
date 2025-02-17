### Install gitlab CentOS 7

```
sudo yum update -y
sudo yum install -y curl openssh-server ca-certificates postfix

curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash

sudo EXTERNAL_URL="http://gitlabsdsv.com" yum install -y gitlab-ce

# Restart config gitlab

sudo gitlab-ctl reconfigure

# Create certificates

mkdir -p /etc/gitlab/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/gitlab/ssl/gitlabsdsv.com.key \
  -out /etc/gitlab/ssl/gitlabsdsv.com.crt \
  -subj "/CN=gitlabsdsv.com/O=SDSV"

# Change config 
  
vi /etc/gitlab/gitlab.rb  

external_url 'https://gitlabsdsv.com'
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlabsdsv.com.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlabsdsv.com.key"

# Restart config gitlab

gitlab-ctl reconfigure

# Check password

cat /etc/gitlab/initial_root_password

https://gitlabsdsv.com

```