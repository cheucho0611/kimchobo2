# 관제 스택 (Grafana · Loki · Prometheus · Promtail)

탐지기가 남긴 이벤트(JSON 한 줄)를 수집해서 Grafana 화면에 보여주는 관제 파트입니다.
STEP 3 관제 목표 "Grafana/Loki/Prometheus/수집기 기동, 샘플 비식별 경보 1건 수신"을 Kali에서 확인 완료했습니다. (2026-10-16)

## 구조

```
탐지기(AI·웹 공격 담당) → events/*.jsonl → Promtail(수집) → Loki(저장) → Grafana(화면)
                                         Prometheus는 위 서비스들이 살아 있는지 감시
```

```
monitoring/
├── docker-compose.yml          # 컨테이너 4개 정의
├── promtail.yml                # events/ 폴더를 읽어 Loki로 전송 + 라벨 부여
├── prometheus.yml              # loki·promtail·prometheus 상태 감시 (15초 간격)
├── grafana/provisioning/datasources/datasources.yml   # Loki·Prometheus 자동 연결
├── events/
│   └── sample_event.jsonl      # 테스트용 비식별 샘플 경보 1건
└── .gitignore                  # 실제 탐지 결과(events/*.jsonl)는 Git 제외
```

## 시작 전에 필요한 것
- Docker + Docker Compose (`docker compose version`으로 확인)
- 비어 있어야 하는 포트: 3000(Grafana), 3100(Loki), 9090(Prometheus)
  - 확인: `sudo ss -tlnp | grep -E ':(3000|3100|9090) '` → 아무것도 안 나오면 OK

## 실행 방법

```bash
cd monitoring
docker compose up -d
docker compose ps            # 4개 모두 Up 이면 정상
```

| 확인 대상 | 방법 | 정상 결과 |
|---|---|---|
| Grafana | 브라우저 `http://localhost:3000` (admin / admin) | 로그인됨 |
| 데이터소스 | Grafana → Connections → Data sources | Loki, Prometheus 자동 등록 |
| Prometheus | 브라우저 `http://localhost:9090/targets` | 3개 모두 UP |
| Loki | `curl localhost:3100/ready` | `ready` |

## 샘플 경보 확인

실행하면 `events/sample_event.jsonl`이 자동으로 수집됩니다.

1. Grafana → **Explore** → 데이터소스 **Loki**
2. 오른쪽 `Builder | Code` 중 **Code** 선택
3. 쿼리 입력 후 Run query:
   ```
   {job="detections"}
   ```
4. `TEST-001` 로그 1건이 보이면 정상

경보를 하나 더 보내 보려면:
```bash
echo '{"event_id":"test-0002","event_time":"2026-10-16T16:30:00+09:00","source":"pii","severity":"medium","detection_type":"pii","rule_id":"TEST-002","sanitized_excerpt":"연락처 [PHONE]","data_origin":"historical_replay"}' >> events/detections.jsonl
```

## 탐지 담당자에게: 이벤트 넘기는 방법 (STEP 4 연결 지점)

탐지 결과를 **아래 8개 필드의 JSON 한 줄**로 `monitoring/events/detections.jsonl`에 **덧붙여(append)** 쓰면 Grafana에 자동으로 표시됩니다.

| 필드 | 설명 | 예시 |
|---|---|---|
| event_id | 이벤트 고유 ID | `test-0001` 또는 UUID |
| event_time | 발생 시각 (ISO 8601, KST) | `2026-10-16T16:00:00+09:00` |
| source | 데이터 출처 | `csic`, `pii`, `chatbot`, `prowlbench`, `shop_web`, `fim` |
| severity | 위험도 | `low`, `medium`, `high`, `critical` |
| detection_type | 탐지 종류 | `sqli`, `xss`, `path_traversal`, `pii`, `secret`, `intrusion`, `file_change` |
| rule_id | 탐지 규칙 ID | `WEB-001`, `PII-003`, `SEC-002` |
| sanitized_excerpt | **마스킹된** 발췌문 (최대 200자) | `연락처 [PHONE]` |
| data_origin | 과거 데이터 재생인지 실시간인지 | `historical_replay`, `live` |

**규칙**
- `sanitized_excerpt`에 원문 개인정보·쿠키·API Key를 절대 넣지 않기 (마스킹 후 기록)
- 파일은 한 줄에 이벤트 하나 (JSONL)
- 덮어쓰기(`>`) 말고 덧붙이기(`>>` / 파이썬은 `open(..., "a")`)

Python 예시:
```python
import json
event = {"event_id": "...", "event_time": "...", "source": "pii", "severity": "high",
         "detection_type": "pii", "rule_id": "PII-003",
         "sanitized_excerpt": "제 번호는 [PHONE]", "data_origin": "historical_replay"}
with open("monitoring/events/detections.jsonl", "a", encoding="utf-8") as f:
    f.write(json.dumps(event, ensure_ascii=False) + "\n")
```

## 유용한 Grafana 쿼리

```
{job="detections"}                                  # 전체 탐지 이벤트
{job="detections", severity="high"}                 # 위험도 high만
{job="detections", data_origin="live"}              # 실시간 이벤트만
sum by (severity) (count_over_time({job="detections"}[5m]))   # 위험도별 건수 그래프
```

## 자주 나는 문제

| 증상 | 원인 / 해결 |
|---|---|
| `address already in use` | 3000/3100/9090 포트를 다른 컨테이너가 사용 중 → `docker ps` 확인 후 정지 |
| Grafana 데이터소스 오류 `connection refused` | URL에 `localhost` 사용 X → 컨테이너 이름 `http://prometheus:9090`, `http://loki:3100` 사용. `/targets` 붙이지 않기 |
| Explore에 아무것도 안 보임 | 시간 범위를 Last 1 hour로, 10~20초 후 다시 Run query. 그래도 없으면 `docker compose logs promtail` 확인 |
| Loki `Ingester not ready` | 시작 직후 정상. 20~30초 기다리기 |
| `loki-config.yaml` 파일이 없음 | 정상. 이미지 내장 설정(`/etc/loki/local-config.yaml`) 사용 |

## 종료
```bash
docker compose down
```

## 다음 할 일
- [ ] AI 담당 탐지 결과를 `events/detections.jsonl`로 연결 (STEP 4)
- [ ] 대시보드 패널: 위험도별 건수, 탐지 유형별 건수, 최근 이벤트 목록
- [ ] `historical_replay` / `live` 구분 패널
