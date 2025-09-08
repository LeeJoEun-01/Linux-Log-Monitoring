# 🖥️ ShellScript_MySQL

> 이 문서는 **우분투에 MySQL** 을 설치하고, 샘플 스키마(EMP/DEPT)를 구성한 뒤 **mysqldump + cron**으로 자동 백업하는 과정을 다룹니다.
> 
> 백업 스크립트는 **테이블 단위**로 덤프 후 `tar.gz`로 압축·보관합니다.

---

## 👥 Contributors

| <img width="150px" src="https://avatars.githubusercontent.com/u/78733700?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/88383179?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/81912226?v=4"/> |
| :---: | :---: | :---: |
| **이조은** | **정다빈** | **정서현** |
| [@LeeJoEun-01](https://github.com/LeeJoEun-01) | [@ddddabi](https://github.com/ddddabi) | [@hyunn522](https://github.com/hyunn522) |

---

## 1. MySQL 설치 및 설정
```shell
# 설치
sudo apt update
sudo apt install -y mysql-server

# 보안 초기 설정 (루트 암호/익명계정 제거 등)
sudo mysql_secure_installation
```

```shell
# 부팅 시 자동 시작
sudo systemctl enable mysql

# 기동/중지/재기동
sudo systemctl start mysql
sudo systemctl stop mysql
sudo systemctl restart mysql
```

- **서비스 상태 확인**
  <br/>
  <img width="1001" height="341" src="https://github.com/user-attachments/assets/3c673ad4-d189-40d7-9125-ccb5e864f176" />

## 2. 데이터베이스 및 테이블 생성

**DEPT**

| 컬럼       | 타입            | 제약     | 설명    |
| -------- | ------------- | ------ | ----- |
| `DEPTNO` | `DECIMAL(10)` | **PK** | 부서 번호 |
| `DNAME`  | `VARCHAR(14)` |        | 부서명   |
| `LOC`    | `VARCHAR(13)` |        | 지역    |

**EMP**

| 컬럼         | 타입             | 제약               | 설명       |
| ---------- | -------------- | ---------------- | -------- |
| `EMPNO`    | `DECIMAL(4)`   | **PK**, NOT NULL | 사번       |
| `ENAME`    | `VARCHAR(10)`  |                  | 이름       |
| `JOB`      | `VARCHAR(9)`   |                  | 직무       |
| `MGR`      | `DECIMAL(4)`   |                  | 상사 사번    |
| `HIREDATE` | `DATE`         |                  | 입사일      |
| `SAL`      | `DECIMAL(7,2)` |                  | 급여       |
| `COMM`     | `DECIMAL(7,2)` |                  | 커미션      |
| `DEPTNO`   | `DECIMAL(2)`   | **FK**           | 소속 부서 번호 |

- **데이터 저장 확인**
  <br/>
  <img width="421" height="313" src="https://github.com/user-attachments/assets/5da8df7e-c1db-4288-ae12-91cc7363857e" />

## 3. 자동화 백업 스크립트 생성 및 등록

1. mysqldump로 특정 테이블을 덤프

2. 결과 .sql을 `tar.gz`로 압축 (용량 절감/전송 편의)

3. 원본 .sql 삭제 (압축본만 보관)

- **백업 디렉토리 생성**
  
  ```shell
  sudo mkdir -p /root/04.mission
  ```
  
- **EMP**
  
  ```shell
    #!/bin/bash
    
    # --- 설정 ---
    DB_NAME="DB명"
    TABLE_NAME="EMP"
    DB_USER="USER명"
    DB_PASS="PW"
    
    BACKUP_DIR="/root/04.mission"
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    
    DUMP_FILE="${BACKUP_DIR}/${TABLE_NAME}_${TIMESTAMP}.sql"
    TAR_FILE="${BACKUP_DIR}/${TABLE_NAME}_${TIMESTAMP}.tar.gz"
    
    # --- 실행 ---
    # 백업 디렉터리가 없으면 생성
    mkdir -p "$BACKUP_DIR"
    
    # 1. mysqldump로 테이블 덤프
    mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" "$TABLE_NAME" > "$DUMP_FILE"
    
    # 2. 덤프 파일을 tar.gz로 압축
    tar -czvf "$TAR_FILE" -C "$BACKUP_DIR" "$(basename "$DUMP_FILE")"
    
    # 3. 원본 .sql 덤프 파일 삭제
    rm "$DUMP_FILE"
    
    echo "Backup completed: $TAR_FILE"
  ```

- **DEPT**
  
  ```shell
    #!/bin/bash
    
    # --- 설정 ---
    DB_NAME="DB명"
    TABLE_NAME="DEPT"
    DB_USER="USER명"
    DB_PASS="PW"
    
    BACKUP_DIR="/root/04.mission"
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    
    DUMP_FILE="${BACKUP_DIR}/${TABLE_NAME}_${TIMESTAMP}.sql"
    TAR_FILE="${BACKUP_DIR}/${TABLE_NAME}_${TIMESTAMP}.tar.gz"
    
    # --- 실행 ---
    # 백업 디렉터리가 없으면 생성
    mkdir -p "$BACKUP_DIR"
    
    # 1. mysqldump로 데이터베이스 덤프
    mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" "$TABLE_NAME" > "$DUMP_FILE"
    
    # 2. 덤프 파일을 tar.gz로 압축
    tar -czvf "$TAR_FILE" -C "$BACKUP_DIR" "$(basename "$DUMP_FILE")"
    
    # 3. 원본 .sql 덤프 파일 삭제
    rm "$DUMP_FILE"
    
    echo "Backup completed: $TAR_FILE"
  ```

- **실행 권한 부여**
  
  ```shell
    # 실행 권한
    sudo chmod +x 04.mission/backup_EMP.sh
    sudo chmod +x 04.mission/backup_DEPT.sh
  ```
  
- **수동 실행 확인**
  
  ```shell
    # 수동으로 실행해 정상 작동 확인
    sudo ./backup_EMP.sh
    sudo ./backup_DEPT.sh
  ```
  
  <br/>
  <img width="866" height="162" src="https://github.com/user-attachments/assets/a24298f9-d9ef-42c3-91d7-19e0960ca466" />
  
## 4. cron에 자동화 작업 등록

운영체제 레벨에서 정해진 시각/주기로 스크립트를 실행할 수 있는 스케줄러

- **crontab에 설정 추가**
  
  ```shell
  sudo crontab -e
  ```
  
- **자동화 시간 지정**

  ```shell
  # 매일 02:00에 EMP 백업
  0 2 * * * /home/ubuntu/04.mission/backup_emp.sh >> /var/log/backup_emp.log 2>&1
  
  # 겹침 방지: 5분 뒤 DEPT 백업
  5 2 * * * /home/ubuntu/04.mission/backup_dept.sh >> /var/log/backup_dept.log 2>&1
  ```
  
- **등록 확인**
  
  ```shell
  sudo crontab -l
  ```
