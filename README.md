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
