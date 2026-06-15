# 2026-06-15 실습 — 데이터 시각화 (Matplotlib / Seaborn)

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (강사 코드 따라치기)
> **관련 강의 파일**: `../lectures/2026-06-15_AI-Security-Course_Lecture_Data-Visualization.md`

---

## 느낀 것 / 배운 것

### 📅 2026-06-15

- matplotlib는 옵션을 일일이 손으로 주는 **저수준** 느낌, seaborn은 데이터프레임 컬럼명만 넘기면 알아서 그려주는 **고수준** 느낌이었다. 같은 산점도라도 `plt.scatter` vs `sns.scatterplot`의 코드량 차이가 컸다.
- `subplot`으로 한 figure에 4개 그래프를 배치해보면서 **Figure(도화지) / axes(개별 칸)**의 구분이 손에 잡혔다.
- `sns.scatterplot`에서 `hue`/`style`/`size`를 동시에 주니 한 그래프에 변수 3~4개를 욱여넣을 수 있었다. 다만 **너무 많이 넣으면 오히려 안 읽힌다**는 것도 느꼈다.

---

## 실습 개요
- **목표**: matplotlib 기본 작도 + seaborn 통계 시각화 손에 익히기.
- **방법**: `plt_01`(matplotlib 기본/꾸미기/pie/DataFrame.plot), `plt_02`(seaborn scatter/line/bar/lmplot).

---

## 진행 과정

### Step 1. Matplotlib 기본 (plt_01)
```python
# subplot 2x2 배치 + tight_layout
plt.subplot(2,2,1); plt.plot(x, y);  plt.title('plot 1')
plt.subplot(2,2,2); plt.plot(x, y1); plt.title('plot 2')
# ...
plt.tight_layout(); plt.show()

# 스타일 문자열 + 꾸미기
plt.plot(x,y,'o--b', x,y1,'+-r', x,y2,'s:k', linewidth=2, markersize=7)
plt.ylim(0,120); plt.grid(True)
plt.legend(['1번','2번','3번'], loc='upper right')

# pie - 점유율
plt.pie(sales, labels=loc, autopct='%0.1f%%',
        explode=[0,0.1,0,0,0,0], shadow=True)
```
- 선 그래프 겹쳐 그리기, 한글 라벨(`plt.rc('font', family='gulim')`).
- `DataFrame.plot`으로 연도별 매출 작도 + **boolean 인덱싱으로 2003~2007 구간만** 필터해서 그리기.

### Step 2. Seaborn 통계 시각화 (plt_02)
```python
# tips 데이터
sns.scatterplot(x='total_bill', y='tip', data=df)
sns.barplot(x='day', y='total_bill', data=df, hue='sex')

# fmri 데이터 - 범주 분리 + 신뢰구간 off
sns.lineplot(x=df['timepoint'], y=df['signal'],
             hue=df['event'], style=df['event'], marker='o', ci=False)

# 변수 다중 매핑
sns.scatterplot(x='total_bill', y='tip', hue='size', style='size',
                sizes=(15,200), alpha=0.5, data=df)

# 회귀선 + facet
sns.lmplot(x='total_bill', y='tip', col='time', row='smoker', data=df)
```
- `hue`/`style`/`size`로 범주 자동 분리, `lmplot`으로 회귀선 + `col`/`row` facet(time/smoker별 분할).

---

## 결과 / 산출물
- matplotlib·seaborn 주요 그래프 작도 코드 세트 (선 / subplot / pie / scatter / line / bar / lm).

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| matplotlib↔seaborn의 코드량/추상화 수준 차이를 체감 | 강사 코드 따라치기 위주라 직접 변형은 적었음 |
| facet(`col`/`row`)으로 조건별 그래프 분할까지 해봄 | `ci`(신뢰구간)의 통계적 의미는 아직 감으로만 |

---

## 다음에 이어서 할 것
- 해당 없음 — 시각화는 곧바로 ML 실습의 EDA(pairplot, corr 히트맵 등) 활용으로 이어짐.

## 다음에 파고들 만한 질문
- (강의 파일에 1개 정리 — 여기선 생략)

---
*작성일: 2026-06-15 | 인공지능 사관학교 AI 보안 과정*
