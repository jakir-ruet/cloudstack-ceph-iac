## Welcome to Apache CloudStack, Ceph Storage IaC

### Architecture Overview

| Role                   | Hostname     | IP Address     | OS           |
| ---------------------- | ------------ | -------------- | ------------ |
| CloudStack Management  | cloudstack   | 192.168.68.151 | Ubuntu 20.04 |
| Ceph MON + OSD Storage | ceph-storage | 192.168.68.152 | Ubuntu 20.04 |

- Hypervisor: KVM
- CloudStack: v4.20.1.0
- Ceph: Reef 18.x
- Storage: Ceph RBD (block)

### CloudStack Management Node – `192.168.68.151`

#### Set Hostname and Update

```bash
sudo hostnamectl set-hostname cloudstack
sudo apt update && sudo apt dist-upgrade -y
echo -e "192.168.68.151 cloudstack\n192.168.68.152 ceph-storage" | sudo tee -a /etc/hosts
```

#### Install Base Dependencies

```bash
sudo apt install -y wget curl gnupg lsb-release openjdk-11-jdk \
qemu-kvm libvirt-daemon-system libvirt-clients virtinst
```

#### Install MariaDB and Configure

```bash
sudo apt install -y mariadb-server
sudo mysql_secure_installation
```

#### Edit MariaDB config `[mysqld]`

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```bash
bind-address = 127.0.0.1
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
max_connections = 1000
innodb_file_per_table = 1
```

```bash
sudo systemctl restart mariadb
```

#### Create CloudStack Database

```bash
sudo mysql -u root -p
```

```bash
CREATE DATABASE cloud DEFAULT CHARACTER SET utf8mb4;
GRANT ALL PRIVILEGES ON cloud.* TO 'cloud'@'localhost' IDENTIFIED BY 'cloudpass';
FLUSH PRIVILEGES;
```

`cloud:cloudpass@localhost` is the DB URI fragment for CloudStack.

#### Add CloudStack Repository and Install

```bash
wget -qO- https://download.cloudstack.org/release.asc | sudo apt-key add -
echo "deb http://download.cloudstack.org/ubuntu focal 4.20" | sudo tee /etc/apt/sources.list.d/cloudstack.list
sudo apt update
sudo apt install -y cloudstack-management cloudstack-common
```

#### Set JAVA_HOME

```bash
echo "JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))" | sudo tee -a /etc/environment
source /etc/environment
```

#### Setup CloudStack

```bash
cloudstack-setup-databases cloud:cloudpass@localhost --deploy-as=root
cloudstack-setup-management
cloudstack-sysvmtemplate -m /var/www/html
```

#### Access CloudStack UI

```bash
URL: http://192.168.68.151:8080/client
Username: admin
Password: password (you will be prompted to change)
```

### Ceph Storage Node – `192.168.68.152`

#### Set Hostname and Update

```bash
sudo hostnamectl set-hostname ceph-storage
sudo apt update && sudo apt dist-upgrade -y
echo -e "192.168.68.151 cloudstack\n192.168.68.152 ceph-storage" | sudo tee -a /etc/hosts
```

#### Install cephadm and Bootstrap Ceph

```bash
sudo apt install -y curl gnupg
curl -fsSL https://download.ceph.com/keys/release.asc | sudo apt-key add -
echo "deb https://download.ceph.com/debian-reef $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/ceph.list
sudo apt update
sudo apt install -y cephadm

sudo cephadm bootstrap \
  --mon-ip 192.168.68.152 \
  --initial-dashboard-user admin \
  --initial-dashboard-password adminpass
```

Access the Ceph Dashboard: `https://192.168.68.152:8443`

#### Add OSD (Example for all available devices)

```bash
sudo ceph orch device ls
sudo ceph orch apply osd --all-available-devices
```

#### Create RBD Pool and Client Key

```bash
sudo ceph osd pool create cloudstack 128
sudo rbd pool init cloudstack

sudo ceph auth get-or-create client.cloudstack \
  mon 'profile rbd' osd 'profile rbd pool=cloudstack' \
  -o /etc/ceph/ceph.client.cloudstack.keyring
```

##### Encode the keyring

```bash
sudo base64 -w0 /etc/ceph/ceph.client.cloudstack.keyring
```

#### Install Ceph Client Tools

```bash
sudo apt install -y ceph-common
sudo ceph config generate-minimal-conf | sudo tee /etc/ceph/ceph.conf
```

### Connect CloudStack to Ceph RBD

#### Add KVM Host in CloudStack

```bash
CloudStack UI → Infrastructure → Zones → Add Zone
```

- Hypervisor: `KVM`
- Host: `192.168.68.152`

#### Add Primary Storage (RBD)

```bash
CloudStack UI → Infrastructure → Primary Storage → Add
```

| Field           | Value                            |
| --------------- | -------------------------------- |
| Name            | Ceph-RBD                         |
| Protocol        | RBD                              |
| Server          | 192.168.68.152:6789              |
| Path            | cloudstack                       |
| RBD ID          | cloudstack                       |
| RBD Secret      | (Base64-encoded client keyring)  |
| RBD Secret UUID | (UUID from `uuidgen`, see below) |

#### Create libvirt Secret on KVM Host

```bash
UUID=$(uuidgen)
cat > /tmp/secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>$UUID</uuid>
  <usage type='ceph'>
    <name>client.cloudstack secret</name>
  </usage>
</secret>
EOF

sudo virsh secret-define --file /tmp/secret.xml
sudo virsh secret-set-value --secret $UUID \
  --base64 $(sudo base64 -w0 /etc/ceph/ceph.client.cloudstack.keyring)
```

- Use the `same UUID` in the CloudStack UI.

#### Placeholder Replacements

| Placeholder                 | Value                       | Description                            |
| --------------------------- | --------------------------- | -------------------------------------- |
| `bind-address`              | `127.0.0.1`                 | MariaDB binds only to localhost        |
| `cloud:cloudpass@localhost` | CloudStack DB credentials   | Used in DB setup and CloudStack config |
| `user@ceph-mon`             | `cloudstack@192.168.68.152` | Ceph client name and MON IP            |
| `your_pool`                 | `cloudstack`                | Name of RBD pool used for CloudStack   |
