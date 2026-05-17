# 회계전표 생성

> 유형: 정책 | 관련 entity: `acc-slip`, `acc-jmt`

## 개요

지출의 각 단계는 회계전표(ACC_SLIP)를 생성하거나 그 상태를 전이시킵니다.
전표는 마스터(ACC_SLIP) + 상세(ACC_SLIPDTS)의 2단 구조이며, 분개유형 매핑(ACC_JMT)에 따라 차변·대변 행으로 분해됩니다.

## 전표 데이터 구조

```
ACC_SLIP (마스터, 1행)
  ├─ SLIP_NO        VARCHAR(20) PK
  ├─ SLIP_AMT       총금액 (공급가 + 부가세)
  ├─ SLIP_STAT      'M' 미승인 / 'S' 승인
  ├─ APP_STAT       '01' 미결재 / '93' 결재완료
  ├─ OCC_DIV_NO     발생구분번호 (예: EXP_OACT_NO)
  └─ ...
ACC_SLIPDTS (상세, N행)
  ├─ DR_CR_DIV      'D' 차변 / 'C' 대변
  ├─ ACNT_CD        회계계정코드 (ACC_JMT로 결정)
  ├─ DTS_AMT        거래처별 총금액
  └─ ...
```

## 금액의 단위 차이

같은 화면에서 두 가지 금액 단위가 공존합니다.

| 단위 | 적용 |
|------|------|
| **총금액** (공급가 + 부가세) | ACC_SLIP.SLIP_AMT, ACC_SLIPDTS.DTS_AMT |
| **공급가액** (부가세 제외) | BDG_EBDG_CPL 잔액 차감 (cross-domain) |

이 둘을 혼동해 SLIP_AMT에 공급가액을 넣거나 예산 차감에 총금액을 사용하면 정합성이 깨집니다.

## 분개유형 매핑 (ACC_JMT)

같은 예산과목이라도 거래의 성격(분개유형 JRNLZ_TY_CD)에 따라 차/대변 회계계정이 달라야 합니다.
ACC_JMT 테이블이 (과목 + 분개유형) → (차변 계정, 대변 계정) 매핑을 들고 있습니다.

```
EXP_CST_SBJ.SBJ_SEQ = 1   (과목 P, 분개유형 J1)
                            │
                            ▼
                     ACC_JMT (P, J1) → 차변 5001 / 대변 7011
                            │
                            ▼
                     ACC_SLIPDTS 2행 (D 5001, C 7011)

EXP_CST_SBJ.SBJ_SEQ = 2   (같은 과목 P, 분개유형 J2)
                            │
                            ▼
                     ACC_JMT (P, J2) → 차변 5001 / 대변 8033
                            │
                            ▼
                     ACC_SLIPDTS 2행 (D 5001, C 8033)
```

## 단계별 전표 변화

| 단계 | 전표 변화 |
|------|----------|
| 품의 저장 (102BMulti) | ACC_SLIP + ACC_SLIPDTS 신규 생성. `SLIP_STAT='M'`, `APP_STAT='01'` |
| 결의 승인 (120Multi) | 같은 ACC_SLIP을 `SLIP_STAT='S'`, `APP_STAT='93'`로 UPDATE |
| 지급명령 결재완료 (106Multi) | SP 호출로 52번 명령전표 신규 생성. EXP_PYM_CMD에 SLIP_NO 기록 |

## FK 해제 순서 (전표 삭제 시)

ACC_SLIP을 참조하는 FK 컬럼을 먼저 NULL로 해제해야 삭제가 됩니다.

| 참조 테이블 | FK 컬럼 | FK 이름 |
|------------|---------|---------|
| EXP_RES | PYM_SLIP_NO | FK_RES_R_SLIP2 |
| EXP_RES | RES_SLIP_NO | FK_RES_R_SLIP |

```java
// ❌ FK 해제 없이 삭제 → DataIntegrityViolationException
accExp106MultiDAO.deleteAccSlipDts(slipParam);
accExp106MultiDAO.deleteAccSlip(slipParam);      // FK 위반
accExp106MultiDAO.updateResPymSlipNo(params);    // 너무 늦음

// ✅ FK 해제 먼저, 그 다음 삭제
accExp106MultiDAO.updateResPymSlipNo(params);    // PYM_SLIP_NO = NULL
accExp106MultiDAO.deleteAccSlipDts(slipParam);
accExp106MultiDAO.deleteAccSlip(slipParam);
```

## 관련 룰

- `~/.claude/rules/hmcl-slip-amount-rules.md`
- `~/.claude/rules/java-coding-style.md` (FK 해제 순서)
