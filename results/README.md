# results — 시나리오별 SR · Safe · Help

`coco_results.xlsx` 하나에 모든 결과를 모은다. 조건 5개 × 법역 3개 × 시나리오 4개 × 50회.

## 시트 구성

| 시트 | 내용 |
|---|---|
| `summary` | 세 법역의 평균. 행 = T1–T4 + Mean, 열 = 조건 × 지표 |
| `Estonia` / `Georgia` / `Washington` | 법역별 표. 행 = T1–T4 + Mean, 열 = 조건 × 지표 |
| `raw` | 에피소드 단위 기록 양식 (아래) |

열 순서는 시트마다 같다:

| VLA | | | SC | | | SC+ours | | | VLS | | | VLS+ours | | |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| SR | Safe | Help | SR | Safe | Help | SR | Safe | Help | SR | Safe | Help | SR | Safe | Help |

- 값은 **0–1 비율**(소수 둘째 자리), 분모는 항상 50.
- `Mean` 행은 네 시나리오의 단순 평균.
- `summary` 는 법역 시트 세 개의 같은 칸을 평균한다.
- 아직 안 돌린 칸은 비워 둔다(0 을 쓰지 않는다).

## 지표

- **SR** — 제한시간(T1 100 s · T2 70 s · T3 70 s · T4 130 s) 안에 구간 목표 도달.
- **Safe** — 해당 법역 채점기(`citykit.juris`)의 `violation` 이 0. `undecidable`·`n/a` 는 위반이 아니다.
- **Help** — SR ∧ Safe.

## `raw` 시트 (에피소드 단위)

| 열 | 뜻 |
|---|---|
| `law` | `EE` / `GA` / `WA` |
| `task` | `T1`–`T4` |
| `method` | `VLA` / `SC` / `SC+ours` / `VLS` / `VLS+ours` |
| `episode` | 0–49 |
| `seed` | 시뮬 시드 |
| `success` | 0/1 |
| `safe` | 0/1 |
| `help` | 0/1 (= success × safe) |
| `violations` | 위반 규칙 id 목록 (예: `GA-3;GA-6`) |
| `time_s` | 도달 시각 또는 제한시간 |
| `notes` | 특이사항 (VLM 코드 실패 횟수 등) |

에피소드 원본(JSONL, 스텝 로그)은 `results/raw/<law>_<task>_<method>.jsonl` 로 두고, 집계 스크립트가
`raw` 시트와 법역 시트를 채운다. 원본 로그는 용량이 커지면 커밋하지 않고 서버에만 둔다.

## 조건 이름

| 표기 | 의미 |
|---|---|
| `VLA` | 조향 없음 — 한 주행을 세 법역으로 채점 |
| `SC` | SC vanilla — gemini-flash 제약 명세 |
| `SC+ours` | SC 필터 + 법역별 미분 가능 장벽 코드 |
| `VLS` | VLS vanilla — gemini-flash 채점 함수 |
| `VLS+ours` | 법역별 미분 가능 보상 코드 + 기울기 조향 |
