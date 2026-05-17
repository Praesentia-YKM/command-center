# 예산 편성과 차수 운영

> 유형: 정책 | 관련 entity: `bdg-cpl-process`, `bdg-bdgdgr`, `bdg-ebdg-cpl`

## 개요

회계연도가 시작되기 전 본예산을 잡고, 회계연도 중간에 추경(추가경정예산)을 통해 조정합니다.
편성은 항상 **요구(Y) → 조정(J) → 확정(H)** 3단계를 거쳐 운영됩니다.

## 차수 구조

```
ACC_YY = 2026
  ├─ BDG_BDGDGR (BDG_CPL_DIV='M', BDG_DGR=1)   ─── 2026년 본예산
  ├─ BDG_BDGDGR (BDG_CPL_DIV='S', BDG_DGR=1)   ─── 2026년 1차 추경
  ├─ BDG_BDGDGR (BDG_CPL_DIV='S', BDG_DGR=2)   ─── 2026년 2차 추경
  └─ ...
```

- `BDG_CPL_DIV`: M(본예산) / S(추경)
- `BDG_DGR`: 차수 번호

각 차수 안에서 편성구분(CPL_DIV)이 Y(요구) / J(조정) / H(확정)로 진행됩니다.

## 편성 단계 (CPL_DIV)

| 단계 | 코드 | 의미 |
|------|------|------|
| 요구 | Y | 부서가 "이만큼 필요하다" 신청 |
| 조정 | J | 예산담당 부서가 검토·조정 |
| 확정 | H | 최종 확정. 지출 통제의 기준 행 |

`BDG_EBDG_CPL`은 같은 차수+사업+과목+부서에 대해 Y / J / H의 행이 별도로 존재합니다.
지출 통제는 `CPL_DIV='H'` 행만 참조합니다.

## 엑셀 업로드 → 반영 흐름

대규모 편성 데이터를 그리드로 한 행씩 입력하는 것은 비현실적입니다.
엑셀 업로드 → 스테이징 → SP 반영의 일괄 처리로 운영합니다.

```
[예산 담당자]
   │ 엑셀 파일 업로드
   ▼
1. deleteUploadData
       BDG_EBDG_CPL_UPLOAD 클리어 (이전 업로드 잔여 정리)
   ▼
2. excelUpload  (batch INSERT, 2000건 단위)
       엑셀 행을 BDG_EBDG_CPL_UPLOAD에 적재
   ▼
3. updateAllLwtYn
       LEAD() 윈도우 함수로 트리의 "마지막 행" 표시 (LWT_YN)
   ▼
4. updateAllCplSeq
       ROW_NUMBER()로 편성순번(CPL_SEQ) 부여
   ▼
5. reflect
       ├─ deleteEbdgCpl       (이번 차수 + 편성구분의 기존 행 삭제)
       ├─ copyInsertEbdgCpl   (스테이징 → BDG_EBDG_CPL 본 테이블)
       └─ callMarkOrdUpdate   (SP_CVR_MARKORD_UPDATE)
```

## BDG_EBDG_CPL PK 분해

```
ACC_YY        + ACC_DIV       + BDG_CPL_DIV  + BDG_DGR
+ CPL_DIV     + DTL_BIZ_CD    + DEPT_CD     + EXP_SBJ      + CPL_SEQ
```

같은 사업+과목+부서라도 차수가 다르면 별도 행. 편성구분이 다르면(Y/J/H) 또 별도 행.
지출 통제는 `CPL_DIV='H'`로 좁혀 검색합니다.

## 본예산 vs 추경

- **본예산(M)**: 회계연도 시작 전 1회 확정
- **추경(S)**: 회계연도 중간에 N차 발생. 차수가 늘어날수록 누적 잔액이 변동

차수가 바뀌어도 사업 위계·과목 체계는 보존되며, 편성행(BDG_EBDG_CPL)만 새 차수로 추가/조정됩니다.

## 참고 룰

- `~/.claude/rules/hmcl-budget-module-structure.md`
- `~/.claude/skills/budget-excel-import` (엑셀 업로드 자동화 스킬)
