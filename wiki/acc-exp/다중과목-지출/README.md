# 다중과목 지출

> 유형: 정책 | 관련 entity: `exp-cst`, `exp-oact` | 화면: AccExp102BMulti / 120Multi / 106Multi

## 개요

기존 지출 화면은 한 품의에 예산과목 1개 / 거래처 1개를 가정했습니다.
실무에서는 한 품의에 여러 과목·여러 거래처가 필요한 경우가 빈번해, 별도 화면 3종을
신규 개발해 분리 운영합니다.

## 데이터 구조

```
EXP_CST  (품의 마스터, 1행)
  ├─ EXP_CST_SBJ × N  (예산과목 + 분개유형 + 사업)
  │     같은 과목이라도 분개유형이 다르면 별도 SBJ_SEQ
  └─ EXP_CST_CLT × M  (거래처)
        SBJ_SEQ로 어느 과목에 속하는지 연결
```

OACT(원인행위)도 동일하게 `EXP_OACT + EXP_OACT_SBJ + EXP_OACT_CLT` 구조를 가집니다.
DDL 변경 핵심:

- `EXP_CST` / `EXP_OACT` 에 `SLIP_NO VARCHAR(20)` 추가
- `EXP_CST_SBJ` / `EXP_OACT_SBJ`에 `JRNLZ_TY_CD VARCHAR(3)` 추가

## 5단계 금액 검증

다중과목/다중거래처는 합계 정합성이 깨지기 쉬워 5단계 검증을 한 트랜잭션에서 수행합니다.

| 단계 | 검증 식 | 의미 |
|------|---------|------|
| 1 | `CLT.AMT = CLT.SUPP + CLT.VAT` | 거래처 행 자체 금액 정합 |
| 2 | `Σ CLT.SUPP (한 SBJ에 속한) = SBJ.SBJ_AMT` | 과목-거래처 공급가 합 |
| 3 | `Σ SBJ.SBJ_AMT = CST.CST_SBJ_AMT` | 몸통-과목 공급가 합 |
| 4 | `Σ CLT.AMT = CST.CST_AMT` | 몸통-거래처 총액 합 |
| 5 | `CST_AMT = CST_SBJ_AMT + CST_VAT_AMT` | 총액 분해 |

하나라도 어긋나면 ProcException으로 즉시 중단합니다.

## 예산통제 기준의 차이

Multi 화면에서는 예산통제 단위를 **공급가액(SUPP)** 기준으로 운영합니다.

- `SBJ_AMT = Σ CLT.SUPP` → BDG_EBDG_CPL 잔액 차감 단위
- `CST_AMT = Σ CLT.AMT` → ACC_SLIP.SLIP_AMT 단위

기존(단일과목) 화면은 총금액 기준 통제를 유지하므로 두 화면이 같은 BDG_EBDG_CPL을 두고 서로 다른 단위로 차감/검증한다는 점이 운영상 주의 포인트입니다.

## 분개유형 다중화

같은 예산과목이라도 거래의 성격에 따라 차변·대변 계정이 달라야 할 때가 있습니다.
화면에서 사용자가 분개유형(JRNLZ_TY_CD)을 선택하면, 동일 EXP_SBJ라도 별도 SBJ_SEQ로 분리되고
ACC_JMT 매핑이 각각 적용되어 ACC_SLIPDTS의 차/대변 행이 다른 계정에 떨어집니다.

```
같은 과목 + 분개유형 1   ──► ACC_JMT 매핑 A   ──► 차변 X / 대변 Y
같은 과목 + 분개유형 2   ──► ACC_JMT 매핑 B   ──► 차변 X / 대변 Z
```

## 트레이드오프

- **장점**: 한 품의에 여러 과목·거래처를 한 번에 처리 → 화면 전환·결재 횟수 감소
- **단점**: 5단계 검증·N×M 그리드·분개유형 매트릭스 복잡도가 매우 높음. 단일과목 화면의 단순함을 잃음
- **결정**: 두 화면을 병행 운영. 실무자가 케이스에 따라 선택

## 관련 룰

- `~/.claude/rules/hmcl-multi-sbj-process.md`
- `~/.claude/rules/hmcl-slip-amount-rules.md`
