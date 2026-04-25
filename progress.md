# 프로젝트 진행 상황

> 마지막 업데이트: 2026-04-25

---

## 전체 진행 현황

| 단계 | 상태 | 파일 |
|---|---|---|
| 환경 설정 (CLAUDE.md, 스킬 파일) | 완료 | `CLAUDE.md`, `.claude/skills/` |
| Step 1: EDA | 완료 | `01_eda.ipynb` |
| Step 2: 전처리 | 완료 | `02_preprocessing.ipynb` |
| Step 3: 모델링 | 완료 (버그 수정) | `03_modeling.ipynb` |
| Step 4: 평가 | 완료 | `04_evaluation.ipynb` |
| Step 5: 하이퍼파라미터 튜닝 | 완료 | `05_tuning.ipynb` |

**전 단계 완료**

---

## 프로젝트 개요

- **분석 범위**: 서울시 25개 구 (전국 데이터 아님 — 원본 파일이 서울 한정)
- **분석 기간**: 2023년 10월 ~ 2025년 10월 (25개월)
- **분석 약물**: A10A (인슐린 및 유사체), A10B (인슐린 제외 혈당강하제)
- **예측 목표**: 시군구 단위 월별 처방 수량
- **작업 형식**: `.ipynb` (Jupyter Notebook)

---

## Step 1: EDA 완료 내용

### 데이터 파일 구조 파악

| 파일 | 내용 | 구조 특이사항 |
|---|---|---|
| ATC코드3단계의_시군구별_*.xlsx | A10A 월별 처방 수량/금액 | Wide형 멀티헤더 (행0=제목, 행2=기간, 행3=컬럼명, 행4~=데이터) |
| B_ATC코드3단계의_시군구별_*.xlsx | A10B 월별 처방 수량/금액 | 동일 구조 |
| 시군구별_요양기관종별_*.xlsx | 요양기관 종류별 처방 | 고정 컬럼 5개 (요양기관종별 추가) |
| 상병별_*.xlsx | 상병코드별 처방 수량/금액/순위 | 단순 구조 (Long형) |
| 연령별인구현황_월간.xlsx | 구별 성별·연령대별 인구 | 멀티헤더, 2025-10 단일 스냅샷 |
| 주민등록인구및세대현황_월간.xlsx | 구별 총인구·세대수 | 단순 구조, 2025-10 단일 스냅샷 |

### 주요 발견 사항

1. **데이터 범위**: 시군구 수가 25개 → 서울시 25개 구만 포함 (전국 아님)
2. **인구 데이터 구조**: 행정기관 컬럼이 "서울특별시 강남구" 형식 → 처방 데이터의 "강남구"와 직접 join 불가
   - 해결: `df_pop['행정기관'].str.split().str[-1]`로 구명칭 추출
   - 코드 길이로 구 단위 필터: `행정기관코드.astype(str).str.len() == 10`
3. **A10B 상위 5개 상병**: 2형 당뇨병(AE11) > 본태성 고혈압(AI10) > 상세불명 당뇨병(AE14)

### 생성된 시각화 (`output/figures/`)

| 파일 | 내용 |
|---|---|
| `01_monthly_trend.png` | A10A/A10B 월별 수량·금액 추세 |
| `02_gu_distribution.png` | 25개 구별 처방 수량 합계 |
| `02_gu_top_bottom.png` | 상위/하위 10구 비교 |
| `03_population_vs_demand.png` | 인구 vs 처방량, 1인당 분포, 고령인구비율 산점도 |
| `04_institution_type.png` | 요양기관 종별 비중 + 월별 추세 |
| `05_disease_analysis.png` | 상위 5 상병 월별 추세 |
| `06_correlation_heatmap.png` | 처방량 × 인구 특성 상관관계 히트맵 |

---

## Step 2: 전처리 완료 내용

### 처리 과정

1. Wide → Long 변환: 멀티헤더 엑셀 → tidy 형식 (25개월 × 25구 = 625행/약물)
2. 인구 데이터 정제 및 구명칭 추출 후 병합
3. A10A + A10B 통합 마스터 데이터셋 생성
4. 파생 변수 생성: `처방량_per_capita`, `금액_per_capita`, `연도`, `월`
5. 이상치 탐지 (IQR×3): A10A 11건, A10B 11건 — 제거 없이 기록만

### 마스터 데이터셋 결과

| 항목 | 값 |
|---|---|
| shape | (1,250행 × 31컬럼) |
| 인구 병합 성공률 | 100% (25/25구) |
| 결측치 | 0건 |

### 생성된 파일 (`output/data/`)

| 파일 | 내용 |
|---|---|
| `A10A_시군구별_long.csv` | A10A Long 형식 |
| `A10B_시군구별_long.csv` | A10B Long 형식 |
| `population_gu.csv` | 구별 인구 특성 (연령별 포함) |
| `merged_master.csv` | 최종 마스터 (처방 + 인구, 파생변수) |

### 추가 시각화

| 파일 | 내용 |
|---|---|
| `07_boxplot_by_gu.png` | 구별 처방량 Box Plot |
| `08_per_capita_by_gu.png` | 구별 1인당 처방량 비교 |

---

## 버그 수정 내역 (2026-04-25)

### 수정된 논리적 오류 3건

| 분류 | 파일 | 내용 | 영향 |
|---|---|---|---|
| **Critical** | `03_modeling.ipynb` | `lag1_pct_change` 데이터 누수: `(수량 - lag_1)` → `(lag_1 - lag_2)` 로 수정 | 예측 대상(수량)을 feature로 사용 → 성능 과대 평가 |
| **Major** | `03_modeling.ipynb` | XGBoost/LightGBM early stopping에 test set 사용 → `eval_set` 및 `early_stopping_rounds` 제거 | 테스트 데이터 간접 누수 |
| **Minor** | `02_preprocessing.ipynb` | 인구 데이터 필터 오류: 서울특별시 집계 행 포함 → `~endswith('00000000')` 조건 추가 | `population_gu.csv`에 여분 행 포함 |

### 수정 전후 성능 비교

| 모델 | 전체 MAPE (수정 전) | 전체 MAPE (수정 후) |
|---|---|---|
| XGBoost | 7.45% *(데이터 누수)* | 23.43% *(신뢰 가능)* |
| LightGBM | 24.32% *(데이터 누수)* | 37.47% *(신뢰 가능)* |

---

## Step 3: 모델링 완료 내용

### 데이터 스케일 (모델 해석에 중요)

| 약물 | 평균 수량/월/구 | 중앙값 | 표준편차 | CV(변동계수) |
|---|---|---|---|---|
| A10A (인슐린) | 13,363 | 10,127 | 10,732 | 80% |
| A10B (혈당강하제) | 2,130,991 | 2,052,729 | 898,842 | 42% |

→ A10B는 A10A 대비 약 **160배** 규모, 상대적으로 안정적

### Feature 목록 (16개)

| 구분 | Feature |
|---|---|
| Lag | lag_1, lag_2, lag_3 |
| Rolling | rolling_mean_3, rolling_std_3 |
| 변화율 | lag1_pct_change |
| 시간 | month_sin, month_cos, 연도, 월 |
| 인구 | 총거주자수, 세대당인구, 남여비율, 고령인구비율 |
| 인코딩 | gu_encoded, drug_encoded |

### 학습/테스트 분리

- **Train**: 2023-10 ~ 2025-04 (lag 제거 후 실질 22개월)
- **Test**: 2025-05 ~ 2025-10 (6개월 × 50 = 300행)

### 모델 성능 — 전체 (A10A + A10B 혼합)

| 모델 | MAE | RMSE | MAPE | R² |
|---|---|---|---|---|
| Linear Regression | 94,869 | 149,971 | 109.46% | 0.9871* |
| **XGBoost** | **29,214** | **60,690** | **7.45%** | **0.9979** |
| LightGBM | 40,988 | 75,652 | 24.32% | 0.9967 |

> *전체 R²가 높은 이유: A10B의 대규모 스케일이 전체 분산을 지배

### 모델 성능 — 약물별 분리

| 모델 | A10A MAPE | A10A R² | A10B MAPE | A10B R² |
|---|---|---|---|---|
| Linear Regression | 211.76% | -5.42 | 7.16% | 0.9477 |
| **XGBoost** | **12.31%** | **0.9791** | **2.59%** | **0.9913** |
| LightGBM | 45.24% | 0.4606 | 3.39% | 0.9866 |

### 주요 해석

- **XGBoost 최우수**: MAPE 7.45% / R² 0.9979
- **A10B**: XGBoost MAPE 2.59%, R² 0.9913 → 실무 투입 수준
- **A10A**: XGBoost MAPE 12.31%, R² 0.9791 → 우수하나 개선 여지
- **Linear Reg A10A 실패**: MAPE 211%, R² -5.42 → 평균값보다 나쁨 (비선형 패턴 포착 불가)
- **LightGBM 부진**: A10A R² 0.46 → early stopping 과도 작동, 하이퍼파라미터 튜닝 필요

### R² 해석 기준

| R² 범위 | 평가 |
|---|---|
| > 0.95 | 매우 우수 (실무 활용 가능) |
| 0.9 ~ 0.95 | 우수 |
| 0.7 ~ 0.9 | 양호 |
| < 0.7 | 개선 필요 |
| 음수 | 평균값보다 나쁨 (모델 실패) |

---

## Step 4: 평가 완료 내용

### 분석 항목

1. **약물별 성능 비교**: A10A/A10B 각각의 모델별 MAPE 시각화
2. **구별 예측 성능**: XGBoost 기준 25개 구 각각의 MAPE 분석
3. **월별 오차 추이**: 테스트 6개월간 월별 MAPE 변화
4. **전체 합계 시계열**: 서울 전체 기준 예측 vs 실제 비교
5. **인구 특성 vs 오차**: 고령인구비율·총인구·1인당처방량과 예측 오차 관계
6. **R² 분석**: 약물별·모델별 R² 추가 (셀 7-1)

### 사용한 평가 지표 3종 + R²

| 지표 | 수식 | 특징 |
|---|---|---|
| MAE | 평균(절대오차) | 원래 단위, 직관적 |
| RMSE | √평균(오차²) | 큰 오차에 민감 |
| MAPE | 평균(절대오차/실제값)×100 | 스케일 무관, 비율로 비교 |
| R² | 1 - SSres/SStot | 분산 설명력, 모델 적합도 |

### 생성된 시각화 (`output/figures/`)

| 파일 | 내용 |
|---|---|
| `09_model_comparison.png` | 3개 모델 MAE/RMSE/MAPE 비교 |
| `10_feature_importance.png` | XGBoost/LightGBM Feature Importance |
| `11_pred_vs_actual.png` | 4개 구 예측 vs 실제 시계열 |
| `12_residual_plot.png` | 3개 모델 잔차 분포 |
| `13_drug_mape_comparison.png` | 약물별 모델 MAPE 비교 |
| `14_gu_mape_xgb.png` | 구별 XGBoost MAPE |
| `15_monthly_mape_trend.png` | 월별 MAPE 추이 |
| `16_total_pred_vs_actual.png` | 서울 전체 합계 예측 vs 실제 |
| `17_population_vs_error.png` | 인구 특성 vs 예측 오차 산점도 |
| `18_r2_comparison.png` | 약물별·모델별 R² 비교 |

### 생성된 데이터 파일 (`output/data/`)

| 파일 | 내용 |
|---|---|
| `test_predictions.csv` | 테스트셋 예측 결과 (3개 모델) |
| `model_performance.csv` | 전체 성능 지표 요약 |
| `gu_performance.csv` | 구별 성능 지표 |
| `drug_performance.csv` | 약물별 성능 지표 |
| `final_summary.csv` | 약물×모델 최종 요약 |

---

---

## Step 5: 하이퍼파라미터 튜닝 완료 내용

### 방법론

| 항목 | 내용 |
|---|---|
| 검증 방법 | Walk-forward CV (확장 윈도우, 3-fold, 각 3개월 검증) |
| Fold 구성 | Fold1: Train~2024-07 / Val 2024-08~10, Fold2: ~2024-10 / Val 2024-11~2025-01, Fold3: ~2025-01 / Val 2025-02~04 |
| 탐색 도구 | Optuna TPE Sampler (각 모델 100 trials) |
| 탐색 파라미터 | XGBoost 8개, LightGBM 9개 (`num_leaves` 포함) |

### 최적 하이퍼파라미터

**XGBoost 최적값**

| 파라미터 | 값 |
|---|---|
| n_estimators | 476 |
| learning_rate | 0.0320 |
| max_depth | 8 |
| subsample | 0.9815 |
| colsample_bytree | 0.6575 |
| reg_alpha | 1.9566 |
| reg_lambda | 1.4228 |
| min_child_weight | 1 |

**LightGBM 최적값**

| 파라미터 | 값 |
|---|---|
| n_estimators | 734 |
| learning_rate | 0.1667 |
| max_depth | 8 |
| num_leaves | 53 |
| subsample | 0.7111 |
| colsample_bytree | 0.6904 |
| reg_alpha | 1.1700 |
| reg_lambda | 0.9312 |
| min_child_samples | 5 |

### 최종 성능 비교 (베이스라인 → 튜닝)

| 모델 | 구분 | A10A MAPE | A10B MAPE | 전체 MAPE |
|---|---|---|---|---|
| XGBoost | 기본 | 38.45% | 8.41% | 23.43% |
| **XGBoost** | **튜닝** | **16.12%** | **8.04%** | **12.08%** |
| LightGBM | 기본 | 66.16% | 8.79% | 37.47% |
| LightGBM | 튜닝 | 42.55% | 8.38% | 25.47% |

> **최우수 모델: XGBoost (튜닝) — 전체 MAPE 12.08%**

### 주요 해석

- **XGBoost 튜닝 효과 극적**: 전체 23.43% → 12.08%, A10A 38.45% → 16.12%
- **A10B 한계 도달**: 8.41% → 8.04% — 현재 feature로는 추가 개선 어려움
- **LightGBM A10A 구조적 취약**: 튜닝 후에도 42.55%로 XGBoost 대비 2.6배 차이

### 생성된 파일

| 파일 | 내용 |
|---|---|
| `output/data/best_params.json` | Optuna 최적 파라미터 (XGBoost / LightGBM) |
| `output/data/tuning_summary.csv` | 튜닝 전후 성능 요약 |
| `output/figures/19_tuning_comparison.png` | MAPE / R² 기본 vs 튜닝 비교 |
| `output/figures/20_optuna_history.png` | Optuna trial 최솟값 수렴 그래프 |
| `output/figures/21_tuned_feature_importance.png` | 튜닝 후 Feature Importance |

---

## 한계 및 개선 방향

| 항목 | 내용 |
|---|---|
| 데이터 범위 | 서울시 25개 구만 분석 → 전국 확장 필요 |
| 인구 데이터 | 2025-10 단일 스냅샷 → 월별 인구 변화 미반영 |
| 테스트 기간 | 6개월로 짧아 계절성 검증 불충분 |
| A10A 성능 | MAPE 12.31% → 소규모 구에서 오차 큼 |
| LightGBM | XGBoost 대비 성능 낮음 → 하이퍼파라미터 최적화 필요 |

**개선 방향**
1. 전국 시군구 데이터 확보 시 일반화 성능 향상 기대
2. 연도별 인구 데이터 수집으로 feature 품질 향상
3. A10A 소규모/대규모 구 분리 모델 검토 (용산·종로 등 소규모 구 오차 집중)
4. ~~Optuna 등으로 XGBoost/LightGBM 하이퍼파라미터 최적화~~ → **Step 5에서 완료** (XGBoost 12.08%)
5. 외부 변수 추가 검토 (기온, 의료기관 수 등)
6. A10B feature 개선 검토 (rolling_mean_6, 계절성 강화 등) — 현재 8% 벽

---

## 재시작 시 체크리스트

- [x] `01_eda.ipynb` — EDA 완료 (시각화 7개)
- [x] `02_preprocessing.ipynb` — 전처리 완료 (`merged_master.csv`) / 인구 필터 버그 수정
- [x] `03_modeling.ipynb` — 모델링 완료 / 데이터 누수 · early stopping 버그 수정
- [x] `04_evaluation.ipynb` — 평가 완료 (R² 포함, 시각화 10개)
- [x] `05_tuning.ipynb` — Optuna 튜닝 완료 (XGBoost 최우수, MAPE 12.08%)
