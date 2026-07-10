# 🧠 NLP 감성 분석 · Value Score 산출
> 올리브영 스킨케어 리뷰 텍스트를 기반으로 Q/E/S 감성 점수를 산출하고,
> 최종 가성비 점수(P Score)를 계산하여 추천 시스템의 근거를 마련한다.

---

## 👤 담당

| 이름 | 역할 |
|------|------|
| 이주아 | NLP 파이프라인 설계 · 감성 분석 · Value Score 산출 |

---

## 📁 파일 구성

| 파일 | 설명 | 상품 수 |
|------|------|---------|
| `product_qesv_scores_revised.csv` | 전체 상품 Q/E/S 점수 | 379개 |
| `product_qesv_scores_promo_revised.csv` | 프로모션 상품 Q/E/S 점수 (avg_review_score 포함) | 164개 |
| `final_p_score_with_id_for_db.csv` | DB 저장용 최종 P Score | 437개 |

---

## 🔬 분석 파이프라인

```
리뷰 텍스트 (skincare_review 컬렉션)
        ↓
텍스트 전처리
(특수문자 제거 · 형태소 분석 · 불용어 제거 · 유의어 통일)
        ↓
키워드 기반 피처 추출
(Q / E / S 축 키워드 매칭)
        ↓
축별 감성 점수 산출 (0~1)
        ↓
최종 P Score 계산
        ↓
DB 저장 (final_p_score_with_id_for_db.csv)
```

---

## 📐 점수 체계

### Q / E / S 3축 구조

| 축 | 의미 | 핵심 키워드 예시 |
|----|------|----------------|
| **Q (Quality)** | 품질 만족도 | 재구매, 인생템, 만족, 추천 |
| **E (Efficacy)** | 효능·효과 | 촉촉, 흡수 빠름, 보습, 진정 |
| **S (Safety)** | 안전성·순함 | 자극 없음, 순함, 트러블 없음, 민감성 ok |

### 컬럼 설명

| 컬럼 | 설명 |
|------|------|
| `product_id` | 상품 고유 ID (master_id) |
| `review_n` | 해당 상품의 리뷰 개수 |
| `product_name_clean` | 올리브영 상품명 |
| `category` | 스킨케어 세부 카테고리 |
| `Q_mass_total` | Q축 키워드 총 언급량 |
| `Q_pos_product` | Q축 긍정 감성 점수 (0~1) |
| `E_mass_total` | E축 키워드 총 언급량 |
| `E_pos_product` | E축 긍정 감성 점수 (0~1) |
| `S_mass_total` | S축 키워드 총 언급량 |
| `S_pos_product` | S축 긍정 감성 점수 (0~1) |
| `avg_review_score` | 평균 별점 (promo 파일만) |

---

## 📊 데이터 현황

### 전체 상품 Q/E/S 점수 분포 (`product_qesv_scores_revised`)

| 지표 | Q Score | E Score | S Score |
|------|---------|---------|---------|
| 상품 수 | 379개 | 379개 | 379개 |
| 평균 | 0.773 | 0.769 | 0.824 |
| 중앙값 | 0.781 | 0.774 | 0.829 |
| 최솟값 | 0.325 | 0.325 | 0.325 |
| 최댓값 | 0.983 | 0.983 | 0.983 |

> S(안전성) 점수가 Q/E 대비 전반적으로 높게 분포
> → 올리브영 스킨케어 제품군의 안전성에 대한 소비자 신뢰가 높음

### 카테고리별 상품 분포

| 카테고리 | 상품 수 |
|----------|---------|
| 에센스/세럼/앰플 | 185개 |
| 크림 | 92개 |
| 스킨/토너 | 60개 |
| 로션 | 24개 |
| 미스트/오일 | 18개 |

### 최종 P Score 분포 (`final_p_score_with_id_for_db`)

| 지표 | 값 |
|------|----|
| 상품 수 | 437개 |
| 평균 P Score | 0.904 |
| 중앙값 | 0.911 |
| 최솟값 | 0.394 |
| 최댓값 | 0.999 |

---

## 💯 P Score 산출 수식

$$
Composite Score = 0.4 \times Sentiment + 0.2 \times Repurchase + 0.2 \times Texture + 0.2 \times SkinSafety
$$

$$
Value Score = \frac{Composite Score \times \log(ReviewCount + 1)}{NormalizedUnitPrice}
$$

$$
NormalizedUnitPrice = \frac{P_i - P_{min}}{P_{max} - P_{min} + \varepsilon}
$$

> ε = 0.01 (분모 0 방지)
> NormalizedUnitPrice는 세부 카테고리 내 단위 가격(ml당) 기준으로 정규화

---

## 🔗 전체 파이프라인에서의 위치

```
웹 크롤링 ✅
        ↓
MongoDB 저장 · 전처리 · EDA ✅
        ↓
NLP 감성 분석 · P Score 산출 ✅ ← 이 파트
        ↓
사용자 맞춤 추천 · 프론트엔드
```

---

## 🔗 EDA → NLP 연결 포인트

| EDA 인사이트 | NLP 적용 |
|-------------|---------|
| 평점 변별력 없음 (4점대 집중) | Q/E/S 감성 점수로 세분화된 품질 평가 |
| skin_type 결측 61% | 리뷰 텍스트 키워드로 피부타입 보완 |
| 불만족할수록 리뷰 길이 증가 | 부정 키워드 가중 분석에 반영 |
| 고가 제품 리뷰 부족 | review_n 기반 베이지안 보정 적용 |
