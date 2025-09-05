# Linux-Log-Monitoring
리눅스 로그 데이터 수집 및 모니터링

# 1. nginx 설치 및 웹 페이지 띄우기

### nginx 설치

### 페이지 구성
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

# 2. 로그 실행 및 수집 스크립트
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
#!/bin/bash

set -euo pipefail

# ===== 설정 =====
OUT_DIR="/home/admin/log-reports"
NGX_LOGS="/var/log/nginx/access.log*"

# 파일명에 사용할 타임스탬프 (YYYYMMDD-HHMM 형식)
FNAME_TS="$(date '+%Y%m%d-%H%M')"
REQ_CSV="$OUT_DIR/requests_${FNAME_TS}.csv"        # 요청 단건 스냅샷
CPU_CSV="$OUT_DIR/cpu_usage_${FNAME_TS}.csv"      # CPU 스냅샷

TMP="/tmp/ngx_analytics.$$"

mkdir -p "$OUT_DIR" "$TMP"

TS="$(date '+%Y-%m-%d %H:%M:%S')"

# ===== 0) 로그 수집 (회전 .gz 포함) =====
zgrep -h -a . $NGX_LOGS > "$TMP/all.log" || true


# ===== 1) 요청 단건 CSV 생성 (스냅샷) =====
# 헤더 작성 후 새로 덮어씀 (최신 스냅샷)
echo "timestamp,ip,method,url,status,bytes,request_time_ms,referer,user_agent" > "$REQ_CSV"

# 일반적으로 combined 로그는 쿼트(")로 구분: "REQUEST" STATUS BYTES "REFERER" "UA"
# 시간은 $4 [$time_local], $5 ]타임존], 요청은 "METHOD PATH HTTP/x.y"
awk -v OFS=',' '
function esc(s){ gsub(/"/,"\"\"",s); return "\"" s "\"" }  # CSV 안전 처리
{
  # split by quote to safely get request/ref/ua even if they contain spaces
  n = split($0, q, "\"");            # q[2]=request, q[6]=referer, q[8]=ua (있을 때)
  req = (n>=2 ? q[2] : "-")
  ref = (n>=6 ? q[6] : "-")
  ua  = (n>=8 ? q[8] : "-")

  # method & url
  method="-"; url="-"
  if (match(req, /^([A-Z]+) ([^ ]+) HTTP\/[0-9.]+$/, m)) { method=m[1]; url=m[2]; }
  # timestamp: $4 is like [05/Sep/2025:11:23:45, $5 like +0900]
  t4=$4; gsub(/\[/,"",t4)
  t5=$5; gsub(/\]/,"",t5)
  ts = t4 " " t5
  ip = $1
  status = $9
  bytes  = $10

  # request_time (Nginx log_format에 포함되어야 함)
  rt_ms="NA"
  if (match($0, /request_time=([0-9.]+)/, r)) {
    rt_ms = sprintf("%.2f", r[1]*1000.0)
  }

  # CSV 출력 (referer/ua는 큰따옴표 이스케이프)
  print esc(ts), esc(ip), esc(method), esc(url), esc(status), esc(bytes), esc(rt_ms), esc(ref), esc(ua)
}
' "$TMP/all.log" >> "$REQ_CSV"


# ===== 2) CPU 사용률 스냅샷 =====
cpu_pct() {
  read -r _ u n s i o irq sirq st _ < /proc/stat
  idle1=$((i+o)); non1=$((u+n+s+irq+sirq+st)); tot1=$((idle1+non1))
  sleep 1
  read -r _ u n s i o irq sirq st _ < /proc/stat
  idle2=$((i+o)); non2=$((u+n+s+irq+sirq+st)); tot2=$((idle2+non2))
  td=$((tot2-tot1)); id=$((idle2-idle1))
  awk -v td="$td" -v id="$id" "BEGIN{if(td<=0){print \"0.00\"}else{printf \"%.2f\", (td-id)/td*100.0}}"
}
CPU="$(cpu_pct)"

# 헤더 없으면 생성
[ -s "$CPU_CSV" ] || echo "timestamp,cpu_percent" > "$CPU_CSV"
echo "$TS,$CPU" >> "$CPU_CSV"

# 정리
rm -rf "$TMP"
echo "[OK] $TS  requests→$REQ_CSV  cpu→$CPU_CSV
```
--

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

--
# 4. AWK
1. 가장 많이 접속된 URL top 3
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
  
2. CPU 사용량의 최대값/최소값
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

3. 응답 시간의 최대값/최소값
```shell
#!/usr/bin/env bash
set -euo pipefail
DIR="/home/admin/log-reports"

# requests_*.csv 전부 모음 (없으면 에러)
shopt -s nullglob
FILES=("$DIR"/requests_*.csv)
[ ${#FILES[@]} -gt 0 ] || { echo "[ERR] no requests_*.csv"; exit 1; }

# CSV 안의 따옴표/콤마 안전 파싱을 위해 gawk FPAT 사용
gawk -v FPAT='([^,]+)|(\"[^\"]*\")' '
  FNR==1 { next }                               # 각 파일의 헤더 스킵
  {
    rt=$7; gsub(/"/,"",rt); if(rt==""||rt=="NA") next
    ts=$1; gsub(/"/,"",ts)                      # 타임스탬프
    url=$4; gsub(/"/,"",url)                    # URL

    v=rt+0
    if(k==0 || v<min){ min=v; mint=ts; minu=url }
    if(k==0 || v>max){ max=v; maxt=ts; maxu=url }
    k++
  }
  END{
    if(k==0){ print "min=NA  max=NA"; exit }
    printf "min=%.2fms (%s, %s)\n", min, mint, minu
    printf "max=%.2fms (%s, %s)\n", max, maxt, maxu
  }
' "${FILES[@]}"
```
- 결과 사진
