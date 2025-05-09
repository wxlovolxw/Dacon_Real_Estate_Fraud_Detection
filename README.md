# Dacon Real Estate Fraud Detection

Dacon 부동산 허위매물 예측 대회에 참여하여 진행한 데이터 분석 프로젝트입니다.  
EDA, 피처 엔지니어링, 모델링, 해석까지 실무 흐름에 맞춰 전 과정을 체계적으로 수행했습니다.

---

## 폴더 구성
---
- `notebooks/`
  - `main_analysis.ipynb`: 전체 분석 흐름이 담긴 메인 노트북
- `images/`
  - SHAP summary, confusion matrix 등 시각화 결과 저장 폴더
- `docs/`
  - 분석 과정 정리 파일 저장 폴더

---

## 분석 개요
---
- 목표: 부동산 매물 데이터에서 **허위매물 여부 예측**
- 데이터셋: Dacon 제공 부동산 매물 데이터
- 타겟 변수: **`허위매물여부`**
- 성능 지표: F1 Score, Precision, Recall
- 최종 모델: LightGBM (Threshold 0.45 적용)

---

## 데이터 전처리
---
### 결측치 처리
- 총 6개 피처에서 결측 발생: 전용면적, 해당층, 총층, 방수, 욕실수, 총주차대수
- `총층`, `방수`, `욕실수`: 결측률 낮아 최빈값 대체
- `해당층`, `총주차대수`: 조건부 결측 판단 → RandomForestRegressor로 보간
- `전용면적`: 범주 기반 중앙값 그룹 보간
- 결측 여부가 타겟과 관련 있는 경우 → 결측 여부 파생 변수 생성

### 이상치 처리 및 수치 변환
- 군집 기반 이상치 탐지 실패 → 분포 기반 IQR 방식으로 이상치 제거
- 왜도 큰 수치형 변수 변환:
  - `보증금(억)`, `관리비`: 로그 변환
  - `총층`, `해당층`, `총주차대수`: Box-Cox 변환
- KDE 플롯과 왜도 지표를 통해 변환 효과 확인

---

## 피처 엔지니어링
---
- 시간 파생: `게재일` → `년월` 생성 및 Label Encoding
- 중개사무소 파생: 상위 5개(`top5`)와 나머지 그룹화, 또는 빈도 기반 그룹화
- 방향 처리: `방향_sin`, `방향_cos` 변환
- 타겟과 무관하거나 독립적인 피처 제거

---

## 파생 변수 조합 실험
---
방향, 중개사무소, 시간, 전용면적, 주차대수 관련 파생 피처 조합 실험을 통해 최적 조합을 선정하였습니다.

- 방향 관련:
  - 방향(원본) vs 방향 선호도 그룹 vs 방향_sin+cos 조합 비교
  - → **방향 선호도 그룹화(feature)** 최종 선택

- 중개사무소 관련:
  - 중개사무소_top5 vs 중개사무소_빈도기반 비교
  - → **중개사무소_빈도기반** 최종 선택

- 시간 관련:
  - 년월 vs 게재일_정책구간 비교
  - → **년월** 사용 최종 선택

- 전용면적 관련:
  - 전용면적_raw vs 전용면적_filled 등 비교
  - → **전용면적_raw + 전용면적_결측여부** 조합(area_B) 최종 선택

- 주차대수 관련:
  - 총주차대수_raw vs 총주차대수_filled_boxcox 등 비교
  - → **총주차대수_filled_boxcox 단일 feature** 선택

### 최종 Feature Engineering 조합
- 방향선호도 + 중개사무소_빈도기반 + 년월 조합
- 전용면적_raw + 전용면적_결측여부
- 총주차대수_filled_boxcox

---

## 클래스 불균형 처리
---
- 허위매물 소수 클래스 비율 매우 낮음
- 실험한 기법:
  - Borderline-SMOTE
  - SMOTETomek
  - Isolation Forest 기반 이상치 제거 (최종 선택)
- Hold-out Validation 및 Cross-Validation 결과,  
  **Isolation Forest 적용 시 F1 Score가 가장 우수하여 최종 선택**

---

## 모델링 및 비교
---
- 실험한 모델:
  - LightGBM (최종 선택)
  - XGBoost
  - CatBoost
  - RandomForest
  - LogisticRegression
- Optuna를 활용해 5-Fold Cross-Validation 기반 하이퍼파라미터 튜닝 수행
- 최종 성능:
  - LightGBM + Isolation Forest + Threshold 0.45 적용
  - F1 Score: 0.8671

---

## 모델 해석
---
- SHAP 기반 피처 중요도 분석 수행:
  - '년월', '중개사무소_빈도기반', '관리비_log', '보증금(억)_log' 등의 변수가 높은 영향력
- 피처 제거 실험은 진행했으나,  
  최종 모델에서는 **결측 여부 파생 피처를 포함하여 학습**
- 추가로, SHAP Summary Scatter Plot을 통해  
  피처별 값 변화가 예측 결과에 미치는 구체적 기여 방향까지 함께 분석하였습니다.

### SHAP Feature Importance (Top 10)
![SHAP Feature Importance](images/shap_top10_features_tuned_lgbm.png)

> 주요 피처들의 평균적인 중요도(Mean Absolute SHAP Value)를 기준으로 정렬한 결과

### SHAP Summary Plot
![SHAP Summary Plot](images/final_model_shap_summary_scatter.png)

> 주요 피처 값 변화가 개별 예측 결과에 미치는 영향을 시각화

---

## Threshold 튜닝
---
- 다양한 threshold(0.1 ~ 0.9) 구간에서 F1, Precision, Recall 변화 관찰
- 최종 선택:
  - Threshold = 0.45 (Precision-Recall 균형 최적화)

### Threshold별 Score 변화
![Threshold Fine Tuning](images/threshold_fine_tuning.png)

> Threshold 값 변화에 따른 F1, Precision, Recall 변화를 시각화하여 최적 지점을 탐색

### Precision-Recall Curve
![Precision Recall Curve](images/precision_recall_curve_045.png)

> Threshold=0.45 설정이 Precision-Recall 트레이드오프 상 합리적인 선택임을 확인

---

## 최종 모델 성능
---
- F1 Score: 0.8519
- Precision: 0.8846
- Recall: 0.8214
- Accuracy: 0.9680
- ROC-AUC: 0.9839

### Confusion Matrix
![Confusion Matrix](images/final_model_confusion_matrix.png)

> 정상 매물과 허위 매물에 대한 분류 성능을 시각화

---

## 최종 요약
---
- EDA → 결측 및 이상치 처리 → 피처 엔지니어링 → 모델링 → 해석 및 튜닝까지  
  실무 흐름에 기반하여 체계적이고 해석 가능한 분석 사이클을 완성한 프로젝트입니다.
- 모델 성능뿐만 아니라, 데이터 품질 이슈(결측, 이상치) 및 클래스 불균형 대응까지 통합 고려한 점이 특징입니다.