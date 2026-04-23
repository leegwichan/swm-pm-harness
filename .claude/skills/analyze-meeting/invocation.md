# invocation — 호출 형태와 모드

`analyze-meeting` skill의 사용자 입력 파싱 규약.

## 호출 형태

```
/analyze-meeting <project> [mode] [input-path]
```

### 인자

- `<project>` (필수): `projects/inputs/` 아래 디렉터리 이름. 예: `stitch`, `sample-project`.
  - 인자 자체를 생략하면 `Glob`으로 `projects/inputs/*/`를 조회해 디렉터리 목록을 보여주고 "어떤 프로젝트를 분석할지" 되묻는다. 임의로 선택하지 않는다.
- `mode` (선택, 기본값 `full`): 아래 모드 표 참조.
- `input-path` (선택): 절대 경로 또는 레포 루트 기준 상대 경로. 생략 시 `projects/inputs/<project>/` 내 가장 최근 수정된 `.md` 파일을 자동 선택.

### 예시

```
/analyze-meeting sample-project                                         # full + 최근 파일
/analyze-meeting sample-project full                                    # 동일
/analyze-meeting sample-project planning                                # planning 모드
/analyze-meeting stitch tech projects/inputs/stitch/stack-v2.md         # 특정 파일 지정
/analyze-meeting                                                        # 프로젝트 목록 안내 후 중단
```

## 모드 정의

현재 등록된 에이전트는 **정확히 4명**이다. 새 에이전트가 추가되면 `full`의 정의와 아래 모드 표를 함께 갱신해야 한다.

### 등록된 에이전트 (4명)

1. `tech-lead-mentor`
2. `target-user-persona`
3. `swm-reviewer`
4. `peer-competitor`

### 모드 표

| mode | 호출할 에이전트 | 인원 | 언제 쓰는가 |
| --- | --- | --- | --- |
| `full` (기본) | tech-lead-mentor, target-user-persona, swm-reviewer, peer-competitor | 4 | 주요 의사결정 회의, 중간평가 직전 |
| `planning` | target-user-persona, swm-reviewer, tech-lead-mentor | 3 | 기획서 초안, 핵심 기능 범위 결정 |
| `tech` | tech-lead-mentor, swm-reviewer | 2 | 스택·아키텍처 결정 |
| `portfolio` | peer-competitor, tech-lead-mentor | 2 | 수료 직전 README·포트폴리오 정비 |

## 검증

호출이 들어오면 다음을 확인하고, 실패 시 안내 후 **즉시 중단**한다.

1. `<project>` 인자가 비어있다 → 프로젝트 목록 안내 후 중단.
2. `projects/inputs/<project>/` 디렉터리가 존재하지 않는다 → 다음 안내 후 중단.
   ```
   프로젝트 디렉터리가 없습니다. 다음 명령으로 생성 후 다시 시도하세요:
   mkdir -p projects/inputs/<project> projects/outputs/<project>
   ```
3. `mode`가 위 표에 없는 값이다 → 지원 모드 목록을 보여주고 중단.
4. `input-path`가 주어졌는데 파일이 없거나 읽을 수 없다 → 경로 확인 안내 후 중단.
5. `input-path` 생략 상태에서 `projects/inputs/<project>/` 내 `.md` 파일이 하나도 없다 → 파일 배치 안내 후 중단.
