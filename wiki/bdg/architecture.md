# Architecture — 회계/예산

> 정책 인덱스 + 데이터 흐름.

## 정책 인덱스

| 주제 | 책임 entity | 핵심 정책 |
|------|-------------|----------|
| [사업 위계](사업-위계/README.md) | `bdg-mng-cls`, `bdg-mng-biz`, `bdg-dtl-biz` | 정책 → 단위 → 세부 3계층 owns |
| [과목 체계](과목-체계/README.md) | `bdg-ebdg-sbj` | 세출 H/M/S 3계층 + MARK_ORD |
| [예산 편성·차수](예산-편성-차수/README.md) | `bdg-cpl-process`, `bdg-bdgdgr`, `bdg-ebdg-cpl` | 본예산/추경 + 요구→조정→확정 |
| [예산 통제](예산-통제/README.md) | `bdg-ctrl-process`, `bdg-ebdg-cpl` | 잔액 검증·차감 (cross-domain) |

## 데이터 흐름

### 편성 — 엑셀 업로드 → 반영

```
[예산 담당자]
   │ 엑셀 파일 업로드 (수천~수만 행)
   ▼
deleteUploadData  (BDG_EBDG_CPL_UPLOAD 클리어)
   ▼
excelUpload  (batch INSERT, 2000건 단위)
   ▼
updateAllLwtYn  (LEAD 윈도우로 마지막 행 표시)
   ▼
updateAllCplSeq  (ROW_NUMBER로 편성순번)
   ▼
reflect
   ├─ deleteEbdgCpl  (이번 차수 + 편성구분 대상 행 제거)
   ├─ copyInsertEbdgCpl  (스테이징 → BDG_EBDG_CPL)
   └─ callMarkOrdUpdate (SP_CVR_MARKORD_UPDATE)
```

### 통제 — 지출이 잔액 차감

```
[지출 화면 (acc-exp)]
   │ 다중과목 + 거래처 입력 → 저장
   ▼
bdg-ctrl-process
   │ 1. BDG_EBDG_CPL 잔액 SELECT (FOR UPDATE)
   │    PK: (ACC_YY, ACC_DIV, BDG_CPL_DIV, BDG_DGR,
   │         CPL_DIV='H', DTL_BIZ_CD, DEPT_CD, EXP_SBJ, CPL_SEQ)
   │ 2. SUPP 합계(공급가) ≤ 잔액 검증 → 미만이면 ProcException
   │ 3. 잔액 차감 UPDATE
   ▼
[지출 도메인이 후속 INSERT 진행]
```

## 사업 위계 트리

```
BDG_MNG_CLS (정책사업)        ─── ACC_YY + MNG_CLS_CD
  └─ BDG_MNG_BIZ (단위사업)   ─── + MNG_BIZ_CD
        └─ BDG_DTL_BIZ (세부) ─── + DTL_BIZ_CD ◄─ EXP_CST_SBJ.DTL_BIZ_CD가 직접 참조
```

## 과목 체계 트리 (세출)

```
BDG_EBDG_SBJ                  LVL  EXP_SBJ 자릿수
  ├─ H 목그룹                  1    (예) 5
  │     └─ M 목                2          5+5
  │           └─ S 세목        3          5+5+5  ◄─ EXP_CST_SBJ.EXP_SBJ가 참조
```

`MARK_ORD`(10자리, 각 LVL 2자리)로 그리드 정렬 순서를 안정화합니다.

## 코드값 요약

| 컬럼 | 코드 | 의미 |
|------|------|------|
| BDG_CPL_DIV | M / S | 본예산 / 추경 |
| CPL_DIV | Y / J / H | 요구 / 조정 / 확정 |
| LVL (EBDG_SBJ) | H / M / S | 목그룹 / 목 / 세목 |
| LVL (IBDG_SBJ) | K / H / M / S | 관 / 항 / 목 / 세목 |
