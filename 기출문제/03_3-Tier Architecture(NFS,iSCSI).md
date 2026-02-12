# Multi-tier Architecture
---

# 1. Multi-tier Architecture

## 1.1 Multi-tier Architecture란?

>**Multi-tier Architecture** 는 애플리케이션을 다계층으로 분리하여 구성하는 시스템 아키텍처 모델입니다. 이를 통해 시스템의 유연성과 확장성을 크게 향상시킵니다.

## 1.2 Multi-tier Architecture 구성도

![topology](./images/iSCSI.png) 

# 2. Vagrantfile

`Vagrantfile`
```ruby  
# -*- mode: ruby -*-
# vi: set ft=ruby :

# common variables
BOX_NAME = "nobreak-labs/rocky-9"
MEMORY = "1024"
CPUS = 2

# IP config
NODES = {
  "WP-LB01" => { 
    networks: [
      { ip: "192.168.56.10" },
      { ip: "192.168.57.10" }
    ]
  },
  "WP-WEB01" => { 
    networks: [
      { ip: "192.168.57.11" },
      { ip: "192.168.58.11" }
    ]
  },
  "WP-WEB02" => { 
    networks: [
      { ip: "192.168.57.12" },
      { ip: "192.168.58.12" }
    ]
  },
  "WP-DB01" => { 
    networks: [
      { ip: "192.168.58.13" }
    ]
  },
  "WP-NFS01" => { 
    networks: [
      { ip: "192.168.57.14" }
    ],
    disk: { size: "10GB", name: "sdb" }
  },
}

Vagrant.configure("2") do |config|
  
  NODES.each do |node_name, node_config|
    config.vm.define node_name do |node|
      node.vm.box = BOX_NAME
      
      # Network config
      node_config[:networks].each do |network|
        node.vm.network "private_network", ip: network[:ip]
      end
      
      node.vm.synced_folder ".", "/vagrant", disabled: true
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = MEMORY
        vb.cpus = CPUS
        vb.name = node_name
      end
      
      node.vm.hostname = node_name
      
      # Add disk
      if node_config[:disk]
        node.vm.disk :disk, size: node_config[:disk][:size], name: node_config[:disk][:name]
      end
    end
  end
end
```

# 3. iSCSI 서버 구축

## 3.1 iSCSI 란?

>**iSCSI** 는 IP 네트워크를 통해 데이터 저장소를 연결하고 관리하는 전송 계층 프로토콜로써 원격 저장 장치를 로컬 시스템에 직접 연결된 디바이스처럼 사용할 수 있게 해주는 블록 기반 스토리지 네트워킹입니다.

## 3.2 패키지 설치

```bash
sudo dnf install -y targetcli
```
- iSCSI 서버 환경을 설정하고 관리하기 위한 관리 도구인 targetcli를 설치합니다.

## 3.3 iSCSI Target 서비스 활성화 및 시작

```bash
sudo systemctl enable target --now
```
iSCSI Target 서비스를 지금 즉시 실행과 동시에 부팅 시 자동 시작할 수 있도록 설정합니다.

## 3.4 iSCSI Target Backstore 생성

```bash
sudo targetcli /backstores/block create name=iscsi_db dev=/dev/sdb
```
- 물리적 디스크 장치인 /dev/sdb를 블록 스토리지로 등록합니다.
  
## 3.5 iSCSI Target IQN 생성

```bash
sudo targetcli /iscsi create iqn.2026-02.local.wordpress:storage
```
- 네트워크에서 iSCSI 서버를 식별하기 위한 고유 주소인 IQN을 생성합니다.

## 3.6 LUN 생성

```bash
sudo targetcli /iscsi/iqn.2026-02.local.wordpress:storage/tpg1/luns \ 
create /backstores/block/iscsi_db
```
- 생성한 IQN과 실제 물리 디스크를 연결합니다.

## 3.7 접근 제어 목록 설정

```bash
sudo targetcli /iscsi/iqn.2026-02.local.wordpress:storage/tpg1/acls \ 
create iqn.2026-02.local.wordpress:db01
```
- DB 서버만 이 스토리지에 접근할 수 있도록 접근 제어 목록을 설정합니다.

## 3.8 포털 생성

```bash
sudo targetcli /iscsi/iqn.2026-02.local.wordpress:storage/tpg1/portals \ 
create 192.168.58.15 3260
```
- iSCSI 서버가 클라이언트의 접속을 기다릴 IP 주소와 port(3260)를 지정합니다.

## 3.9 iSCSI Target 구성 상태 확인

```bash
sudo targetcli ls
```

```text
o- / ..................................................................... [...]
  o- backstores .......................................................... [...]
  | o- block .............................................. [Storage Objects: 1]
  | | o- iscsi_db .................... [/dev/sdb (10.0GiB) write-thru activated]
  | |   o- alua ............................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ................... [ALUA state: Active/optimized]
  | o- fileio ............................................. [Storage Objects: 0]
  | o- pscsi .............................................. [Storage Objects: 0]
  | o- ramdisk ............................................ [Storage Objects: 0]
  o- iscsi ........................................................ [Targets: 1]
  | o- iqn.2026-02.local.wordpress:storage ........................... [TPGs: 1]
  |   o- tpg1 ........................................... [no-gen-acls, no-auth]
  |     o- acls ...................................................... [ACLs: 1]
  |     | o- iqn.2026-02.local.wordpress:db01 ................. [Mapped LUNs: 1]
  |     |   o- mapped_lun0 .......................... [lun0 block/iscsi_db (rw)]
  |     o- luns ...................................................... [LUNs: 1]
  |     | o- lun0 ............... [block/iscsi_db (/dev/sdb) (default_tg_pt_gp)]
  |     o- portals ................................................ [Portals: 1]
  |       o- 0.0.0.0:3260 ................................................. [OK]
  o- loopback ..................................................... [Targets: 0]
```

- Backstore 등록, Target 생성, 접근 제어 설정 등이 정상적으로 완료되었는지 확인합니다.

## 3.10 방화벽 설정

### 3.10.1 방화벽 적용

```bash
sudo firewall-cmd --add-service=iscsi-target --permanent && \
sudo firewall-cmd --reload
```
- iSCSI port(3260)를 영구 개방하고 변경된 방화벽 규칙을 즉시 시스템에 반영합니다.

### 3.10.2 방화벽 확인

```bash
sudo firewall-cmd --list-services
```
- 방화벽 서비스 목록에 iscsi-target이 정상적으로 등록되었는지 확인합니다.

# 4. DB 서버 구축

## 4.1 DB 란?

>**데이터베이스(Database, DB)** 란 여러 사용자나 애플리케이션이 동시에 공유할 수 있도록 체계적으로 구조화하여 저장한 데이터들의 집합체입니다.

## 4.2 패키지 설치

```bash
sudo dnf install -y iscsi-initiator-utils mysql-server
``` 
- iSCSI 클라이언트 패키지와 MySQL 서버 패키지를 시스템에 설치합니다.

## 4.3 iSCSI 클라이언트 서비스 활성화 및 시작

```bash
sudo systemctl enable iscsid --now
```
- iSCSI 클라이언트 서비스를 지금 즉시 실행과 동시에 부팅 시 자동 시작할 수 있도록 설정합니다.

## 4.4 iSCSI 구성 및 Target 연동

### 4.4.1 IQN 설정

```bash
sudo tee /etc/iscsi/initiatorname.iscsi << EOF
InitiatorName=iqn.2026-02.local.wordpress:db01
EOF
```
- DB 서버의 고유 IQN 이름을 설정합니다. 앞서 iSCSI 서버의 접근 제어 목록에서 설정한 이름과 동일해야 접속이 가능합니다.

### 4.4.2  Target 검색

```bash
sudo iscsiadm -m discovery -t sendtargets -p 192.168.58.15
```
- iSCSI 서버의 IP를 조회하여 제공하는 타겟 목록을 확인합니다.

### 4.4.3  Target 로그인

```bash
sudo iscsiadm -m node -T iqn.2026-02.local.wordpress:storage \ 
-p 192.168.58.15 --login
```
- 검색된 타겟에 로그인하면 원격 디스크가 로컬 장치처럼 인식됩니다.

## 4.5 디스크 볼륨 구성

### 4.5.1 디스크 포맷

```bash
sudo mkfs.xfs /dev/sdb1
```
- 네트워크를 통해 연결된 새 디스크 장치를 XFS 파일 시스템으로 포맷합니다.

### 4.5.2 디스크 마운트

```bash
sudo mount /dev/sdb /var/lib/mysql
```
- 포맷된 디스크를 MySQL의 데이터 저장 기본 경로인 `/var/lib/mysql`에 마운트합니다.

### 4.5.3 자동 마운트 설정

```bash
sudo tee /etc/fstab << EOF
/dev/sdb /var/lib/mysql xfs defaults,_netdev 0 0
EOF
```

### 4.5.4 소유권 변경

```bash
sudo chown -R mysql:mysql /var/lib/mysql
```
- MySQL 서비스가 해당 디렉터리에 데이터를 기록할 수 있도록 소유권을 변경합니다.

## 4.6 MySQL 서비스 활성화 및 시작

```bash   
sudo systemctl enable mysqld --now
```
- 부팅 시 자동 시작 등록과 함께 지금 즉시 서비스를 실행합니다.

## 4.7 MySQL 초기 보안 설정

```bash  
sudo mysql_secure_installation  
```
- MySQL 설치 후 보안상 취약할 수 있는 기본 설정들을 한 번에 개선할 수 있는 명령어입니다. 입력 후 아래와 같은 대화형 스크립트가 나옵니다.

### 4.7.1 익명 사용자 제거  
Remove anonymous users? (Press y|Y for Yes, any other key for No) : `y`  
- 사용자 인증 과정을 거치지 않은 비인가자의 접근을 원천 차단하기 위해 익명 계정을 삭제합니다.

### 4.7.2 root 원격 접속 차단  
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : `y`
- DB(데이터베이스)의 모든 권한을 가진 root 계정이 외부 네트워크에서 접속하는 것을 차단하여 보안 위협을 최소화합니다.
- 관리자 작업은 서버 로컬에서만 수행하거나, 별도의 일반 사용자 계정을 생성하여 원격 접속하도록 유도하는 것이 보안 원칙입니다.

### 4.7.3 테스트 DB 삭제  
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : `y`
- 기본으로 생성되는 test DB(데이터베이스)는 누구나 접근할 수 있도록 권한이 느슨하게 설정되어 있어 최소 권한 및 최소 기능의 원칙에 따라 기본 DB(데이터베이스)를 삭제하여 공격 지점을 줄입니다.

### 4.7.4 권한 테이블 즉시 적용  
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : `y`
- 재시작되기 전까지 변경된 보안 설정이 비활성화 상태이므로 보안 규칙들이 즉시 반영되어 적용되도록 합니다.

## 4.8 MySQL 접속 및 설정

```bash
mysql -u root -p
```
- MySQL에 접속하는 명령어이며, u 옵션은 계정의 이름이고 p 옵션은 패스워드입니다.

### 4.8.1 WordPress DB 생성

```sql
CREATE DATABASE wordpress;
```
- wordpress DB(데이터베이스)를 생성합니다.

### 4.8.2 사용자 생성

```sql
CREATE USER 'wpuser'@'192.168.58.%' IDENTIFIED BY 'qwer1234';
```
- 사용자를 생성하고 네트워크 대역과 암호를 설정합니다.
- 192.168.58.%에서 %는 192.168.58로 시작하는 모든 IP 주소에서의 접속을 허용한다는 뜻입니다.

### 4.8.3 WordPress DB에 대한 권한 부여

```sql
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'192.168.58.%';
```
- 사용자 wpuser에 해당 DB(데이터베이스)에 대한 모든 권한을 부여합니다.

### 4.8.4 권한 즉시 적용

```sql
FLUSH PRIVILEGES;
```
- 변경된 권한을 즉시 적용합니다.

### 4.8.5 DB 목록 확인

```sql
SHOW DATABASES;
```

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress          |
+--------------------+
```
- wordpress DB(데이터베이스)가 생성된 것을 확인할 수 있습니다.

### 4.8.6 계정 목록 확인

```sql
SELECT user, host FROM mysql.user;
```

```
+------------------+--------------+
| user             | host         |
+------------------+--------------+
| wpuser           | 192.168.58.% |
| mysql.infoschema | localhost    |
| mysql.session    | localhost    |
| mysql.sys        | localhost    |
| root             | localhost    |
+------------------+--------------+
```
- 사용자와 네트워크 대역을 확인할 수 있습니다.

### 4.8.7 권한 설정 확인

```sql
SHOW GRANTS FOR 'wpuser'@'192.168.58.%';
```

```
+------------------------------------------------------------------+
| Grants for wpuser@192.168.58.%                                   |
+------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `wpuser`@`192.168.58.%`                    |
| GRANT ALL PRIVILEGES ON `wordpress`.* TO `wpuser`@`192.168.58.%` |
+------------------------------------------------------------------+
```
- 사용자 wpuser에 모든 권한이 부여된 것을 확인할 수 있습니다.

## 4.9 방화벽 설정

### 4.9.1 방화벽 적용

```bash
sudo firewall-cmd --add-service=mysql --permanent && \ 
sudo firewall-cmd --reload
```
- MySQL port(3306)를 영구 개방하고 변경된 방화벽 규칙을 즉시 시스템에 반영합니다.

### 4.9.2 방화벽 확인

```bash
sudo firewall-cmd --list-services
```
- 방화벽 서비스 목록에 mysql이 정상적으로 등록되었는지 확인합니다.

# 4. Application Tier (WEB,NFS) 서버 구축

## 4.1 NFS 서버 구축 

### 4.1.1 파티션 생성
```
bash
sudo fdisk /dev/sdb
```
- 파티션 관리 도구인 fdisk를 사용하여 두 번째 하드디스크(/dev/sdb)의 파티션 테이블을 편집합니다.
- 명령어 입력후 아래와 같은 대화형 스크립트가 나옵니다.

#### 4.1.1.1 새 파티션 생성
Command (m for help): `n`
- 하드디스크 내에 데이터를 담을 신규 파티션을 생성합니다. 

#### 4.1.1.2 파티션 타입 선택 

p   primary (0 primary, 0 extended, 4 free)  
e   extended  
Select (default p): `p`
- 부팅이 가능하고 데이터를 직접 저장할 수 있는 기본 단위인 주 파티션(Primary)을 선택합니다.

#### 4.1.1.3 파티션 번호 선택
Partition number (1-4, default 1): `1`
- 생성할 파티션에 번호를 부여하는 과정으로 첫 번째 파티션이므로 관례상 1번을 할당하며 시스템에서 /dev/sdb1이라는 명칭으로 인식하게 됩니다.

#### 4.1.1.4 시작 섹터 설정
First sector (2048-..., default 2048):
- 디스크의 물리적 시작 지점을 정합니다.

#### 4.1.1.5 섹터 크기 설정
Last sector, +/-sectors or +/-size{K,M,G,T,P} (...., default ....):
- 파티션이 끝나는 지점을 지정 즉 용량의 크기를 할당할 수 있습니다.

#### 4.1.1.6 저장 및 종료
Command (m for help): `w`
- 파티션 설정을 영구적으로 저장하고 도구를 종료합니다.

### 4.1.2 디스크 포맷 및 마운트

#### 4.1.2.1 디스크 포맷
```
bash
sudo mkfs.xfs /dev/sdb1
```
- 생성한 파티션(/dev/sdb1)에 데이터를 저장할 수 있는 규칙인 XFS 파일 시스템으로 포맷(초기화)합니다.

#### 4.1.2.2 마운트 포인트 생성
```
bash
sudo mkdir -p /mnt/nfs
```
- 하드디스크라는 물리적 장치를 리눅스 디렉토리 구조와 연결하기 위해 빈 디렉토리를 생성합니다.

#### 4.1.2.3 디스크 마운트
```
bash
sudo mount /dev/sdb1 /mnt/nfs
```
- 포맷된 파티션을 생성한 디렉토리(/mnt/nfs)에 마운트합니다.

#### 4.1.2.4 자동 마운트 설정
```
bash
echo '/dev/sdb1 /mnt/nfs xfs defaults 0 0' | sudo tee -a /etc/fstab
```

```
bash
vi /etc/fstab
```
```
test
/dev/sdb1 /mnt/nfs xfs defaults 0 0
```
- 시스템 설정 파일인 /etc/fstab에 정보를 등록하여 부팅 시마다 자동으로 디스크가 지정된 경로에 마운트되도록 설정합니다.

### 4.1.3 NFS 설치 및 설정

#### 4.1.3.1 패키지 설치
```
bash
sudo dnf install -y nfs-utils
```
- 리눅스 시스템 간 파일 공유를 가능하게 하는 NFS 패키지를 설치합니다.

#### 4.1.3.2 WordPress 공유 디렉터리 생성
```
bash
sudo mkdir -p /mnt/nfs/wordpress
```
- 워드프레스의 정적 파일이 실제로 저장되고 공유될 NFS 전용 루트 디렉터리를 생성합니다.

#### 4.1.3.3 권한 설정
```
bash
sudo chmod -R 775 /mnt/nfs/wordpress
```
- 여러 웹 서버가 해당 디렉터리에 파일을 쓰고 수정할 수 있도록 하기 위해 권한을 부여합니다.

### 4.1.4 NFS Export 설정 및 서비스 활성화

#### 4.1.4.1 /etc/exports 설정
```
bash
sudo tee /etc/exports << EOF
/mnt/nfs/wordpress 192.168.57.11(rw,sync,no_root_squash)
/mnt/nfs/wordpress 192.168.57.12(rw,sync,no_root_squash)
EOF
```

```
bash
vi /etc/exports
```

```
text
/mnt/nfs/wordpress 192.168.57.11(rw,sync,no_root_squash)
/mnt/nfs/wordpress 192.168.57.12(rw,sync,no_root_squash)
```
- 클라이언트 서버가 접근할 수 있도록 공유 디렉터리 접근 권한 설정을 등록합니다.

#### 4.1.4.2 Export 적용
```
bash
sudo exportfs -rav
```
- /etc/exports의 변경 내용을 서비스 재시작 없이 즉시 커널의 export 테이블에 반영합니다.

#### 4.1.4.3 적용 확인
```
bash
sudo exportfs -v
```
- 현재 공유 중인 목록을 상세히 확인합니다.

#### 4.1.4.4 NFS 서비스 활성화
```
bash
sudo systemctl enable nfs-server --now
```
- 부팅 시 NFS 서버가 자동 시작되도록 설정하고, 현재 시점부터 서비스를 가동합니다.

### 4.1.5 방화벽 설정 및 확인

#### 4.1.5.1 NFS 방화벽 영구 적용
```
bash
sudo firewall-cmd --add-service=nfs --permanent
```
- 방화벽 설정을 변경하여 NFS port(2049)를 개방합니다.
- permanent 옵션으로 영구 적용합니다.

#### 4.1.5.2 mountd 방화벽 영구 적용
```
bash
sudo firewall-cmd --add-service=mountd --permanent
```
- 방화벽 설정을 변경하여 mountd Port를 개방합니다.
- mountd는 실제 파일 시스템 마운트 요청을 처리하는 서비스입니다.
- permanent 옵션으로 영구 적용합니다.

#### 4.1.5.3 rpc-bind 방화벽 영구 적용
```
bash
sudo firewall-cmd --add-service=rpc-bind --permanent
```
- 방화벽 설정을 변경하여 rpc-bind Port를 개방합니다.
- rpc-bind는 클라이언트가 서버의 서비스를 찾을 수 있게 도와주는 port mapper입니다.
- permanent 옵션으로 영구 적용합니다.

#### 4.1.5.4 방화벽 설정 확인
```
bash
sudo firewall-cmd --list-services
```
- 방화벽의 Port가 정상적으로 개방되었는지 확인합니다.

### 4.1.6 SELinux NFS export 허용
```
bash
sudo setsebool -P nfs_export_all_rw 1
```
- SELinux정책상 기본적으로 차단되어 있는 읽기/쓰기 공유 기능을 허용하도록 보안 정책을 변경합니다.

## 4.2 WEB 서버 구축 

### 4.2.1 패키지 설치
```
bash
sudo dnf install -y httpd php php-mysqlnd mysql nfs-utils
```
- 웹 서비스 운영을 위한 APM(Apache, PHP, MySQL)과 NFS 패키지를 설치합니다.

### 4.2.2 Apache 서비스 활성화 및 시작
```
bash
sudo systemctl enable httpd --now
```
- 시스템 부팅 시 웹 서버가 자동으로 실행되도록 enable 옵션을 설정하고 now 옵션을 통해 현재 상태에서 서비스를 즉시 가동합니다.

### 4.2.3 WordPress 다운로드 및 배치

#### 4.2.3.1 다운로드
```
bash
curl -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
```
- 워드프레스 공식 홈페이지에서 가장 최신 버전를 아카이브 파일 형태로 다운받습니다.

#### 4.2.3.2 압축 해제
```
bash
sudo tar -xvf wordpress.tar.gz -C /var/www/html
```
- 웹 서버의 기본 문서 루트 경로인 /var/www/html 위치에 내려받은 소스 파일의 압축을 해제합니다.

#### 4.2.3.3 소유권 설정
```
bash
sudo chown -R apache:apache /var/www/html/wordpress
```
- apache 사용자가 워드프레스 디렉토리에 직접 접근하여 파일을 읽고 쓸 수 있도록 소유권을 변경합니다.

#### 4.2.3.4 권한 부여
```
bash
sudo chmod -R 775 /var/www/html/wordpress
```
- 원활한 플러그인 설치 및 미디어 업로드를 위해 권한을 부여합니다.

### 4.2.4 WordPress 설정 및 DB 연동

#### 4.2.4.1 wp-config.php 설정 파일 생성
```
bash
sudo cp /var/www/html/wordpress/wp-config-sample.php \ 
/var/www/html/wordpress/wp-config.php
```
- 워드프레스에서 제공하는 기본 샘플 설정 파일을 실제 설정 파일로 복사합니다.

#### 4.2.4.2 DB 정보 입력
```
bash
sudo sed -i \
  "s/define( 'DB_NAME', '.*' );/define( 'DB_NAME', 'wordpress' );/" \
  /var/www/html/wordpress/wp-config.php

sudo sed -i \
  "s/define( 'DB_USER', '.*' );/define( 'DB_USER', 'wpuser' );/" \
  /var/www/html/wordpress/wp-config.php

sudo sed -i \
  "s/define( 'DB_PASSWORD', '.*' );/define( 'DB_PASSWORD', 'qwer1234' );/" \
  /var/www/html/wordpress/wp-config.php

sudo sed -i \
  "s/define( 'DB_HOST', '.*' );/define( 'DB_HOST', '192.168.58.13' );/" \
  /var/www/html/wordpress/wp-config.php
```

```
bash
sudo vi /var/www/html/wordpress/wp-config.php
```

```
text
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'qwer1234' );
define( 'DB_HOST', '192.168.58.13' );
```

- 워드프레스가 MySQL서버에 접속할 수 있도록 DB 이름, 계정 이름, 비밀번호, DB 서버의 IP 주소를 등록합니다.

#### 4.2.4.3 WordPress VirtualHost 생성
```
bash
sudo tee /etc/httpd/conf.d/wordpress.conf << EOF
<VirtualHost *:80>
    ServerName wordpress.local
    DocumentRoot /var/www/html/wordpress

    <Directory /var/www/html/wordpress>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
EOF
```

```
bash
sudo vi /etc/httpd/conf.d/wordpress.conf
```

```
text
<VirtualHost *:80>
    ServerName wordpress.local
    DocumentRoot /var/www/html/wordpress

    <Directory /var/www/html/wordpress>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```
- wordpress.local으로 들어오는 요청을 워드프레스 디렉토리로 정확히 연결해 주는 VirtualHost 설정을 생성합니다.
  - Require all granted: 사이트 접속 허용
  - AllowOverride All: 내부 동작 허용

### 4.2.5 NFS 마운트 설정

#### 4.2.5.1 WordPress 디렉터리 백업
```
bash
sudo cp -r /var/www/html/wordpress /var/www/html/wordpress.bak
```
- 공유 디렉터리를 마운트하면 기존 로컬 파일이 가려지기 때문에 이를 방지하기 위해 wordpress폴더를 백업합니다.

#### 4.2.5.2 NFS 마운트
```
bash
sudo mount -t nfs 192.168.57.14:/mnt/nfs/wordpress /var/www/html/wordpress
```
- 공유 디렉터리를 웹 서버의 워드프레스 경로와 마운트합니다. 이를 통해 웹 서버가 NFS 저장소를 바라보게 됩니다.

#### 4.2.5.3 WordPress 디렉터리 복사
```
bash
sudo rsync -av /var/www/html/wordpress.bak /var/www/html/wordpress
```
- 백업해 두었던 워드프레스 파일을 마운트된 공유 디렉터리에 복사합니다.이때 rsync의 -av 옵션을 사용하여 파일의 권한과 속성을 그대로 유지하며 복사합니다.

#### 4.2.5.4 자동 마운트 설정
```
bash
echo '192.168.57.14:/mnt/nfs/wordpress /var/www/html/wordpress nfs defaults,_netdev 0 0' | sudo tee -a /etc/fstab
```

```
bash
vi /etc/fstab
```

```
text
192.168.57.14:/mnt/nfs/wordpress /var/www/html/wordpress nfs defaults,_netdev 0 0
```
- 부팅 시 웹 서버가 자동으로 NFS 스토리지에 연결되도록 설정합니다.
- `_netdev` 옵션은 부팅 시 네트워크 서비스가 수행되고 나서 `/etc/fstab`에 있는 항목을 마운트 시도합니다.

#### 4.2.5.5 마운트 상태 확인
```
bash
df -h | grep wordpress
```
- 파일 시스템 목록을 조회하여 NFS 경로가 정상적으로 워드프레스 디렉터리에 연결되었는지 확인합니다.

### 4.2.6 방화벽 설정

#### 4.2.6.1 방화벽 적용

```bash
sudo firewall-cmd --add-service=http --permanent && \ 
sudo firewall-cmd --reload
```
- 외부 사용자가 로드밸런서를 통해 웹 서비스에 접속할 수 있도록 HTTP port(80) 를 개방합니다.

#### 4.2.6.3 방화벽 확인

```bash
sudo firewall-cmd --list-services
```
- 방화벽 서비스 목록에 http가 정상적으로 등록되었는지 확인합니다.

### 4.2.7 SELinux 설정

#### 4.2.7.1 MySQL 접근 허용
```
bash
sudo setsebool -P httpd_can_network_connect_db 1
```
- 웹 서버가 DB(데이터베이스) 서버에 접속할 수 있도록 보안 정책을 변경합니다.

#### 4.2.7.2 NFS 접근 허용
```
bash
sudo setsebool -P httpd_use_nfs 1
```
- 웹 서버가 NFS 마운트된 디렉터리 내의 파일을 읽고 쓸 수 있도록 보안 정책을 변경합니다.

### 4.2.8 Apache 재시작
```
bash
sudo systemctl restart httpd
```
- 모든 설정을 마친 후 웹 서버를 재시작하여 변경 사항을 적용합니다.

### 4.2.9 MySQL 접속 확인
```
bash
sudo mysql -h 192.168.58.13 -u wpuser -pqwer1234 wordpress
```
- 웹 서버에서 DB 서버로 원격 접속을 확인합니다.

### 4.2.10 NFS 동작 확인

#### 4.2.10.1 파일 생성(WEB01) 
```
bash
sudo tee /var/www/html/wordpress/from_web01.txt << EOF
Created from WEB01
EOF
```
- 웹 서버(WEB01)의 워드프레스 디렉터리에 테스트용 텍스트 파일을 생성합니다.

#### 4.2.10.2 파일 확인(WEB02)
```
bash
curl http://192.168.57.12/from_web01.txt
```
- 웹 서버(WEB02)의 IP 주소로 앞서 생성한 텍스트 파일에 HTTP 요청을 보냅니다.

# 5. Presentation Tier HAProxy 서버 구축

## 5.1 패키지 설치
```
bash
sudo dnf install -y haproxy
```
- 웹 서버 전면에서 트래픽을 효율적으로 배분하고 장애 발생 시 서버를 제외하는 로드밸런서 HAProxy 패키지를 설치합니다.

## 5.2 HAProxy 환경 설정 및 백업

### 5.2.1 HAProxy 설정 파일 백업
```
bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```
- 설정 오류 발생 시 즉시 복구할 수 있도록 기본 설정 파일을 백업합니다.

### 5.2.2  HAProxy 환경 설정
```
bash
sudo tee /etc/haproxy/haproxy.cfg << EOF
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    timeout connect         5s
    timeout client          50s
    timeout server          50s

frontend http_front
    bind *:80
    default_backend wordpress_backend

backend wordpress_backend
    balance roundrobin
    option httpchk GET /
    http-response set-header X-Server %[srv_name]
    server web01 192.168.57.11:80 check
    server web02 192.168.57.12:80 check
EOF
```
- 사용자의 요청을 받아 웹 서버로 전달하는 설정을 등록합니다.
  - frontend: 80번 Port로 들어오는 외부 요청을 수신합니다.
  - backend: 실제 요청을 처리할 웹 서버 그룹(web01, web02)과 로드밸런싱 방식을 지정합니다.
  - check: 각 웹 서버의 상태를 주기적으로 체크하여 정상적인 서버에만 요청을 보냅니다.
  - http-response set-header: 응답 헤더에 어떤 서버가 처리했는지(X-Server) 표시하여 로드밸런싱 동작 여부를 확인할 수 있습니다.

## 5.3 설정 검사 및 HAProxy 서비스 시작

### 5.3.1 설정 파일 문법 검사
```
bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
- 설정 파일에 오타가 없는지 미리 검증합니다.

### 5.3.2 HAProxy 서비스 활성화 및 시작
```
bash
sudo systemctl start haproxy --now
```
- HAProxy 서비스를 즉시 실행하고, 시스템 재부팅 시에도 자동으로 구동되도록 설정합니다.

## 5.4 방화벽 설정

### 5.4.1 방화벽 적용
```bash
sudo firewall-cmd --add-service=http --permanent && \
sudo firewall-cmd --reload
```
- 외부 사용자가 로드밸런서를 통해 웹 서비스에 접속할 수 있도록 HTTP port(80) 를 개방합니다.

### 5.4.3 방화벽 확인
```bash
sudo firewall-cmd --list-services
```
- 방화벽 서비스 목록에 http가 정상적으로 등록되었는지 확인합니다.

### 5.4.4 SELinux haproxy 아웃바운드 허용
```
bash
sudo setsebool -P haproxy_connect_any 1
```
- 로드밸런서가 웹 서버로 요청을 전달할 수 있게 보안 정책을 변경합니다.

## 5.5 로드밸런싱 동작 확인
```
bash
for i in {1..10}; do
  echo "Request $i"
  curl -sI http://192.168.56.10/ | grep -i "^X-Server"
done
```

```
Request 1
x-server: web01
Request 2
x-server: web02
Request 3
x-server: web01
Request 4
x-server: web02
Request 5
x-server: web01
Request 6
x-server: web02
Request 7
x-server: web01
Request 8
x-server: web02
Request 9
x-server: web01
Request 10
x-server: web02
```
- 반복문을 활용해 Round Robin 로드밸런싱의 정상 동작 유무를 확인합니다.