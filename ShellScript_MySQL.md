# 🖥️ ShellScript_MySQL

---
## 👥 Contributors

| <img width="150px" src="https://avatars.githubusercontent.com/u/78733700?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/88383179?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/81912226?v=4"/> |
| :---: | :---: | :---: |
| **이조은** | **정다빈** | **정서현** |
| [@LeeJoEun-01](https://github.com/LeeJoEun-01) | [@ddddabi](https://github.com/ddddabi) | [@hyunn522](https://github.com/hyunn522) |



---

## 1. MySQL 설치 및 설정
```
# 설치
sudo apt update
sudo apt install -y mysql-server

# 보안 초기 설정 (루트 암호/익명계정 제거 등)
sudo mysql_secure_installation
```

```
# 서비스 등록 상태
systemctl status mysql

# 부팅 시 자동 시작
sudo systemctl enable mysql

# 기동/중지/재기동
sudo systemctl start mysql
sudo systemctl stop mysql
sudo systemctl restart mysql
```
- **서비스 등록 상태 확인**
  <br/>
  <img width="1001" height="341" src="https://github.com/user-attachments/assets/3c673ad4-d189-40d7-9125-ccb5e864f176" />


## 2. 데이터베이스 및 테이블 생성
``` 
# 데이터 저장
DROP DATABASE IF EXISTS sqld;

CREATE DATABASE sqld default CHARACTER SET UTF8; 

USE sqld;

CREATE TABLE DEPT
       (DEPTNO DECIMAL(10),
        DNAME VARCHAR(14),
        LOC VARCHAR(13) 
);

INSERT INTO DEPT VALUES (10, 'ACCOUNTING', 'NEW YORK');
INSERT INTO DEPT VALUES (20, 'RESEARCH',   'DALLAS');
INSERT INTO DEPT VALUES (30, 'SALES',      'CHICAGO');
INSERT INTO DEPT VALUES (40, 'OPERATIONS', 'BOSTON');

CREATE TABLE EMP (
 EMPNO               DECIMAL(4) NOT NULL,
 ENAME               VARCHAR(10),
 JOB                 VARCHAR(9),
 MGR                 DECIMAL(4) ,
 HIREDATE            DATE,
 SAL                 DECIMAL(7,2),
 COMM                DECIMAL(7,2),
 DEPTNO              DECIMAL(2) 
 );

INSERT INTO EMP VALUES (7839,'KING','PRESIDENT',NULL,STR_TO_DATE('1981-11-17','%Y-%m-%d'),5000,NULL,10);
INSERT INTO EMP VALUES (7698,'BLAKE','MANAGER',7839,STR_TO_DATE('1981-05-01','%Y-%m-%d'),2850,NULL,30);
INSERT INTO EMP VALUES (7782,'CLARK','MANAGER',7839,STR_TO_DATE('1981-05-09','%Y-%m-%d'),2450,NULL,10);
INSERT INTO EMP VALUES (7566,'JONES','MANAGER',7839,STR_TO_DATE('1981-04-01','%Y-%m-%d'),2975,NULL,20);
INSERT INTO EMP VALUES (7654,'MARTIN','SALESMAN',7698,STR_TO_DATE('1981-09-10','%Y-%m-%d'),1250,1400,30);
INSERT INTO EMP VALUES (7499,'ALLEN','SALESMAN',7698,STR_TO_DATE('1981-02-11','%Y-%m-%d'),1600,300,30);
INSERT INTO EMP VALUES (7844,'TURNER','SALESMAN',7698,STR_TO_DATE('1981-08-21','%Y-%m-%d'),1500,0,30);
INSERT INTO EMP VALUES (7900,'JAMES','CLERK',7698,STR_TO_DATE('1981-12-11','%Y-%m-%d'),950,NULL,30);
INSERT INTO EMP VALUES (7521,'WARD','SALESMAN',7698,STR_TO_DATE('1981-02-23','%Y-%m-%d'),1250,500,30);
INSERT INTO EMP VALUES (7902,'FORD','ANALYST',7566,STR_TO_DATE('1981-12-11','%Y-%m-%d'),3000,NULL,20);
INSERT INTO EMP VALUES (7369,'SMITH','CLERK',7902,STR_TO_DATE('1980-12-09','%Y-%m-%d'),800,NULL,20);
INSERT INTO EMP VALUES (7788,'SCOTT','ANALYST',7566,STR_TO_DATE('1982-12-22','%Y-%m-%d'),3000,NULL,20);
INSERT INTO EMP VALUES (7876,'ADAMS','CLERK',7788,STR_TO_DATE('1983-01-15','%Y-%m-%d'),1100,NULL,20);
INSERT INTO EMP VALUES (7934,'MILLER','CLERK',7782,STR_TO_DATE('1982-01-11','%Y-%m-%d'),1300,NULL,10);

COMMIT;

SELECT * FROM DEPT;

SELECT * FROM EMP;

```
- **데이터 저장 확인**
  <br/>
  <img width="421" height="313" src="https://github.com/user-attachments/assets/5da8df7e-c1db-4288-ae12-91cc7363857e" />

## 3. 자동화 백업 스크립트 생성 및 등록
- **백업 디렉토리 생성**
  ```
  sudo mkdir -p /root/04.mission
  ```
- **EMP**
  ```
    #!/bin/bash
    
    # --- 설정 ---
    DB_NAME="DB 명"
    TABLE_NAME="EMP"
    DB_USER="USER 명"
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

- **DEPT**
  ```
    #!/bin/bash
    
    # --- 설정 ---
    DB_NAME="DB 명"
    TABLE_NAME="DEPT"
    DB_USER="USER 명"
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
  ```
    # 실행 권한
    sudo chmod +x 04.mission/backup_EMP.sh
    sudo chmod +x 04.mission/backup_DEPT.sh
  ```
  
- **수동 실행 확인**
  ```
    # 수동으로 실행해 정상 작동 확인
    sudo ./backup_EMP.sh
    sudo ./backup_DEPT.sh
  ```
  <br/>
  <img width="866" height="162" src="https://github.com/user-attachments/assets/a24298f9-d9ef-42c3-91d7-19e0960ca466" />


  
## 4. cron에 자동화 작업 등록
- **crontab에 설정 추가**
  ```
  sudo crontab -e
  ```
- **자동화 시간 지정**
  ```
  # 매일 02:00에 EMP 백업
  0 2 * * * /home/ubuntu/04.mission/backup_emp.sh >> /var/log/backup_emp.log 2>&1
  
  # 겹침 방지: 5분 뒤 DEPT 백업
  5 2 * * * /home/ubuntu/04.mission/backup_dept.sh >> /var/log/backup_dept.log 2>&1
  ```
- **등록 확인**
  ```
  sudo crontab -l
  ```
