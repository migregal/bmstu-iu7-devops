## Connection

Create ssh key after first connect to skip password prompt in future. Replace values between `<>` with correct ones.
```console
$ ssh-keygen -t rsa
$ ssh-copy-id -i ~/.ssh/<id_rsa>.pub <user>@<host>
```

## VM1

```console
$ sudo su -m
```

### Docker

Install pre-requirements.
```console
$ sudo apt update
$ sudo apt install curl software-properties-common ca-certificates apt-transport-https -y
```

Install necessary certs.
```console
$ sudo mkdir -m 0755 -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Finally install docker packages.
```console
$ sudo apt update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Gitlab Runner

Install gitlab-runner package.

```console
$ sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

$ sudo chmod +x /usr/local/bin/gitlab-runner

$ sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

$ sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
$ sudo gitlab-runner start
```

Register gitlab-runner to your gitlab instance. Insert your host and registration token values.
```console
$ sudo gitlab-runner register --url $HOST --registration-token $REGISTRATION_TOKEN
```

### Port Forwarding

Install pre-requirements.
```console
$ apt-get install iptables-persistent
```

Enable forwarding for IPv4.
```console
$ echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/gateway.conf
$ sysctl -p /etc/sysctl.d/gateway.conf
```

Configure port forwarding for default HTTP port. `ens15` - is an interface name so it may differ from one VM to another.

```console
$ sudo iptables -t nat -A PREROUTING -i ens15 -p tcp -d 192.168.3.23 --dport 80 -j DNAT --to 192.168.3.25:80

$ sudo iptables -t nat -A POSTROUTING -o ens15 -j MASQUERADE

$ netfilter-persistent save
```

## VM2

```console
$ sudo su -m
```

Install nginx.

```console
sudo apt update
sudo apt install nginx
```

Install PostgreSQL.

```
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql.service
```

### Enable ufw firewall

```
sudo ufw allow from 192.168.3.23 to any port 22
sudo ufw allow from 192.168.3.23 to any port 80
```
