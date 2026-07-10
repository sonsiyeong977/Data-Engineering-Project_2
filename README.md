# 🧴 올리브영 스킨케어 가성비 추천 시스템
> 올리브영 스킨케어 제품의 리뷰·가격 데이터를 크롤링하여 NoSQL DB에 저장하고,
> NLP 기반 감성 분석과 가성비 점수(Value Score)를 산출해 사용자 맞춤형 제품을 추천하는 데이터 엔지니어링 프로젝트

---

## 👥 팀 구성 및 역할

| 이름 | 역할 |
|------|------|
| 서연 | 웹 크롤링 (Playwright + Crawlee) |
| **손시영** | **DB 설계 · 전처리 · EDA · 회귀분석 · 가중치 도출** |
| 이주아 | NLP 감성 분석 · Q/E/S/P Score 산출 |
| 최현서 | 프론트엔드 구현 |

---

## 🗂️ 프로젝트 구조

```
Data-Engineering-Project_2/
├── EDA_oyskincare.ipynb             # DB 설계 · 전처리 · EDA (손시영)
├── README.md
├── nlp/
│   ├── nlp_README.md                # NLP 파트 설명
│   ├── product_qesv_scores_revised.csv
│   ├── product_qesv_scores_promo_revised.csv
│   └── final_p_score_with_id_for_db.csv
└── regression/
    ├── regression_README.md         # 회귀분석 파트 설명
    ├── DE_2__회귀분석.ipynb          # 일반 상품 회귀분석 (손시영)
    └── promo_회귀분석.ipynb          # 프로모션 상품 회귀분석 (손시영)
```

---

## 🔄 전체 파이프라인

```
웹 크롤링 (Playwright + Crawlee)
        ↓
MongoDB 저장 · 전처리
        ↓
EDA (탐색적 데이터 분석)
        ↓
NLP 감성 분석 · Q/E/S/P Score 산출
        ↓
회귀분석 · 가중치 도출
        ↓
사용자 맞춤 추천 · 프론트엔드
```

---

## ✋ 손시영 기여 파트

### 1️⃣ MongoDB 스키마 설계

비정형 리뷰 텍스트와 정형 상품 데이터를 함께 관리하기 위해 MongoDB를 선택, 두 컬렉션으로 분리 설계.

**컬렉션 1 — `skincare` (상품 정보)**
```json
{
  "product_id": "string",
  "product_name": "string",
  "category": "string",
  "original_price": 15000,
  "sale_price": 12000,
  "discount_rate": 20,
  "final_price": 12000,
  "volume": "150ml",
  "price_per_ml": 80.0,
  "average_rating": 4.8,
  "review_count_total": 1200,
  "badges": ["베스트", "올영픽"]
}
```

**컬렉션 2 — `skincare_review` (리뷰 정보)**
```json
{
  "review_id": "string",
  "product_id": "string",
  "review_score": 5,
  "review_text": "촉촉하고 흡수가 빨라요",
  "date": "2026-03-01",
  "skin_type": "건성",
  "skin_tone": "밝은편",
  "skin_trouble": "건조함"
}
```

---

### 2️⃣ 데이터 전처리

| 처리 항목 | 내용 |
|-----------|------|
| 가격 정규화 | 문자열 → 숫자 변환, `final_price` 파생 변수 생성 |
| ml당 가격 계산 | 카테고리 내 공정 비교를 위한 단위 가격 산출 |
| 결측치 처리 | `original_price` 결측(15%) 제거, `skin_type` 결측(61%) 텍스트 키워드로 보완 |
| 중복 제거 | 동일 상품·동일 리뷰 중복 제거 |
| 리뷰 텍스트 정제 | 특수문자·이모지 제거, 공백 정리 |

---

### 3️⃣ EDA 핵심 인사이트

| 인사이트 | 내용 | 모델링 적용 |
|----------|------|------------|
| 평점 변별력 없음 | 전체 평점이 4점대 후반 집중 | 감성 점수로 세분화 |
| 인기는 중저가 집중 | 1~3만원대에 리뷰 집중 | 베이지안 보정 적용 |
| 가격-만족도 무관 | 상관계수 ≈ 0 | 가격 가중치 독립 설정 |
| 할인율-만족도 무관 | 상관계수 ≈ 0 | 할인율 핵심 피처 제외 |
| 불만족 = 긴 리뷰 | 1점 리뷰 평균 122자 | 부정 리뷰 가중 분석 |

---

### 4️⃣ 회귀분석 · 가중치 도출

**회귀 모델**

$$
y = b_0 + b_1Q + b_2E + b_3S + 1.5 \cdot b_4P
$$

**일반 상품 회귀 결과 (378개)**

| 변수 | 계수 (b) |
|------|---------|
| 품질 (Q) | 7,765.83 |
| 가치 (E) | 885.02 |
| 감성 (S) | -18,086.09 |
| 가성비 (P) | **8,218.74** |

**최종 가중치 (보정 후 정규화)**

| 요소 | 가중치 |
|------|--------|
| 가성비 (P) | **47.32%** |
| 품질 (Q) | **44.71%** |
| 사회적 가치 (E) | 5.10% |
| 감성 (S) | 2.87% |

> 가성비 + 품질 합산 **약 91%** → 제품 인기 결정 핵심 요소 확인

---

## 📦 데이터 현황

| 컬렉션 | 문서 수 | 주요 컬럼 |
|--------|---------|----------|
| `skincare` | 100개 | 제품명, 가격, 할인율, 평점, 리뷰수 등 (21개) |
| `skincare_review` | 60,165개 | 리뷰 텍스트, 점수, 피부타입 등 (11개) |

> MongoDB Atlas (`oliveyoung_db`) 에 저장

---

## 💯 Value Score 수식

$$
User\_Value\_Score = (b_1 \cdot w_Q)Q + (b_2 \cdot w_E)E + (b_3 \cdot w_S)S + (b_4 \cdot w_P \cdot 1.5)P
$$

$$
w_i = \frac{s_i}{\sum_{j \in \{Q,E,S,P\}} s_j}
$$

---

## ⚙️ 실행 환경

- Google Colab
- Python 3.x
- MongoDB Atlas

```bash
pip install pymongo pandas matplotlib seaborn scikit-learn statsmodels
```
