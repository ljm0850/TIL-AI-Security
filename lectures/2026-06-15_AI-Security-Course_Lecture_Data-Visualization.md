# 2026-06-15 강의 정리 — 데이터 시각화 (Matplotlib / Seaborn)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **강의 주제**: Matplotlib · Seaborn 시각화 (지난주 입문에 이어 마무리)
> **관련 실습 파일**: `../practice/2026-06-15_AI-Security-Course_Practice_Data-Visualization.md`

---

## 핵심 인사이트 모음

### 📅 2026-06-15

- 그래프 종류 선택은 결국 **"내가 보려는 데이터가 수치형이냐 범주형이냐"**에서 갈린다. 수치 분포를 보고 싶으면 히스토그램, 범주별 개수를 보고 싶으면 막대그래프(countplot), 비율을 보고 싶으면 파이, 이상치를 보고 싶으면 박스플롯. **차트는 취향이 아니라 데이터 타입에 따라 결정된다**는 게 정리됐다.
- Seaborn은 결국 **matplotlib 위에 Pandas를 끼워 편하게 만든 래퍼**다. matplotlib로 한참 옵션 줄 걸 sns는 `hue`/`style` 한 줄로 범주별 색/모양을 나눠준다. 바닥(matplotlib)을 알아야 sns가 안 될 때 손을 댈 수 있겠다 싶었다.

---

## 1. Matplotlib

### 핵심 개념
- **Figure**: 그림이 그려지는 전체 범위(Scope) / 도화지. 그 안에 `subplot`(axes)을 배치한다.

### 상세 내용
- `plot` / `subplot` / `tight_layout`, 선 스타일(`'o--b'` 등), `xlim`/`ylim`, `title`/`xlabel`/`ylabel`, `grid`, `legend(loc=)`.
- 한글 폰트: `plt.rc('font', family='gulim')`.
- 그래프별 용도:
  - **히스토그램**: **연속형 수치 데이터**를 구간(bin)으로 나눠 빈도(분포)를 본다.
    - ⚠️ "예/아니오 같은 범주형 값의 개수"를 세는 건 히스토그램이 아니라 **막대그래프 / seaborn `countplot`**이다. (히스토그램 = 수치 분포, 막대그래프 = 범주 개수 — 헷갈리기 쉬운 지점이라 정리해 둠)
  - **파이(pie)**: 전체 100% 기준 점유율. `autopct`로 비율 표기, `explode`로 조각 분리.
  - **박스플롯**: 사분위수와 **이상치(outlier) 확인**에 좋음.
- `DataFrame.plot(x=, y=, style=, title=)`로 Pandas에서 바로 작도 가능 (boolean 인덱싱으로 특정 구간만 필터해서 그릴 수도 있음).

---

## 2. Seaborn

### 핵심 개념
- Pandas 데이터 시각화를 위한 **matplotlib 기반** 라이브러리.

### 상세 내용
- `scatterplot` / `lineplot` / `barplot`: `hue`(색 구분), `style`(모양 구분), `size` 등으로 범주를 자동 분리.
- `lineplot`의 `ci` 옵션으로 신뢰구간 표시 on/off.
- **회귀 시각화**: `lmplot(x, y)` — 산점도 + 회귀선. `col`/`row`로 조건별 분할(facet). **값을 예측하는 관계를 눈으로 확인할 때** 쓴다.

---

## 다음에 파고들 만한 질문
> 강의가 다시 다루지 않을 곁가지 호기심만.

- `sns.lmplot`과 `regplot`은 회귀선을 둘 다 그리는데 뭐가 다른가? (`col`/`row` facet이 되고 안 되고의 차이인지 궁금)

---

## 찾아본 것
> 오늘 직접 파고든 항목은 없음.

| 주제 | 찾아본 날짜 | 내용 요약 또는 링크 | 출처 |
|---|---|---|---|
| | | | |
|||||

---
*작성일: 2026-06-15 | 인공지능 사관학교 AI 보안 과정*
