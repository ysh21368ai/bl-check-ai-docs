# 모니터링

운영 상태를 빠르게 확인할 수 있는 SQL / 명령어 모음.

## 매일 1분 점검

```bash
# 1. cron 최근 발화 (운영 OS)
journalctl -u cron --no-pager -n 10 | grep bl_check

# 2. 최근 사이클 로그
tail -30 "/home/dev01/seunghyun/project/고객지원팀/bl_check_management_arrange - final_kr/logs/bl_check_$(date +%Y-%m-%d).log" \
  | grep -E "START|END|완료:|FAIL|DPY-"
```

## DB 모니터링 SQL

### 1) 큐 상태

```sql
SELECT CHECKTP, COUNT(*) AS cnt
FROM LINER.T_AICHECK_TARGET
GROUP BY CHECKTP;
```

| CHECKTP | 정상 | 비정상 |
|---|---|---|
| `'I'` | 0 ~ 수십 건 | 100건+ = 처리 지연 |
| `'U'` | 0 ~ 20 | 50건+ = hang 의심 |

### 2) 좀비 세션

```sql
SELECT sid, serial#, status, last_call_et, machine, sql_id, event, wait_class
FROM v$session
WHERE username = 'LINER' AND machine = 'isteam1'
  AND status = 'ACTIVE' AND last_call_et > 60
ORDER BY last_call_et DESC;
```

→ 0건이면 정상.

### 3) 시간당 처리량

```sql
SELECT TO_CHAR(INPDATE, 'YYYY-MM-DD HH24') AS hour,
       COUNT(DISTINCT BLNO) AS bl_cnt
FROM LINER.T_AICHECK_RESULT
WHERE INPDATE >= SYSTIMESTAMP - INTERVAL '24' HOUR
GROUP BY TO_CHAR(INPDATE, 'YYYY-MM-DD HH24')
ORDER BY hour;
```

운영 기준: 시간당 100~200건. 0건 시간대 발생 시 hang 의심.

### 4) 룰별 FAIL 률 (최근 7일)

```sql
SELECT RULE_CODE,
       COUNT(*) AS total,
       SUM(CASE WHEN CHECK_RESULT='N' THEN 1 ELSE 0 END) AS fail_cnt,
       ROUND(SUM(CASE WHEN CHECK_RESULT='N' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 1) AS fail_pct
FROM LINER.T_AICHECK_RESULT
WHERE INPDATE >= SYSTIMESTAMP - INTERVAL '7' DAY
GROUP BY RULE_CODE
ORDER BY fail_cnt DESC;
```

→ FAIL 률 비정상 ↑ 시 룰 본문 또는 적용 범위 점검.

### 5) AI vs Human 비교 (False Positive)

AI 가 FAIL → 휴먼이 PASS 로 뒤집은 BL:

```sql
SELECT DISTINCT ai.BLNO, ai.PASS AS ai_pass, hm.PASS AS human_pass,
       hm.SRC AS human_src, hm.INPUSER, hm.INPDATE AS human_time
FROM LINER.T_BLCHECK_AUTO_H ai
JOIN LINER.T_BLCHECK_AUTO_H hm ON ai.BLNO = hm.BLNO
WHERE ai.SRC = 'AI' AND ai.PASS = 'N'
  AND hm.SRC <> 'AI' AND hm.PASS = 'Y'
  AND hm.INPDATE > ai.INPDATE
ORDER BY ai.BLNO;
```

→ False Positive 분석. 모델 튜닝 / 룰 조정 후보.

### 6) 토큰 사용량

cron 사이클별 토큰 사용량은 `output/runYYYYMMDD_HHMM_token_usage.json` 에 저장됨.

```bash
ls -la output/run*_token_usage.json | tail -10
cat output/run20260520_0935_token_usage.json
```

```json
{
  "calls": 40,
  "input_tokens": 187645,
  "output_tokens": 14321,
  "reasoning_tokens": 4813,
  "cache_read_tokens": 137728
}
```

월간 토큰 사용량 집계:

```bash
# (수동 합산 — 추후 자동화 스크립트 가능)
ls output/run20260520_*_token_usage.json | xargs cat | \
  python3 -c "import sys,json; t=[json.loads(l) for l in sys.stdin if l.strip().startswith('{')]; print('input:', sum(x['input_tokens'] for x in t))"
```

## 시스템 자원

### 디스크

```bash
df -h /home/dev01
du -sh /home/dev01/seunghyun/project/고객지원팀/bl_check_management_arrange*/
du -sh /home/dev01/seunghyun/project/고객지원팀/bl_check_management_arrange*/output/
ls /home/dev01/seunghyun/project/고객지원팀/bl_check_management_arrange*/output/run* | wc -l
```

⚠ output/run* 디렉토리 영구 누적. 7일 이상 정기 정리 권장 ([운영 매뉴얼](runbook.md)).

### 메모리

```bash
ps -eo pid,rss,cmd | grep bl_check | grep -v grep
```

cron 사이클 진행 중인 Python 프로세스의 RSS 메모리 확인. 일반적으로 500MB 이하.

### Oracle 측

```sql
-- 우리 user 의 현재 세션 수
SELECT username, COUNT(*) FROM v$session
WHERE username='LINER' GROUP BY username;

-- 우리 user 의 SessionPool 최대 = 5 → 보통 2~5개 보임

-- 좀비 후보 (LINER + 1시간+ ACTIVE)
SELECT sid, serial#, last_call_et/60 AS min,
       sql_id, status, event
FROM v$session
WHERE username='LINER' AND status='ACTIVE' AND last_call_et > 3600;
```

## 대시보드 자동화 (향후 작업)

현재는 수동 SQL 조회. 향후:

- **Prometheus + Grafana** — 메트릭 수집 (큐 크기, 처리량, 토큰 사용량)
- **OpenAPI / FastAPI** 로 메트릭 endpoint 노출
- **Slack 알림** — 좀비 발생 / 큐 누적 / SHELL_TIMEOUT 시 자동 통보

## 관련 문서

- [일상 운영 매뉴얼](runbook.md)
- [트러블슈팅](troubleshooting.md)
