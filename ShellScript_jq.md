# 📜 jq 기반 명언 출력기

> 리눅스 학습 과정에서 지칠 때마다 따뜻한 위로와 통찰을 건네는 치유형 명령어 구현 — 명언 API를 활용해 콘솔에 귀엽게 명언 메시지 출력하기
> 실시간 API 호출, `jq` 파싱, CSV 저장까지 자동화!


---

## 👥 Contributors

| <img width="150px" src="https://avatars.githubusercontent.com/u/78733700?v=4"/> | <img width="150px" src="https://avatars.githubusercontent.com/u/88383179?v=4"/> |
| :---: | :---: |
| **이조은** | **정다빈** |
| [@LeeJoEun-01](https://github.com/LeeJoEun-01) | [@ddddabi](https://github.com/ddddabi) |

---

## 1. jq 기반 명언 API 출력기 (CLI 도구)

### 🔗 사용 API

- **API 주소**: `https://korean-advice-open-api.vercel.app/api/advice`
- **응답 예시**:
  ```
    {
      "author": "헬렌 켈러",
      "authorProfile": "미국 작가, 사회운동가, 강연가",
      "message": "세상은 고통으로 가득하지만 그것을 이겨내는 사람들로도 가득하다."
    }
  ```

### 🛠 사용 기술
  | 도구/명령어             | 설명                    |
  | ------------------ | --------------------- |
  | `curl`             | API 호출                |
  | `jq`               | JSON 파싱               |
  | `mkdir -p`         | 날짜별 로그 디렉토리 자동 생성     |
  | `echo >> file.csv` | 로그 CSV 저장             |

### 📄 CSV 로그 예시
| Timestamp           | Author     | Profile | Message         |
| ------------------- | ---------- | ------- | --------------- |
| 2025-09-09 16:20:52 | 알베르트 아인슈타인 | 물리학자    | 상상력은 지식보다 중요하다. |

- **저장 경로**: ``` ~/.../korean_quotes.csv ```

- **로그 형식**: ``` 2025-09-09 16:20:52","알베르트 아인슈타인","물리학자","상상력은 지식보다 중요하다.```

## 2. 사용 방법
- CLI 명령어로 쉽게 실행할 수 있어요.
  ```
    # 1. 실행 권한 부여
    chmod +x korean_quote.sh

    # 2. 직접 실행
    ./korean_quote.sh
    
    # 3. helpme 명령어로 실행하려면
    echo "alias helpme='$HOME/04.mission/02.jq/korean_quote.sh'" >> ~/.bashrc
    source ~/.bashrc
  
    # 이후
    helpme
  ```

## 3. 출력 예시
-  케이크 고양이
    ```
      {\___/}
      ( • - •)
       />🍰 세상은 고통으로 가득하지만 그것을
             이겨내는 사람들로도 가득하다.
    
     👤 헬렌 켈러 (미국 작가, 사회운동가, 강연가)
     📅 2025-09-09 16:15:12
    ```
- 도시락 고양이 세트
    ```
     /\_/\      (\ __ /)        A__A
     ( ˶•o•˶)    ( •ω• )       ( •⤙•  )
     ଘ( ა🍱 )   ( ა🍙૮ )｡     ( 🍜٩  )੭
  
      세상은 고통으로 가득하지만 그것을
      이겨내는 사람들로도 가득하다.
  
   👤 헬렌 켈러 (미국 작가, 사회운동가, 강연가)
   📅 2025-09-09 16:15:12
  ```
- 고양이 기차<br>
  | IMAGE     | GIF                    |
  | ------------------ | --------------------- |
  | <img width="350" alt="image" src="https://github.com/user-attachments/assets/323ba3ca-ed1d-4b13-be66-c59610c096b9" /> |<img width="500" alt="image" src="https://github.com/user-attachments/assets/1cc46d23-8948-42be-a01f-4a280e8e10cb" />             |

