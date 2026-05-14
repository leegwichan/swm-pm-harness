# SWM PM Harness

SW 마에스트로 연수 프로젝트의 **기획·기술 의사결정 분석** + **기획 심의 통과를 위한 기획서 초안 작성**을 돕는 Claude Code 하네스. 모든 판정의 기준은 **"2026년 신입 개발자 취업 성공"**.

팀 회의 RAW DATA(회의록, 기획서 초안, 기술 스펙 등)를 입력으로 주면 두 가지 산출물을 만들 수 있다.

### 1. 4페르소나 분석 리포트 — `/analyze-meeting`

- 4명의 서브에이전트가 **각자 독립된 컨텍스트**에서 병렬 분석
- 메인 Claude가 **정(正)·반(反)·합(合) + 팀원별 취업 레버리지 점검** 구조로 종합
- `projects/outputs/<project>/{YYYY-MM-DD-HHmm}-{mode}-report.md`로 저장

### 2. AI·SW마에스트로 17기 기획서 초안 — `/draft-proposal`

- `projects/forms/` 의 17기 양식·체크리스트(외부 후기 5단계 반영) 기반으로 초안 작성
- 본문 `02-main.md` (10p) + 요약 `01-summary.md` (1p)
- **11개 섹션 묶음 인터리브 질의응답**: 한 섹션마다 RAW 기반 초안 → 1~3개 질문 → 답변 반영 → 부분 저장
- "팀 논의 안 됨" 류 답변은 본문 직후 `<!-- ⚠️ 추가 논의 필요 -->` 블록으로 인라인 누적, 멈추지 않고 다음 섹션 진행
- 시스템 구성도·AI 데이터 플로우 등 시각 자료 권장 섹션은 **Mermaid 초안 + placeholder 제작 가이드**가 디폴트로 함께 들어감
- `projects/outputs/<project>/{YYYY-MM-DD-HHmm}-proposal/` 에 저장. `--resume <ts>` 로 중간 재개 가능

**최우선 목표 (고정):**

> "SWM에서의 활동을 바탕으로 2026년 백엔드/프론트 신입 개발자 서비스/대기업 취업에 성공한다."

합(合) 섹션의 모든 판정은 이 목표를 기준으로 내려진다. 기술적 재미·유저 PMF·평가 점수는 보조 지표.

## 비공식 툴 디스클레이머

- 본 레포는 **SW 마에스트로 연수생 개인이 학습 목적으로 만든 비공식 툴**이다.
- **과학기술정보통신부, IITP, SW 마에스트로 운영진과 무관**하다.
- 본 툴이 산출하는 어떤 평가·예측도 **실제 단계평가/수료평가 결과를 보장하지 않는다.**

## 4명의 서브에이전트

| 에이전트 | 역할 | 핵심 질문 |
|---|---|---|
| `tech-lead-mentor` | 스타트업 CTO 10년차 + 면접관 | "3개월 안에 UT·발표 준비·발표까지 실사용 가능한 완성도로 나와? 면접 레버리지 있어?" |
| `target-user-persona` | 관심 없음이 디폴트인 실제 유저 | "내가 이걸 지금 쓰는 대안보다 **10배** 좋은가?" |
| `swm-reviewer` | 드라이한 단계평가/수료평가 위원 | "세금 지원 3개월 압축 일정의 결과물로 타당한가? 불통과 사유는?" |
| `peer-competitor` | 같은 자리 노리는 동기/경쟁자 | "README 한 줄이 GitHub 수천 개와 구분되는가?" |

정의는 `.claude/agents/*.md` 각 파일 참고. 현재 정확히 **4명**이며, 새 에이전트가 추가되면 `/analyze-meeting`의 모드 정의도 함께 갱신해야 한다.

## 설치

별도 설치 과정 없음. Claude Code가 이 폴더 안에서 실행되기만 하면 된다.

```bash
cd swm-pm-harness
claude
```

Claude Code가 `.claude/agents/`의 서브에이전트와 `.claude/skills/`의 스킬을 자동으로 인식한다.

## 멀티 프로젝트 구조

팀이 **동시에 여러 기획**을 진행하는 것을 전제로 설계되어 있다.

```
projects/
├── inputs/
│   ├── sample-project/           # 샘플 (동작 검증용)
│   │   └── sample-meeting.md
│   ├── <project-A>/              # 사용자가 직접 생성
│   └── <project-B>/
└── outputs/
    ├── sample-project/
    ├── <project-A>/
    └── <project-B>/
```

입력과 출력이 최상위에서 나뉘고, 각각 아래에 **프로젝트 이름 디렉터리**가 들어가는 구조다.

### 새 프로젝트 추가

사용자가 **명시적으로** 디렉터리를 만든다. 하네스가 자동으로 만들지 않는다.

```bash
mkdir -p projects/inputs/<project-name> projects/outputs/<project-name>
```

그 다음 `projects/inputs/<project-name>/`에 회의록·기획서·기술 스펙 `.md` 파일을 넣으면 된다.

## 사용법 — `/analyze-meeting` (4페르소나 분석)

### 호출 형태

```
/analyze-meeting <project> [mode] [input-path]
```

- `<project>` (필수): `projects/inputs/` 아래 디렉터리 이름. 예: `sample-project`, `stitch`.
  - 완전히 생략하면 하네스가 `projects/inputs/` 아래 디렉터리 목록을 보여주고 선택을 되묻는다.
- `mode` (선택, 기본 `full`)
- `input-path` (선택): 생략 시 `projects/inputs/<project>/` 내 가장 최근 수정된 `.md` 자동 선택.

### 예시

```
/analyze-meeting sample-project                                        # 가장 최근 파일, full 모드
/analyze-meeting sample-project full                                   # 동일
/analyze-meeting sample-project planning                               # planning 모드, 최근 파일
/analyze-meeting stitch tech projects/inputs/stitch/stack-v2.md        # 특정 파일 지정
```

### 모드

| mode | 참여 에이전트 | 인원 | 언제 쓰는가 |
|---|---|---|---|
| `full` (기본) | 4명 전원 | 4 | 주요 의사결정 회의, 중간평가 직전 |
| `planning` | target-user, swm-reviewer, tech-lead | 3 | 기획서 초안, 핵심 기능 범위 결정 |
| `tech` | tech-lead, swm-reviewer | 2 | 스택·아키텍처 결정 |
| `portfolio` | peer-competitor, tech-lead | 2 | 수료 직전 README·포트폴리오 정비 |

### 리포트 확인

생성된 리포트는 `projects/outputs/<project>/{YYYY-MM-DD-HHmm}-{mode}-report.md`에 저장.

구조:
- **한 줄 요약** (취업 임팩트 관점)
- **정(正)**: 각 에이전트의 **원문 전체** (축약 없음, 톤 보존)
- **반(反)**: 에이전트 간 충돌 지점 (3개 프리셋 충돌축 체크)
- **합(合)**: 취업 임팩트 기준 종합 권고 + 면접 카드 2~3장 (백엔드/프론트 루트 분리)
- **팀원별 취업 레버리지 점검**: 각자 자기 지분으로 면접에서 말할 수 있는가
- **다음 회의 전까지 팀이 답해야 할 질문**

## 사용법 — `/draft-proposal` (17기 기획서 초안)

### 호출 형태

```
/draft-proposal <project> [--resume <ts>]
```

- `<project>` (필수): `projects/inputs/` 아래 디렉터리. 예: `debate-zip`.
- `--resume <ts>` (선택): 이전 실행의 타임스탬프(`YYYY-MM-DD-HHmm`)로 이어서 진행.

### 진행 방식

1. `projects/inputs/<project>/` 의 RAW(`.md`/`.html`/`.txt`/이미지 메타)를 시간순(파일명 오름차순)으로 합쳐서 읽는다.
2. `projects/forms/02-main.md` 양식 골격을 기반으로 **11개 섹션 묶음을 인터리브로** 진행:
   - 각 섹션마다 (a) RAW 기반 초안 제시 → (b) [`questions-bank.md`](.claude/skills/draft-proposal/questions-bank.md) 후보에서 1~3개 핵심 질문 → (c) 답변 반영 → (d) 즉시 부분 저장.
   - "팀 논의 안 됨/판단 불가/모름" 류 답변은 본문 직후 `<!-- ⚠️ 추가 논의 필요 -->` 블록을 인라인으로 누적하고 placeholder로 본문을 채운 뒤 **멈추지 않고** 다음 섹션 진행.
3. 본문 11개 섹션이 끝나면 1p 요약(`01-summary.md`)을 본문 압축으로 자동 도출.
4. 시각 자료 권장 섹션(시스템 구성도·AI 데이터 플로우·일정 등)에서는 **Mermaid 초안 + placeholder 제작 가이드**를 함께 본문에 삽입. 사용자가 "Mermaid로 충분"이면 가이드는 추가논의로 이동, "직접 그릴게"면 Mermaid 삭제 후 `[디자인 결정]` 누적. **최종 PDF 제출 전에 Mermaid 코드는 렌더링한 PNG로 교체**해야 함이 마무리에서 안내된다.

### 산출물 위치

```
projects/outputs/<project>/{YYYY-MM-DD-HHmm}-proposal/
├── 02-main.md       # 10p 본문 (각 섹션 + 인라인 추가논의)
└── 01-summary.md    # 1p 요약 (말미에 종합 추가논의 한 블록)
```

### 외부 후기 반영 체크 (`projects/forms/`)

`projects/forms/02-main-checklist.md` 의 "🌐 5단계: 외부 후기 반영 추가 체크" 에는 SWM 11·13·15·16기 합격 후기에서 자주 지적된 항목을 정리해두었다.

- 사용자 인터뷰·검증 트레이스 (영상 또는 인용 카드)
- 차별점 한 문장 진술 ("경쟁사를 죽이는 이유")
- AI 활용의 필연성 ("AI 떼면 동일 서비스가 만들어지는가")
- 멘토 보완점 코멘트 반영 흔적 (전부 칭찬은 형식적)
- MVP 데모 가능 시점(보통 8~9월) 일정표 한 셀 명시
- 발표 PPT 첫 페이지와 본 기획서 / 요약본 톤 일관성
- 팀 3인 역할 중복 검사 (백/프/AI 분리)

`draft-proposal` 스킬은 매 섹션 진입 시 이 체크리스트를 자가검증에 사용한다.

### 예시

```
/draft-proposal debate-zip                              # 새 실행
/draft-proposal debate-zip --resume 2026-05-14-1830     # 이어서 진행
```

## 페르소나 (자동 처리)

`target-user-persona`는 **하드코딩된 페르소나를 쓰지 않는다**. 대신:

1. **입력에 페르소나가 명시되어 있으면** → 그것을 구체적 1인으로 인스턴스화
2. **명시되어 있지 않으면** → 기획·핵심 기능에서 가장 그럴듯한 1인을 **추론**해 생성 (추론 근거 명시)

리포트 상단에 "사용한 페르소나 카드"가 항상 박히므로, 추론 결과가 실제 타겟과 다르면 그 사실을 보고 다음 번 입력 파일에 페르소나를 명시하면 된다.

## 디렉터리 구조

```
.
├── CLAUDE.md                            # 전역 규칙
├── README.md                            # 이 파일
├── .gitignore                           # projects/outputs/ 제외
├── .claude/
│   ├── agents/
│   │   ├── tech-lead-mentor.md
│   │   ├── target-user-persona.md
│   │   ├── swm-reviewer.md
│   │   └── peer-competitor.md
│   └── skills/
│       ├── analyze-meeting/             # 4페르소나 분석
│       │   ├── SKILL.md                 # 진입점
│       │   ├── invocation.md            # 호출 형태·모드
│       │   ├── execution.md             # 실행 절차 6단계
│       │   ├── output-format.md         # 리포트 포맷·마스킹
│       │   └── constraints.md           # 에러·금지 사항
│       ├── synthesis-framework/         # 정반합 + 취업 임팩트 규약
│       │   └── SKILL.md
│       └── draft-proposal/              # 17기 기획서 초안 작성기
│           ├── SKILL.md                 # 진입점 (파일 지도)
│           ├── invocation.md            # 호출 형태·재개 인자
│           ├── execution.md             # 11개 섹션 인터리브 절차
│           ├── output-format.md         # 인라인 추가논의 / Mermaid → PNG 안내
│           ├── questions-bank.md        # 섹션별 질문 후보 (외부 후기 ⭐)
│           ├── image-suggestions.md     # 섹션별 Mermaid 패턴 + placeholder
│           └── constraints.md
└── projects/
    ├── forms/                           # 17기 양식·체크리스트 (비공식 작업본)
    │   ├── 02-main.md                   # 본문 양식 (10p)
    │   ├── 02-main-checklist.md         # 본문 체크 + 외부 후기 5단계
    │   ├── 01-summary.md                # 요약 양식 (1p)
    │   └── 01-summary-checklist.md      # 요약 체크
    ├── inputs/
    │   └── sample-project/
    │       └── sample-meeting.md        # 의도적 문제점 심어둠
    └── outputs/
        └── sample-project/              # gitignored
```

## 동작 원칙 (요약)

- **병렬 호출**: 서브에이전트 간 의견 오염 방지.
- **원문 보존**: 정(正) 섹션에 에이전트 응답 전문을 그대로 붙인다(축약 금지).
- **톤 보존**: swm-reviewer가 드라이했으면 리포트도 드라이하다.
- **충돌 명시**: 반(反) 섹션에서 숨김없이 드러낸다.
- **취업 임팩트 기준**: 합(合)의 모든 판정은 "2026 신입 취업 성공"에 정렬. 백엔드/프론트 루트 면접 카드 분리.
- **멀티 프로젝트**: `projects/inputs/<name>/` · `projects/outputs/<name>/` 단위로 분리. 자동 생성 금지.
- **개인정보 마스킹**: 최종 리포트에서 이름/연락처/이메일을 마스킹.

## 첫 테스트

### `/analyze-meeting` 시도

```
/analyze-meeting sample-project
```

샘플 회의록에는 의도적으로 여러 문제점을 심어뒀다. 하네스가 다음을 잡아내는지 확인:

- [ ] 기능 9개 스코프 크리프 (tech-lead-mentor)
- [ ] 팀 전원 신규 스택 러닝 함정 (tech-lead-mentor)
- [ ] "AI 기반 개인 맞춤 X 플랫폼" 양산형 한 줄 (peer-competitor)
- [ ] "편리하고 혁신적" 추상 차별점 (swm-reviewer)
- [ ] 10배 법칙 미충족 (target-user-persona)
- [ ] **페르소나 자동 추론** (입력에 타겟 유저가 불명확할 경우 target-user-persona가 기획에서 추론하는가)
- [ ] **백엔드/프론트 면접 카드 분리** (합 섹션에서)
- [ ] **팀원별 취업 레버리지 점검 표** (합 섹션에서)
- [ ] 개인정보 마스킹 (`김OO`, `박OO` 등)

### `/draft-proposal` 시도

실제 회의 자료가 누적된 프로젝트에서:

```
/draft-proposal debate-zip
```

다음을 확인:

- [ ] `projects/inputs/debate-zip/` 의 자료를 시간순으로 읽었는가
- [ ] 11개 섹션 묶음을 한 섹션씩 인터리브로 진행하는가 (한 번에 토해내지 않음)
- [ ] 매 섹션 진입 시 외부 후기 5단계 체크가 자가검증에 사용되는가
- [ ] 시스템 구성도 섹션에서 **Mermaid 초안 + placeholder**가 둘 다 본문에 들어왔는가
- [ ] "팀 논의 안 됨"으로 답했을 때 인라인 `<!-- ⚠️ 추가 논의 필요 -->` 블록이 그 자리에 생기는가
- [ ] 부분 저장이 동작해서 중단 후 `--resume <ts>` 로 이어지는가
- [ ] 최종 산출물이 `projects/outputs/debate-zip/<ts>-proposal/` 에 `02-main.md` + `01-summary.md` 둘 다 생기는가
- [ ] 마무리 메시지에서 "Mermaid 코드는 PNG로 교체" 안내가 나오는가

## 튜닝 팁

- 한 번에 완벽한 페르소나가 나오긴 어렵다. **2~3회 반복 튜닝**을 예상할 것.
- 출력이 마음에 안 들면 안 맞는 부분을 지목해서 해당 에이전트 파일(`.claude/agents/*.md`)만 수정 요청하면 된다.
- `swm-reviewer`와 `peer-competitor`는 둘 다 드라이해지지 않도록 톤을 구분시켜 둘 것.
- 종합 리포트의 "합(合)"이 모호해지면 `synthesis-framework/SKILL.md`의 4원칙과 품질 체크리스트를 구체화한다.
- **페르소나 추론이 계속 빗나가면**, 프로젝트 input에 "## 타겟 유저" 섹션을 명시적으로 넣어 준다.
- **팀원별 레버리지 점검이 공란이면**, input에 팀원 역할·지원 루트(백/프)를 써둔다.
