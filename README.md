# TEAM ISNLP2 · AI-Driven Legal Demand Mining

### 이슈 군집화와 법령-이슈 간 유사도를 활용한 법률 수요 추출 및 제안 모델

> 뉴스·SNS 담론을 자동으로 **이슈 군집화**하고 관련 **법령 Top-k** 를 매핑한 뒤, 근거 점수(BM25/임베딩/NLI Entailment)를 융합해 **우선 순위화 된 입법 수요(demand_score)**를 산출한다. 최종적으로 산출된 입법 수요 지표와 Top1 법령을 함께 제공하여, 데이터 기반의 입법/정책 의사결정을 돕는다.

---

## 목차

- [개요]
- [핵심 기능]
- [아키텍처]
- [스코어 정의]
- [입출력 형식]
- [평가 지표]

---

## 개요

기사·게시물·커뮤니티 글 등 **비정형 대규모 데이터**는 실제 시민의 관심과 요구를 반영합니다. 그러나

1. 핵심 이슈의 **자동 추출/요약이 어렵고**,
2. 이슈↔법령 **연결 근거가 부족**하며,
3. **시급성/중요도**를 판단할 **객관 지표가 부재**합니다.

본 프로젝트는 다음을 통해 문제를 해결합니다.

- **이슈 군집화**로 담론 구조화
- **법령-이슈 유사도**(BM25/임베딩/NLI)로 규범적 대응 지점 특정
- **지표화 및 우선순위화**로 정책·입법 실무 활용성 제고

---

## 핵심 기능

- **플랫폼별 전처리**: 뉴스/트위터/인스타/커뮤니티 노이즈 제거, 해시태그·이모지 처리, 중복 제거
- **이슈 탐지**: sBERT 임베딩 → UMAP/HDBSCAN(필요시 K-means fallback), 대형·이질 군집 자동 재분할
- **키워드 라벨링**: c-TF-IDF + NPMI/KL + **MMR** 로 군집 대표 키워드 Top-k
- **법령 매핑**: 사법공개포털 기반 법령 **카탈로그** 구축 → BM25(1차) → 임베딩(2차) → 의미적 추론 기반 **엔테일먼트** 검증
- **스코어 융합**: 빈도/증가율/참여도 × (BM25/sBERT/NLI) → **demand_score(0–1, 매핑된 법령 별 수요 근거 지표)**
- **산출물**: 이슈-법령 랭킹 테이블, 근거 스니펫, 요약 리포트/대시보드

---

## 아키텍처

```python
[Raw Data] ──► [Preprocessing] ──► [Embedding(sBERT)]
                                  │
                                  └─► [UMAP] ─► [HDBSCAN] ─► [Keyword Labeling(c-TF-IDF + NPMI/KL + MMR)]
                                                                   │
                                               [Law Catalog(SQLite)] ──► [BM25] ─► [Embedding] ─► [NLI]
                                                                                                  │
                                                                                      [Scoring & Ranking]
                                                                                                  │
                                                                                            [CSV / Dash]

```

---

## 스코어 정의

```
salience_score = wF*F + wG*G + wE*E + wK*K + wM*M
link_score     = α*bm25_norm + β*sbert_norm + γ*nli_entail_norm + δ*conf_norm

# ε: 수치 안정화 상수, normalize(): 0–1 정규화
demand_score   = normalize( λ*salience_score + (1-λ)*link_score )
geo_blend      = sqrt( (salience_score+ε) * (link_score+ε) )
final_score    = normalize( θ*demand_score + (1-θ)*geo_blend )
```

- **F**: 클러스터 규모, **G**: 시계열 증가율, **E**: 참여도(댓글·공유 등), **K**: 키워드 응집도, **M**: 군집 분리 마진
- 가중치(α,β,γ,δ,λ,θ)는 실험 로그로 튜닝

---

## 입출력 형식

**입력 (전처리 완료 .xlsx)**

`title_norm, content_norm, date, likes, comments, shares, ...`

**중간 산출**

- `_table.xlsx` : 클러스터 요약(키워드, 대표문서, 지표)
- `_outputs.json` : 상위 이슈 요약(Top-k)
- `law_catalog.sqlite` : 법령 ID/명/본문/약칭 인덱스

**최종 산출**

- `issue_with_rank5.csv` : 이슈-법령 Top-k + demand_score
- `_cat_metrics.csv` : 전역 메트릭

---

## 평가 지표

- **클러스터 품질**: silhouette, DBCV, intra/inter similarity, separation margin
- **키워드 품질**: NPMI, diversity, c-TF-IDF 상위항목 일관성
- **매핑 품질**: BM25@k, Embedding@k, **NLI entailment** 정/재현
- **최종 성과**: 상위 N 이슈의 전문가 판정 일치율, 리포트 유용성
