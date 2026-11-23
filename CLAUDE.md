# CLAUDE.md — wtoon 웹툰 제작 프로젝트 규칙

이 저장소는 웹툰 자동 제작 하네스다(에이전트 27개, 스킬 6개). 진입점은 `webtoon-orchestrator` 스킬이고, 제작 절차의 원문은 그 SKILL.md에 있다. 이 파일은 **이 프로젝트(작가님 1인, Claude 프로젝트 "wtoon", 클라우드 실행)에 맞춘 실행 규칙**을 정한다. 하네스 원문과 이 파일이 다르면 이 파일을 따른다.

## 1. 실행 환경

- 모든 작업은 클라우드 환경 **"wtoon 웹툰 하네스"**에서 돈다(OpenAI 접속 허용, codex CLI와 `codex-image`·`codex-cli` 스킬 자동 설치).
- 이 환경에는 `TeamCreate`/`TeamDelete`/팀원 간 `SendMessage`가 없다. 오케스트레이터는 각 단계 팀원을 **Agent 도구로 서브에이전트(`subagent_type`=에이전트 이름, `model: "opus"`)로 차례대로 호출**한다.
  - 에이전트 정의의 "SendMessage로 받는다/보낸다"는 이 환경에서 **"파일로 남기고, 최종 보고에 요지를 적어 오케스트레이터에게 돌려준다"**로 읽는다. 오케스트레이터가 다음 에이전트 프롬프트에 그 파일 경로와 요지를 넣어 전달한다.
  - 서로 의존하지 않는 작업(리서치 4인 조사, 샷리스트+레터링)은 한 번에 여러 Agent 호출로 병렬 실행한다.
- 작가님과는 한국어로, 쉽고 짧게 이야기한다.

## 2. 산출물 저장 위치 (세 군데, 역할이 다르다)

| 위치 | 용도 | 비고 |
|---|---|---|
| 로컬 `_workspace/` (저장소 루트) | 실제 작업 폴더. 하네스의 모든 상대경로가 여기를 가리킨다 | 컨테이너가 지워지면 사라진다. git에 올리지 않는다(.gitignore) |
| `/mnt/project-files/wtoon/_workspace/` | **원본(SSOT).** 스레드끼리 이어받는 공유 폴더 | 모든 스레드가 보고 오래 남는다 |
| 구글 드라이브 `Wtoon` 폴더 (id `1Ke1c-I0gL7JqFJdMIHoKycbty5n38yvp`) | 작가님이 보는 결과물 | 드라이브 커넥터로 올린다 |

동기화 규칙:
- **작업 시작 시** 공유 폴더를 로컬로 내려받는다: `mkdir -p _workspace && cp -a /mnt/project-files/wtoon/_workspace/. _workspace/ 2>/dev/null || true`
- **단계(Phase)가 끝날 때마다** 로컬을 공유 폴더로 올린다: `mkdir -p /mnt/project-files/wtoon/_workspace && cp -a _workspace/. /mnt/project-files/wtoon/_workspace/`
- 공유 폴더는 네트워크 마운트다. 경로 끝에 `/`를 붙이고, 폴더 전체를 검색하지 않는다. 다른 스레드도 쓰므로 고치기 직전에 다시 읽는다.
- codex는 로컬 `_workspace/`에 렌더한다(공유 폴더에 직접 쓰지 않는다). 렌더가 끝나고 검증까지 통과하면 공유 폴더와 드라이브에 올린다.

드라이브 `Wtoon` 폴더 구조:
```
Wtoon/
  00_기획/        trend-brief.md, concept.md, characters.md, series-arc.md (기획 확인용 사본)
  refs/           캐릭터 레퍼런스 시트 PNG + INDEX.md
  ep{NN}/         script_final.md, panel_001.png … (검증 통과본)
  RELEASE/ep{NN}/ index.html, panels/, qa_report.md, RELEASE_NOTES.md
```
- 하위 폴더가 없으면 먼저 만든다. 재렌더한 패널은 같은 이름 파일을 갱신(`update_file`)해 중복을 만들지 않는다.

## 3. 스레드 분담

| 스레드 | 맡는 일 |
|---|---|
| **회차 제작 스레드** (회차마다 하나, 예: "EP01 제작") | Phase 0~3(준비·리서치·시나리오), Phase 4의 글 작업(art-director 스타일 바이블·캐릭터 시트, panel-director 샷리스트, letterer 레터링, prompt-smith 프롬프트), Phase 5(조립·검수·패키징) |
| **이미지 생성 전용 스레드** ("클라우드에서 codex 로그인", 고정) | codex가 필요한 모든 일: ref-sheet-artist 레퍼런스 시트, panel-artist-a/b/c 렌더, panel-validator 검증-재생성 루프(REGEN 때 prompt-smith 보강 포함) |

- codex 로그인은 이미지 스레드 컨테이너에만 있다. **다른 스레드에서 codex를 돌리지 않는다.** 새 이미지 스레드를 만들지도 않는다.
- 넘기는 방법: 회차 제작 스레드가 `ep{NN}_prompts.md`까지 만들고 공유 폴더에 올린 뒤, 이미지 스레드에 "EP{NN} 렌더 준비됨"과 경로를 알린다. 이미지 스레드는 `validation.md`까지 끝내고 공유 폴더·드라이브에 올린 뒤 회차 제작 스레드에 알린다.
- 이미지 스레드 컨테이너가 바뀌어 `codex login status`가 미로그인이면 `codex login --device-auth`를 실행하고 링크와 코드를 작가님께 드린다.

## 4. 제작 절차 요약 (하네스 원문 기준)

| 단계 | 담당 에이전트 | 스킬 | 입력 → 출력 (`_workspace/` 기준) | 통과 기준 |
|---|---|---|---|---|
| P0 컨텍스트 | 오케스트레이터 | orchestrator | `_workspace/` 유무 + 요청 → 실행 모드(초기/다음 화/부분 재실행/새 기획) | — |
| P1 준비 | 오케스트레이터 | orchestrator | 요청 → `00_input/brief.md` | 회차 번호·장르·제약 기록 |
| P2 리서치 | trend-scout, platform-ranker, audience-analyst, hook-analyst(병렬) → trend-synthesizer | trend-research | brief → `01_research/*.md` → `trend-brief.md` | 모든 주장에 출처·관측일 |
| P3 시나리오 | concept-architect → worldbuilder → character-designer → series-plotter → {twist-master, tension-engineer} → episode-outliner → dialogue-writer → script-editor | scenario | trend-brief → `02_story/{concept,world,characters,series-arc,twist-plan,tension-curve}.md` → `03_episode/ep{NN}_{beatsheet,script,script_final}.md` | 매 회차 반전 1개+, 50+ 패널 분량, 대사 위주, 캐릭터 외형 토큰 블록 |
| P4a 아트 디렉션 | art-director | panel-breakdown | characters·script_final → `04_visual/style-bible.md`(장소 토큰 LOC_*, 말풍선 규약), `character-sheets.md` | 스타일 바이블 8섹션, 캐릭터별 불변 토큰+식별 표식 |
| P4b 레퍼런스 시트 | ref-sheet-artist (이미지 스레드) | panel-render | character-sheets → `04_visual/refs/*.png`, `refs/INDEX.md` | 각도 간 동일인, 텍스트 없음, 최대 3회 재렌더. 후속 회차는 재사용 |
| P4c 콘티+레터링 | panel-director, letterer (병렬) | panel-breakdown / assembly | script_final → `ep{NN}_shotlist.md`(50+, scene_id/location), `ep{NN}_lettering.md` | 패널 50+, 모든 패널에 LOC_*, 한글 대사 짧게 |
| P4d 프롬프트 | prompt-smith | panel-render | shotlist·lettering·refs → `ep{NN}_prompts.md`(scene 그룹 A/B/C) | 스타일+장소+캐릭터 토큰+레퍼런스 앵커+말풍선 베이크. `no text` 금지 |
| P4e 렌더 | panel-artist-a/b/c (이미지 스레드) | panel-render | prompts → `05_panels/ep{NN}/panel_NNN.png` | codex 동시 실행 합계 5 이하, 0바이트·손상·md5 중복 없음 |
| P4f 검증 루프 | panel-validator (+prompt-smith, panel-artist) (이미지 스레드) | panel-render | PNG·refs·shotlist·lettering → `ep{NN}_validation.md` | 6축(C1 캐릭터, C2 배경, C3 말풍선·한글·통합 레터링, C4 프롬프트 충실도, C5 대사 흐름, C6 무결성). 패널당 재생성 최대 3회, 넘으면 ACCEPT-FLAG. 레터링 원장 필수 |
| P5 조립·검수 | episode-compositor → quality-reviewer → continuity-manager → showrunner | assembly | 패널 → `06_assembly/ep{NN}/index.html`, `qa_report.md`, `06_assembly/continuity.md`(누적) → `RELEASE/ep{NN}/` | 8항목 PASS/FIX/REDO, 재작업 루프 최대 2회, REDO 있으면 사인오프 금지 |
| P6 마무리 | 오케스트레이터 | orchestrator | — → 작가님께 요약(제목·로그라인·반전·패널 수·뷰어·다음 화 시드) | 피드백 요청 |

## 5. 작가님 확인 지점 (여기서는 멈추고 승인을 받는다)

하네스 원문은 끝까지 자동으로 달리지만, 이 프로젝트는 비용이 큰 단계 앞에서 작가님 확인을 받는다. 확인 요청은 짧게, 선택지가 있으면 결정 카드로 한다. 승인 전에는 다음 단계를 시작하지 않는다.

1. **기획 확인** — P3에서 concept·characters·series-arc가 나오면(대본 쓰기 전). 보여 줄 것: 로그라인, 주요 캐릭터 3~5명 한 줄씩, 시리즈 엔진, 이번 화 반전 한 줄. 후속 회차는 생략.
2. **대본 확인** — `ep{NN}_script_final.md`가 나오면(비주얼 들어가기 전). 보여 줄 것: 회차 줄거리 5줄, 반전·클리프행어, 예상 패널 수.
3. **레퍼런스 시트 확인** — `refs/`가 나오면(패널 렌더 전). 드라이브 `Wtoon/refs/` 링크. 여기서 확정된 외형이 시리즈 전체 기준이 된다. 후속 회차는 신규 캐릭터만.
4. **완성본 확인** — `RELEASE/ep{NN}/`가 나오면. 드라이브 링크와 ACCEPT-FLAG 패널 목록.

작가님이 "확인 없이 끝까지"라고 하면 그 회차는 1~3을 건너뛴다.

## 6. 꼭 지킬 규칙

- codex 동시 세션은 **전체 합계 5개 이하**.
- 모든 텍스트(말풍선·효과음·화면 문자)는 이미지에 그린다(in-image 베이크). HTML 오버레이·후작업 금지. 레퍼런스 시트만 텍스트 없이.
- 일관성 토큰·레퍼런스·장소 토큰이 빠진 프롬프트는 렌더하지 않는다.
- 후속 회차는 `02_story/`, `style-bible.md`, `character-sheets.md`, `refs/`, `continuity.md`를 재사용한다(다시 만들지 않는다).
- 부분 재실행은 해당 단계와 영향받는 하위 단계만 다시 한다. 합격한 산출물은 보존한다.
- 같은 종류의 피드백이 2번 이상 나오면 스킬/에이전트 정의 개선을 제안하고, 바꾸면 아래 변경 이력에 적는다.

## 변경 이력

- 2026-11-23: 프로젝트 실행 규칙 추가(클라우드 실행, 서브에이전트 순차 호출, 저장 위치 3곳, 스레드 분담, 작가님 확인 지점 4곳). 렌더 스킬의 macOS 경로·md5 명령을 리눅스용으로 수정, 드라이브 업로드 절차 추가.
