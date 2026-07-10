# 🧴 올리브영 스킨케어 가성비 추천 시스템
> 올리브영 스킨케어 제품의 리뷰·가격 데이터를 크롤링하여 NoSQL DB에 저장하고,
> NLP 기반 감성 분석과 가성비 점수(Value Score)를 산출해 사용자 맞춤형 제품을 추천하는 데이터 엔지니어링 프로젝트

---

## 👥 팀 구성 및 역할

| 역할 |
|------|------|
| 웹 크롤링 (Playwright + Crawlee) |
| **DB 설계 및 저장 · EDA · 시각화** |
| NLP 감성 분석 · Value Score 설계 |
| 프론트엔드 구현 |

---

## 🗂️ 프로젝트 구조

```
oliveyoung-skincare/
├── crawling/
│   └── crawler.js                  # 올리브영 크롤러 (Playwright + Crawlee)
├── database/
│   └── EDA_oyskincare.ipynb        # DB 저장 · 전처리 · EDA ← 손시영 담당
├── nlp/
│   └── nlp_pipeline.ipynb          # 감성 분석 · Value Score 계산
├── frontend/
│   └── ...                         # 추천 UI
└── README.md
```

---

## ✋ 내 기여 파트 (손시영)

### 1️⃣ MongoDB 스키마 설계

비정형 리뷰 텍스트와 정형 상품 데이터를 함께 관리하기 위해 MongoDB를 선택하고, 두 컬렉션으로 분리 설계했다.

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

> 상품과 리뷰를 분리함으로써 NLP 처리 효율성 확보 및 대용량 확장 가능

---

### 2️⃣ 데이터 전처리

| 처리 항목 | 내용 |
|-----------|------|
| 가격 정규화 | 문자열 → 숫자 변환, `final_price` 파생 변수 생성 |
| ml당 가격 계산 | 카테고리 내 공정 비교를 위한 단위 가격 산출 |
| 결측치 처리 | `original_price` 결측(15%) 제거, `skin_type` 결측(61%) 텍스트 키워드로 보완 |
| 중복 제거 | 동일 상품·동일 리뷰 중복 제거 |
| 이상치 제거 | 비정상 가격 필터링 |
| 리뷰 텍스트 정제 | 특수문자·이모지 제거, 공백 정리 |

---

### 3️⃣ EDA (탐색적 데이터 분석)

**데이터 현황**

| 컬렉션 | 문서 수 | 주요 컬럼 수 |
|--------|---------|-------------|
| `skincare` | 100개 | 21개 |
| `skincare_review` | 60,165개 | 11개 |

**기술 통계**

| 지표 | 값 |
|------|----|
| 평균 판매가 | 24,876원 |
| 평균 할인율 | 19.5% |
| 평균 리뷰수 | 2,940개 (std: 12,107) |
| 평균 평점 | 4.8점 |
| 평균 리뷰 길이 | 112자 |

---

### 4️⃣ 핵심 인사이트

#### 📊 인사이트 1 — 평점만으로는 변별이 안 된다
- 전체 제품 평점이 **4점대 후반에 극단적으로 집중**
- 평점 기반 추천 시스템의 한계 → **리뷰 텍스트 감성 분석 필요성** 근거 제시

#### 💰 인사이트 2 — 인기는 중저가에 집중된다
- **1~3만원대**에 리뷰가 집중, 고가 제품은 리뷰수 적음
- 콜드스타트 문제 가능성 → **베이지안 보정으로 신뢰도 조정** 제안

#### 📉 인사이트 3 — 가격과 만족도는 무관하다
- 가격-평점 상관계수 ≈ **0**
- 고가 제품이 반드시 높은 만족도를 보이지 않음
- → **중저가 제품도 추천 대상으로 유효**하다는 근거

#### 🏷️ 인사이트 4 — 할인율도 만족도와 무관하다
- 할인율-평점 상관계수 ≈ **0**
- 프로모션이 실제 제품 품질과 무관
- → 할인율을 추천 핵심 피처로 사용하는 것은 신중해야 함

#### 📝 인사이트 5 — 불만족할수록 더 길게 쓴다
- 1점 리뷰 평균 길이: **122자** vs 5점 리뷰: **113자**
- 부정 리뷰 분석이 감성 점수 계산에 특히 유효함

---

### 5️⃣ EDA → 모델링 연결 포인트

| EDA 결과 | 모델링 적용 |
|----------|------------|
| 평점 변별력 없음 | 감성 점수를 Quality Score에 포함 |
| 고가 제품 리뷰 부족 | 베이지안 보정으로 판매량 proxy 산출 |
| 가격-평점 무관계 | 가격 가중치 독립적으로 설정 |
| skin_type 결측 61% | 리뷰 텍스트 키워드로 보완 후 조건부 리랭킹 |

---

## 🔄 전체 파이프라인

```
웹 크롤링 (Playwright)
        ↓
MongoDB 저장 (skincare / skincare_review)
        ↓
전처리 & EDA 
        ↓
NLP 감성 분석 · Value Score 계산
        ↓
사용자 맞춤 추천 · 프론트엔드
```

---

## 💯 Value Score 수식

$$
Value Score = \frac{Composite Score \times \log(ReviewCount + 1)}{NormalizedUnitPrice}
$$

$$
Composite Score = 0.4 \times Sentiment + 0.2 \times Repurchase + 0.2 \times Texture + 0.2 \times SkinSafety
$$

---

## ⚙️ 실행 환경

- Google Colab
- Python 3.x
- MongoDB Atlas

```bash
pip install pymongo pandas matplotlib seaborn
```

> MongoDB 연결 시 본인 Atlas 연결 문자열로 교체 필요
> ```python
> client = MongoClient("your_mongodb_connection_string_here")
> ```
