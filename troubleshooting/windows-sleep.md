# Windows 절전 모드 막기 — 노드 상시 가동

> 노드(`isannd` + provider/consumer)는 **항상 깨어 있어야** 메시에서 동작한다. Windows가 절전(대기)·최대 절전으로 들어가면 노드는 사실상 오프라인이 된다. 이 문서는 절전을 끄는 방법을 정리한다.

---

## 왜 막아야 하나

노드는 RV(랑데뷰)에 **5초마다 punch keepalive**를 보내 자기 공인 UDP NAT 매핑을 유지한다. 시스템이 절전에 들어가면:

- keepalive가 끊김 → RV가 노드를 **stale**로 판정(cutoff 약 60초) → 다른 노드가 나를 **못 찾거나 hole-punch에 실패**한다.
- 깨어나도 매핑이 만료됐으면 **재학습 + 재펀치**까지 갭이 생겨 첫 요청이 실패할 수 있다.
- provider 노드면 그 사이 들어온 추론 요청을 통째로 놓친다.

> **화면만 꺼지는 것(monitor off)은 괜찮다** — 시스템은 깨어 있다. 막아야 할 건 **절전(sleep/standby)** 과 **최대 절전(hibernate)** 이다.

---

## 빠른 적용 (powercfg — 권장)

**관리자 PowerShell**에서 한 번에:

```powershell
# 전원 연결(AC) 상태에서 절대 잠들지 않게 — 0 = "안 함(Never)"
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0

# 노트북: 배터리(DC)에서도 막으려면 함께
powercfg /change standby-timeout-dc 0
powercfg /change hibernate-timeout-dc 0

# 화면은 꺼도 무방(전력 절약) — 15분 후 화면만 off, 시스템은 계속 가동
powercfg /change monitor-timeout-ac 15
```

`powercfg`는 GUI보다 확실하고 스크립트화하기 좋다. `0`은 해당 타임아웃 비활성(절대 안 함).

---

## GUI로 하기

**Windows 11**: 설정 → 시스템 → 전원 및 배터리 → **화면 및 절전**
- "전원 연결 시 다음 시간이 지나면 장치를 절전 상태로 전환" → **안 함**

**Windows 10**: 설정 → 시스템 → 전원 및 절전 → **절전**
- "전원 사용 시 다음 시간이 지나면 PC를 절전 상태로 전환" → **안 함**

또는 제어판 → 전원 옵션 → (현재 전원 관리 옵션) **설정 변경** → 고급 전원 관리 옵션 설정 변경 → **절전 → 다음 시간 이후 최대 절전 상태로 전환 → 안 함**.

---

## 노트북 — 덮개 닫기 / 전원 버튼

덮개를 닫으면 기본적으로 절전된다. 전원 연결 시 동작을 **아무 것도 안 함**으로:

제어판 → 전원 옵션 → **덮개를 닫으면 / 전원 단추를 누르면** → (전원 연결) **아무 것도 안 함**

powercfg로도 가능:

```powershell
# 덮개 닫아도 아무 동작 안 하게 (AC) — LIDACTION 0 = Do nothing
powercfg /setacvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
powercfg /setactive SCHEME_CURRENT
```

(LIDACTION 값: `0`=안 함 · `1`=절전 · `2`=최대 절전 · `3`=종료)

---

## ★ 네트워크 어댑터 절전 끄기 (놓치기 쉬움)

시스템이 안 자도 **NIC(랜카드)가 절전되면 노드가 네트워크에서 사라진다.** 노드 운영에서 가장 자주 빠뜨리는 항목이다.

장치 관리자 → **네트워크 어댑터** → (사용 중인 어댑터) 속성 → **전원 관리** 탭 →
- "전원을 절약하기 위해 컴퓨터가 이 장치를 끌 수 있음" **체크 해제**

유선/무선 둘 다 쓰면 양쪽 모두 해제한다.

---

## 적용 확인

```powershell
powercfg /a          # 이 PC가 지원하는 절전 상태 (S3 / Modern Standby 등)
powercfg /requests   # 지금 무엇이 절전을 막고/요청하고 있는지
powercfg /lastwake   # 마지막에 PC를 깨운 원인 (예기치 않은 wake 추적)
```

현재 절전 타임아웃만 확인:

```powershell
powercfg /query SCHEME_CURRENT SUB_SLEEP STANDBYIDLE
# "현재 AC 전원 설정 색인: 0x00000000" 이면 절전 꺼짐
```

---

## (선택) 고성능 전원 계획

추론 노드의 지연/스로틀을 줄이고 싶으면 전원 계획 자체를 고성능으로:

```powershell
# 고성능
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c

# Ultimate Performance (목록에 없으면 먼저 노출시킨 뒤 setactive)
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
```

> 전원 계획만 바꿔도 그 계획의 절전 타임아웃이 0이 아니면 잠들 수 있다 — 위 `standby-timeout` 0 설정과 **함께** 적용하는 게 안전하다.

---

## Modern Standby (S0) 주의

일부 최신 노트북은 S3(대기)를 빼고 **Modern Standby(S0 Low Power Idle)** 만 지원한다 — `powercfg /a` 결과에 "대기(S3)"가 없고 "대기(S0 짧음/김)"만 보이면 그 경우다. 이때도 위의 **standby-timeout 0 + 덮개 동작 없음 + NIC 절전 해제**면 충분히 가동 상태가 유지된다. 화면만 끄고 싶으면 monitor-timeout만 둔다.

---

## 요약 체크리스트

- [ ] `standby-timeout-ac 0` · `hibernate-timeout-ac 0` (powercfg 또는 GUI "안 함")
- [ ] (노트북) 덮개 닫기 동작 = 아무 것도 안 함, 배터리(DC)도 동일 적용
- [ ] **네트워크 어댑터 전원 관리 절전 체크 해제** ← 필수
- [ ] `powercfg /a` · `powercfg /requests`로 확인
- [ ] (선택) 고성능 전원 계획

> 절전을 끈 노드는 전기를 더 쓴다. 상시 가동이 목적인 노드(데스크톱·전원 연결 PC)에만 적용하고, 잠깐 쓰는 PC는 화면만 끄는 쪽을 권장한다.

---

← Back to **[README](../README.md)** · **[Q&A](../qna/qna.md)** · **[Docker Desktop](docker-desktop.md)** · **[Port policy](../reference/ports.md)**
