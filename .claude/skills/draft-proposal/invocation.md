# invocation — 호출 형태와 인자

`draft-proposal` skill의 사용자 입력 파싱 규약.

## 호출 형태

```
/draft-proposal <project> [--resume {ts}] [--sections <spec>]
```

### 인자

- `<project>` (필수): `projects/inputs/` 아래 디렉터리 이름. 예: `debate-zip`, `sample-project`.
  - 인자 자체를 생략하면 `Glob`으로 `projects/inputs/*/`를 조회해 디렉터리 목록을 보여주고 "어떤 프로젝트의 기획서를 작성할지" 되묻는다. 임의 선택 금지.
- `--resume {ts}` (선택): `projects/outputs/<project>/{ts}-proposal/` 디렉터리가 이미 있으면 그 작업을 이어서 진행. `{ts}`는 `YYYY-MM-DD-HHmm` 포맷.
  - 생략 시 신규 디렉터리 생성.
  - 이어 작업할 때는 기존 `02-main.md`의 어디까지 채워졌는지 읽어 다음 비어있는 섹션부터 진행한다.
- `--sections <spec>` (선택): 본문 11개 섹션 중 일부만 진행하는 **분담 모드**. 팀원끼리 섹션을 나눠 작업할 때 사용.
  - `<spec>` 형식: 범위(`6-9`), 콤마 리스트(`6,8,9`), 혼합(`6-7,9`), 단일(`9`).
  - 값은 1~11 사이. 섹션 번호는 [`execution.md`](./execution.md) 3절의 11개 섹션 표 기준.
  - 지정 시: (a) 해당 섹션만 인터리브 진행, (b) **요약본(`01-summary.md`) 자동 도출 단계는 건너뜀**, (c) 마무리 메시지에 부분 진행 모드임을 표기.
  - 생략 시: 1~11번 전체 + 요약본 도출 (기본 동작).

### 예시

```
/draft-proposal debate-zip                                           # 신규. 1~11번 전체 + 요약본
/draft-proposal debate-zip --resume 2026-05-14-1830                  # 기존 작업 이어서 (전체)
/draft-proposal debate-zip --sections 6-9                            # 6~9번만 (분담). 요약본 도출 안 함.
/draft-proposal debate-zip --resume 2026-05-14-1830 --sections 9     # 재개 + 9번 묶음만
/draft-proposal                                                       # 프로젝트 목록 안내 후 중단
```

## 입력 소스

- RAW는 항상 `projects/inputs/<project>/**/*.{md,html,txt,csv,png,jpg,jpeg}` 전체를 시간순(파일명 오름차순)으로 묶은 합본.
  - `.csv`는 본문 분석 대상. 텍스트로 그대로 읽어 합본에 포함한다. 내용은 인터뷰/설문 응답일 수도 있고, 시장 데이터·벤치마크 수치·기능 비교표·로그 추출본 등일 수도 있으므로 **파일명·헤더·앞 몇 행으로 성격을 먼저 추론**한 뒤 어느 섹션 근거로 쓸지 결정한다. 성격이 모호하면 그 시점에 사용자에게 확인하고, 추측한 분류를 단정해서 본문에 박지 않는다.
  - 이미지 파일은 본문 분석 대상은 아니지만, "이런 시각 자료가 input에 있음"이라는 메타 정보로 인지하고 시스템 구성도 등에 재활용 가능 여부를 사용자에게 묻는다.
- `analyze-meeting` skill의 결과물(`projects/outputs/<project>/{...}/synthesis.md`)은 이번 skill에서는 **사용하지 않는다**. 사용자 선택대로 inputs만.

## 검증

호출이 들어오면 다음을 확인하고, 실패 시 안내 후 **즉시 중단**한다.

1. `<project>` 인자 누락 → `Glob`으로 `projects/inputs/*/` 디렉터리 목록 출력 후 중단.
2. `projects/inputs/<project>/` 디렉터리 부재 → 다음 안내 후 중단.
   ```
   프로젝트 디렉터리가 없습니다. 다음 명령으로 생성 후 RAW 자료를 넣고 다시 시도하세요:
   mkdir -p projects/inputs/<project>
   ```
3. `projects/inputs/<project>/` 내 분석 가능 파일이 하나도 없다 → 파일 배치 안내 후 중단.
4. `--resume {ts}` 가 지정되었는데 `projects/outputs/<project>/{ts}-proposal/` 디렉터리가 없다 → 다음 안내 후 중단.
   ```
   resume 대상 디렉터리가 없습니다: projects/outputs/<project>/{ts}-proposal/
   `ls projects/outputs/<project>/` 로 후보 타임스탬프를 확인 후 다시 시도하세요.
   ```
5. `--sections <spec>` 파싱 실패(형식 오류, 1~11 범위 벗어남, 빈 spec) → 다음 안내 후 중단.
   ```
   --sections 형식이 올바르지 않습니다.
   허용 형식: 범위(6-9), 콤마 리스트(6,8,9), 혼합(6-7,9), 단일(9). 값은 1~11.
   섹션 번호는 execution.md 3절의 11개 섹션 표 참조.
   ```

## 신규 vs 재개

| 모드 | 동작 |
| --- | --- |
| 신규 | `projects/outputs/<project>/{YYYY-MM-DD-HHmm}-proposal/` 디렉터리를 새로 만들고, 빈 `02-main.md`(양식 골격)과 빈 `01-summary.md`(양식 골격)을 만들어 두고 1번 섹션부터 시작. |
| 재개 | 기존 `02-main.md`를 읽어 마지막으로 확정된 섹션 번호를 식별. 그 다음 섹션부터 진행. 사용자에게 "현재 N번 섹션까지 진행됨. N+1번부터 이어갑니다" 를 한 줄 알린다. |

확정된 섹션의 식별은 `02-main.md` 본문의 마커로 한다 — 자세한 마커 규칙은 [`output-format.md`](./output-format.md) 참조.

> `--sections` 가 함께 지정되면 신규/재개 어느 쪽이든 **지정된 섹션만** 인터리브 진행한다. 지정되지 않은 섹션은 양식 골격 그대로 둔다(신규) 또는 기존 내용 그대로 둔다(재개). 시작 시 사용자에게 "분담 모드: 섹션 {spec} 만 진행합니다. 요약본 도출은 건너뜁니다" 를 한 줄 알린다.
