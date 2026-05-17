# 예산 통제 (잔액 검증 + 차감)

> 유형: 정책 | 관련 entity: `bdg-ctrl-process`, `bdg-ebdg-cpl` | cross-domain 진입점

## 개요

예산 통제는 지출 도메인(acc-exp)이 호출하는 인터페이스입니다.
지출이 발생할 때마다 BDG_EBDG_CPL의 편성 잔액을 검증하고 즉시 차감합니다.
잔액이 부족하면 ProcException으로 지출 등록을 중단합니다.

## 진입점

acc-exp 도메인의 다음 process가 통제를 호출합니다.

- `exp-cst-process` (품의)
- 원인행위·결의 단계도 잔액 차감/복원의 영향을 받음

cross-domain.yaml에 명시된 관계:

```
acc-exp/exp-cst-process  ── references ──►  bdg/bdg-ebdg-cpl
acc-exp/exp-cst-process  ── references ──►  bdg/bdg-ebdg-sbj  (과목 코드 검증)
acc-exp/exp-cst-process  ── references ──►  bdg/bdg-dtl-biz   (세부사업 코드 검증)
```

## 통제 단위 — 공급가액

Multi 화면 기준으로 통제 단위는 **공급가액(SUPP)**입니다.

```
거래처별 SUPP 합계   = SBJ_AMT (한 과목의 공급가 합)
SBJ_AMT 합계         ≤ BDG_EBDG_CPL.편성잔액
```

부가세는 통제 대상이 아닙니다. ACC_SLIP에 들어가는 총금액(SLIP_AMT)과 혼동하면 안 됩니다.

## 잔액 검증 흐름

```
[지출 화면이 trigger]
   │
   ▼
1. BDG_EBDG_CPL 잔액 조회
   PK = (ACC_YY, ACC_DIV, BDG_CPL_DIV, BDG_DGR='최신',
         CPL_DIV='H',  DTL_BIZ_CD, DEPT_CD, EXP_SBJ, CPL_SEQ)
   잔액 = 편성액 - (이미 차감된 액)

2. 검증
   if (요청 SUPP 합계 > 잔액)
     → throw ProcException("예산 잔액 부족: ...")

3. 차감 UPDATE
   UPDATE BDG_EBDG_CPL
   SET 차감액 = 차감액 + 요청 SUPP 합계
   WHERE PK = ...
```

## 동시성

같은 BDG_EBDG_CPL 행을 동시에 여러 사용자가 차감하려는 상황을 막기 위해 `SELECT ... FOR UPDATE`로 행 잠금을 잡습니다.
DB 방언별 락 구문이 다르므로 매퍼가 분기됩니다.

| DB | 락 구문 |
|----|--------|
| MariaDB | `FOR UPDATE` |
| Oracle | `FOR UPDATE` |
| MSSQL | `WITH (UPDLOCK, ROWLOCK)` 또는 `XLOCK` |

## 차감 복원

지출이 반려·취소되면 차감했던 액을 BDG_EBDG_CPL에 되돌려야 합니다.

| 시점 | 차감/복원 |
|------|----------|
| 품의 저장 | 차감 |
| 품의 반려 | 복원 |
| 품의 수정 (예전 액 → 새 액) | 차이만큼 재차감/복원 |
| 결의 반려 | 복원 (CST_STAT을 15로 되돌리는 흐름과 동기) |

## 흔한 실수

- Multi 화면에서 통제 단위를 총금액으로 잘못 잡아 부가세까지 차감 → 잔액 부족이 과도하게 발생
- 차감만 하고 반려 흐름의 복원을 빠뜨림 → 잔액이 점점 줄어 결산 시점에 미사용 잔액이 남음
- `BDG_DGR='최신'` 결정 로직 누락 — 추경 차수가 늘어나도 본예산 잔액만 보고 있는 케이스

## 참고 룰

- `~/.claude/rules/hmcl-multi-sbj-process.md` (5단계 검증)
- `~/.claude/rules/hmcl-slip-amount-rules.md` (통제 단위 vs 전표 단위)
