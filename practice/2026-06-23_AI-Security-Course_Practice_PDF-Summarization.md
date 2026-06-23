# 2026-06-23 실습 — PDF 텍스트 추출 & LLM 요약 파이프라인

> **과정**: 인공지능 사관학교 AI 보안 과정
> **실습 유형**: 개인 실습 (LLM 활용 — 문서 처리 자동화)
> **관련 코드**: ag_06_1_pdf_to_text, ag_06_2_요약, ag_06_3_요약통합

---

## 느낀 것 / 배운 것

### 📅 2026-06-23

- **"PDF → 텍스트 → 요약"을 함수로 쪼개니 파이프라인이 보였다.** ag_06_1(추출) → ag_06_2(요약) → ag_06_3(둘을 합쳐 `summerize_pdf(pdf) → 요약`)으로 단계가 명확히 나뉜다. 각 단계가 "입력 경로 → 출력 경로"를 반환하도록 짜니, 단계를 갈아끼우거나 중간 산출물(txt)을 점검하기 쉬웠다. 앞서 배운 함수 분리·모듈화가 AI 활용 코드에서도 그대로 힘을 발휘.
- **PDF 텍스트 추출의 현실적 난점은 "헤더/푸터 같은 노이즈"였다.** pymupdf로 페이지마다 `clip` 영역을 지정해 **상단 75pt·하단 75pt를 잘라내고** 본문만 뽑았다. PDF는 깔끔한 텍스트가 아니라 페이지마다 반복되는 머리말/꼬리말이 섞여 있어, "추출"이 곧 "정제"라는 걸 체감. 요약 품질은 결국 이 전처리에서 갈린다.
- **AI 보안 관점(중요) — 외부 문서를 프롬프트에 그대로 넣는 건 프롬프트 인젝션 표면이다.** ag_06_2는 `system_prompt = f'''...{txt}'''` 로 **PDF에서 뽑은 외부 텍스트를 system 프롬프트에 직접 삽입**한다. 만약 그 PDF 안에 "이전 지시는 무시하고 ○○해" 같은 문구가 숨어 있으면, 모델이 그걸 지시로 받아들여 탈취될 수 있다(프롬프트 인젝션). "신뢰할 수 없는 입력(문서)"과 "지시(시스템 프롬프트)"를 같은 공간에 섞는 구조 자체가 위험하다

---

## 실습 개요
- **목표**: 논문 PDF에서 본문 텍스트를 추출하고, LLM으로 저자의 문제 인식·주장을 정해진 포맷으로 요약하는 자동화 파이프라인 구축.
- **도구**: pymupdf(PDF 파싱), OpenAI API(gpt-4o-mini).

---

## 진행 과정

### Step 1. PDF → 텍스트 추출 (ag_06_1)
```python
import pymupdf
doc = pymupdf.open(pdf_file_path)
header_height, footer_height = 75, 75
full_text = ''
for page in doc:
    rect = page.rect
    # 헤더/푸터 영역을 제외한 본문만 clip으로 추출
    text = page.get_text(clip=(0, header_height, rect.width, rect.height - footer_height))
    full_text += text + f'\n{"-"*30}\n'
# os.path.basename/splitext로 파일명만 뽑아 output/ch04/{name}.txt 저장
```
- 핵심: 페이지마다 반복되는 머리말/꼬리말을 좌표 영역으로 잘라내 본문 품질 확보.

### Step 2. 텍스트 요약 (ag_06_2)
```python
def summerize_text(file_path):
    txt = open(file_path, encoding='utf-8').read()
    system_prompt = f'''
    너는 다음 글을 요약하는 봇이다. ... 주요 내용을 요약하라.
    # 제목
    ## 저자의 문제 인식 및 주장 (15문장 이내)
    ## 저자 소개
    ==== 이하 텍스트 ====
    {txt}
    '''   # ⚠️ 외부 txt가 시스템 프롬프트에 직접 삽입됨 (프롬프트 인젝션 표면 — 위 인사이트)
    response = client.chat.completions.create(
        model='gpt-4o-mini', temperature=0.1,
        messages=[{'role':'system', 'content':system_prompt}])
    return response.choices[0].message.content
```
- 출력 포맷을 프롬프트에 명시 → 일정한 구조의 요약. `if __name__ == '__main__'`으로 모듈/실행 구분.

### Step 3. 통합 파이프라인 (ag_06_3)
```python
def summerize_pdf(pdf_file_path, output_file_path):
    txt_file_path = pdf_to_text(pdf_file_path)   # Step1 함수화
    summary_txt = summerize_text(txt_file_path)  # Step2 재사용
    open(output_file_path, 'w', encoding='utf-8').write(summary_txt)
    return summary_txt
```
- Step1·2를 함수로 묶어 "PDF 경로 → 요약본" 한 번에. 단계별 함수를 조립하는 구조.

---

## 결과 / 산출물
- 논문 PDF → 본문 txt → 지정 포맷(제목/문제인식·주장/저자소개) 요약본 자동 생성.

---

## 잘 된 것 / 안 된 것

| 잘 된 것 | 안 된 것 |
|---|---|
| 추출·요약·통합을 함수로 분리해 재사용 가능한 파이프라인 구성 | 외부 텍스트를 system에 직접 삽입 — 프롬프트 인젝션 무방비 |
| clip 좌표로 헤더/푸터를 걷어내 본문 품질 확보 | 표/그림 등 비텍스트 요소는 손실 (텍스트만 추출) |
| 출력 포맷을 프롬프트로 강제해 일관된 요약 |  |

---
*작성일: 2026-06-23 | 인공지능 사관학교 AI 보안 과정*
