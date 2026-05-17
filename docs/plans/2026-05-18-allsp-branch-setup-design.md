# allsp 브랜치 셋업 디자인 (2026-05-18)

## 배경

- 사내 발표용 참고 자료로 `command-center`의 wiki/ontology/GitHub Pages 사이트를 ALLSP 회계 ERP 도메인(`acc/exp` 지출, `bdg` 예산) 기반으로 보여주고 싶다.
- main 브랜치는 현재 이커머스 데모(commerce-catalog / commerce-order / commerce-inventory)를 유지한다.
- 새 `allsp` 브랜치를 만들어 ALLSP 도메인 데모로 운영한다.

## 결정 사항 (브레인스토밍 결과)

| 항목 | 결정 |
|------|------|
| 브랜치명 | `allsp` (main에서 분기) |
| 범위/깊이 | `acc-exp` + `bdg` 둘 다, 얕게 |
| 기존 commerce-* | allsp 브랜치에서 완전 삭제 (main에는 그대로 유지) |
| 도메인 ID 명명 | `acc-exp`, `bdg` (ALLSP 실제 패키지명 그대로) |
| GitHub Pages 노출 | `deploy-pages.yml`의 trigger branches에 `allsp` 추가 |
| 미커밋 변경 | `git stash`로 main에 보관 후 allsp 분기 |

## 변경 범위

| 영역 | 처리 |
|------|------|
| `ontology/index.yaml` | commerce-* 3개 → acc-exp + bdg 2개 |
| `ontology/abox/commerce-*.yaml` × 3 | 삭제 |
| `ontology/abox/acc-exp.yaml` | 신규 (entity 10개) |
| `ontology/abox/bdg.yaml` | 신규 (entity 8개) |
| `ontology/abox/cross-domain.yaml` | 관계 3개로 갱신 |
| `ontology/abox/infra.yaml` | mariadb/oracle/mssql/spring/mybatis/dhtmlx/tomcat/jsp |
| `wiki/commerce-*` × 3 폴더 | 삭제 |
| `wiki/acc-exp/` | 신규 (골격 4 + 토픽 4) |
| `wiki/bdg/` | 신규 (골격 4 + 토픽 4) |
| `wiki/README.md`, `wiki/glossary.md` | 회계 ERP로 갱신 |
| `wiki/ontology-design.md` | 유지 (도메인 중립 메타) |
| `.github/workflows/deploy-pages.yml` | `branches: [main]` → `[main, allsp]` |
| `site/` | **변경 없음** (build-data.mjs가 ontology + wiki를 자동 수집) |
| `projects/`, `worktrees/`, `.claude/` | **변경 없음** |

## ontology 엔티티 카탈로그

### acc-exp (지출)
| Entity | 설명 |
|--------|------|
| EXP_CST | 지출품의 마스터 (CST_STAT 01→91→93) |
| EXP_CST_SBJ | 품의 예산과목 (다중과목 + JRNLZ_TY_CD 분개유형) |
| EXP_CST_CLT | 품의 거래처 (다중거래처) |
| EXP_OACT | 원인행위 마스터 (OACT_STAT) |
| EXP_OACT_SBJ / EXP_OACT_CLT | 원인행위 예산과목/거래처 |
| EXP_RES | 지출결의 (RES_STAT) |
| EXP_PYM_CMD | 지급명령 (CMD_STAT) |
| ACC_SLIP / ACC_SLIPDTS | 회계전표 마스터/상세 |
| ACC_JMT | 분개유형 매핑 (과목+유형 → 차/대변 계정) |

### bdg (예산)
| Entity | 설명 |
|--------|------|
| BDG_BDGDGR | 예산차수 (본예산 M / 추경 S) |
| BDG_MNG_CLS | 정책사업 |
| BDG_MNG_BIZ | 단위사업 (FK: MNG_CLS_CD) |
| BDG_DTL_BIZ | 세부사업 (FK: MNG_BIZ_CD, MNG_CLS_CD) |
| BDG_EBDG_SBJ | 지출과목 (H 목그룹 / M 목 / S 세목) |
| BDG_IBDG_SBJ | 수입과목 (K 관 / H 항 / M 목 / S 세목) |
| BDG_EBDG_CPL | 세출예산편성 (요구 Y / 조정 J / 확정 H) |
| BDG_IBDG_CPL | 세입예산편성 |

### cross-domain 관계
- `EXP_CST_SBJ → BDG_EBDG_CPL` (`reduces_remaining`): 지출이 예산편성 잔액을 차감
- `EXP_CST_SBJ → BDG_EBDG_SBJ` (`references_subject`): 지출과목 참조
- `EXP_CST_SBJ → BDG_DTL_BIZ` (`references_business`): 세부사업 참조

## wiki 토픽

### acc-exp 하위 토픽 4
- 지출-라이프사이클 (품의 → 원인행위 → 결의 → 지급명령 → 전표 4단계)
- 다중과목-지출 (AccExp102BMulti / 120Multi / 106Multi)
- 결재-워크플로우 (상태전이 + esaApp 연동 + SP 영향)
- 회계전표-생성 (ACC_SLIP + ACC_JMT 분개유형)

### bdg 하위 토픽 4
- 사업-위계 (정책 → 단위 → 세부 3계층)
- 과목-체계 (세출/세입 트리 + MARK_ORD)
- 예산-편성-차수 (요구/조정/확정 + 본예산/추경)
- 예산-통제 (지출 차감 + cross-domain 연결)

## 실행 단계

1. **준비**: stash + 브랜치 분기 + 디자인 문서 작성 (이 문서)
2. **삭제**: commerce-* 콘텐츠 제거
3. **ontology 작성**: index/abox/cross-domain/infra
4. **wiki 작성**: acc-exp + bdg + 루트 갱신
5. **workflow 갱신**: deploy-pages.yml 브랜치 trigger
6. **로컬 검증**: build:data + test + vite build
7. **커밋 + 푸시**: 한국어 커밋, push origin allsp → GitHub Pages 자동 배포

## 비범위 (Out of Scope)

- `site/` 코드 변경 (build-data.mjs가 자동 처리)
- PR 생성·머지 (필요 시 사용자가 별도 요청)
- ALLSP_STD 실제 코드 변경 (참조만)
- 깊이 있는 도메인 모델링 (발표용 얕은 깊이)

## Trade-offs

- **얕은 깊이의 장점**: 발표 톤에 부합, 작업량 적음, 빠른 배포
- **얕은 깊이의 단점**: 실제 운영 wiki로는 부족 (참고 자료 목적이라 OK)
- **commerce-* 완전 삭제 vs 보존**: 삭제 선택 — allsp 브랜치만 ALLSP 도메인으로 깔끔. main에는 이력 보존됨
- **acc-exp vs accounting-expense ID**: ALLSP 실제 패키지명 채택 — 내부 친화도가 우선
