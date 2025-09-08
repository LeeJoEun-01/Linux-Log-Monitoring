# 🐧 Service-Traffic-Analyzer
> 운영 서비스에서 발생하는 **트래픽 과부하를 CPU 사용량과 응답 시간 기반으로 모니터링**하고 분석하는 프로젝트입니다.
> 
> 스크립트와 awk를 활용해 **Nginx access.log**를 분석하고, cron을 이용해 작업을 자동화하여 시간대별 **CSV 리포트**를 생성합니다.
> 생성된 리포트를 통해 CPU 사용률과 응답 시간 지표를 종합적으로 파악할 수 있으며, 이를 기반으로 **트래픽 패턴을 분석**하고 **잠재적인 성능 이슈를 신속하게 식별**할 수 있습니다.

---
## 👥 Contributors

| <img width="150px" src="https://avatars.githubusercontent.com/u/78733700?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/88383179?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/81912226?v=4"/> |
| :---: | :---: | :---: |
| **이조은** | **정다빈** | **정서현** |
| [@LeeJoEun-01](https://github.com/LeeJoEun-01) | [@ddddabi](https://github.com/ddddabi) | [@hyunn522](https://github.com/hyunn522) |

## 🛠️ Tech Stacks & Tools
- **Linux (Ubuntu 24.04.2)**
  : Nginx 서버 구동 및 로그 분석 스크립트 실행을 위한 기본 OS 환경으로 사용
- **Nginx**
  : 실제 운영 환경과 유사한 웹 서버 접속 로그(access.log) 생성
- **Shell Script (Bash)**
  : awk, cron 등 리눅스 명령어를 조합하여 로그 분석 및 리포팅 과정 자동화
- **awk**
  : 로그 파일에서 원하는 데이터 필드를 정밀하게 추출 및 가공
- **cron**
  : 분석 스크립트의 주기적인 실행을 위한 스케줄링

| MobaXterm | Visual Studio Code |
|---|---|
|<img width="500" alt="image" src="https://github.com/user-attachments/assets/204d40a4-0533-4a2e-bd88-271b49dedabe"/>| <img width="500" alt="image" src="https://github.com/user-attachments/assets/5ef88a91-d894-4bff-9d31-a0d7fdb48b1d" /> |
| 리눅스 서버 원격 접속 및 관리를 위해 사용 | 생성된 CSV 리포트의 데이터를 확인하고 분석하기 위해 사용 |

---

## 1. 🌐 Nginx 설치 & 페이지 띄우기
실제 웹 서비스와 동일한 환경을 시뮬레이션하고, 트래픽 과부하 분석에 필요한 `접속 로그(access.log)`를 생성하기 위해 사용했습니다.

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
- 해당 구조 설계 이유: 모니터링 시스템 테스트를 위해 /main/과 /detail/ 경로를 의도적으로 분리했습니다.

  이를 통해 <U>특정 페이지(예: 상세 페이지)에 과부하가 걸리는 시나리오를 재현</U>하고, 스크립트가 다양한 트래픽 패턴을 정확히 감지하는지 검증하고자 했습니다.
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
> Nginx 로그는 보통 `adm` 그룹에서 읽을 수 있기 때문에, 다음 명령어를 통해 `admin` 사용자를 `adm` 그룹에 추가해야 합니다.
> ```bash
> sudo usermod -aG adm admin
> # 새 로그인 세션에서 반영됨
> ```

---

## 2. 📦 CSV 리포트 스키마

Nginx 접근 로그에서 파싱한 결과를 시각별 스냅샷 CSV로 저장할 때 적용되는 스키마입니다. 각 파일명은 `YYYYMMDD-HHMM` 형태이며, 한 주기의 수집 결과만 담습니다.

**`🧾 requests-YYYYMMDD-HHMM.csv`** : `Nginx access.log`를 요청 단위별로 파싱한 결과

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

**`🧠 cpu_usage-YYYYMMDD-HHMM.csv`** : 시스템 전체 CPU 사용률 스냅샷

> 웹 요청 단위가 아닌 시스템 전체 지표이므로, 동일 시각의 요청 통계와 상관관계 분석에 활용할 수 있습니다.

| 컬럼명 | 의미 | 예시 |
|---|---|---|
| `timestamp` | 수집 시각(로컬) | `2025-09-05 14:47:07` |
| `cpu_percent` | 1초 샘플 기준 총 CPU 사용률(%) | `7.25` |

---

## 2. 📜 셸 스크립트

####  셸 스크립트를 사용한 이유는❓

> **1. 가볍고 빠른 실행**
> 
> 별도의 런타임 설치 없이 모든 리눅스에서 즉시 실행됩니다. 리소스를 거의 차지하지 않아 운영 서버에 부담을 주지 않고 모니터링하기에 이상적입니다.
> 
> **2. 강력한 텍스트 처리 능력**
> 
> `awk`, `grep` 등 강력한 텍스트 처리 명령어들을 유연하게 조합하여 사용 가능합니다.
>
> **3. 시스템 제어 및 자동화의 용이성**
>
> 셸 스크립트는 운영체제의 명령어를 직접 제어하는 데 특화되어 `cron`을 이용한 스케줄링 등 자동화 작업을 가장 간단하고 직관적으로 구현합니다.

### 🌳 로그 실행 리스트
<img width="460" alt="image" src="https://github.com/user-attachments/assets/9c0f37a6-fc46-450f-ace0-d4e4047e5773" />

### 🔎 트래픽/로그 생성 스크립트 (`createLog.sh`)
Nginx 웹 서버에 의도적으로 접속하여 분석할 `access.log` 데이터를 생성
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

### 🔎 로그를 CSV로 전환하는 스크립트 (`analyzer.sh`)
모든 Nginx 접속 로그를 분석하여, **개별 요청 내역과 현재 CPU 사용률을 각각 타임스탬프가 찍힌 CSV 파일로 생성**하는 작업을 수행
- 생성 파일:
  - `requests-YYYYMMDD-HHMM.csv`
  - `cpu_usage-YYYYMMDD-HHMM.csv`

```bash
#!/usr/bin/env bash

set -euo pipefail
# ❇️ 의도치 않은 오류나 데이터 손상을 방지하는 중요한 설정

# ===== 설정 =====
OUT_DIR="/home/admin/log-reports"
NGX_LOGS="/var/log/nginx/access.log*"

# 파일명에 사용할 타임스탬프 (YYYYMMDD-HHMM)
FNAME_TS="$(date '+%Y%m%d-%H%M')"
REQ_CSV="${OUT_DIR}/requests_${FNAME_TS}.csv"   # 요청 단건 스냅샷 
CPU_CSV="${OUT_DIR}/cpu_usage_${FNAME_TS}.csv"  # CPU 스냅샷

TMP="$(mktemp -d /tmp/ngx_analytics.XXXXXX)"
trap 'rm -rf "$TMP"' EXIT
# ❇️ 임시 파일/디렉터리 자동 정리

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
  # ❇️ CPU 카운터 값을 1초 간격으로 두 번 읽어 그 차이를 계산함으로써,실제 1초 동안의 평균 CPU 사용률을 비교적 정확하게 측정
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

## 4. ⏰ Crontab을 통한 로그 수집 자동화

`crontab`을 활용해 트래픽 발생 및 `Nginx`의 `access.log`를 수집·기록하는 작업을 자동화합니다.

```bash
crontab -e
```

```bash
# 3분 주기로 트래픽 발생 스크립트 실행
*/3  * * * * /home/admin/nginx-log-scripts/createLog.sh

# 10분 주기로 Nginx 로그 분석 스크립트 실행
*/10 * * * * /home/admin/nginx-log-scripts/analyzer.sh
```

> 🔎 동작 확인  
> ```bash
> watch -n 30 'ls -lh /home/admin/log-reports/*.csv | tail -n 6'
> # 또는
> grep CRON /var/log/syslog | tail
> ```

---

## 5. 🧰 AWK 분석 스크립트

crontab이 쌓아둔 CSV 스냅샷을 모아 간단한 인사이트를 즉시 산출합니다.

### ⭐ 가장 많이 접속된 URL TOP 3

여러 `requests_*.csv`를 합쳐 URL별 요청 수를 집계하고, 상위 3개를 콘솔 표로 출력합니다.

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
  
  <img height="140" alt="image" src="https://github.com/user-attachments/assets/e8172b30-ea5c-49dc-b85a-ef58e3b84de6" />

### 📈 CPU 사용률 최솟값/최댓값

여러 `cpu_usage_*.csv`의 cpu_percent 값을 모아 전체 기간(min/max)과 해당 시각을 출력합니다.

이 값은 요청 단위가 아니라 시스템 전체 스냅샷(1초 표본)입니다.

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
