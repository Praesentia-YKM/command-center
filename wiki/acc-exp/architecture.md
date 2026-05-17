# Architecture — 회계/지출

> 정책 인덱스 + 데이터 흐름. 코드 구조/ERD는 ontology와 ALLSP_STD 코드에 위임합니다.

## 정책 인덱스

| 주제 | 책임 entity | 핵심 정책 |
|------|-------------|----------|
| [지출 라이프사이클](지출-라이프사이클/README.md) | `exp-cst-process`, `exp-oact-process`, `exp-res-process`, `exp-pym-cmd-process` | 4단계 상태 전이 + 단계 간 게이트 |
| [다중과목 지출](다중과목-지출/README.md) | `exp-cst`, `exp-oact` | EXP_*_SBJ(과목+JRNLZ_TY_CD) + EXP_*_CLT(거래처) 5단계 금액 검증 |
| [결재 워크플로우](결재-워크플로우/README.md) | `exp-cst`, `exp-oact`, `exp-res`, `exp-pym-cmd` | 상태코드 A61/A64/A68/A69 + esaApp + SP 영향 |
| [회계전표 생성](회계전표-생성/README.md) | `acc-slip`, `acc-jmt` | ACC_SLIP/ACC_SLIPDTS 생성 + 분개유형 매핑 |

## 데이터 흐름

### 4단계 라이프사이클

```
[요청부서]
   │ 다중과목 + 거래처 입력
   ▼
exp-cst-process (품의)
   │ 1. 예산잔액 검증 (bdg 도메인 BDG_EBDG_CPL 조회)
   │ 2. EXP_CST + EXP_CST_SBJ + EXP_CST_CLT INSERT
   │ 3. ACC_JMT로 분개유형 → 차/대변 매핑 조회
   │ 4. ACC_SLIP + ACC_SLIPDTS 부채전표 생성 (SLIP_STAT='M')
   │ 5. 결재상신 → CST_STAT 01→91 (SP가 즉시 91 반영)
   ▼
exp-oact-process (원인행위)
   │ 품의 결재 완료(93) 이후 OACT 등록
   │ 다중과목 102BMulti에서는 같은 트랜잭션에 OACT도 동시 생성
   ▼
exp-res-process (결의)
   │ AccExp120Multi에서 요청 → 검토 → 승인
   │ 승인 시 ACC_SLIP.SLIP_STAT 'M'→'S', APP_STAT 93
   ▼
exp-pym-cmd-process (지급명령)
   │ AccExp106Multi에서 명령 등록 → 결재완료/전결
   │ 52번 명령전표 생성 (SP 호출), PYM_STAT M→J
   ▼
[자금 집행 완료]
```

### 결재 단계의 SP 영향

```
사용자가 "결재상신" 클릭
   │
   ▼
Service: CST_STAT = '01' UPDATE
   │
   ▼
SP PKG_EA_RECOM$EA_RECOM_GUBUN 실행
   │ 결재 단계별로 즉시 덮어씀
   │   - 상신 직후 → 91
   │   - 중간결재 → 92
   │   - 최종결재 → 93
   │   - 반려     → 15
   ▼
서버 검증: '01' 또는 '91' 둘 다 허용해야 안전
```

### 예산 차감의 단위 (cross-domain)

```
지출 화면 입력값
  ├─ 거래처별 SUPP (공급가)  ──► 예산 차감 단위
  └─ 거래처별 AMT  (총금액)   ──► ACC_SLIP.SLIP_AMT 단위

검증:
  Σ CLT.SUPP = SBJ.SBJ_AMT = CST.CST_SBJ_AMT  (공급가 정합)
  Σ CLT.AMT  = SBJ.AMT     = CST.CST_AMT      (총액 정합)
```

cross-domain 관계는 `ontology/abox/cross-domain.yaml`에 정의됩니다.

## 레이어 구조

```
JSP (WebContent/jsp/acc/exp/*.jsp)
  ├─ DHTMLX Grid (app_dhtmlxApi.js 확장)
  ├─ AllspAlert.confirm / quickConfirm (Promise 기반 모달)
  └─ appJs.httpSend / httpAsyncSendJson (동기/비동기 AJAX)
        ▼
Controller (@Controller, /accExp102BMulti/*.do)
        ▼
Service (@Service, DefaultVO 자동 주입)
        ▼
DAO (extends DefaultDAO)
        ▼
MyBatis sqlmap/{maria|oracle|mssql}/acc/exp/*.xml
        ▼
DB (FN_NVL / FN_COMCD_NM 등 커스텀 함수 포함)
```
