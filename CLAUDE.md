# CLAUDE.md — practice.html 개발 가이드

세종대 HICS Lab 학부 IC 설계 수업용 web SPICE 실습 사이트.
새 세션에서 이 파일을 먼저 읽으면 전체 구조와 함정을 빠르게 파악할 수 있다.

---

## 1. 한 줄 요약

- **단일 파일** [practice.html](practice.html) (≈140 KB) 에 HTML + CSS + JS 가 전부 인라인.
- 브라우저에서 ngspice WASM 을 Web Worker 로 돌려 `.tran` / `.dc` / `.ac` / `.noise` 4개 분석 수행.
- ETRI 0.5μm BSIM3v3 PDK 가 JS 상수 `ETRI_LIB` 로 임베드되어 매 시뮬마다 `/model.lib` 로 가상 FS 에 마운트됨.
- 외부 라이브러리는 CDN (CodeMirror 5, Chart.js 4, chartjs-plugin-zoom) — 빌드 단계 없음, 그냥 파일 열거나 GitHub Pages 로 서빙.
- 배포: **https://hics-lab.github.io/icdesign/practice.html** (repo: `hics-lab/icdesign`, main 브랜치)

---

## 2. 파일 구조

```
OPAMP_design/                      ← 작업 디렉토리 (repo root 이름과 다름)
├── practice.html                  ★ 실습 사이트 본체 — 거의 모든 작업이 여기서
├── ngspice.js                     ngspice WASM 로더 (Emscripten 산출물)
├── ngspice.wasm                   ngspice WebAssembly 바이너리
├── README.md                      간단 소개 + 사용법 + 라이센스
├── LICENSE                        MIT — Sejong Univ. HICS Lab (+ 3rd party 고지)
├── PDK_차이.md                    ETRI 원본 PDK vs 임베드 모델 차이 (7개 파라미터 제거, 누설 전류 함정)
├── opamp_design.html              ★ 별개 페이지 — 5T OTA 설계 실습 (손계산 vs SPICE). §11 참고
├── etri_mos.lib                   ETRI MOS 모델 텍스트 (참고용, 사이트는 임베드 사용)
├── Model_Parameter/               ETRI 원본 PDK 파일 (참고용)
│   └── 05cmos_model_260210.lib    원본 모델 파일
├── NSPL_05CMOS_PDK_260224.zip     ETRI 배포 zip (참고용)
├── 5T_OTA/                        5T OTA 실습 자료 (사이트와 무관)
└── .claude/                       테스트 / 헬퍼 스크립트들 (Node.js)
    ├── ngspice_runner.js          ngspice WASM 을 Node 에서 직접 돌리는 헬퍼
    ├── test_etri.js               갤러리 전 회로 자동 검증
    ├── test_*.js / debug_*.js     특정 회로 / 이슈 디버깅용
    └── settings.json              Claude Code 권한 설정
```

**핵심 원칙: practice.html 한 파일에 다 넣는다.**
빌드 단계도 모듈 분리도 안 했다. 학생이 파일 하나 받아도 돌아가야 함.

---

## 3. 로컬에서 열기 / 디버깅

브라우저로 그냥 열어도 WASM 로딩이 fetch 로 가능하지만 일부 브라우저는 `file://` 에서 WASM 을 막음. 권장:

```powershell
# 어떤 정적 서버든 OK
npx http-server -p 8000
# 또는
python -m http.server 8000
```

그리고 `http://localhost:8000/practice.html` 접속.

Node 에서 ngspice WASM 만 돌려보고 싶을 때 → `.claude/ngspice_runner.js` 참고.

---

## 4. practice.html 내부 구조 (라인 번호는 변동 가능, Grep 으로 함수명 검색 추천)

### 4.1 CSS / HTML (≈ 1~600 라인)
- CSS 변수: `--primary`, `--secondary`, `--text`, `--muted`, `--border`
- Grid 레이아웃: 좌측 (netlist + log) / 우측 (results + tabs)
- Netlist 영역: `.nl-tabbar` + CodeMirror 컨테이너 `.editor-wrap`
- Results 영역: `.tabbar` (tran/dc/ac/noise/log) → 각 `#cc-<kind>` 안에 view 들
- Modal: `#help-modal`, `#gallery-modal` — backdrop 클릭으로 닫힘

### 4.2 ETRI PDK 임베드 ([practice.html:637](practice.html#L637))
```js
const ETRI_LIB = `... 약 230 라인 .MODEL ...`;
```
원본 PDK 에서 **NMOS/PMOS 만 7개 파라미터 제거** (HSPICE ACM 전용 + ngspice WASM 이 거부하는 FC):
`RSH, RD, RS, ACM, LDIF, HDIF, FC`
S/D 접합 누설 `JS`, `JSW` 는 원본 값 그대로 넣었고, 원본 `N` 은 BSIM3 이름 `NJ` 로 바꿔 넣었다 (2026-11).
예전처럼 빼 두면 ngspice 기본값 JS = 1e-4 A/m², JSW = 0 이 쓰여 접합 누설이 원본보다 70~140배 크게 나왔다.
모델 파라미터 값 변경은 0개. 자세한 내용은 [PDK_차이.md](PDK_차이.md).

**RES (`p2res`) 는 식 하나를 고쳤다**: 원본 `.param dT='(TEMP - tnom)'` 은 ngspice 가 `.param` 에서
`TEMP` 를 몰라 `Formula() error` → fatal. 그래서 `.lib "/model.lib" RES` 를 쓰는 netlist 가 전부 죽었다.
→ `rbody` 식 안에서 ngspice 온도 변수 `temper` 로 직접 계산 (계수·식 동일, 1350 Ω / 77 °C 1385.9 Ω 손계산 일치).

PDK 소자 사용법(인스턴스 문법·함정)은 도움말 `🧩 PDK 소자` 탭(`data-tab="pdk"`) 과 갤러리 6번에 있다.
함정 요약: 다이오드는 **W/L 필수**(생략 시 면적 1 m² → 0.6 V 에서 683 A), 다이오드 RS 는 면적에 반비례
(10×10 µm ≈ 16 kΩ), PNP 컬렉터 = 기판 = GND, 저항은 `X` 로 호출(subckt), 커패시터는 값 대신 W/L.

### 4.3 갤러리 ([practice.html:869~](practice.html#L869))
`GALLERY = [{cat, title, tag, stars, desc, netlist, tab?, run?, sweep?}, ...]`, 카테고리 `GALLERY_CATEGORIES`.
- `intern` (연구실인턴 실습) — `tab`/`run`/`sweep` 프리셋으로 누르면 바로 실행
- `tutorial` (SPICE 튜토리얼) 6개:
  1. 저항 분배기 (.dc)
  2. RC step 응답 (.tran)
  3. RC 주파수 응답 (.ac)
  4. NMOS Id–Vgs (.dc)
  5. NMOS 증폭기 종합 (.dc + .ac + .noise)
  6. 마이칩 PDK 소자 종합 (.dc + .ac) — MOS/DIODE/BJT/RES/CAP 전 섹션 사용

### 4.4 전역 상태 ([practice.html:1032~](practice.html#L1032))
```js
chartViews    = {tran: [], dc: [], ac: [], noise: []}   // 각 탭의 서브뷰들
activeViewIdx = {tran: 0, ...}                          // 각 탭의 활성 서브뷰
varSelections = {tran: {}, ...}                         // 변수 체크박스 상태 (재실행 후에도 유지)
lastResults   = {tran: null, ...}                       // 마지막 시뮬 결과 parsed
currentWorker = null                                    // 실행 중 Web Worker
activeTab     = 'tran'                                  // 현재 결과 탭

// Netlist 멀티 탭
netlistTabs    = [{id, title, content, userTitle?}, ...]
activeNetTab   = 0
nextNetTabId   = 2
```

### 4.5 Netlist 탭 시스템 ([practice.html:1052~1245](practice.html#L1052))
- `deriveTabTitle(content, n)` — 첫 `* ...` 라인에서 22자 추출
- `renderNetlistTabs()` — 탭바 다시 그리기, 더블클릭 → `startTabRename`
- `switchNetlistTab(idx)` — 활성 변경 시 현재 cm 내용 저장 후 새 탭 setValue
- `_updateActiveTabTitle()` — netlist 첫 줄 바뀌면 자동 재명명 (단 `userTitle === true` 면 스킵)
- `newNetlistTab(content?)` / `closeNetlistTab(idx)`

### 4.6 state persistence ([practice.html:1304~](practice.html#L1304))
- `STATE_VERSION = 2` — `netlistTabs` 배열로 저장 (v1 은 단일 `netlist` string)
- `serializeState()` / `applyState(state)` — v1 → v2 마이그레이션 포함
- `saveToLocalStorage()` — debounce 700 ms
- `flushSave()` — 즉시 저장
- `localStorage` key: `'practice-state'`
- export/import: JSON 파일 다운로드/업로드

### 4.7 분석 라인 동기화 ([practice.html:1633~](practice.html#L1633))
- `detectAnalyses(src)` — netlist 에서 `.tran/.dc/.ac/.noise` 존재 여부
- `buildAnalysisLine(kind)` — 파라미터 입력란 → 라인 문자열
- `writeAnalysisLineToNetlist(kind, silent)` — Run 직전 자동으로 netlist 갱신
- `readParamsFromNetlist(kind)` — netlist 의 분석 라인을 파싱해 입력란 복원
- `refreshDetected()` / `refreshAllParams()` — netlist 변경 시 호출됨

### 4.8 시뮬레이션 실행 ([practice.html:1978~](practice.html#L1978))
핵심 흐름:
```
runKind(kind)
  → writeAnalysisLineToNetlist(kind, true)        // 입력란 → netlist
  → buildSingleAnalysisNetlist(src, kind)         // 나머지 분석 라인 제거 + wrapper 삽입
  → Web Worker 생성 → importScripts('ngspice.js')
  → Module.preRun: FS.writeFile('/sim.cir', netlist), FS.writeFile('/model.lib', ETRI_LIB)
  → Module.arguments = ['-b', '/sim.cir']
  → Module.postRun: FS.readFile('/<kind>.txt') → postMessage('done', data)
  → handleResult(kind, data, dt)
```
- Timeout: 30 초 (worker terminate)
- `stopSim()` 으로 수동 중단

### 4.9 wrapper / control 블록 ([practice.html:1782~](practice.html#L1782))
사용자 netlist 에 `.control...endc` 가 있으면 **제거** 한 뒤 자동 wrapper 삽입:
```spice
.control
set wr_vecnames
set wr_singlescale
run
echo "==NGSP_CURPLOT_BEGIN=="
echo $curplot
echo "==NGSP_CURPLOT_END=="
* noise:
setplot noise1
wrdata /noise.txt inoise_spectrum onoise_spectrum
* else:
echo "==NGSP_VECTORS_BEGIN=="
display
echo "==NGSP_VECTORS_END=="
wrdata /<kind>.txt all
quit
.endc
```
- `NGSP_CURPLOT` 마커 → 분석 실패 감지 (값이 `constants` 면 실패)
- `NGSP_VECTORS` 마커 → 변수 타입 (voltage/current/...) 추출

### 4.10 결과 파싱 ([practice.html:1851~](practice.html#L1851))
- `parseWrdataWithHeader(txt, complexMode)` — singlescale + vecnames 형식 파서
- `buildRealParsed(txt, vecInfo, xfb)` — tran/dc/noise 용
- `buildAcParsed(txt, vecInfo)` — AC (complex → mag dB, phase deg)
- `parseRealWrdata(txt)` — noise 전용 (xName 자동 frequency)
- `parseVectorNamesFromLog(log)` — display 출력에서 `name : type, real|complex` 추출
- `annotateTypesByName(vars, vecInfo)` — voltage/current 정보 매핑

### 4.11 결과 렌더링 (View 시스템) ([practice.html:2285~](practice.html#L2285))
- `chartViews[tab]` = view 배열, 각 view = `{id, kind, chart, chart2?, element, varList, ...}`
- `view.kind`:
  - `'V'` (전압), `'I'` (전류) — tran/dc/noise/일반용
  - `'bode'` — AC 전용. mag(상) + phase(하) 2개 chart, x축 동기화
  - `'noise'` — Noise 전용. spectrum chart 위 + integration panel 아래
- `renderResults(tab)` — tab 의 view 들 전부 새로 만듦
- `addView(tab, kind, isDuplicate?)` — view 추가
- `buildViewChart(tab, view, canvas, canvas2)` — Chart.js 인스턴스
- `buildViewVarList(tab, view, listEl)` — 우측 변수 체크박스 리스트
- `varSelections[tab][varName]` 으로 체크 상태 재실행 후에도 유지
- 그래프 조작: 휠 zoom 은 chartjs-plugin-zoom. **왼쪽 드래그 pan(`setupChartMousePan`)·오른쪽 드래그 영역 zoom·터치(`setupChartTouch`)는 직접 구현**
  — plugin 의 pan 은 Hammer.js 가 있어야 하는데 Hammer 는 터치까지 가로채므로 싣지 않음 (`pan.enabled:false`).
  pan/터치는 `_touchAdjustAxis` 로 원래 데이터 범위 안에서만 움직이고, Bode 위/아래는 `_syncBodeSibling` 으로 x 축 맞춤.

### 4.12 Noise 적분 ([practice.html:2691~](practice.html#L2691))
- `integrateNoiseBand(rows, varIdx, f1, f2)` — 로그-로그 보간 + 사다리꼴 적분
  - 입력 spectrum 단위: **V/√Hz** (ngspice noise 출력)
  - 적분: ∫S²df → √(...) = RMS V (검증: 4kT·R 이론과 일치)
- `parseSiNum(s)` — SPICE 식 숫자 파싱 (Meg/k/m/u/n/p/f), Hz/Ω/ohm/s 단위 무시
- `spiceNumFormat(v, sig)` — Meg/u (engFormat 의 M/μ 와 다름 — SPICE 호환)
- `buildNoiseIntegratePanel(view)` — f1/f2 input + 결과 span
- `fillNoiseIntegrationDefaults(view)` — 첫 렌더 시 fmin/fmax 자동 채움
- `recalcNoiseIntegration(view)` — input/change/keyup/paste 모든 이벤트에 리스닝, 입력값 절대 덮어쓰지 않음

### 4.13 에러 감지 / stale view 정리 ([practice.html:2129~](practice.html#L2129))
- `detectNgspiceErrors(stdout, stderr)` — `error:`, `not in circuit`, `singular matrix`, `no convergence`, `unknown parameter`, `no such vector/parameter/device`, `timestep too small`, `simulation aborted` 등 패턴 매치
- `detectCurplotFailure(log)` — `$curplot` 값이 `constants` 면 분석 미실행
- `clearStaleResults(kind)` — `lastResults[kind] = null`, `clearViews(kind)`, empty panel 복귀
- `handleResult` 가 실패 감지 시 위 함수 호출

---

## 5. SPICE / ngspice 관련 함정 (정확히 알고 있어야 함)

### 5.1 단위 prefix — SPICE 와 SI 가 다르다
SPICE 는 case-insensitive 라서 다음을 **반드시** 알아야 함:
- `1M` = **1 milli** (1e-3) — SI 의 mega 가 아님
- `1Meg` = 1 mega (1e6)
- `1u` = `1U` = 1 micro
- `1m` = `1M` = 1 milli
- `1k` = `1K` = 1 kilo

`engFormat` 은 SI 의 M (mega) 를 쓰므로 SPICE netlist 에 넣을 값은 `spiceNumFormat` 으로 포맷해야 함.
`parseSiNum` 은 양쪽 다 받음 (대소문자 무관, `meg` 우선 매치).

### 5.2 ETRI PDK 의 빠진 파라미터들이 시뮬에 미치는 영향
- **`AD/PD/AS/PS` 안 적으면 junction cap = 0** → 출력단 cap 이 거의 안 잡힘
- **`KF`, `AF` 없음** → flicker (1/f) 노이즈 모델링 안 됨, noise 가 평탄(white-only)
- **`FC` 없음** → reverse-bias junction cap 만 계산 (forward-bias 영역 보정 X), 일반 동작에는 영향 없음
- 동작점 / 채널 전류 / 게이트 cap (CGDO/CGSO/CGBO) 은 정확
- **접합 누설**은 JS/JSW/NJ 를 넣어 원본과 같다. 단 `AD/PD/AS/PS` 를 안 적으면 접합마다 1e-14 A 고정이고,
  사이트는 `.options` 를 안 주므로 기본 gmin 1e-12 S 가 접합마다 1 pA/V 를 더한다 → fA~nA 급 회로는 둘 다 챙겨야 함
  (PDK_차이.md "누설 전류 함정")

→ 학생이 "왜 노이즈가 평탄해?" / "왜 cap 이 영향이 없어?" 물어보면 위 두 가지 확인:
1. Cload 명시했는지
2. AD/PD/AS/PS 명시했는지

### 5.3 ngspice 의 silent failure — `const` plot
`.dc Vfake 0 5 0.1` 처럼 존재하지 않는 소스를 쓰면 `run` 이 조용히 실패하고 current plot 이 built-in 상수 plot 으로 남음. `wrdata all` 은 그 plot 의 상수 (`boltz`, `echarge`, `c`, `planck`, `pi`, `e`, `TRUE`, `FALSE`, `kelvin`, `i`, `yes`, `no`) 를 dump.
→ **함정**: `echo $curplot` 이 출력하는 plot **이름은 `const`** 다 (`display` 의 타입 표기는 `constants` 지만 이름은 짧게 `const`). 그래서 `detectCurplotFailure` 정규식은 `/^const/i` 여야 한다 (`/^constants/` 로 쓰면 못 잡음 — 실제로 이 버그가 있었음).
→ 2중 방어: `detectCurplotFailure` (curplot 이름 검사) + `looksLikeConstantsDump(headers)` (파싱된 신호명이 전부 상수 집합이면 실패). 둘 중 하나라도 걸리면 실패 처리 + view 클리어.

### 5.4 wrdata 의 복잡한 컬럼 매핑
- `set wr_singlescale` 켜야 x 가 한 번만 나옴 (안 켜면 각 변수마다 x 반복)
- `set wr_vecnames` 켜야 첫 줄에 변수명 (안 켜면 컬럼이 익명)
- AC (complex) 는 각 변수가 **2 컬럼** (real, imag), 헤더에 변수명이 2번 반복됨
- `wrdata all` 은 default-scale 변수 (x축) 도 포함하므로 파싱 시 `xName == varName` 제거

### 5.5 Noise 의 plot 전환
`.noise` 실행 후 ngspice 는 **plot 2개** 생성:
- `noise1` — frequency 별 spectrum (inoise_spectrum, onoise_spectrum) ← 우리가 원하는 것
- `noise2` — 적분된 total (inoise_total, onoise_total) ← run 직후 current plot

→ wrapper 에 `setplot noise1` 필수.

### 5.6 ngspice noise 출력 단위
- voltage source `Vin` 으로 noise sim → output: **V/√Hz** (V²/Hz 가 아님, 검증됨)
- 적분 시 S² 로 제곱해서 V²/Hz 만든 뒤 적분 → √(...) → Vrms
- voltage divider 1k+1k, T=300K 로 검증: 2.88 nV/√Hz @ 1 kHz = √(4kT·500Ω) ✓

---

## 6. 자주 하는 작업 패턴

### 6.1 새 분석 종류 추가 (예: noise2 / pz / sens)
1. HTML 탭 추가 (`.tabbar`, `#cc-<kind>`, `#empty-<kind>`)
2. 파라미터 입력 row 추가
3. `KIND_LABEL`, `lastResults`, `chartViews`, `activeViewIdx`, `varSelections` 에 키 추가
4. `detectAnalyses` 정규식 보강
5. `buildAnalysisLine` / `readParamsFromNetlist` 보강
6. `buildWrapper` 분기 추가
7. `handleResult` 분기 추가 (특수 파서 필요 시)
8. `addView` 에서 새 view kind 처리

### 6.2 갤러리 회로 추가/수정
[practice.html:869~](practice.html#L869) 의 `GALLERY` 배열 수정.
각 요소: `{cat: 'tutorial'|'circuit', title, tag, stars, desc, netlist}`.
**중요**: netlist 는 `.lib "/model.lib" MOS` 줄을 포함해야 ETRI 모델 사용 가능.

### 6.3 도움말 추가
`#help-modal` 안의 사이드바 탭 (`.help-side a`) + 컨텐츠 영역 (`.help-content section`) 짝지어 추가.
Credits / 라이센스 정보는 마지막 탭 (`#help-credits`).

### 6.4 차트 색 / dash 패턴 바꾸기
[practice.html:2253~](practice.html#L2253) 의 `COLORS`, `DASH_PATTERNS`.
- 같은 view 안에서 trace 가 많을 때 색 + dash 조합으로 구분
- `getColor(k)`, `getDash(k)` 사용

### 6.5 테스트 (Node)
`.claude/test_etri.js` 를 실행하면 갤러리 전 회로의 모든 해석 케이스 자동 검증 (2026-11 기준 14 케이스).
GALLERY 가 참조하는 `INTERN_*` 공용 netlist 상수는 스크립트가 자동으로 추출해 넣는다.
```powershell
node .claude/test_etri.js
```
새 회로 추가 시 이 스크립트도 갱신.

---

## 7. 라이센스 / 출처 (이미 정리됨, 건드릴 일 적음)

- **사이트 코드**: MIT License (© 2026 Sejong University HICS Lab)
  - [LICENSE](LICENSE) 파일, README, practice.html `<script>` 헤더, 헤더 subtitle, help 모달 Credits 탭 — 5곳에 표시
- **ngspice**: BSD-3-Clause
- **ETRI 0.5μm PDK**: 모아팹(MoaFab) myChip 공지사항 공개 배포자료
- **CodeMirror 5 / Chart.js / chartjs-plugin-zoom**: 모두 MIT

ngspice.wasm 의 자체 빌드/공개 출처 의문은 — 이 사이트에서 직접 호스팅 (`/ngspice.wasm`). BSD-3 의 의무 (source 가용성) 는 `npm @holdenvision/ngspice` 등 공개 빌드 경로가 있고, 우리 repo 의 wasm 파일은 그 패키지의 것과 동일.

---

## 8. 작업 시 주의

1. **practice.html 외 파일은 거의 안 건드림** — UI/시뮬 로직은 전부 거기 있음.
2. **CSS 변경 시 grid 레이아웃 깨지지 않게** — `.left/.right`, `.editor-wrap`, `#cc-<kind>` 의 flex/grid 의존성 큼.
3. **CodeMirror 인스턴스는 1개** — 탭 전환 시 `cm.setValue()` 로 내용 갈아치움. 무한 change 이벤트 방지하려 `_suspendCmChange` 플래그 사용.
4. **state 변경 후 `flushSave()` 호출** — debounce 가 늦으면 즉시 저장 안 됨.
5. **Web Worker 는 매 Run 마다 새로 생성** — netlist + model 매번 전송. 캐싱 안 함 (worker code 안에 `${baseUrl}` 가 들어가 있어 closure 가 복잡).
6. **localStorage 키는 `'practice-state'` 하나** — version 필드로 마이그레이션.
7. **에러 메시지는 한국어** — 학생용 사이트.
8. **커밋 메시지는 영어** — git log 일관성. 예: `practice: <변경 요약>`
9. **푸시 직전 자동 테스트 없음** — 큰 변경 시 직접 브라우저에서 확인하거나 `.claude/test_etri.js` 돌리기.

---

## 9. 최근 주요 변경 (커밋 로그 요약)

| 커밋 prefix | 내용 |
|---|---|
| `practice:` | 사이트 본체 변경 (대부분) |
| `license:` | 라이센스 / 저작권 표시 변경 |

주요 마일스톤:
- 다중 netlist 탭 + 더블클릭 이름 수정 (`cf015fe`)
- noise 대역 적분 (`c7141b2`, `1ef758b`)
- AC Bode 통합 차트 (`344a34d`)
- 변수 선택 영속 (`597082b`)
- MIT 라이센스 적용 (`6bec2bb`)
- ngspice 에러 감지 + curplot 검사 (`50daed4`, `713d2ed`)

---

## 10. 자주 받는 사용자 요청 패턴

- "X 가 안 보여" → 보통 stale view 또는 데이터 0 rows. `handleResult` 흐름 따라가기.
- "M / Meg 단위 헷갈려" → 5.1 참고. parseSiNum 은 양쪽 다 받지만 SPICE 표기는 `Meg` 권장.
- "노이즈 shaping 이 평탄해" → 5.2 참고. Cload 명시 권유.
- "이상한 신호 (boltz, FALSE 등) 가 결과에 나옴" → 5.3 참고. 분석 실패. wrapper 에서 검사 중.
- "f1 바꿔도 결과 안 변해" → thermal-dominated 면 정상. 자릿수 단위 변경 추천.
- "탭 이름 바꾸고 싶어" → 더블클릭 (이미 구현됨, `cf015fe`).

---

## 11. opamp_design.html — 5T OTA 설계 실습 (2026-11 재작성)

5T OTA 를 사이징하면서 **손계산(제곱법칙)** 과 **SPICE(ngspice + ETRI BSIM3v3)** 를 나란히 비교하는 페이지. 한 파일, 빌드 없음, Chart.js 4.4.0.

- **손계산 엔진** `handCalc(D, H)`: PMOS 입력형은 전압을 VDD−V 로 뒤집고 n↔p 파라미터를 바꿔 NMOS 입력식으로 푼다 (`buildQ`).
  균형 동작점은 꼬리 노드 전압 이분법 (`solveBalancedN`). λ(L) = λ@1µm / L, body effect 선택. 파라미터 `FIT` 는
  ETRI BSIM3 (TT) 를 ngspice 로 피팅한 값 (W=20µm, |VDS|=2.5V, Vov 0.15~0.45V): k'n 115, k'p 38.6 µA/V², VTN 0.72, |VTP| 0.96,
  λn 0.027, λp 0.053 (L=1µm), γn 0.70 (2φF 0.86), γp 0.50 (2φF 0.78).
- **SPICE 엔진** `engineNetlist` → `runWorker` → `analyzeSpice`: ngspice 1회 실행에 (1) 개루프 .op + .ac, (2) 단위이득 버퍼 사본
  DC 스윕(ICMR), (3) 차동 입력 사본 DC 스윕 ±50mV/±1V 를 모두 넣는다 (약 0.1초). 슬라이더 변경 300ms debounce.
- **`ETRI_MOS` 는 practice.html `ETRI_LIB` 의 MOS 섹션을 그대로 복사한 것** — 모델을 고치면 두 파일을 같이 바꿀 것.
  코너(TT/SS/FF/SF/FS)는 `CORNERS` 의 TOX/VTH0/XL/XW 를 텍스트 치환 (`modelLib`). 값 출처: Model_Parameter/05cmos_mosfet_corner.lib.
  코너 파일은 JS/JSW 를 바꾸지 않으므로 접합 누설은 모든 코너에서 TT 값.
- 측정 정의: ICMR = 버퍼에서 |Vout−Vin| ≤ 20mV 구간, 선형 출력 범위 = 차동 전달 기울기 ≥ 최대의 50% 구간, PM = 180° + ∠H(0 dB).
  손계산 ICMR 은 버퍼 비교용으로 입력단 한계 ∩ 출력 한계(M4 포화) ∩ 레일 (`R.icmr`), 교과서식 입력단 한계는 `R.icmrIn`.
- **설계 도우미** `helperCalc`: Allen-Holberg 순서. ICMR 이 허락하는 최대 Vov 대신 `Vov 상한`(기본 0.3V), 그리고 손계산이
  전류·gm 을 10~25% 크게 잡으므로 `설계 여유`(기본 25%). 이 두 가지가 없으면 최소폭 소자가 나와 SPICE 에서 목표 미달.
- **SPICE 실습 페이지로 보내기**: `practice.html#import=<base64url(UTF-8 JSON {netlist, tab})>` — practice.html 의
  `importFromHash()` 가 새 탭으로 열고 주소에서 지운다.
- 이전 버전에서 뺀 것: 페이지 안 JS 로 만든 BSIM3 근사 (실제 ngspice 대비 ISS +70%, GBW ×2.8 오차), 공정 파일 가져오기(UI 없던 코드),
  Vid 스윕의 'V+만 / V−만' 모드. 이전 파일은 git `55a0978` 에 있음.
- **화면 배치** (2026-11 두 번째 개편): `main.app` 3열 = 왼쪽 `#kpi` 핵심 성능 카드 7개(`.kc`, 큰 값 SPICE / 주황 손계산 / `.t` 상태·목표)
  | 가운데 회로도 + 설계 변수 | 오른쪽 그래프 탭(`#ctabs`) + 자세히 탭(`#dtabs`: 진단·성능 비교·동작점·수식·손계산 모델).
  - ≥1100px 이고 높이 ≥600px 이면 한 화면 앱 모드 (body 스크롤 없음, 열마다 따로 스크롤). 높이 ≤800px 이면 슬라이더 2열(`.vcells`)·
    카드 수식 줄 숨김 — 1280×650(1920 노트북 150% 배율)에서도 스크롤 없이 다 보이게 맞춤. 900~1099px 는 [카드 | 한 열], <900px 는 카드가 위쪽 가로 띠.
  - 카드 클릭 → `data-g`(그래프 탭)·`data-d`(자세히 탭)·`data-eq`(수식 `.eq[data-k]` 깜빡임). 회로도 소자 클릭 → 동작점 탭.
    슬라이더 묶음 hover → 회로도 해당 소자 주황 테두리(`ST.hl`). 회로도/동작점 SPICE·손계산 토글은 하나의 상태(`ST.src`, `.src-tg` 두 곳).
  - 탭·토글 상태는 `ota5t-state-v2` 의 `ui` 에 저장. 회로도 SVG 는 W=380, H=340, 범례는 SVG 안 빈 구석.

---

이상. 새 세션 시작 시 이 파일 + [PDK_차이.md](PDK_차이.md) + 최근 `git log` 만 보면 충분.
