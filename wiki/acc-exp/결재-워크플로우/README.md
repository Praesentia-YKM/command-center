# 결재 워크플로우

> 유형: 정책 | 관련 entity: `exp-cst`, `exp-oact`, `exp-res`, `exp-pym-cmd`

## 개요

ALLSP 결재는 전자결재 시스템(esaApp) + 상태코드(공통코드 A61/A64/A68/A69) + SP(PKG_EA_RECOM)의 3가지가 맞물려 동작합니다.
화면 코드만 봐서는 의도를 알기 어려운 이유는 **SP가 사용자 모르게 상태값을 덮어쓰기** 때문입니다.

## 상태 코드 체계

| 컬럼 | 공통코드 | 용도 |
|------|---------|------|
| CST_STAT | A64 | 품의 상태 |
| OACT_STAT | A68 | 원인행위 상태 |
| RES_STAT | A69 | 결의 상태 (02 등록 / 88 검토완료 / 93 승인 / 15 반려) |
| CMD_STAT | A61 | 지급명령 상태 (01 미결재 / 91 상신 / 93 전결 / 15 반려) |

같은 코드값이 컬럼별로 다른 의미인 경우가 있어 함정이 됩니다.

| 코드값 | 의미 |
|--------|------|
| A64.01 | 품의등록 (초기 상태) |
| A68.01 | 품의승인 (A64와 전혀 다른 의미) |
| A69.01 | 원인행위승인 |
| A61.01 | 지급요청 (Service가 설정하지만 SP가 즉시 91로 덮어씀) |

## SP가 변경하는 상태값

`PKG_EA_RECOM$EA_RECOM_GUBUN` SP는 결재 단계에 따라 상태값을 즉시 변경합니다.

| 결재 단계 | SP가 설정 | 이후 처리 |
|----------|----------|----------|
| 상신 직후 | 91 | Service가 01로 잡은 직후 SP가 91로 덮어씀 |
| 중간결재 | 92 | |
| 최종결재 | 93 | confirmPopEasApp 콜백이 업무 상태로 다시 덮어씀 |
| 반려 | 15 | |

**핵심**: SP가 잠시 거치는 93은 과도기 값. 공통코드(A61)에 93이 등록되지 않은 이유.

## 전자결재 연동 (esaApp)

`appJs.esaApp()`은 결재선 팝업(easElcAppPopup.do)을 띄웁니다.
팝업은 부모 창의 `parent.fn_eaRecom0011()`과 `parent.fn_search0011()`을 호출하므로,
부모 JSP에 두 함수가 반드시 `function` 선언문으로 존재해야 합니다.

```jsp
<%-- 반드시 function 선언문 (const/let 금지) --%>
function fn_eaRecom0011() {
    const paramObj = { accYy: ..., accDiv: ..., formCd: "KAD18", pgmId: "ACCEXP106" };
    return appJs.esaApp(grid0011, paramObj, "pymCmdNo", "cmdNo", "otln", "cmdNo");
}

function fn_search0011() {
    ApprovalHandler.search();
}
```

`const`/`let`은 모듈 스코프 변수여서 `window`/`parent`의 프로퍼티가 되지 않아 콜백 시점에 `is not a function`이 됩니다.

## 정방향 / 역방향 매트릭스

정방향 1개 ↔ 역방향 1개. 다중과목 102BMulti 화면 예시:

| 기능 | CST_STAT → | OACT_STAT → | 허용 입력 |
|------|------------|-------------|----------|
| 신규 저장 | 01 | 02 | - |
| 수정 저장 | 01 (리셋) | 02 (리셋) | 01 또는 15 |
| 삭제 | - | - | 01 또는 15 |
| 결재상신 | SP가 91 | SP가 91 | 01 |
| 상신취소 | 15 | 15 | 91 |
| 전결처리 | 93 | 93 | ≠ 93 |
| 전결취소 | 15 | 15 | OACT_STAT=93 |
| 반려 | 15 | 15 | 91 |

## 흔한 실수

- 서버 검증에서 `01`만 허용하면 SP가 91로 바꾼 직후 모든 액션이 차단됨 → `01`과 `91`을 함께 허용
- 옵셔널 단계(검토 등) 통과 여부에 따른 역방향 복귀 분기 누락
- 프론트 validator의 allowed 상태값과 백엔드 Service의 if 조건이 불일치 → silent failure

## 관련 룰

- `~/.claude/rules/hmcl-approval-workflow.md`
- `~/.claude/rules/hmcl-exp-status-codes.md`
