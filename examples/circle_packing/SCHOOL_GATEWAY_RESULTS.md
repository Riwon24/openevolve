# Circle Packing — 학교 게이트웨이(Gemini 3.7/3.1) 실행 기록

`examples/circle_packing`의 n=26 원 패킹 문제를 학교 Gateway API로 돌린 결과 정리.
(로그 원본은 `openevolve_output_school/logs/`, 프로그램별 상세는 `openevolve_output_school/checkpoints/`에 있음 — 둘 다 `.gitignore`로 저장소에는 올라가지 않음)

## 설정 (`config_phase_1_school.yaml`)

- **API**: 학교 게이트웨이 `https://factchat-cloud.mindlogic.ai/v1/gateway`
- **모델 앙상블**: `gemini-3.7-flash` (weight 0.8, 주력) + `gemini-3.1-pro-preview` (weight 0.2, `reasoning_effort: low`)
- **평가**: cascade 2단계 (threshold 0.5 / 0.75), `parallel_evaluations: 4`, `timeout: 60s`
- **DB**: `population_size: 60`, `num_islands: 4`, `elite_selection_ratio: 0.3`, `exploitation_ratio: 0.7`
- **진화 방식**: `diff_based_evolution: false` (매번 EVOLVE-BLOCK 전체를 다시 작성하는 full-rewrite 모드)
- **random_seed**: `null` — 게이트웨이가 OpenAI의 `seed` 파라미터를 인식하지 못해서 끔 (아래 시도 1 참고)

## 실행 이력 (총 3번, 체크포인트로 이어짐)

| 시각 | 결과 |
|---|---|
| 17:45:40 | **1차 시도** — `seed` 파라미터를 게이트웨이가 400으로 거부(`Unknown name "seed"`). iteration 4에서 전부 실패, 개선 없이(0.3642) 수동 중단 → config에서 `random_seed: null`로 수정 |
| 17:48:10 | **2차 시도** — 정상 진화 시작. iteration 3부터 바로 개선, iteration 18에서 최고점 도달 후 iteration ~14 부근 수동 중단(checkpoint_10, 20 저장) |
| 17:58:38 | **3차 시도** (체크포인트 재개) — checkpoint_30까지 도달했지만 이 로그 파일은 12줄에서 끊겨 있음(강제 종료로 로그 버퍼가 디스크에 flush되기 전에 프로세스가 죽은 것으로 추정). 최고점 갱신은 없었음 |

## 점수 진행

| iteration | combined_score | 비고 |
|---|---|---|
| 0 | 0.3642 | 초기 프로그램 (원 8+16개 링 배치, sum_radii=0.9598) |
| 3 | 0.9838 | 완전히 다른 배치 발견 |
| 4 | 0.9904 | |
| 8 | 0.9800 | |
| 10 | 0.9985 | checkpoint_10 시점 최고 |
| 12 | 0.9913 | |
| 14 | 0.9962 | |
| **18** | **1.0003** ⭐ | 최종 best (아래 참고) |
| 20 ~ 30 | 1.0003 (변화 없음) | 추가 개선 못 찾고 정체 |

## 최종 결과 (`openevolve_output_school/best/best_program_info.json`)

```
sum_radii    = 2.6359830849175645
target_ratio = 1.0003730872552428   (목표의 100.04%)
validity     = 1.0
```

AlphaEvolve 논문의 n=26 목표값(2.635)을 **근소하게 초과**. 18번의 반복, 총 소요 시간 약 10분.

## 진화 중 관찰된 개별 실패 (정상적인 노이즈 — 전체를 죽이지 않음)

- `module 'program' has no attribute 'run_packing'` — LLM이 고정 래퍼 함수까지 건드려서 발생
- `unterminated triple-quoted string literal` — 생성 코드 문법 오류
- `not enough values to unpack (expected 3, got 2)` — 반환값 개수 불일치
- stage1 timeout (60초 초과)

이런 개체들은 `combined_score`가 0에 가깝게 처리되고 그냥 버려질 뿐, 나머지 워커/섬의 진화는 계속 진행됨.

## 함께 삭제한 실행

- `openevolve_output_gemini/` — OpenRouter 유료 Gemini 2.5 Flash/Pro 조합. 1번째 반복부터 크레딧 부족(HTTP 402)으로 진화가 전혀 진행되지 않음
- `openevolve_output_free/` — OpenRouter 무료 모델(`nex-agi/*:free`) 조합. 대부분 빈 응답("LLM returned None response"), 드물게 나온 응답도 배열 shape 오류. 관련 config(`config_phase_1_free.yaml`, `config_phase_1_gemini.yaml`)는 나중에 유료 키로 재시도할 수 있어 남겨둠, 결과 폴더만 삭제
