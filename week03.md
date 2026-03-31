# 3회차 - Pandas 심화: 통계 요약 / 결측치 / 파생변수 / Reshaping / 합치기 / GroupBy / 시계열

## 강의 범위
- 섹션 5: Summarize Data (기본 통계, apply/lambda)
- 섹션 6: Handling Missing Data (결측치 처리)
- 섹션 7: Make New Columns (파생변수, binning)
- 섹션 8: Reshaping Data (정렬, melt/pivot, concat)
- 섹션 9: Combine Data Sets (merge)
- 섹션 10: Group Data (groupby)
- 섹션 11: 시계열 데이터 분석 (rolling, expanding)

## 실습 데이터
Spotify 트랙 데이터 (spotify_2015_2025_85k.csv)
- 총 85,000행, 19열
- 주요 컬럼: track_name, artist_name, genre, popularity, danceability, energy, loudness, tempo, stream_count, country, label, release_date 등
- 결측치: track_name 21개, album_name 46개

---

## 1. Summarize Data — 기본 통계 요약

### 개념
데이터를 하나의 숫자로 압축하는 것. 분포를 파악하고 이상치를 발견하는 출발점.

### 코드

```python
# 열별 고유값 수
df['genre'].nunique()       # 12  → 장르 종류가 12개
df['country'].nunique()     # 10  → 데이터에 포함된 국가 수

# 빈도 집계
df['genre'].value_counts()
# genre
# Metal       7200
# Jazz        7177
# Hip-Hop     7160
# Classical   7158
# Rock        7113

# 기본 통계
df['popularity'].mean()     # 48.16
df['popularity'].median()   # 47.0
df['stream_count'].sum()    # 18,220,148,000
df['track_name'].count()    # 84979  → 결측치 21개 제외된 수

# 전체 수치형 컬럼 한 번에 요약
df.describe()
```

### apply + lambda — 익명함수로 열 변환

```python
# ms → 분 변환
df['duration_min'] = df['duration_ms'].apply(lambda x: round(x / 60000, 2))
# 0    3.90
# 1    6.26
# 2    4.82

# 조건 분기도 가능
df['is_long'] = df['duration_ms'].apply(lambda x: 'Long' if x > 300000 else 'Short')
```

### 포인트
- `count()` = 결측치 제외한 수 / `len(df)` = 전체 행 수 → 차이로 결측치 확인 가능
- `value_counts()` = 범주형 열의 빈도 파악할 때 항상 먼저 씀
- `apply(lambda)` = 열 전체에 함수를 적용할 때. 반복문 없이 한 줄로 처리

---

## 2. Handling Missing Data — 결측치 처리

### 개념
빈칸(NaN)이 있는 채로 분석하면 집계 오류나 왜곡이 생긴다.
분석 목적에 따라 제거(dropna)할지 채울지(fillna) 판단하는 것이 핵심.

### 코드

```python
# 결측치 확인
df.isnull().sum()
# track_name    21
# album_name    46

# 결측치 채우기
df['track_name'] = df['track_name'].fillna('Unknown')
df['album_name'] = df['album_name'].fillna('Unknown Album')

# fillna 후 확인 → 결측치 0개
df.isnull().sum()   # Series([], dtype: int64)

# 결측치 행 제거 (해당 행이 분석에 필수일 때)
df_clean = df.dropna(subset=['track_name'])
```

### 실무 판단 기준
| 상황 | 선택 |
|------|------|
| 결측 행 자체가 분석 불필요 | `dropna(subset=['열'])` |
| 수치형 → 전체 흐름 유지하고 싶을 때 | `fillna(mean 또는 median)` |
| 범주형 (이름, 레이블 등) | `fillna('Unknown')` |

### 포인트
- `fillna` 후 반드시 `isnull().sum()` 으로 처리됐는지 재확인
- 이 데이터는 track_name · album_name만 결측 → `fillna('Unknown')` 으로 처리하면 충분

---

## 3. Make New Columns — 파생변수 만들기

### 개념
기존 컬럼을 조합해 새로운 의미의 컬럼을 만드는 것. 실무에서 가장 자주 쓰는 기술.

### 코드

```python
# assign() — 체이닝 방식 (코드 가독성 높음)
df = df.assign(duration_min=lambda x: round(x['duration_ms'] / 60000, 2))

# qcut — 데이터를 동일 비율(분위수)로 나누기
df['pop_tier'] = pd.qcut(df['popularity'], q=4,
                          labels=['Bronze', 'Silver', 'Gold', 'Platinum'])
# pop_tier
# Silver      22444
# Bronze      22330
# Gold        20620
# Platinum    19606

# cut — 균등 구간으로 나누기 (직접 경계값 지정)
df['pop_range'] = pd.cut(df['popularity'], bins=[0, 30, 60, 100],
                          labels=['Low', 'Mid', 'High'])
```

### Spotify 데이터 적용 예시

```python
# 발매 연도 추출
df['release_year'] = pd.to_datetime(df['release_date']).dt.year

# 댄스 + 에너지 복합 지수
df['club_score'] = round(df['danceability'] * df['energy'], 3)

# explicit 여부 → 텍스트로
df['explicit_label'] = df['explicit'].apply(lambda x: 'Yes' if x == 1 else 'No')
```

### qcut vs cut 차이
| | `qcut` | `cut` |
|--|--------|-------|
| 기준 | 데이터 분위수 (개수 균등) | 값의 범위 균등 분할 |
| 언제 | 분포 무관하게 균등 그룹화 | 0~100 같은 의미 있는 구간이 있을 때 |
| 주의 | 중복 bin edge 시 `duplicates='drop'` 필요 | - |

---

## 4. Reshaping Data — 데이터 구조 바꾸기

### 개념
분석 목적에 맞게 데이터의 모양을 변환하는 것.
정렬, 이름 변경, melt/pivot, concat 모두 여기에 해당.

### sort_values / rename / reset_index

```python
# 인기도 내림차순 정렬
df.sort_values('popularity', ascending=False)[['track_name','artist_name','popularity']].head(5)
#                        track_name       artist_name  popularity
#              Against recently    Edward Ramirez         100
#               Sort write save  Alejandra Santos         100

# 컬럼명 변경
df.rename(columns={'stream_count': 'streams', 'track_name': 'title'})

# groupby 결과 → 인덱스를 일반 컬럼으로 변환
df.groupby('genre')['popularity'].mean() \
  .sort_values(ascending=False) \
  .reset_index()
#        genre  popularity
# 0        Pop   48.37
# 1        R&B   48.36
```

### melt / pivot — Wide ↔ Long 변환

```python
# melt: 여러 열 → 하나의 열로 (Wide → Long)
df_small = df[['track_name', 'danceability', 'energy', 'tempo']].head(3)
melted = df_small.melt(id_vars='track_name', var_name='feature', value_name='value')

# track_name             feature   value
# Agent every (0)    danceability    0.15
# Night respond      danceability    0.44
# Agent every (0)          energy    0.74
# Agent every (0)           tempo   73.12

# pivot: Long → Wide 복원
pivot = melted.pivot(index='track_name', columns='feature', values='value')
# feature                 danceability  energy   tempo
# Agent every (0)                 0.15    0.74   73.12
# Future choice whatever          0.62    0.80   71.03
```

### concat — 데이터프레임 이어 붙이기

```python
# 장르별로 분리된 데이터 합치기 (행 기준)
df_pop  = df[df['genre'] == 'Pop'].head(3)
df_rock = df[df['genre'] == 'Rock'].head(3)
combined = pd.concat([df_pop, df_rock]).reset_index(drop=True)

#                 track_name genre
# 0          Agent every (0)   Pop
# 1      Bad fall pick those   Pop
# 2    Song body court movie   Pop
# 3   Future choice whatever  Rock
# 4                Move each  Rock

# 열 기준으로 합치기
pd.concat([df1, df2], axis=1)
```

### 포인트
- `melt` = 분석·시각화 전처리에 필수. 여러 feature를 한 축으로 비교할 때
- `pivot` = melt의 반대. 보고서용 표 만들 때 사용
- `concat` 후에는 `reset_index(drop=True)` 로 인덱스 정리하는 습관

---

## 5. Combine Data Sets — merge로 테이블 합치기

### 개념
엑셀의 VLOOKUP = pandas의 `merge`. 공통 키(key)를 기준으로 두 테이블을 연결.

### 코드

```python
# 장르별 평균 인기도 테이블 생성 후 원본에 붙이기
genre_avg = df.groupby('genre')['popularity'].mean().reset_index()
genre_avg.columns = ['genre', 'genre_avg_pop']

merged = pd.merge(df, genre_avg, how='left', on='genre')
merged['pop_vs_genre'] = round(merged['popularity'] - merged['genre_avg_pop'], 2)

#                track_name  genre  popularity  genre_avg_pop  pop_vs_genre
#         Agent every (0)    Pop          55      48.37          6.63
#           Night respond  Metal          45      48.19         -3.19
#  Future choice whatever   Rock          55      47.97          7.03
```

### 조인 종류
| 옵션 | 설명 | 언제 |
|------|------|------|
| `how='left'` | 왼쪽 기준, 오른쪽에 없으면 NaN | 원본 유지하면서 정보 추가할 때 |
| `how='inner'` | 양쪽 모두 있는 행만 | 확실히 매칭되는 것만 분석할 때 |
| `how='outer'` | 양쪽 전부 포함 | 누락 없이 전체 합칠 때 |
| `how='right'` | 오른쪽 기준 | 드물게 사용 |

### 포인트
- 실무에서 `how='left'` 가 가장 많이 쓰임 → 원본 행 수 유지 보장
- merge 후 `shape` 로 행 수 변화 반드시 확인 (inner 조인 시 행이 줄 수 있음)

---

## 6. Group Data — groupby로 그룹별 집계

### 개념
엑셀 피벗 테이블과 동일. "장르별로", "나라별로", "연도별로" 나눠서 집계.

### 코드

```python
# 단순 집계
df.groupby('genre')['popularity'].mean()

# 여러 함수 동시에 (named aggregation)
df.groupby('genre').agg(
    avg_popularity=('popularity', 'mean'),
    total_streams=('stream_count', 'sum'),
    track_count=('track_id', 'count')
).sort_values('avg_popularity', ascending=False)

#            avg_popularity  total_streams  track_count
# Pop                 48.37  1,538,151,000         7096
# R&B                 48.36  1,625,619,000         7084
# Classical           48.36  1,558,461,000         7158
# Hip-Hop             48.36  1,906,762,000         7160
# ...
# Jazz                47.85  1,447,452,000         7177

# 두 컬럼 기준 그룹화
df.groupby(['genre', 'country'])['stream_count'].mean()
```

### 실무 인사이트 도출 예시
```
분석: 장르별 총 스트리밍 수 비교
→ Hip-Hop 1위 (1.9조), R&B 2위 (1.6조), Classical 3위 (1.5조)
→ 평균 인기도는 장르 간 차이가 거의 없음 (47~48 대)
→ 인사이트: popularity 지표는 장르 경쟁력 비교에 부적합
  stream_count 기반 분석이 더 의미 있음
```

### 포인트
- `agg()` 안에 `결과열명=('대상열', '함수')` 형태로 쓰면 결과 컬럼명이 바로 정리됨
- groupby 결과는 인덱스가 그룹키 → 이후 작업 위해 `reset_index()` 습관화

---

## 7. 시계열 데이터 분석 — rolling / expanding

### 개념
시간 순서가 있는 데이터를 분석할 때 사용.
- `rolling(n)` : 최근 n개 구간의 이동 평균 → 단기 추세 파악
- `expanding()` : 누적 평균 → 전체 기간 기준 트렌드 파악

### 코드

```python
# 연도별 평균 인기도 추출
df['release_year'] = pd.to_datetime(df['release_date']).dt.year
yearly = df.groupby('release_year')['popularity'].mean().sort_index()

# rolling(3) — 3년 이동 평균
yearly.rolling(3).mean().round(2)
# release_year
# 2015      NaN   ← 데이터 3개 미만이면 NaN
# 2016      NaN
# 2017    48.07
# 2018    48.02
# ...
# 2025    48.25

# expanding() — 누적 평균
yearly.expanding().mean().round(2)
# release_year
# 2015    48.23
# 2016    48.11
# 2017    48.07
# ...
# 2025    48.16
```

### rolling vs expanding 차이
| | `rolling(n)` | `expanding()` |
|--|-------------|--------------|
| 계산 범위 | 최근 n개 행 | 처음부터 현재까지 전체 |
| NaN 발생 | 초반 n-1개 행 | 없음 |
| 활용 | 최근 트렌드 모니터링 | 기준선(baseline) 파악 |

### 실무 인사이트 도출 예시
```
분석: 연도별 인기도 이동 평균 (rolling 3)
→ 2017~2019: 48.0 수준 유지
→ 2021~2023: 48.2 → 48.3 소폭 상승
→ 인사이트: popularity 지표 자체가 연도별 변화가 거의 없음
  → 이 데이터에서는 popularity보다 stream_count로
    연도 트렌드를 분석하는 것이 더 유효함
```

---

## 핵심 정리

| 목적 | 코드 |
|------|------|
| 고유값 개수 | `df['열'].nunique()` |
| 빈도 집계 | `df['열'].value_counts()` |
| 기본 통계 | `df.describe()` |
| 열 전체에 함수 적용 | `df['열'].apply(lambda x: ...)` |
| 결측치 확인 | `df.isnull().sum()` |
| 결측치 채우기 | `df['열'].fillna(값)` |
| 결측치 행 제거 | `df.dropna(subset=['열'])` |
| 새 컬럼 추가 | `df.assign(새열=lambda x: ...)` |
| 분위수 구간화 | `pd.qcut(df['열'], q=4, labels=[...])` |
| 균등 구간화 | `pd.cut(df['열'], bins=[...], labels=[...])` |
| 컬럼명 변경 | `df.rename(columns={'구':'신'})` |
| Wide → Long | `df.melt(id_vars='키열', var_name=..., value_name=...)` |
| Long → Wide | `df.pivot(index=..., columns=..., values=...)` |
| 행 기준 합치기 | `pd.concat([df1, df2])` |
| 테이블 조인 | `pd.merge(df1, df2, how='left', on='키')` |
| 그룹별 집계 | `df.groupby('열').agg(결과명=('대상열','함수'))` |
| 이동 평균 | `series.rolling(n).mean()` |
| 누적 평균 | `series.expanding().mean()` |
