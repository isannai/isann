# Docker Desktop 충돌 — iSANN은 WSL 네이티브 docker만 쓴다

> iSANN 노드는 Windows에서 **WSL Ubuntu 안의 네이티브 docker-ce**만 사용한다. Docker Desktop은 **백엔드로도, 폴백으로도 쓰지 않는다.** Docker Desktop이 깔려 있거나 실행 중이면 같은 `/var/run/docker.sock`을 두고 충돌할 수 있다 — 이 문서는 증상·진단·해결을 정리한다.

---

## 왜 네이티브 WSL docker만 쓰나

iSANN의 엔진 컨테이너는 `ivm setup`이 WSL Ubuntu에 설치한 네이티브 docker-ce 위에서 돈다. Docker Desktop은 자체 `docker-desktop` distro 안에서 별도 데몬을 돌리는 **다른 제품**이다. 둘을 섞으면 docker 명령이 어느 데몬으로 갈지 모호해지고, 노드가 엉뚱한 데몬(Desktop)에 컨테이너를 올리거나 "준비됨"을 오판한다.

그래서 iSANN은 **distro·바이너리·소켓을 못박아** 항상 네이티브 dockerd만 겨냥한다:

```
wsl -d <Ubuntu> -- /usr/bin/docker -H unix:///var/run/docker.sock <명령>
```

- `-d <Ubuntu>` — `docker-desktop` distro가 아니라 진짜 Ubuntu distro에서 실행
- `/usr/bin/docker` — Docker Desktop이 WSL에 끼워넣는 **PATH shim** 우회(절대경로)
- `-H unix:///var/run/docker.sock` — Docker Desktop이 잡아둔 **docker context**(`desktop-linux`)를 무시하고 네이티브 dockerd 소켓에 직접 연결

---

## "WSL Integration → Ubuntu 토글"이란 — 켜면 / 끄면

Docker Desktop은 자기 엔진을 **숨겨진 전용 distro(`docker-desktop`)** 에서 돌린다. **Settings(⚙) → Resources → WSL Integration** 의 distro별 토글은, **켜면** Docker Desktop이 *그 distro 안으로 손을 뻗어*:

- `docker` CLI **shim**을 PATH에 깔고,
- docker **context** / `/var/run/docker.sock` 을 **Docker Desktop 엔진**으로 연결한다.

→ 그래서 **Ubuntu 토글이 켜져 있으면**, Ubuntu 안에서 `docker …` 는 네이티브 dockerd가 아니라 **Docker Desktop 엔진**으로 간다. iSANN은 같은 Ubuntu에 깐 **자기 네이티브 docker-ce**(`ivm setup`)를 쓰므로, 토글이 켜져 있으면 Docker Desktop이 같은 distro·소켓 위에 올라앉은 셈이다.

**토글을 끄면(또는 Docker Desktop 종료)** = Docker Desktop이 그 distro에 shim/소켓을 안 끼워넣어, 네이티브 dockerd가 `/var/run/docker.sock` 을 온전히 소유한다.

- iSANN은 런타임에 절대경로(`/usr/bin/docker`)+명시 소켓(`-H unix:///var/run/docker.sock`)으로 shim을 우회하므로 **반드시 꺼야 하는 건 아니다** — 깨끗한 상태일 뿐.
- 단 `ivm check`가 `/var/run/docker.sock is served by Docker Desktop` 을 보고하면(소켓을 실제로 Desktop이 점유 → 네이티브 dockerd 도달 불가) **끄는 게 필수**.
- Docker Desktop 종료는 토글 off와 같은 효과(꺼져 있으면 끼워넣을 게 없음).

---

## Docker Desktop이 가로채는 3가지 지점

Docker Desktop의 **WSL Integration**(distro별 토글)을 켜면 그 distro 안에서 세 군데로 docker 요청을 가로챈다:

| # | 가로채기 | iSANN의 우회 |
|:--|:--|:--|
| ① | **PATH shim** — `docker`가 Desktop CLI로 잡힘 | `/usr/bin/docker` 절대경로 |
| ② | **docker context** — `currentContext=desktop-linux` | `-H unix:///var/run/docker.sock` (context보다 우선) |
| ③ | **socket 점유**(드묾) — `/var/run/docker.sock`을 Desktop이 서빙 | 정체성 가드로 감지 후 거부 |

①②는 위 3중 핀으로 항상 우회된다. ③(소켓 자체를 Desktop이 점유)이 발생하면 iSANN은 **조용히 Desktop으로 새지 않고 명확히 거부**한다(아래 진단 참고).

---

## 진단 — `ivm check`

`ivm check`가 Docker Desktop **설치/실행 여부 + 소켓 점유**를 표시한다(읽기 전용, 아무것도 안 바꿈).

**깔려 있지 않음 (정상):**
```
  Docker Desktop:
    [ -] not installed (no conflict with native WSL docker)
```

**설치됨 · 정지:**
```
  Docker Desktop:
    [ ?] installed, not running
         iSANN uses native docker in WSL, not Docker Desktop.
         Disable its WSL integration for Ubuntu-22.04 so it never takes /var/run/docker.sock.
```

**실행 중 + 소켓 충돌:**
```
  Docker Desktop:
    [!!] running
    [!!] /var/run/docker.sock is served by Docker Desktop, not native docker-ce
         -> Quit Docker Desktop (tray -> Quit), or disable its WSL integration
            for Ubuntu-22.04, then run 'ivm check' again.
```
이때 위쪽 `Docker Engine` 행도 `[OK]`가 아니라 **native 미준비**로 표시된다(소켓이 네이티브 dockerd가 아니므로).

> WSL이 정지 상태면 소켓 점유 여부는 cold-boot 회피를 위해 판정하지 않는다(unknown). 설치/실행만 표시되며, 필요하면 `isann docker warmup` 후 다시 확인한다.

---

## 해결

### 1) `ivm setup` 전 — Docker Desktop을 종료한다

`ivm setup`은 Docker Desktop이 **실행 중이면 설치를 막는다**(설치 중 Desktop integration이 PATH/소켓을 오염시키기 때문):

```
$ ivm setup
  [!!] Docker Desktop is running.
       iSANN uses native docker inside WSL, not Docker Desktop.
       Quit Docker Desktop (tray icon -> Quit), then run 'ivm setup' again.
```

**작업 표시줄(트레이) → Docker 아이콘 우클릭 → Quit Docker Desktop** 후 다시 `ivm setup`.

### 2) 장기 공존 — WSL Integration 끄기 (권장)

Docker Desktop을 계속 쓰고 싶다면 **Ubuntu distro에 대한 WSL Integration만 꺼서** 노드 distro에 손대지 않게 한다:

**Docker Desktop → Settings(⚙) → Resources → WSL Integration** → 사용하는 **Ubuntu** distro 토글 **off** → Apply & Restart.

이렇게 하면 Desktop이 그 distro에 shim/소켓을 끼우지 않아, iSANN의 네이티브 docker와 안전하게 공존한다(둘은 서로 다른 distro·소켓).

### 3) 소켓 충돌이 떠 있으면

`ivm check`가 `/var/run/docker.sock is served by Docker Desktop`을 보고하면:
- **Docker Desktop을 종료**하거나, 위 **WSL Integration을 끄고**,
- `isann docker warmup`으로 네이티브 dockerd를 다시 띄운 뒤,
- `ivm check`로 `Docker Engine [OK] ... (WSL)` 확인.

---

## 동작별 정리

| 동작 | Docker Desktop 상태 | iSANN의 처리 |
|:--|:--|:--|
| `ivm setup` | **실행 중** | 차단 + "종료 후 재시도" 안내 |
| `ivm setup` | 정지 / 미설치 | 정상 — WSL에 네이티브 docker 설치(빠진 것만) |
| `isann docker warmup` | 무관 | 항상 WSL 기동 → 네이티브 dockerd 기동 |
| `isann docker create` / `pull` / `ps` | 무관 | 항상 `/usr/bin/docker -H …` = 네이티브 dockerd |
| `isann model pull` | 무관 | **docker 안 씀** — 모델 가중치를 디스크로 직접 다운로드 |

> **`isann model pull`은 Docker Desktop과 무관하다.** 모델 가중치 다운로드는 fetcher가 URL→디스크로 직접 받는 작업이라 docker를 거치지 않는다. Desktop 충돌은 **엔진 이미지 `docker pull` / `docker create`** 등 실제 docker 명령에만 해당한다.

---

## 검증 — 어느 데몬에 붙었는지 직접 확인

같은 핀으로 `docker info`를 찍어 네이티브인지 확인할 수 있다:

```powershell
wsl -d Ubuntu-22.04 -- /usr/bin/docker -H unix:///var/run/docker.sock info --format "{{.OperatingSystem}} / {{.Name}}"
# 네이티브   -> "Ubuntu 22.04... / <호스트명>"
# Desktop   -> "Docker Desktop / docker-desktop"   (충돌 — 위 해결 적용)
```

---

## 요약 체크리스트

- [ ] `ivm setup` 전에 **Docker Desktop 종료**(트레이 → Quit)
- [ ] 계속 쓰려면 **Ubuntu distro의 WSL Integration off**(Settings → Resources → WSL Integration)
- [ ] `ivm check`에서 `Docker Desktop [ -] not installed` 또는 충돌 없음 확인
- [ ] `ivm check`에서 `Docker Engine [OK] ... (WSL)` = 네이티브 docker-ce 준비됨
- [ ] (충돌 시) Desktop 종료/Integration off → `isann docker warmup` → 재확인

---

← Back to **[README](../README.md)** · **[Q&A](../qna/qna.md)** · **[`ivm` reference](../cli/ivm-reference.md)** · **[`isann` reference](../cli/cli-reference.md)**
