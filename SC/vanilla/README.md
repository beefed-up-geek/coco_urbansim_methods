# SC vanilla — 안전 제약 필터, 제약은 VLM 이 자연어 조문에서 뽑는다

원리(`../paper.pdf`): 정책이 낸 행동을 **안전 집합 안으로 투영**한다. vanilla 에서는 그 안전 집합을
**gemini-flash** 가 법 조문(자연어)과 특권 상태를 읽고 **제약 명세(JSON)** 로 쓴다. 정책 가중치는 손대지 않는다.

## 입력

- 법역 하나(`GA|OH|EE`)의 조문 요약(영어, `environment/readme.md` §4 의 임계값을 그대로 쓴 문장들).
- 태스크 설명 한 줄.
- 특권 상태(`/priv`) 의 `robot, surface, crosswalk, signal, section, pedestrians(list 제외)`.
- 현재 에고 프레임 1장(선택).

## VLM 이 내야 하는 것 — 제약 어휘(필터가 아는 것만)

| 키 | 값 | 필터 동작 |
|---|---|---|
| `signal_gate` | `"green"` / `"green_and_finish"` / null | 횡단보도 가장자리 2 m 이내·진행방향이 건너는 쪽일 때, 조건 미충족이면 `vx=0` |
| `no_stop_on_cw` | bool | 도색 위에서 `vx≤0` 이면 `vx=1.2` |
| `full_speed_on_cw` | bool | 도색 위에서 `vx=1.2`, `kph=0` |
| `keep_half` | `"right"` / null | 도색 위에서 `right_half=false` 면 `wz=min(wz,−0.4)` |
| `min_free_m` | 숫자 / null | 보도에서 `free_m` 미달이면 가장자리 쪽으로 조향 |
| `yield_dist_m` | 숫자 / null | 전방 보행자가 이 거리 안(측방 0.9 m 이내)이면 `vx=0` |
| `overtake_lat_m` | 숫자 / null | 앞지르는 중 측방 미달이면 바깥으로 `±0.5` |
| `roadway` | `"forbid"` / `"right"` / `"allow"` / null | 차도(도색 위 제외)에서 후진 / 우측 조향 |
| `beacon_on_roadway` | bool | 차도에 있으면 `/beacon on` |

출력은 JSON 하나 + `reason`. **추론 모델이라 `max_tokens` 를 1500 이상** 줘야 한다 — 작으면 JSON 이 잘려 빈 필터가 된다.

## 언제 부르나

에피소드 시작 때 1회(**동기로 기다린다** — 필터 없이 출발하면 의미가 없다), 면이 `roadway`/`shared` 로 바뀔 때 1회 더
(비동기). 한 호출 4~6 s. 예산 30회/에피소드.

## 필터

`apply(vx, wz, priv) → (vx', wz', beacon, kph, note)`. 위 표의 규칙을 위에서 아래로 적용한다. 도색 위에서는
차도 규칙을 적용하지 않는다(횡단은 차도 주행이 아니고 "우측"이 무의미하다).
