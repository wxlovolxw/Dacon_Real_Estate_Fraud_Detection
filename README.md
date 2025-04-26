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
- 목표: 부동산 매물 데이터에서 허위매물 여부 예측
- 데이터셋: Dacon 제공 부동산 매물 데이터
- 타겟 변수: `허위매물여부`
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

##  파생 변수 조합 실험
---
- 방향(`방향` vs `방향_sin+방향_cos`)
- 중개사무소(`top5` vs `빈도기반`)
- 게재일(`년월` vs `게재일_정책구간`)

→ 최종 선택:  
방향=원본 방향 / 중개사무소=빈도 기반 그룹 / 시간=년월

---

## 클래스 불균형 처리
---
- 허위매물 소수 클래스 비율 매우 낮음
- 실험한 기법:
  - Borderline-SMOTE
  - SMOTETomek
  - Isolation Forest 기반 이상치 제거 (최종 선택)
- Hold-out Validation 및 Cross-Validation 결과,  
  Borderline-SMOTE 적용 시 F1 Score가 가장 안정적으로 향상됨

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

## 모델 해석 및 피처 제거
---
- SHAP 기반 피처 중요도 분석 수행
- Recursive Feature Elimination (SHAP-RFE) 적용
  - 중요도 낮은 파생 피처 (`해당층_결측여부`, `총주차대수_결측여부`, `전용면적_결측여부`) 제거
  - 제거 후에도 모델 성능 유지 또는 소폭 향상

---

## Threshold 튜닝
---
- 다양한 threshold(0.1 ~ 0.9) 구간에서 F1, Precision, Recall 변화 관찰
- 최종 선택:
  - Threshold = 0.45 (Precision-Recall 균형 최적화)


---

## 최종 요약
---
- EDA → 결측 및 이상치 처리 → 피처 엔지니어링 → 모델링 → 해석 및 튜닝까지  
  실무 흐름에 기반하여 체계적이고 해석 가능한 분석 사이클을 완성한 프로젝트입니다.
- 모델 성능뿐만 아니라, 데이터 품질 이슈(결측, 이상치) 및 클래스 불균형 대응까지 통합 고려한 점이 특징입니다.