# 🗄️ MySQL 설치 및 자동 백업 실습
## 📌 1. MySQL 설치 및 데이터 생성
``` bash
# 패키지 업데이트 및 MySQL 설치
sudo apt-get update
sudo apt-get install mysql-server

# 상태 확인
sudo systemctl status mysql
```


**데이터베이스 및 테이블 생성**
``` sql
DROP DATABASE IF EXISTS sqld;
CREATE DATABASE sqld DEFAULT CHARACTER SET UTF8; 
USE sqld;

-- 부서 테이블
CREATE TABLE DEPT (
  DEPTNO DECIMAL(10),
  DNAME  VARCHAR(14),
  LOC    VARCHAR(13)
);

INSERT INTO DEPT VALUES (10, 'ACCOUNTING', 'NEW YORK');
INSERT INTO DEPT VALUES (20, 'RESEARCH',   'DALLAS');
INSERT INTO DEPT VALUES (30, 'SALES',      'CHICAGO');
INSERT INTO DEPT VALUES (40, 'OPERATIONS', 'BOSTON');

-- 직원 테이블
CREATE TABLE EMP (
  EMPNO     DECIMAL(4) NOT NULL,
  ENAME     VARCHAR(10),
  JOB       VARCHAR(9),
  MGR       DECIMAL(4),
  HIREDATE  DATE,
  SAL       DECIMAL(7,2),
  COMM      DECIMAL(7,2),
  DEPTNO    DECIMAL(2)
);

-- 데이터 입력
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
```

테이블 확인:
``` sql
SELECT * FROM DEPT;
SELECT * FROM EMP;
```

## 📌 2. 자동화: MySQL 덤프 + 압축
**스크립트 작성 (/root/script/mysql-dump.sh)**
``` bash
#!/bin/bash

# MySQL 덤프
mysqldump -u root -pubuntu sqld > /root/db.sql

# 백업 디렉토리
SQL_DUMP_DIR="/root/script"

# 날짜 형식: YYYYMMDD_HHMMSS
SQL_NAME="MYSQL_$(date +%Y%m%d_%H%M%S).tar.gz"

# 백업 수행 (db.sql 포함)
tar -czf "$SQL_DUMP_DIR/$SQL_NAME" /root/db.sql

echo "backup_success: $SQL_DUMP_DIR/$SQL_NAME"
```

실행 권한 부여:
```bash
chmod +x /root/script/mysql-dump.sh
```

## 📌 3. Crontab 설정 (1시간 주기 덤프)
``` bash
sudo crontab -e

1 * * * * /root/script/mysql-dump.sh >> /root/script/backup.log 2>&1
```

- 매 시간 1분마다 자동 백업 실행

- 실행 결과는 /root/script/backup.log 에 기록됨
