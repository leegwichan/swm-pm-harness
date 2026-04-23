# SWM PM Harness

SW 마에스트로 연수 프로젝트의 기획·기술 의사결정을 4명의 전문가 페르소나 관점에서 분석하고, **"2026년 신입 개발자 취업 성공"** 기준으로 종합해주는 Claude Code 하네스.

팀 회의 RAW DATA(회의록, 기획서 초안, 기술 스펙 등)를 입력으로 주면:

1. 4명의 서브에이전트가 **각자 독립된 컨텍스트**에서 병렬 분석
2. 메인 Claude가 **정(正)·반(反)·합(合) + 팀원별 취업 레버리지 점검** 구조로 종합
3. 결과를 `projects/outputs/<project>/{타임스탬프}-{mode}-report.md`로 저장

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

## 사용법

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
│       ├── analyze-meeting/             # 여러 파일로 분할된 스킬
│       │   ├── SKILL.md                 # 진입점 (파일 지도)
│       │   ├── invocation.md            # 호출 형태·모드
│       │   ├── execution.md             # 실행 절차 6단계
│       │   ├── output-format.md         # 리포트 포맷·마스킹
│       │   └── constraints.md           # 에러·금지 사항
│       └── synthesis-framework/
│           └── SKILL.md                 # 정반합 + 취업 임팩트 규약
└── projects/
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

## 튜닝 팁

- 한 번에 완벽한 페르소나가 나오긴 어렵다. **2~3회 반복 튜닝**을 예상할 것.
- 출력이 마음에 안 들면 안 맞는 부분을 지목해서 해당 에이전트 파일(`.claude/agents/*.md`)만 수정 요청하면 된다.
- `swm-reviewer`와 `peer-competitor`는 둘 다 드라이해지지 않도록 톤을 구분시켜 둘 것.
- 종합 리포트의 "합(合)"이 모호해지면 `synthesis-framework/SKILL.md`의 4원칙과 품질 체크리스트를 구체화한다.
- **페르소나 추론이 계속 빗나가면**, 프로젝트 input에 "## 타겟 유저" 섹션을 명시적으로 넣어 준다.
- **팀원별 레버리지 점검이 공란이면**, input에 팀원 역할·지원 루트(백/프)를 써둔다.
