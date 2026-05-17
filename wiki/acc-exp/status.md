# Status — 회계/지출

> 진행 상황 추적용. ALLSP_STD 작업 흐름과 동기화.

## 운영 중

- AccExp102BMulti / 120Multi / 106Multi 3종 화면 운영
- 3개 DB 방언 매퍼 동기화 (MariaDB / Oracle / MSSQL)
- esaApp 전자결재 연동 (easElcAppPopup.do + iframe 콜백)
- SP PKG_EA_RECOM$EA_RECOM_GUBUN 결재 단계 상태값 자동 반영

## 최근 변경

- 다중 예산과목 + 다중 분개유형 + 다중 거래처 구조 도입 (기존 단일 과목 화면과 병행)
- 예산통제 기준을 다중과목 화면 한정으로 **공급가액 기준**으로 전환
- 전표 삭제 시 EXP_RES.PYM_SLIP_NO / RES_SLIP_NO 선행 NULL 해제 룰 추가

## 알려진 이슈

- 결재 워크플로우 역방향(상신취소·반려) 시 옵셔널 단계(검토 등) 복귀 분기 누락 주의
- 공유 SELECT가 표시 정규화(MIN/GROUP_CONCAT) 결과를 반환할 때, 호출 흐름별로 원본 PK 의미 vs 정규화 의미를 구분 필요
- JSP의 ES6 템플릿 리터럴 `${}`가 JSP EL과 충돌. 문자열 연결 사용
- iframe 팝업 콜백 함수는 `function` 선언문 필수 (`const`/`let` 금지)

## 참고 룰

- `~/.claude/rules/hmcl-multi-sbj-process.md`
- `~/.claude/rules/hmcl-exp-status-codes.md`
- `~/.claude/rules/hmcl-approval-workflow.md`
- `~/.claude/rules/hmcl-slip-amount-rules.md`
