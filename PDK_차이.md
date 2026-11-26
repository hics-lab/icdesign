# 마이칩 ETRI 0.5μm PDK vs 시뮬레이션 임베드 모델 비교

본 문서는 [Model_Parameter/05cmos_model_260210.lib](Model_Parameter/05cmos_model_260210.lib)
(원본, ETRI/모아팹 myChip 배포) 와
[practice.html](practice.html) 의 `ETRI_LIB` 상수 (브라우저 ngspice WASM 시뮬레이션에 마운트되는 모델) 의 차이를 정리한 것이다.
[opamp_design.html](opamp_design.html) 의 `ETRI_MOS` 는 `ETRI_LIB` 의 MOS 섹션과 같다.

비교 결과 — 값이 변경된 파라미터는 **단 하나도 없다**. NMOS / PMOS 에서 ngspice 로 옮길 수 없는 7개 파라미터
(HSPICE ACM 전용 6개 + `FC`)만 제외했고, 원본 `N` 은 BSIM3 이름 `NJ` 로 바꿔 넣었다. 나머지는 100% 동일하다.

> **누설 전류 (2026-11 갱신).** 예전 모델은 `JS`/`JSW`/`N` 까지 빼서, 그 자리에 ngspice BSIM3 기본값이 들어가
> 접합 누설이 원본보다 70~140배 크게 나왔다. 지금은 원본 값을 넣어 같은 조건이면 원본 PDK 와 같은 누설이 나온다.
> 그래도 `AD/PD` 를 생략하거나 기본 `gmin` 을 쓰면 생기는 함정은 남아 있다 → [누설 전류 함정](#누설-전류-함정).

검증은 자동 diff (`.MODEL` 블록 단위로 KEY=VALUE 추출 후 비교) 로 수행했다.

---

## 정량적 요약

| 모델 | 종류 | 원본 파라미터 수 | 임베드 | 제거 | 값 변경 |
|------|------|-----------------|--------|------|---------|
| `nmos4` | BSIM3v3 NMOS | 112 | 105 | 7 | **0** |
| `pmos4` | BSIM3v3 PMOS | 112 | 105 | 7 | **0** |
| `pdiode` | P+/Nwell diode | 26 | 26 | 0 | 0 |
| `ndiode` | N+/Pwell diode | 26 | 26 | 0 | 0 |
| `pnp10` | vertical PNP BJT (10×10 μm²) | 39 | 39 | 0 | 0 |
| `pnp5` | vertical PNP BJT (5×5 μm²) | 39 | 39 | 0 | 0 |
| `pnp2` | vertical PNP BJT (2×2 μm²) | 39 | 39 | 0 | 0 |
| `p2res` (.subckt) | POLY2 N+ 저항 | — | — | — | 값 동일, 온도식 표기만 변경 (아래) |
| `pipcap_i` | POLY1/POLY2 캡 | — | — | — | 동일 |

MOS 임베드 105개 중 `NJ` 는 원본 `N` 의 이름만 바꾼 것이다 (값 동일).

### p2res 온도식 변경 (2026-11)

원본:
```spice
.param dT = '(TEMP - tnom)'
rbody n1 mid r='(rsh*l/w) * (1 + tc1*dT + tc2*dT*dT)'
```
ngspice 는 `.param` 식에서 `TEMP` 를 모르므로 `Formula() error` → fatal 로 시뮬이 죽는다
(그 전까지 `.lib "/model.lib" RES` 를 쓰는 netlist 는 전부 실패했음). 임베드 모델은:
```spice
rbody n1 mid r='(rsh*l/w) * (1 + tc1*(temper-tnom) + tc2*(temper-tnom)*(temper-tnom))'
```
`temper` = ngspice 의 소자 식용 온도 변수. rsh/rcont/tc1/tc2/tnom 값과 식 형태는 원본과 동일.
검증: W=2u L=20u → 1350 Ω (=125·10+2·50), ncont=2 → 1300 Ω, `set temp=77` → 1385.9 Ω (손계산 일치).

---

## NMOS / PMOS 에서 제거된 7개 파라미터

```
RSH, RD, RS, ACM, LDIF, HDIF, FC
```

### 그룹 1 — HSPICE 전용 ACM(Area Calculation Method) 그룹 (6개)

| 파라미터 | 의미 | 왜 제거? |
|----------|------|----------|
| `RSH=50/150` | S/D diffusion sheet resistance | HSPICE 의 ACM 모델이 W/L 에서 자동으로 RD/RS 를 계산할 때 사용. ngspice 는 이 자동 계산을 지원하지 않음 |
| `RD=50/150` | drain series resistance | ngspice 는 BSIM3 의 `RDSW` 를 사용해 S/D 저항을 모델링 (그대로 유지) |
| `RS=50/150` | source series resistance | 동상 |
| `ACM=2` | Area Calculation Method 선택 | HSPICE/Spectre 전용 옵션. ngspice 는 인식 못 함 |
| `LDIF` / `HDIF` | 확산 영역 lateral / head 길이 | ACM=2 에서 자동 계산용. ACM 없이는 무의미 |

→ 동작점/소신호 동작은 **`RDSW`** (BSIM3 표준 파라미터, 그대로 살아 있음) 로 동일하게 모델링된다.
- NMOS: `RDSW = 1.19434E3`
- PMOS: `RDSW = 3.60374E3`

### 그룹 2 — 제외하지 않음 (2026-11 다시 넣음): S/D-bulk 접합 누설 파라미터 (3개)

| 원본 | 임베드 | 값 (NMOS / PMOS) | 의미 |
|------|--------|------------------|------|
| `N` | **`NJ`** | 1.013 / 0.999 | junction emission coefficient |
| `JS` | `JS` | 3.6051E-7 / 2.1749E-7 A/m² | bulk junction 면적당 saturation current |
| `JSW` | `JSW` | 6.9937E-13 / 3.1329E-13 A/m | sidewall saturation current |

→ MOS 채널에 붙은 source/drain-bulk 다이오드의 누설 전류를 정한다. 예전에는 ACM 과 함께 뺐었다.
이 세 개는 ngspice 가 **거부하는 파라미터가 아니다**. `JS`, `JSW` 는 BSIM3 표준 이름이고, `N` 만 BSIM3 이름이 `NJ` 다.
빼 두면 ngspice 기본값(JS = 1e-4 A/m², JSW = 0, NJ = 1)이 쓰여 누설이 원본보다 70~140배 커졌다 (아래 표 ⑤).

다시 넣은 뒤 일반 회로 결과는 그대로다:
- `.claude/test_etri.js` 갤러리 14 케이스 OK 14 / FAIL 0. 전후 출력을 숫자로 비교하면 13 케이스는 완전히 같았다.
  '6. 마이칩 PDK 소자 종합' DC 만 가지 전류가 최대 1e-14 A 달라졌다 (그 점의 전류 3.07 µA 대비 3e-9).
- opamp_design.html 5T OTA 기본 설계(TT): 동작점·ISS·이득·Rout·ICMR·출력 범위는 출력 자릿수까지 같다.
  GBW 15.675036 MHz 와 PM 88.3208° 만 상대 1e-11 이하로 달라졌다.

### 그룹 3 — BSIM3 표준이지만 ngspice WASM 미지원 (1개)

| 파라미터 | 의미 | 왜 제거? |
|----------|------|----------|
| `FC=0.999` | forward-bias depletion cap coefficient | BSIM3 공식 파라미터인데 ngspice WASM 이 model parse 단계에서 거부함. MOS junction cap 의 forward-bias 영역 보정 계수 — 회로가 항상 reverse-biased 일반 동작에서는 영향 없음 |

흥미롭게도 같은 `FC=0.999` 가 **다이오드 모델(pdiode/ndiode) 에서는 살아 있다**.
ngspice 가 D 모델에서는 받아들이고 BSIM3 에서는 거부하는 파싱 차이.

---

## 시뮬레이션 영향 분석

| 시뮬레이션 종류 | 차이 |
|-----------------|------|
| `.op` (DC 동작점) | 없음 — 채널 전류는 RDSW 로 동일 |
| `.dc` (Id-Vgs, Id-Vds) | 없음 |
| `.ac` 저주파 (~수 MHz) | 거의 없음 — 게이트/오버랩 cap (CGDO/CGSO/CGBO) 모두 동일 |
| `.ac` 고주파 / `.tran` 빠른 edge | 미세하게 빠를 수 있음 — junction cap 자동 계산이 안 되어 학생이 PD/PS/AD/AS 를 명시하지 않으면 0 으로 처리 |
| `.noise` (열잡음, flicker) | 영향 없음 — KF/AF 같은 1/f 노이즈 파라미터는 원본에도 없음, 열잡음은 RDSW 로 계산 |
| 누설 전류 (fA~nA 급: 오프 누설, 유지 시간, 서브-nA 기준전압 등) | 모델 값은 원본과 같음. 단 AD/PD 를 꼭 주고 gmin 을 낮춰야 맞게 나옴 → [누설 전류 함정](#누설-전류-함정) |

### 학생이 `M1 d g 0 0 nmos4 W=20u L=1u` 식으로 작성하는 경우

- 동일하게 동작
- 단, `Warning: Pd = 0 is less than W` 경고가 ngspice 로그에 나옴 (무해 — junction cap 이 0 으로 모델링됨)
- 정확도 향상이 필요하면:
  ```spice
  M1 d g 0 0 nmos4 W=20u L=1u AD=10p AS=10p PD=21u PS=21u
  ```
  - `AS = AD = W × Ldiff` (Ldiff 는 0.4~0.6 μm)
  - `PS = PD = 2×Ldiff + W`
  - 이렇게 하면 junction cap 과 junction 누설이 모두 원본 PDK 값으로 계산된다.

---

## 누설 전류 함정

모델 파라미터가 맞아도 아래 두 가지는 시뮬레이터 쪽에서 생긴다. 세 번째는 2026-11 이전 모델에만 있던 문제다.

| 항목 | ngspice 가 하는 일 | 결과 |
|------|------|------|
| AD=AS=PD=PS=0 (생략) | 접합마다 포화전류를 **1e-14 A 로 고정**. JS 값·온도와 무관 — 원본 모델을 넣어도 같다 | 누설이 엉뚱한 값 |
| `gmin` (실습 페이지는 `.options` 를 안 줘서 1e-12 S) | 각 접합에 병렬로 붙음 | 1 V 역바이어스에서 접합당 1 pA. 상온 실제 누설(0.02 fA 수준)을 완전히 덮는다 |
| (2026-11 이전) `JS`/`JSW`/`N` 생략 | 기본값 JS = 1e-4 A/m² (원본의 280배 / 460배), JSW = 0, NJ = 1 | 누설 70~140배 과대. **지금은 해결** |

### 시뮬레이션 확인 (2026-11, 사이트와 같은 ngspice.wasm)

NMOS/PMOS W/L = 10/10 µm, TT. 게이트·소스·드레인을 묶어 채널 전류를 0 으로 두고, S/D 와 벌크 사이에 1 V 역바이어스를 건 뒤
벌크 전류(두 접합 합)를 읽었다. 기하를 줄 때는 AD = AS = 15p, PD = PS = 23u.
넷리스트: `..\누설기준전압_마이칩검증\sim\js_check\` — `practice_model_new.lib` 은 지금의 `ETRI_LIB`,
`practice_model_old.lib` 은 2026-11 이전 `ETRI_LIB` 를 그대로 뽑은 것.

| 조건 | NMOS 27 °C | NMOS 100 °C | PMOS 27 °C | PMOS 100 °C |
|------|-----------|------------|-----------|------------|
| ① 지금 모델, AD/PD 생략, 옵션 없음 (사이트에서 그냥 돌린 경우) | 2.02 pA | 2.02 pA | 2.02 pA | 2.02 pA |
| ② 지금 모델, AD/PD 생략, gmin = 1e-21 | 20 fA | 20 fA | 20 fA | 20 fA |
| ③ 지금 모델, AD/PD 명시, 옵션 없음 | 2.00 pA | 2.65 pA | 2.00 pA | 2.36 pA |
| ④ **지금 모델, AD/PD 명시, gmin = 1e-21** (= 원본 PDK) | 0.043 fA | 0.65 pA | 0.021 fA | 0.36 pA |
| ⑤ 예전 모델 (JS/JSW/N 없음), AD/PD 명시, gmin = 1e-21 | 3.0 fA | 51 pA | 3.0 fA | 51 pA |
| ⑤ ÷ ④ | 70배 | 79배 | 143배 | 142배 |

- ①은 gmin 전류(1e-12 S × 1 V × 2) 가 거의 전부라 온도에 따라 변하지도 않는다. 실제 누설이 아니다.
- ②의 20 fA 는 1e-14 A × 2 접합이다. 원본 PDK 모델로 AD/PD 를 생략해도 똑같이 20 fA 가 나왔다.
  값이 모델·온도와 무관하게 정확히 1e-14 A 라서 BSIM3 코드에 박힌 고정값으로 보인다 (ngspice 소스는 직접 확인하지 않았다).
- ③: AD/PD 를 줘도 기본 gmin 이면 gmin 전류(2 pA)가 실제 누설을 덮는다. 100 °C 의 실제 누설 0.65 pA 도 2 pA 에 묻힌다.
- ④는 원본 JS/JSW/NJ 를 살린 PDK 모델(`..\누설기준전압_마이칩검증\sim\mos_TT_j.lib`)과 7자리까지 같다 (NMOS 27 °C 4.298832e-17 A).
  손계산: (3.6051e-7 × 15e-12 + 6.9937e-13 × 23e-6) × 2 = 4.3e-17 A.
- ⑤ ÷ ④ 가 JS 비(280/460배)보다 작은 것은 기본 JSW = 0 이라 sidewall 몫이 빠졌기 때문이다. 손계산: 1e-4 × 15e-12 × 2 = 3.0e-15 A.

### 영향 범위

- **영향 없음**: 증폭기·OTA·전류 미러 등 바이어스가 µA 급인 일반 실습. 위 값은 pA 이하라 상대 오차 1e-6 이하.
- **주의가 필요함**: 서브-nA 기준전압/전류원, 스위치 오프 누설, 커패시터 유지 시간(샘플-홀드, 다이내믹 노드), 고온 누설 해석.

### 누설 회로를 시뮬레이션해야 할 때

1. 모든 MOS 에 AD/AS/PD/PS 를 준다 (생략하면 1e-14 A 고정).
2. `.options gmin=1e-21` 이하, `abstol` 은 회로 전류보다 충분히 작게 주고, 결과마다 KCL 을 확인한다.
   실습 페이지도 netlist 의 `.options` 줄은 그대로 ngspice 에 넘긴다 (wrapper 는 `.control` 과 분석 줄만 바꿈).
3. 회로 전류가 약 1 fA 아래면 Newton 이 틀린 해를 '수렴' 으로 보고할 수 있다.
   실제 사례와 우회 방법: `..\누설기준전압_마이칩검증\보고서.md` §2 (ISSCC'19 192 pW 기준전압 재현 검증).
4. 원본 PDK 에도 없는 것: n-well/p-기판 접합 누설 모델, Monte Carlo 모델, 코너별 JS 변화. 모델이 fA 이하에서 맞는지도 PDK 문서로는 확인되지 않는다.

---

## 검증

2026-11 JS/JSW/NJ 를 다시 넣은 뒤: [.claude/test_etri.js](.claude/test_etri.js) 갤러리 14 케이스 **OK: 14, FAIL: 0**.
ngspice 경고는 넣기 전과 같다 (Pd/Ps = 0, 소스 DC 값 없음 — 모두 원래 있던 것, unknown parameter 없음).

처음 임베드할 때의 검증 — 갤러리 5개 회로의 7가지 시뮬레이션 케이스(.dc, .tran, .ac, .noise) 를 실제 ngspice WASM 으로 돌려 모두 정상 결과를 확인했다 — **OK: 7, FAIL: 0**.

검증된 회로:
1. 저항 분배기 (.dc)
2. RC step 응답 (.tran)
3. RC 주파수 응답 (.ac)
4. NMOS Id–Vgs (.dc, MOSFET 모델 사용)
5. NMOS 증폭기 종합 (.dc + .ac + .noise)

또한 voltage divider 의 열잡음을 ngspice WASM 으로 측정한 결과 (1 kΩ 두 개 직렬, R₁∥R₂=500Ω, T=300 K):

| 항목 | 값 |
|------|-----|
| ngspice 출력 | 2.88E-9 V/√Hz @ 1 kHz |
| 이론값 √(4kT × 500Ω) | 2.88 nV/√Hz |

→ 정확히 일치. 모델이 실제 칩 동작을 정확히 반영함을 의미한다.

---

## 결론

마이칩 PDK 와 본 페이지에 임베드된 모델은 **모든 BSIM3 핵심 파라미터** (VTH0, U0, K1, RDSW, CGDO/CGSO/CGBO, CJ/CJSW, JS/JSW, ...) **와 모든 다이오드/BJT/저항/캡 모델이 100% 동일**하며, 단지 ngspice 가 처리 못 하는 HSPICE ACM 확장 + `FC` 만 빠져 있다. 학부 교육용 시뮬레이션 결과는 마이칩 풀 PDK 를 쓴 것과 사실상 차이가 없다.
누설 전류를 다루는 회로는 AD/PD 와 gmin 을 챙겨야 한다 ([누설 전류 함정](#누설-전류-함정)).
