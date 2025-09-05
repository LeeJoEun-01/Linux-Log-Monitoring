# 🐧Linux-Log-Monitoring
리눅스 로그 데이터 수집 및 모니터링 <br>
- Nginx 웹 서버의 접속 로그(access.log)를 자동으로 분석하는 시스템입니다.
- 셸 스크립트와 awk를 이용해 로그 파일에서 유의미한 데이터를 추출하고, cron을 통해 분석 작업을 주기적으로 **자동화**합니다.
- 이 프로젝트를 통해 생성된 일일 리포트로 서버의 트래픽 패턴과 잠재적인 이슈를 손쉽게 파악하고 모니터링할 수 있습니다.

### 🛠️Tech Stacks
- Linux (Ubuntu 24.04.2)
- Nginx
- Shell Script (Bash)
- awk
- cron
  
### ⚙️Tools
| MobaXterm | Visual Studio Code |
|---|---|
|<img width="500" alt="image" src="https://github.com/user-attachments/assets/204d40a4-0533-4a2e-bd88-271b49dedabe"/>| <img width="500" alt="image" src="https://github.com/user-attachments/assets/5ef88a91-d894-4bff-9d31-a0d7fdb48b1d" /> |

# 1. nginx 설치 및 웹 페이지 띄우기

**1. Nginx 설치하기**
   ```shell
   # 패키지 목록 업데이트
   sudo apt-get update

   # Nginx 설치
   sudo apt-get install -y nginx
   ```
**2. Nginx 서비스 시작 및 활성화**
  ```shell
  # Nginx 서비스 시작
  sudo systemctl start nginx

  # 재부팅 시 자동 시작 활성화
  sudo systemctl enable nginx
  ```

### 📁 Nginx 폴더 구성
```shell
/var/www/html/
├─ main/
│  └─ index.html
└─ detail/
   ├─ 1/
   │  └─ index.html
   ├─ 2/
   │  └─ index.html
   ├─ 3/
   │  └─ index.html
   ├─ 4/
   │  └─ index.html
   └─ 5/
      └─ index.html
```
<br>

# 2. 로그 실행 및 수집 스크립트
### 🌳로그 실행 Tree
<img width="460" alt="image" src="https://github.com/user-attachments/assets/9c0f37a6-fc46-450f-ace0-d4e4047e5773" />

### 📄createLog.sh - 로그 생성 스크립트
```shell
#!/bin/bash

echo "새로운 파일 구조에 맞춰 로그 생성을 시작합니다..."

# --- 1. 메인 페이지와 특정 상세 페이지에 집중적으로 접속 ---
echo "메인 페이지와 특정 상세 페이지(1, 3)에 접속 중..."
for i in {1..10}; do
    # 메인 페이지 접속
    curl http://localhost/main/

    # 1번 상세 페이지 접속
    curl http://localhost/detail/1/

    # 3번 상세 페이지에 더 많이 접속해서 패턴 만들기
    curl http://localhost/detail/3/
done


# --- 2. 모든 상세 페이지에 무작위로 접속하고, 404 에러도 발생시키기 ---
echo "모든 상세 페이지에 무작위 접속"
for i in {1..15}; do
    # 1부터 5까지의 숫자 중 하나를 무작위로 선택
    RANDOM_PAGE=$((RANDOM % 5 + 1))

    echo "Accessing /detail/${RANDOM_PAGE}/"
    curl "http://localhost/detail/${RANDOM_PAGE}/"
done

echo "로그 생성 완료."
```

### 📄analyzer.sh - 로그 수집 스크립트
- 모니터링 데이터 수집 후 csv 파일로 추출
  - `requests-{날짜-시간}.csv`
  - `cpu_usage-{날짜-시간}.csv`

```shell
#!/usr/bin/env bash
set -euo pipefail

# ===== 설정 =====
OUT_DIR="/home/admin/log-reports"
NGX_LOGS="/var/log/nginx/access.log*"

# 파일명에 사용할 타임스탬프 (YYYYMMDD-HHMM)
FNAME_TS="$(date '+%Y%m%d-%H%M')"
REQ_CSV="${OUT_DIR}/requests_${FNAME_TS}.csv"   # 요청 단건 스냅샷 (응답시간 X)
CPU_CSV="${OUT_DIR}/cpu_usage_${FNAME_TS}.csv"  # CPU 스냅샷

TMP="$(mktemp -d /tmp/ngx_analytics.XXXXXX)"
trap 'rm -rf "$TMP"' EXIT

TS="$(date '+%Y-%m-%d %H:%M:%S')"

mkdir -p "$OUT_DIR"

# ===== 0) 로그 수집 (회전 .gz 포함) =====
zgrep -h -a . $NGX_LOGS > "$TMP/all.log" || true

# ===== 1) 요청 단건 CSV 생성 =====
# 응답시간 컬럼 제거:  timestamp,ip,method,url,status,bytes,referer,user_agent
printf 'timestamp,ip,method,url,status,bytes,referer,user_agent\n' > "$REQ_CSV"

# combined 로그 파싱: "REQUEST" STATUS BYTES "REFERER" "UA"
# $4=[time_local  $5=+TZ]
awk -v OFS=',' -v Q='"' '
function esc(s){ gsub(/"/, Q Q, s); return Q s Q }
{
  # 큰따옴표 기준 split
  n = split($0, q, /"/)           # q[2]=request, q[6]=referer, q[8]=ua
  req = (n>=2 ? q[2] : "-")
  ref = (n>=6 ? q[6] : "-")
  ua  = (n>=8 ? q[8] : "-")

  # method & url
  method="-"; url="-"
  if (match(req, /^([A-Z]+) ([^ ]+) HTTP\/[0-9.]+$/, m)) { method=m[1]; url=m[2] }

  # timestamp
  t4=$4; gsub(/\[/, "", t4)
  t5=$5; gsub(/\]/, "", t5)
  ts = t4 " " t5

  ip = $1
  status = $9
  bytes = $10

  # 응답시간 수집 제거 (request_time 파싱 안 함)

  print esc(ts), esc(ip), esc(method), esc(url), esc(status), esc(bytes), esc(ref), esc(ua)
}
' "$TMP/all.log" >> "$REQ_CSV"

# ===== 2) CPU 사용률 스냅샷 =====
cpu_pct() {
  read -r _ u n s i o irq sirq st _ < /proc/stat
  idle1=$((i+o)); non1=$((u+n+s+irq+sirq+st)); tot1=$((idle1+non1))
  sleep 1
  read -r _ u n s i o irq sirq st _ < /proc/stat
  idle2=$((i+o)); non2=$((u+n+s+irq+sirq+st)); tot2=$((idle2+non2))
  td=$((tot2 - tot1)); id=$((idle2 - idle1))
  awk -v td="$td" -v id="$id" 'BEGIN { if (td<=0) print "0.00"; else printf "%.2f", (td-id)/td*100.0 }'
}
CPU="$(cpu_pct)"

printf 'timestamp,cpu_percent\n' > "$CPU_CSV"
printf '%s,%s\n' "$TS" "$CPU" >> "$CPU_CSV"

echo "[OK] $TS requests=>$REQ_CSV cpu=>$CPU_CSV"
```
<br>

# 3. Crontab 스케줄링
- 실행방법: **`crontab -e`**
- createLog.sh를 3분에 한 번씩 실행 (로그 생성)
- analyzer.sh를 10분에 한 번씩 실행 →  (모니터링)
    - URL별 페이지 접속 횟수, CPU 사용량, 응답 시간
``` shell
# 10분마다 Nginx 로그 분석 스크립트 실행
*/10 * * * * /home/admin/nginx-log-scripts/analyzer.sh

# 3분마다 테스트 로그 생성 스크립트 실행
*/3 * * * * /home/admin/nginx-log-scripts/createLog.sh
```

<br>

# 4. AWK
**1. 가장 많이 접속된 URL top 3**
```shell
#!/usr/bin/env bash
set -euo pipefail
DIR="/home/admin/log-reports"
shopt -s nullglob
FILES=("$DIR"/requests_*.csv)
[ ${#FILES[@]} -gt 0 ] || { echo "[ERR] no requests_*.csv"; exit 1; }

gawk -v FPAT='([^,]+)|(\"[^\"]*\")' '
  FNR==1 { next }                          # 각 파일 헤더 스킵
  {
    u=$4; gsub(/"/,"",u); sub(/\?.*/,"",u) # 쿼리스트링 제거(원치 않으면 이 줄 삭제)
    c[u]++
  }
  END{
    t1c=t2c=t3c=-1; t1u=t2u=t3u=""
    for(u in c){
      v=c[u]
      if(v>t1c){ t3c=t2c; t3u=t2u; t2c=t1c; t2u=t1u; t1c=v; t1u=u }
      else if(v>t2c){ t3c=t2c; t3u=t2u; t2c=v; t2u=u }
      else if(v>t3c){ t3c=v; t3u=u }
    }
    print "rank,count,url"
    if(t1c>=0) printf "1,%d,%s\n", t1c, t1u
    if(t2c>=0) printf "2,%d,%s\n", t2c, t2u
    if(t3c>=0) printf "3,%d,%s\n", t3c, t3u
  }
' "${FILES[@]}"
```
- 결과 사진
<img height="110" alt="image" src="https://github.com/user-attachments/assets/e8172b30-ea5c-49dc-b85a-ef58e3b84de6" />     

<br>

**2. CPU 사용량의 최대값/최소값**
```shell
#!/usr/bin/env bash
set -euo pipefail
DIR="/home/admin/log-reports"
shopt -s nullglob
FILES=("$DIR"/cpu_usage_*.csv)
[ ${#FILES[@]} -gt 0 ] || { echo "[ERR] no cpu_usage_*.csv"; exit 1; }

awk -F, '
  FNR==1 { next }                 # 헤더 스킵
  {
    ts=$1; v=$2+0
    if(n==0||v<min){min=v; mint=ts}
    if(n==0||v>max){max=v; maxt=ts}
    n++
  }
  END{
    if(n==0){print "min=NA, max=NA"; exit}
    printf "min=%.2f%% (%s)\nmax=%.2f%% (%s)\n", min,mint,max,maxt
  }
' "${FILES[@]}"
```
- 결과 사진
<img height="50" alt="image" src="https://github.com/user-attachments/assets/6e57783f-4cf4-4a59-8b3b-31e0815b4dfc" />
