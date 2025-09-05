# 🐧 Linux-Log-Monitoring
> Nginx 웹 서버의 접속 로그(`access.log`)를 자동 분석합니다.  
> 셸 스크립트와 `awk`로 유의미한 데이터를 추출하고, `cron`으로 **주기적 자동화**합니다.  
> 일자별 CSV 리포트로 **트래픽 패턴**과 **잠재 이슈**를 빠르게 파악할 수 있어요.

---
## 👥 Contributors

| <img width="150px" src="https://avatars.githubusercontent.com/u/78733700?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/81912226?v=4"/> |
| :---: | :---: |
| **이조은** | **정서현** |
| [@LeeJoEun-01](https://github.com/LeeJoEun-01) | [@hyunn522](https://github.com/hyunn522) |

## 🛠️ Tech Stacks & Tools
- `Linux (Ubuntu 24.04.2)`
- `Nginx`
- `Shell Script (Bash)`
- `awk`
- `cron`

| MobaXterm | Visual Studio Code |
|---|---|
|<img width="500" alt="image" src="https://github.com/user-attachments/assets/204d40a4-0533-4a2e-bd88-271b49dedabe"/>| <img width="500" alt="image" src="https://github.com/user-attachments/assets/5ef88a91-d894-4bff-9d31-a0d7fdb48b1d" /> |

---

## 1) 🌐 Nginx 설치 & 페이지 띄우기

```bash
# 패키지 목록 업데이트
sudo apt-get update

# Nginx 설치
sudo apt-get install -y nginx

# 서비스 시작 + 부팅시 자동 실행
sudo systemctl start nginx
sudo systemctl enable nginx
```

**문서 루트 구조**
```
/var/www/html/
├─ main/
│  └─ index.html
└─ detail/
   ├─ 1/ └─ index.html
   ├─ 2/ └─ index.html
   ├─ 3/ └─ index.html
   ├─ 4/ └─ index.html
   └─ 5/ └─ index.html
```

> 🔐 로그 읽기 권한  
> Nginx 로그는 보통 `adm` 그룹에서 읽을 수 있음. `admin` 사용자로 분석한다면:
> ```bash
> sudo usermod -aG adm admin
> # 새 로그인 세션에서 반영됨
> ```

---

## 2) 📜 스크립트

### 🌳 로그 실행 Tree
<img width="460" alt="image" src="https://github.com/user-attachments/assets/9c0f37a6-fc46-450f-ace0-d4e4047e5773" />

### 🔎 `createLog.sh` (트래픽/로그 생성)
```bash
#!/bin/bash

echo "새로운 파일 구조에 맞춰 로그 생성을 시작합니다..."

# --- 1. 메인 + 상세(1,3) 집중 접속 ---
echo "메인 페이지와 특정 상세 페이지(1, 3)에 접속 중..."
for i in {1..10}; do
  curl -sS http://localhost/main/     >/dev/null
  curl -sS http://localhost/detail/1/ >/dev/null
  curl -sS http://localhost/detail/3/ >/dev/null
done

# --- 2. 상세 페이지 무작위 접속 ---
echo "모든 상세 페이지에 무작위 접속"
for i in {1..15}; do
  RANDOM_PAGE=$((RANDOM % 5 + 1))
  echo "Accessing /detail/${RANDOM_PAGE}/"
  curl -sS "http://localhost/detail/${RANDOM_PAGE}/" >/dev/null
done

echo "로그 생성 완료."
```

### 🔎 `analyzer.sh` (로그 → CSV)
- 생성 파일:
  - `requests-YYYYMMDD-HHMM.csv`
  - `cpu_usage-YYYYMMDD-HHMM.csv`

```bash
#!/usr/bin/env bash
set -euo pipefail

# ===== 설정 =====
OUT_DIR="/home/admin/log-reports"
NGX_LOGS="/var/log/nginx/access.log*"

# 파일명에 사용할 타임스탬프 (YYYYMMDD-HHMM)
FNAME_TS="$(date '+%Y%m%d-%H%M')"
REQ_CSV="${OUT_DIR}/requests_${FNAME_TS}.csv"   # 요청 단건 스냅샷 
CPU_CSV="${OUT_DIR}/cpu_usage_${FNAME_TS}.csv"  # CPU 스냅샷

TMP="$(mktemp -d /tmp/ngx_analytics.XXXXXX)"
trap 'rm -rf "$TMP"' EXIT

TS="$(date '+%Y-%m-%d %H:%M:%S')"
mkdir -p "$OUT_DIR"

# ===== 0) 로그 수집 (회전 .gz 포함) =====
zgrep -h -a . $NGX_LOGS > "$TMP/all.log" || true

# ===== 1) 요청 단건 CSV 생성 =====
# 응답시간 컬럼 제거: timestamp,ip,method,url,status,bytes,referer,user_agent
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

  ip=$1; status=$9; bytes=$10
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

echo "[OK] $TS  requests=>$REQ_CSV  cpu=>$CPU_CSV"
```

---

## 3) ⏰ Crontab 자동화

```bash
crontab -e
```

```bash
# 10분마다 Nginx 로그 분석 스크립트 실행
*/10 * * * * /home/admin/nginx-log-scripts/analyzer.sh

# 3분마다 테스트 로그 생성 스크립트 실행
*/3  * * * * /home/admin/nginx-log-scripts/createLog.sh
```

> 🔎 동작 확인  
> ```bash
> watch -n 30 'ls -lh /home/admin/log-reports/*.csv | tail -n 6'
> # 또는
> grep CRON /var/log/syslog | tail
> ```

---

## 4) 📦 CSV 스키마 (컬럼 설명)

**🧾 requests-YYYYMMDD-HHMM.csv**
| 컬럼명 | 의미 | 예시 |
|---|---|---|
| `timestamp` | Nginx `$time_local` + 타임존 | `05/Sep/2025:14:47:10 +0900` |
| `ip` | 클라이언트 IP (프록시 앞단이면 XFF 고려) | `10.0.2.2`, `::1` |
| `method` | HTTP 메서드 | `GET`, `POST` |
| `url` | 요청 경로(쿼리 포함) | `/main/`, `/detail/3/` |
| `status` | HTTP 상태 코드 | `200`, `301`, `404` |
| `bytes` | `$body_bytes_sent` (응답 바디 바이트) | `1146` |
| `referer` | HTTP Referer | `-` 또는 URL |
| `user_agent` | 클라이언트 UA 문자열 | `Mozilla/5.0 ...` |

**🧠 cpu_usage-YYYYMMDD-HHMM.csv**
| 컬럼명 | 의미 | 예시 |
|---|---|---|
| `timestamp` | 수집 시각(로컬) | `2025-09-05 14:47:07` |
| `cpu_percent` | 1초 샘플 기준 총 CPU 사용률(%) | `7.25` |

---

## 5) 🧰 AWK 분석 스크립트

### ⭐ 가장 많이 접속된 URL TOP 3
```bash
#!/usr/bin/env bash
set -euo pipefail
DIR="/home/admin/log-reports"
shopt -s nullglob
FILES=("$DIR"/requests_*.csv)
[ ${#FILES[@]} -gt 0 ] || { echo "[ERR] no requests_*.csv"; exit 1; }

gawk -v FPAT='([^,]+)|(\"[^\"]*\")' '
function rep(ch,n,  s,i){ s=""; for(i=0;i<n;i++) s=s ch; return s }
function line(){ printf("+-%s-+-%s-+-%s-+\\n", rep("-",rw), rep("-",cw), rep("-",uw)) }
function row(rank,count,url){ printf("| %-*s | %*s | %-*s |\\n", rw, rank, cw, count, uw, url) }

FNR==1 { next }
{
  u=$4; gsub(/"/,"",u); sub(/\\?.*/,"",u)   
  c[u]++
}
END{
  if (!length(c)) { print "No data."; exit }

  # Top3 고르기
  t1c=t2c=t3c=-1; t1u=t2u=t3u=""
  for(u in c){
    v=c[u]
    if(v>t1c){ t3c=t2c; t3u=t2u; t2c=t1c; t2u=t1u; t1c=v; t1u=u }
    else if(v>t2c){ t3c=t2c; t3u=t2u; t2c=v; t2u=u }
    else if(v>t3c){ t3c=v; t3u=u }
  }

  # 폭 계산
  rw=length("Rank"); cw=length("Count"); uw=length("URL")
  if(t1c>=0){ if(length(t1u)>uw) uw=length(t1u); if(length(t1c)>cw) cw=length(t1c) }
  if(t2c>=0){ if(length(t2u)>uw) uw=length(t2u); if(length(t2c)>cw) cw=length(t2c) }
  if(t3c>=0){ if(length(t3u)>uw) uw=length(t3u); if(length(t3c)>cw) cw=length(t3c) }

  print "🔥 Top 3 URLs by Hits"
  line(); row("Rank","Count","URL"); line()
  if(t1c>=0) row(1, t1c, t1u)
  if(t2c>=0) row(2, t2c, t2u)
  if(t3c>=0) row(3, t3c, t3u)
  line()
}' "${FILES[@]}"
```

- 결과 사진
  
  <img height="110" alt="image" src="https://github.com/user-attachments/assets/e8172b30-ea5c-49dc-b85a-ef58e3b84de6" />

### 📈 CPU 사용률 최솟값/최댓값
```bash
#!/usr/bin/env bash
set -euo pipefail
DIR="/home/admin/log-reports"
shopt -s nullglob
FILES=("$DIR"/cpu_usage_*.csv)
[ ${#FILES[@]} -gt 0 ] || { echo "[ERR] no cpu_usage_*.csv"; exit 1; }

awk -F, '
  FNR==1 { next }                     # 헤더 스킵
  {
    ts=$1; v=$2+0
    if(n==0||v<min){min=v; mint=ts}
    if(n==0||v>max){max=v; maxt=ts}
    n++
  }
  END{
    if(n==0){print "min=NA / max=NA"; exit}
    printf "🟢 min = %.2f%%  @ %s\\n", min, mint
    printf "🔴 max = %.2f%%  @ %s\\n", max, maxt
  }
' "${FILES[@]}"
```

- 결과 사진

  <img height="50" alt="image" src="https://github.com/user-attachments/assets/6e57783f-4cf4-4a59-8b3b-31e0815b4dfc" />  
