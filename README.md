# SecLab: AWS 기반 취약점 분석 및 방어 실습
## 📌 _실습 환경 구성_

![](https://github.com/user-attachments/assets/53f7ca33-b61c-4d7b-b897-1aecd5f9ca7a)

실습 환경을 구성하기 위한 과정

- VPC 생성
- 보안 그룹 생성
- 인스턴스 생성 및 설정
- 로드 밸런서 생성

## VPC 생성

- 이름 : SecLab
- IPv4 CIDR 블록 : 10.0.0.0/16
- IPv6 CIDR 블록 : 없음
- 테넌시 : 기본값
- 가용 영역 수 : 2(us-east-1a, us-east-1b)
- 퍼블릭 서브넷 수 : 2(10.0.1.0/24, 10.0.2.0/24)
- 프라이빗 서브넷 수 : 0
- NAT 게이트웨이 : 없음
- VPC 엔드포인트 : 없음
- DNS 옵션 : DNS 호스트 이름 활성화, DNS 확인 활성화

이 설정은 외부 접근이 가능한 퍼블릭 서브넷의 EC2 인스턴스를 위한 간단한 네트워크 구조로, 취약점 분석 및 보안 실습을 위해 최적화 됨. 실습에 필요한 최소한의 구성으로 비용을 절감하면서도 안정적인 환경을 제공

## 보안 그룹 생성

- DVWA 보안 그룹
    - 이름 : SecLab-DVWA-SG
    - 목적 : DVWA 서버에 외부 접근 차단, ALB를 통한 웹 트래픽만 허용
        - 인바운드 규칙
            - HTTP(80) : ALB 보안 그룹에서만 접근 가능하도록 설정
                - 소스 : ALB 보안 그룹(SecLab-ALB-SG)
        - 아웃바운드 규칙
            - 모두 허용(0.0.0.0/0)
                - 설명 : DVWA 서버가 인터넷에 액세스 할 수 있어야 하므로 제한 X
- Attacker 보안 그룹
    - 이름 : SecLab-Attacker-SG
    - 목적 : 공격 서버가 DVWA 서버에 접근하여 공격을 수행할 수 있도록 설정
        - 인바운드 규칙
            - SSH(20) : 관리자의 IP에서만 접근 허용
                - 소스 : 관리자 IP 주소
        - 아웃바운드 규칙
            - 모두 허용(0.0.0.0/0)
                - 설명 : 공격 서버가 인터넷과 DVWA 서버에 트래픽을 보낼 수 있도록 허용
- ALB 보안 그룹
    - 이름 : SecLab-ALB-SG
    - 목적 : 외부에서 HTTP 트래픽을 허용하여 ALB를 통해 DVWA 서버로 연결할 수 있도록 하기 위함
        - 인바운드 규칙
            - HTTP(80) : 모든 외부 IP에서 접근 허용
                - 소스 : 0.0.0.0/0
        - 아웃바운드 규칙
            - HTTP(80) : DVWA 보안 그룹으로의 트래픽 허용
                - 소스 : DVWA 보안 그룹
                - 설명 : DVWA 보안 그룹에만 트래픽을 전달하도록 하여 보안성을 높임

---
---
![](https://github.com/user-attachments/assets/f6c362a4-4c6f-455c-a8e8-62b9dc4d7c82)
> 추가로 Route 53에서 레코드를 생성하여 해당 레코드로도 DVWA 접속이 가능하게 설정 가능
---
---
## DVWA 인스턴스 생성

- 이름 : DVWA
- 인스턴스 유형 : t2.micro
- 플랫폼 : Amazon Linux 2
- 보안 그룹 : SecLab-DVWA-SG
- 스토리지 : 30GB / gp3

## Attacker 인스턴스 생성

- 이름 : Attacker
- 인스턴스 유형 : t2.micro
- 플랫폼 : Amazon Linux 2
- 보안 그룹 : SecLab-Attacker-SG
- 스토리지 : 20GB / gp3

## DVWA 인스턴스 설정

우선 ssh로 인스턴스에 접속해야 하기 때문에
새로 생성한 .pem 키가 있는 폴더에서 아래 명령어로 접속을 한다.

```bash
chmod 400 DVWA.pem
ssh -i DVWA.pem ec2-user@<public ip>
```

이제 접속이 되었다면 DVWA를 접속하기 위한 설정을 해야 함

```bash
sudo yum update -y # 시스템 업데이트

sudo yum install -y httpd # Apache 설치
sudo systemctl start httpd # Apache 시작
sudo systemctl enable httpd # Apach 부팅 시 자동 시작 설정

sudo yum install -y mysql-server # MySQL 서버 설치
sudo systemctl start mysqld # MySQL 시작
sudo systemctl enable mysqld # MySQL 부팅 시 자동 시작 설정
sudo mysql_secure_installation # MySQL 보안 설정
sudo mysql -u root -p # MySQL에 루트 사용자로 접속하여 데이터베이스 및 사용자 생성
```

MySQL 셸에서 다음 명령 실행
```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa_user'@'localhost' IDENTIFIED BY 'YourPassword';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

```bash
cd /var/www/html # Apache 웹 서버 루트 디렉토리
sudo git clone https://github.com/digininja/DVWA.git # DVWA 다운
cd /var/www/html/DVWA/config # DVWA 설정 파일 디렉토리
sudo cp config.inc.php.dist config.inc.php
sudo nano config.inc.php # DVWA 설정 파일 수정
```

설정 파일의 데이터베이스 정보를 MySQL 설정에 맞게 변경
```php
$_DVWA['db_server']   = 'localhost';
$_DVWA['db_database'] = 'dvwa';
$_DVWA['db_user']     = 'dvwa_user';
$_DVWA['db_password'] = 'YourPassword';
```

DVWA의 디렉터리 및 파일 권한을 설정하여 Apache가 올바르게 접근할 수 있도록 함
```bash
sudo chown -R apache:apache /var/www/html/DVWA
sudo chmod -R 755 /var/www/html/DVWA
sudo chmod -R 777 /var/www/html/DVWA/hackable/uploads
sudo chmod -R 777 /var/www/html/DVWA/external/phpids/0.6/lib/IDS/tmp
sudo chmod -R 777 /var/www/html/DVWA/config
sudo systemctl restart httpd # Apache를 재시작하여 변경 사항을 반영
```

위 설정을 모두 하고 웹 브라우저에서 http://<ALB 주소> 또는 레코드로 DVWA에 접속

![ALB 주소](https://github.com/user-attachments/assets/38ee6424-0ac3-4541-aace-3ee80fa97b31)

---

![레코드](https://github.com/user-attachments/assets/7f00ff98-6f61-4aad-b63f-d2aabb02dcb2)

설정 페이지(/setup.php)로 이동 후 Create / Reset Database 버튼을 클릭하여 DVWA 데이터베이스를 초기화

![](https://github.com/user-attachments/assets/10070653-9cd9-4e70-8b92-2a2f966df99a)
