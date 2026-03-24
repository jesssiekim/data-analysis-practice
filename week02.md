[week02.md](https://github.com/user-attachments/files/26203285/week02.md)
# 2회차 - Pandas 기초: DataFrame & Subset

## 강의 범위
- 섹션 2: DataFrame & Series
- 섹션 3: Subset Observations (Rows)
- 섹션 4: Subset Variables (Columns)

## 실습 데이터
워싱턴 주 전기차 등록 데이터 (Electric_Vehicle_Population_Data.csv)
- 총 279,780행, 16열
- 출처: 미국 워싱턴 주 정부 공개 데이터

---

## 1. Creating DataFrames — 데이터프레임 만들기

### 개념
DataFrame = 엑셀의 시트 전체 표. 행(row)과 열(column)로 구성된 2차원 표.
Series = 표의 열 하나.

### 코드

```python
import pandas as pd  # 항상 맨 위에 한 번만 실행

# 방법 1: 열(column) 기준으로 만들기
df = pd.DataFrame(
    {"a": [4, 5, 6],
     "b": [7, 8, 9],
     "c": [10, 11, 12]},
    index=[1, 2, 3])

# 방법 2: 행(row) 기준으로 만들기
df = pd.DataFrame(
    [[4, 7, 10],
     [5, 8, 11],
     [6, 9, 12]],
    index=[1, 2, 3],
    columns=['a', 'b', 'c'])

# 실전: CSV 파일 불러오기
df = pd.read_csv("파일명.csv")
```

### 포인트
- `pd` = pandas의 별명 (import pandas as pd 로 지정)
- `df` = 불러온 데이터의 변수명 (관례적으로 df 사용, 다른 이름도 가능)
- 행 인덱스는 **0부터 시작** (엑셀의 1행 헤더 제외)

---

## 2. Subset Observations (Rows) — 행 필터링

### 개념
조건에 맞는 행만 뽑아보는 것. 엑셀의 필터 기능과 동일.

### 코드

```python
# 조건으로 필터링
df[df['Electric Range'] > 200]
# → 주행거리 200마일 초과하는 차량만 추출 (31,243개)

# 중복 제거
df.drop_duplicates()
# → 동일한 행이 여러 번 있을 경우 하나만 남김

# 랜덤 샘플링
df.sample(frac=0.5)   # 전체의 50% 랜덤 추출 → 139,890개
df.sample(n=10)       # 랜덤으로 10개만 추출

# 크기 순 정렬 후 추출
df.nlargest(5, 'Electric Range')   # 주행거리 상위 5개 → Tesla Model S (337마일 = 약 542km)
df.nsmallest(5, 'Electric Range')  # 주행거리 하위 5개

# 미리보기
df.head()     # 위에서 5행
df.head(10)   # 위에서 10행
df.tail()     # 아래에서 5행
df.tail(10)   # 아래에서 10행

# 위치로 행 선택
df.iloc[10:20]  # 10번째~19번째 행 (pandas는 0부터 시작)
```

### 실무 활용 예시
- `nlargest` → "주행거리 상위 차량은 어떤 브랜드인가?" → 브랜드별 기술력 비교
- `sample` → 대용량 데이터 전체를 보기 전에 일부만 먼저 파악할 때
- 조건 필터링 → "2020년 이후 등록된 차량만 분석" → 최근 트렌드 파악

---

## 3. Subset Variables (Columns) — 열 선택

### 개념
필요한 열(컬럼)만 선택해서 보는 것. 엑셀에서 특정 열만 골라보는 것과 동일.

### 코드

```python
# 여러 열 선택
df[['Make', 'Model', 'City']]

# 단일 열 선택
df['Make']
df.Make  # 동일한 결과

# 열 이름 패턴으로 선택 (정규식)
df.filter(regex='Electric')  # 'Electric'이 포함된 열 모두 선택
```

### 실무 활용 예시
- 16개 열 중 분석에 필요한 3~4개만 골라 가볍게 작업
- 클라이언트에게 전달할 때 불필요한 열 제거 후 정리

---

## 4. Subsets — 행과 열 동시 선택 (loc / iloc)

### 개념
행과 열을 동시에 조건으로 지정해서 원하는 부분만 정확히 뽑는 것.

### loc vs iloc 차이

| | loc | iloc |
|--|-----|------|
| 방식 | 이름(label)으로 선택 | 숫자 위치로 선택 |
| 열 선택 | `'City'` | `1` (두 번째 열) |
| 실무 빈도 | 자주 사용 | 가끔 사용 |

### 코드

```python
# iloc — 숫자 위치로 선택
df.iloc[10:20]          # 10~19번째 행
df.iloc[:, 1]           # 모든 행의 1번째 열 (= County)
df.iloc[:, [1, 2, 5]]  # 1, 2, 5번째 열

# loc — 이름으로 선택
df.loc[:, 'City']                          # City 열 전체
df.loc[:, 'City':'Make']                   # City부터 Make 사이 열 전체
df.loc[df['Electric Range'] > 200, ['Make', 'City']]  # 조건 + 특정 열만

# 단일 값 접근
df.iat[1, 2]      # 1번행 2번열 값 하나 (숫자 위치)
df.at[4, 'Make']  # 4번행 'Make' 열 값 하나 (이름)
```

### 포인트
- pandas 인덱스는 **0부터 시작** → 엑셀 6번째 줄 = pandas 4번 행
- `loc`는 열 이름을 알 때, `iloc`는 몇 번째 열인지 알 때 사용

---

## 5. 결과 저장 및 활용

```python
# 결과를 변수에 저장 후 확인
result = df.loc[df['Make'] == 'TESLA', 'City'].value_counts()
result  # 확인 후 괜찮으면 아래 실행

# CSV로 저장
result.to_csv("tesla_by_city.csv", index=True)
# → Jupyter 홈 화면에서 다운로드 가능
```

### 실무 인사이트 도출 예시
```
분석: 테슬라 차량이 많은 도시 TOP 5
→ Seattle 14,164대, Bellevue 7,749대, Redmond 5,319대...
→ 인사이트: 테슬라의 60% 이상이 킹카운티(시애틀 일대) 집중
→ 해석: 고소득 지역일수록 테슬라 비중 높음 → 테슬라는 프리미엄 브랜드 포지셔닝
```

---

## 핵심 정리

| 목적 | 코드 |
|------|------|
| 데이터 불러오기 | `pd.read_csv("파일명.csv")` |
| 전체 미리보기 | `df.head()` |
| 조건으로 행 필터링 | `df[df['열'] > 값]` |
| 특정 열만 선택 | `df[['열1', '열2']]` |
| 행+열 동시 선택 | `df.loc[조건, ['열']]` |
| 결과 저장 | `result.to_csv("파일명.csv")` |
