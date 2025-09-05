# AWK 사용자/계정 로그 분석

## 👥팀원
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/Hyunsoo1998">
        <img src="https://github.com/Hyunsoo1998.png" width="100px;" alt="Hyunsoo1998"/><br />
        <sub><b>김현수</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/menzzi">
        <img src="https://github.com/menzzi.png" width="100px;" alt="menzzi"/><br />
        <sub><b>서민지</b></sub>
      </a>
    </td>
  </tr>
</table>
<br>

## 🔍 프로젝트 개요

**리눅스 서버에서 사용자 및 계정 관련 로그를 수집하고 분석**   

- 로그 수집: journalctl, 시스템 로그 파일

- 주기적 로그 수집: cron

- 필요한 정보 추출 및 통계: awk

- 대상: SSH 로그인, 사용자 생성/변경, 그룹 변경, 권한 관련 기록   

<br>

## 📌 추출 대상 데이터
#### 사용자 로그인 기록

- 로그인 실패 기록

- 로그인 성공 기록

#### 사용자 관리

- 사용자 그룹 생성/변경 기록

- 사용자 생성 기록

#### 권한 관리

- sudo 권한 사용

- sudo 인증 실패

<br>

# 💻 실습 내용
## 👨‍💻 로그인 기록
> 💡 로그인 기록은 수동으로 로그인 기록을 저장하고, 이후에는 cron을 이용해 주기적으로 자동 수집하도록 설정했습니다.
#### 1. 저장 (실패 / 성공)

```shell
# 실패
sudo journalctl | grep -i "failed password" | sudo tee -a /var/log/failed_password.log > /dev/null

# 성공
sudo journalctl | grep -i "Accepted password" | sudo tee -a /var/log/success_password.log > /dev/null
```

#### 2. cron으로 주기적 수집 (sudo crontab -e)
  ⚠️주의 : 1분마다 `journalctl --since "today"`를 실행하면 오늘 시작부터 로그 전체를 매번 읽어 append하므로 로그 중복 발생 가능
  ```shell
  */1 * * * * journalctl --since "today" | grep -i "failed password" | tee -a /var/log/failed_password.log > /dev/null
  ```

하루 기준으로 최근 1분 로그만 기록하려면:
```shell
# 실패
*/1 * * * * journalctl _COMM=sshd --since "1 minute ago" --output=short-iso | grep -i "failed password" >> /var/log/failed_password.log

# 성공
*/1 * * * * journalctl _COMM=sshd --since "1 minute ago" --output=short-iso | grep -i "accepted password" >> /var/log/success_password.log
```

- _COMM=sshd → SSH 데몬(sshd) 로그만 가져옴

- --output=short-iso → ISO 8601 형식 (YYYY-MM-DDTHH:MM:SS+TZ)으로 출력

#### 3. AWK를 활용한 분석
**3.1 사용자별 로그인 실패/성공 횟수**
```shell
awk '{count[$(NF-5)]++} END {for(user in count) printf "%-10s : %d\n", user, count[user]}' /var/log/failed_password.log | sort -k3 -nr

awk '{count[$(NF-5)]++} END {for(user in count) printf "%-10s : %d\n", user, count[user]}' /var/log/success_password.log | sort -k3 -nr
```

- $(NF-5) = 마지막에서 5번째 필드 (NF = 현재 줄의 필드 수(Number of Fields))

- count[$(NF-5)]++ = 사용자 수 집계

- END {for(user in count) printf "%-10s : %d\n", user, count[user]} = 사용자별로 "사용자이름 : 횟수" 형식으로 출력

- sort -k3 -nr : 출력 결과를 횟수 기준 내림차순 정렬

  - -k3 → 3번째 컬럼(횟수) 기준

  - -n → 숫자 정렬

  - -r → 내림차순
  
**3.2 IP별 실패/성공 횟수**
```shell
awk '{count[$(NF-3)]++} END {for(ip in count) printf "%-10s : %d\n", ip, count[ip]}' /var/log/failed_password.log | sort -k3 -nr

awk '{count[$(NF-3)]++} END {for(ip in count) printf "%-10s : %d\n", ip, count[ip]}' /var/log/success_password.log | sort -k3 -nr
```
**3.3 특정 날짜 기록 추출**
```shell
awk '/2025-09-05/ {printf "%-30s %-15s %-12s %-15s\n", $1, $2, $(NF-5), $(NF-3)}' /var/log/failed_password.log

awk '/2025-09-05/ {printf "%-30s %-15s %-12s %-15s\n", $1, $2, $(NF-5), $(NF-3)}' /var/log/success_password.log
```
**3.4 특정 사용자 기록 추출**
```shell
awk -v user="사용자 아이디" '$0 ~ user {printf "%-30s %-15s %-12s %-15s\n", $1, $2, $(NF-5), $(NF-3)}' /var/log/failed_password.log

awk -v user="사용자 아이디" '$0 ~ user {printf "%-30s %-15s %-12s %-15s\n", $1, $2, $(NF-5), $(NF-3)}' /var/log/success_password.log
```
**3.5 특정 시간대 실패/성공 로그 집계**
```shell
awk '{
    split($1, dt, "T");
    split(dt[2], tm, "+");
    split(tm[1], t, ":");
    if(t[1]=="12")
        printf "%-30s %-15s %-12s %-15s\n", dt[1], tm[1], $(NF-5), $(NF-3)
}' /var/log/failed_password.log

awk '{
    split($1, dt, "T");
    split(dt[2], tm, "+");
    split(tm[1], t, ":");
    if(t[1]=="12")
        printf "%-30s %-15s %-12s %-15s\n", dt[1], tm[1], $(NF-5), $(NF-3)
}' /var/log/success_password.log
```
- split($1, dt, "T") = 날짜와 시간 부분을 T 기준으로 나눔

  - 예: 2025-09-05T12:16:33+09:00 → dt[1]=2025-09-05, dt[2]=12:16:33+09:00  

- split(dt[2], tm, "+") = 시간과 타임존 분리

- split(tm[1], t, ":") = 시, 분, 초 분리

<br>

## 🔐 사용자 그룹 생성/변경 기록
> 💡 지금부터는 cron으로 주기적 수집만 진행했습니다. (수동으로 수집한 뒤 확인하고 진행 or 바로 주기적 수집 진행)
#### 1. cron으로 주기적 수집 (sudo crontab -e)
```shell
# 그룹 생성
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso _COMM=groupadd >> /var/log/user_activity/group.log

# 그룹 변경
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso _COMM=usermod >> /var/log/user_activity/group.log
```
#### 2. awk로 그룹 생성/변경 기록 추출
```shell
awk '/-- No entries --/||NF<4{next}
{
    gsub("T"," ",$1);
    ts=substr($1,1,19);
    split($3,cmd,"[");
    if(cmd[1]=="groupadd"&&$4=="new"){
        sub(/name=/,"",$6); sub(/,/,"",$6); sub(/GID=/,"",$7);
        printf "%-21s  %-10s  Group \x27%s\x27 (GID: %s) created\n",ts,cmd[1],$6,$7
    }
    if(cmd[1]=="usermod"&&$6=="to"){
        gsub(/\x27/,"",$5); gsub(/\x27/,"",$8);
        printf "%-21s  %-10s  User \x27%s\x27 added to group \x27%s\x27\n",ts,cmd[1],$5,$8
    }
}' /var/log/user_activity/group.log
```
- /-- No entries --/||NF<4{next} = 로그가 비어있거나 필드가 부족하면 건너뜀

- gsub("T"," ",$1) = 날짜와 시간 구분 문자 T를 공백으로 바꿈

- ts=substr($1,1,19) = 타임스탬프 추출 (YYYY-MM-DD HH:MM:SS)

- split($3,cmd,"[") = 명령어(groupadd 또는 usermod) 추출

- 그룹 생성:

  - if(cmd[1]=="groupadd"&&$4=="new") = 새 그룹 생성 기록이면

  - sub(/name=/,"",$6); sub(/,/,"",$6); sub(/GID=/,"",$7) = 그룹 이름과 GID 정리

- 그룹 변경:

  - if(cmd[1]=="usermod"&&$6=="to") = 사용자가 그룹에 추가된 기록이면

  - gsub(/\x27/,"",$5); gsub(/\x27/,"",$8) = 따옴표 제거


<br>

## 👤 사용자 생성 기록
#### 1. cron으로 주기적 수집 (sudo crontab -e)
```shell
# useradd 명령어로 유저 생성
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso _COMM=useradd >> /var/log/user_activity/useradd.log

# adduser 명령어로 유저 생성
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso _COMM=adduser >> /var/log/user_activity/useradd.log
````
#### 2. awk로 사용자 생성 기록 추출
```shell
awk '/useradd/&&/new user:/'{
    gsub("T"," ",$1);
    ts=substr($1,1,19);
    split($3,cmd,"[");
    sub(/name=/,"",$6); sub(/,/,"",$6);
    sub(/UID=/,"",$7); sub(/,/,"",$7);
    sub(/GID=/,"",$8); sub(/,/,"",$8);
    printf "%-21s  %-10s  User \x27%s\x27 (UID: %s, GID: %s) created\n",ts,cmd[1],$6,$7,$8
}' /var/log/user_activity/useradd.log
```
- /useradd/&&/new user:/ = useradd 명령어로 새 계정이 생성된 로그만 선택

- gsub("T"," ",$1) = 날짜와 시간 구분 문자 T를 공백으로 바꿈

- ts=substr($1,1,19) = 타임스탬프 추출 (YYYY-MM-DD HH:MM:SS)

- split($3,cmd,"[") = 실행된 명령어(useradd) 추출

- sub(/name=/,"",$6); sub(/,/,"",$6) = 사용자 이름 정리

- sub(/UID=/,"",$7); sub(/,/,"",$7) = UID 정리

- sub(/GID=/,"",$8); sub(/,/,"",$8) = GID 정리

<br>

## 🛡️ 권한(sudo) 관련 기록
#### 1. cron으로 주기적 수집 (sudo crontab -e)
```shell
# sudo 로그 (모든 사용자)
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso -t sudo >> /var/log/user_activity/sudo_all.log
```
#### 2. awk로 sudo 권한 사용기록 추출
```shell
awk '/COMMAND=/'{
    gsub("T"," ",$1);
    ts=substr($1,1,19);
    user=$4;
    split($0,a,"COMMAND=");
    cmd=a[2];
    if($0~/user NOT in sudoers/){status="FAILED";detail="(No Perms)"}
    else if($0~/incorrect password attempt/){status="FAILED";detail="(Bad Pass)"}
    else if($0~/TTY=/){status="SUCCESS";detail=""}
    if(status){
        printf "%-21s  %-10s  %-8s  %-12s %s\n",ts,user,status,detail,cmd
    }
}' /var/log/user_activity/sudo_all.log
```
- gsub("T"," ",$1) = 타임스탬프에서 T를 공백으로 변경

- ts=substr($1,1,19) = YYYY-MM-DD HH:MM:SS 형태로 타임스탬프 추출

- user=$4 → 명령을 실행한 사용자

- split($0,a,"COMMAND="); cmd=a[2] = 실행된 명령어 추출

- 로그 분석:

  - "user NOT in sudoers" = 권한 없음 → FAILED (No Perms)

  - "incorrect password attempt" = 비밀번호 오류 → FAILED (Bad Pass)

  - "TTY=" → 정상 실행 = SUCCESS
  
<br>

## 🔑 인증 실패 기록
#### 1. cron으로 주기적 수집 (sudo crontab -e)
```shell
# 인증 실패 로그
*/1 * * * * journalctl --since "1 minute ago" --until "now" -o short-iso SYSLOG_FACILITY=10 | grep -E "authentication failure|Failed password" >> /var/log/user_activity/auth_fail.log
```
#### 2. awk로 원격 접속 실패 및 sudo 권한 실패 기록 추출
```shell
awk '/authentication failure/'{
    gsub("T"," ",$1);
    ts=substr($1,1,19);
    if($0~/sshd/){
        split($0,u,"user="); user=u[2];
        split($0,r,"rhost="); split(r[2],rp," "); rhost=rp[1];
        if(user=="") user="N/A";
        if(rhost=="") rhost="N/A";
        printf "%-21s  %-8s  %-10s  From %-15s  Authentication Failure\n",ts,"sshd",user,rhost
    }
    if($0~/sudo/){
        split($0,u,"user="); user=u[2];
        split($0,t,"tty="); split(t[2],tp," "); tty=tp[1];
        if(user=="") user="N/A";
        if(tty=="") tty="N/A";
        printf "%-21s  %-8s  %-10s  On %-15s  Authentication Failure\n",ts,"sudo",user,tty
    }
}' /var/log/user_activity/auth_fail.log
```
- 타임스탬프 변환 (gsub, substr)
- SSH 로그 처리:
  - user= → 사용자, rhost= → 원격 IP 추출
  - 값이 없으면 "N/A"로 처리
- sudo 로그 처리:
  - user= → 사용자, tty= → 터미널 정보 추출
  - 값이 없으면 "N/A"로 처리
 
<br>

## ⚡ 트러블 슈팅

### 1. 로그 기록 저장 중 `Permission denied` 발생

```bash
ubuntu@myserver00:~$ sudo journalctl | grep -i "failed password" > /var/log/failed_password.log
-bash: /var/log/failed_password.log: Permission denied
```

원인

- /var/log/ 디렉토리는 root 전용

- sudo journalctl까지만 root 권한이고, > 리다이렉션은 현재 쉘에서 실행되므로 root 권한이 적용되지 않음

**해결법: sudo tee 사용**

```bash
sudo journalctl | grep -i "failed password" | sudo tee -a /var/log/failed_password.log > /dev/null
```

- tee : 표준 입력을 파일로 저장하는 명령어

- -a : append 모드 (>>와 동일)

- /dev/null : 화면 출력 억제, 파일에만 저장


### 2. 사용자별 적용되는 crontab 범위

일반 사용자(ubuntu)의 crontab -e

- 실행 권한: 해당 사용자 권한으로 실행

- 제한: 홈 디렉터리(/home/ubuntu)와 같이 접근 가능한 파일에만 동작

- 주요 목적: 개인 파일 백업, 스크립트 실행, 알림 등 개인 자동화

root 사용자의 crontab -e

- 실행 권한: 시스템 전체에 접근 가능한 root 권한으로 실행

- 가능 작업: 모든 파일 수정/삭제, 시스템 서비스 관리

- 주요 목적: 시스템 전체 백업, 로그 관리(logrotate), 서비스 재시작, 보안 업데이트 등 운영 및 관리 자동화


| **구분** | **일반 사용자** | **root 사용자** |
| --- | --- | --- |
| 실행 명령어 | `crontab -e` (일반 유저) | `sudo crontab -e` (root 유저) |
| 실행 권한 | **제한된 사용자 권한** | **시스템의 모든 권한 (관리자)** |
| 주요 목적 | 개인 작업 자동화 | 시스템 관리 및 운영 자동화 |
| 작업 파일 위치 | `/var/spool/cron/crontabs/<사용자명>` | `/var/spool/cron/crontabs/root` |
