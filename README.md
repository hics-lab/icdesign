# 🧪 SPICE 실습 페이지

세종대학교 **HICS Lab** 의 학부 IC 설계 수업용 web SPICE 실습 도구.
ngspice WASM + ETRI 0.5μm PDK 를 브라우저에서 바로 시뮬레이션 — 설치 / 서버 / 계정 모두 불필요.

## 🌐 사용

**https://hics-lab.github.io/icdesign/practice.html**

5T OTA 설계 실습 (손계산 vs SPICE 비교, 설계 도우미): **https://hics-lab.github.io/icdesign/opamp_design.html**

## ✨ 특징

- `.tran`, `.dc`, `.ac`, `.noise` 4가지 분석 — 탭별 독립 실행
- 다중 netlist 탭 (한 화면에 여러 회로 동시 관리)
- AC: Bode plot 자동 표시 (mag / phase 위아래로, x축 동기화)
- Noise: 대역 적분으로 RMS noise 자동 계산
- 갤러리: SPICE 튜토리얼 6개 (마이칩 PDK 소자 MOS·다이오드·PNP·저항·커패시터 종합 예제 포함) + 연구실인턴 실습 예제
- 폰 / 태블릿 화면 지원 (편집 · 결과 · 로그 전환, 그래프 두 손가락 확대)
- 자동 저장, JSON export / import, 상세 도움말

## 🖥️ 동작

전부 클라이언트(브라우저) 에서 실행됩니다.
서버로 netlist 가 전송되지 않으며, ngspice 가 WebAssembly 로 컴파일되어 학생의 노트북에서 직접 동작.

## 📚 라이센스

본 페이지 코드 — **MIT License** ([LICENSE](LICENSE) 참고).
저작권 표시만 유지하면 사용 / 수정 / 재배포 자유.

사용된 외부 자산:
- **ngspice** — BSD-3-Clause ([ngspice.sourceforge.io](https://ngspice.sourceforge.io/))
- **CodeMirror 5**, **Chart.js**, **chartjs-plugin-zoom** — MIT
- **ETRI 0.5μm PDK** — ETRI / 모아팹(MoaFab) 공개 자료. 출처: [moafab.kr myChip 공지사항](https://www.moafab.kr/css/board/myChip_notice/read)

## 📬 문의

세종대학교 HICS Lab
