# Jetson Orin Nano Super 개발 환경 구축 기록 (NVMe SSD 부팅)

Windows PC만 있는 환경에서 Jetson Orin Nano Super Developer Kit에 **JetPack 6.2.1**을 **NVMe SSD**에 설치하고, CUDA / TensorRT / PyTorch(GPU) 및 VS Code 원격 개발 환경까지 구축한 전체 과정을 기록한 문서입니다.

---

## 목차

0. [환경 및 배경](#0-환경-및-배경)
1. [JetPack 6.2.1 SD 카드 이미지 다운로드](#1-jetpack-621-sd-카드-이미지-다운로드)
2. [SSD를 외장 케이스에 넣고 Etcher로 굽기](#2-ssd를-외장-케이스에-넣고-etcher로-굽기)
3. [SSD 장착 후 첫 부팅](#3-ssd-장착-후-첫-부팅)
4. [트러블슈팅: `ERROR: mmcblk0p1 not found` (비상 셸에서 부팅 설정 수정)](#4-트러블슈팅-error-mmcblk0p1-not-found)
5. [Ubuntu 초기 설정 (oem-config)](#5-ubuntu-초기-설정-oem-config)
6. [SSD 부팅 확인](#6-ssd-부팅-확인)
7. [시스템 업데이트 (apt update / upgrade)](#7-시스템-업데이트)
8. [CUDA / cuDNN / TensorRT 설치 (nvidia-jetpack)](#8-cuda--cudnn--tensorrt-설치)
9. [전원 모드 MAXN SUPER 설정](#9-전원-모드-maxn-super-설정)
10. [PyTorch (GPU) 설치](#10-pytorch-gpu-설치)
11. [트러블슈팅: `libcudss.so.0` 에러 (cuDSS 설치)](#11-트러블슈팅-libcudssso0-에러)
12. [VS Code Remote-SSH 원격 개발 환경](#12-vs-code-remote-ssh-원격-개발-환경)
13. [GPU 동작 테스트](#13-gpu-동작-테스트)
14. [jtop 설치 (모니터링 도구)](#14-jtop-설치)
15. [GUI(데스크톱) 끄기](#15-gui데스크톱-끄기)
16. [팬 프로필 cool 설정 (온도 낮추기)](#16-팬-프로필-cool-설정)
17. [기타 메모](#17-기타-메모)

---

## 0. 환경 및 배경

### 하드웨어 / 환경

| 항목 | 내용 |
|---|---|
| 보드 | NVIDIA Jetson Orin Nano Engineering Reference Developer Kit Super (8GB) |
| 저장장치 | M.2 NVMe SSD 256GB (Key M, PCIe) |
| 호스트 PC | Windows (Ubuntu 호스트 PC 없음) |
| 모니터 | DisplayPort 연결 (Jetson은 DP 출력만 있음, HDMI 없음) |
| 기타 | USB 키보드, USB-NVMe 외장 케이스 (Realtek RTL9210 칩), USB 메모리 |
| 보드 출고 펌웨어 | UEFI **36.4.7** (`36.4.7-gcid-42132812`) |

### 최종 구성

| 항목 | 버전 |
|---|---|
| JetPack | 6.2.1 (L4T 36.4.7, Ubuntu 22.04) |
| 부팅 장치 | NVMe SSD (`/dev/nvme0n1p1`, 234GB) |
| 전원 모드 | MAXN SUPER |
| CUDA | 12.6 |
| cuDNN | 9.3 |
| TensorRT | 10.3.0 |
| PyTorch / torchvision | 2.11.0 / 0.26.0 (GPU) |
| 추가 라이브러리 | cuDSS (`libcudss0-cuda-12`) |

### 왜 JetPack 7.2가 아니라 6.2.1인가

- 2026년 8월 기준 NVIDIA 공식 다운로드 페이지에는 **JetPack 7.2.1 (Jetson Linux r39.2.1)** 통합 ISO만 있고, Orin Nano용 SD 카드 이미지는 더 이상 제공되지 않음.
- ISO(`jetsoninstaller-r39.2.1-...-arm64.iso`)를 USB에 구워 SSD에 설치했으나, **펌웨어 업데이트(capsule update)가 적용되지 않고 36.4.7에 머무름**. 설치 중 `Y` 확인 창도 나타나지 않았음.
- 그 결과 Ubuntu 24.04는 부팅되지만(`localhost login:` 텍스트 로그인까지 진행) **GUI가 검은 화면**으로 나옴. OS(r39)와 펌웨어(r36)의 버전이 맞지 않기 때문.
- NVIDIA 포럼에도 펌웨어 36.4.x 보드에 USB ISO로 7.2를 설치하면 capsule update가 적용되지 않는 사례가 여러 건 보고되어 있음. 해결하려면 Ubuntu 호스트(또는 WSL2)에서 SDK Manager로 리커버리 플래시가 필요함.
- 현재 펌웨어(36.4.7)가 **원래 JetPack 6.2.x용**이라 펌웨어 업데이트 없이 바로 맞고, Ubuntu 22.04 기반이라 YOLO / TensorRT 관련 자료가 많음 → **JetPack 6.2.1 선택**.

> 펌웨어 버전 확인 방법: 전원을 켜고 NVIDIA 로고에서 `ESC` → UEFI 설정 메뉴 상단에 `36.4.7-gcid-...`처럼 표시됨.

---

## 1. JetPack 6.2.1 SD 카드 이미지 다운로드

JetPack 아카이브(<https://developer.nvidia.com/embedded/jetpack-archive>) → JetPack 6.2.2 페이지에 있는 **"Download JetPack 6.2.1 SD card image for Jetson Orin Nano Developer Kit"** 링크에서 다운로드.

```
https://developer.nvidia.com/downloads/embedded/L4T/r36_Release_v4.4/jp62-r1-orin-nano-sd-card-image.zip
```

- 파일 이름: `jp62-r1-orin-nano-sd-card-image.zip`
- **압축을 풀 필요 없음.** Etcher가 zip을 바로 읽음.
- NVIDIA 안내: JetPack 6.x를 쓰는 기기는 6.2.1 SD 카드 이미지로 설치한 뒤 APT로 6.2.2까지 올릴 수 있음.

> 이 이미지는 원래 microSD용이지만, SSD에 직접 구워서 사용함 (4장의 수정 작업이 필요해짐).

---

## 2. SSD를 외장 케이스에 넣고 Etcher로 굽기

### 2-1. 준비

1. Jetson 전원 어댑터를 뽑음.
2. 설치용 USB 메모리가 Jetson에 꽂혀 있다면 뽑음.
3. 보드 아래쪽 SSD 고정 나사를 풀고 SSD를 뺌.
4. SSD를 USB-NVMe 외장 케이스에 넣음.

> SSD 안의 기존 내용은 Etcher가 덮어쓰기 때문에 미리 지울 필요 없음.

### 2-2. Balena Etcher로 굽기

Etcher 다운로드: <https://etcher.balena.io> → "Etcher for Windows (x86|x64) (Installer)"

1. 외장 케이스에 넣은 SSD를 PC에 연결. Windows가 **"포맷하시겠습니까?"** 창을 띄우면 전부 **취소**.
   - 기존 Linux 파티션 때문에 드라이브 문자가 여러 개(E: ~ R:) 생길 수 있음. 정상.
2. 대상 목록이 헷갈리지 않게 **USB 메모리는 PC에서 뽑아 둠**.
3. Etcher 실행 → **Flash from file** → `jp62-r1-orin-nano-sd-card-image.zip` 선택.
4. **Select target** → 대상 선택.

   | 목록에 보인 장치 | 설명 |
   |---|---|
   | `Realtek RTL9210...CSI Disk Device` 256GB | **외장 케이스에 넣은 Jetson SSD → 이것 선택** |
   | `SHGP42-1000GM` 1TB (C:\) | PC 내부 SSD (Windows) → **절대 선택 금지** |
   | `SHGP42-1000GM` 1TB (D:\) | PC 내부 SSD → **절대 선택 금지** |

   - Realtek RTL9210은 USB-NVMe 외장 케이스에 쓰이는 칩 이름.
5. **Select 1** → **Flash!**
6. `WARNING! You are about to erase an unusually large drive` 경고가 뜸.
   - SD 카드보다 용량이 커서 확인하는 것. 대상이 256GB SSD인지 확인 후 **`Yes, I'm sure`** 클릭.
7. 관리자 권한 창 → **예**.
8. 굽기(Flashing)와 검증(Validating)이 끝나고 **"Flash Completed!"** 가 뜨면 완료.
9. 작업 표시줄에서 **"하드웨어 안전하게 제거"** 후 외장 케이스를 뽑음.

---

## 3. SSD 장착 후 첫 부팅

1. Jetson 전원이 **꺼진 상태**에서 SSD를 케이스에서 꺼내 보드에 다시 장착, 나사로 고정.
2. **설치용 USB 메모리가 Jetson에 꽂혀 있지 않은지** 확인 (꽂혀 있으면 JetPack 7.2 설치 프로그램이 뜰 수 있음).
3. **DP 케이블과 키보드를 먼저 연결**하고 모니터 입력을 DP로 맞춤.
   - Jetson은 켜지는 순간 연결된 모니터만 인식하는 경우가 많음.
4. 마지막으로 전원 어댑터를 꽂음 (전원 버튼 없음, 어댑터를 꽂으면 바로 켜짐).

> JetPack 6.2.1은 현재 펌웨어(36.4.7)와 같은 36.x 세대라서 **펌웨어 업데이트 질문이 나오지 않는 것이 정상**.

### 결과

Ubuntu 초기 설정 화면 대신, 커널 로그 끝에 아래 에러가 나오고 **비상 셸(`bash-5.1#`)** 에서 멈춤.

```
ERROR: mmcblk0p1 not found
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.1#
```

---

## 4. 트러블슈팅: `ERROR: mmcblk0p1 not found`

### 원인

- SD 카드용 이미지라서 부팅 설정 파일(`/boot/extlinux/extlinux.conf`)에 **루트 파일시스템 위치가 `root=/dev/mmcblk0p1`(microSD)** 로 적혀 있음.
- 실제 OS는 **NVMe SSD(`/dev/nvme0n1p1`)** 에 있으므로 루트를 찾지 못하고 비상 셸(initrd)에서 멈춤.
- 해결: 설정 파일에서 `mmcblk0p1` → `nvme0n1p1`로 변경. **비상 셸에서 직접 수정 가능.**

> `nvme0n1p1`은 전부 소문자 영어 + 숫자. `nvme` + 숫자 `0` + `n` + 숫자 `1` + `p` + 숫자 `1`.
> (`nvme0` = 첫 번째 NVMe 장치, `n1` = 첫 번째 네임스페이스, `p1` = 첫 번째 파티션)

### 4-1. SSD가 보이는지 확인

```bash
ls /dev/nvme*
```

`No such file or directory` → 비상 셸 단계에서는 **SSD(PCIe/NVMe) 드라이버가 로드되지 않은 상태**.

### 4-2. 드라이버 파일이 있는지 확인

```bash
ls /lib/modules/5.15.148-tegra/kernel/drivers/nvme/host/
ls /lib/modules/5.15.148-tegra/kernel/drivers/pci/controller/dwc/
```

- `nvme-core.ko`, `nvme.ko`, `pcie-tegra194.ko`가 모두 있음 → 직접 로드 가능.

경로를 줄여 쓰기 위한 변수 설정 후 PHY 드라이버도 확인:

```bash
M=/lib/modules/5.15.148-tegra/kernel/drivers
ls $M/phy/tegra/
```

- `phy-tegra194-p2u.ko` 있음 (PCIe 드라이버가 의존하는 PHY 드라이버).

### 4-3. `insmod`가 없을 때 (`command not found`)

```bash
insmod $M/phy/tegra/phy-tegra194-p2u.ko
# bash: insmod: command not found
```

검색 경로를 추가해도 `which insmod` 결과가 없음:

```bash
export PATH=$PATH:/sbin:/usr/sbin:/bin:/usr/bin
which insmod
```

비상 셸에 어떤 명령어가 있는지 확인:

```bash
ls /sbin /bin /usr/sbin
```

- `/bin`에 **`kmod`**, `busybox`, `mount`, `sed`, `umount` 등이 있음.
- `kmod`는 `insmod`의 본체지만, **`kmod insmod ...` 형태로는 동작하지 않음** (`invalid command 'insmod'`).
- `kmod`는 **`insmod`라는 이름으로 호출되어야** 동작함 (도움말의 "kmod also handles gracefully if called from following symlinks" 부분).

→ `insmod`라는 이름의 심볼릭 링크를 만들어 사용:

```bash
ln -s /bin/kmod /tmp/insmod
ls -l /tmp/insmod
```

> 주의: `ls -s`가 아니라 **`ln -s`** (소문자 L + N, 링크 만들기). `/tmp`가 없으면 `mkdir -p /tmp` 먼저.

### 4-4. 드라이버 로드 (순서 중요, 한 줄씩 실행)

PCIe 연결 드라이버를 먼저 올리고, 그 위에서 NVMe 드라이버를 올림.

```bash
/tmp/insmod $M/phy/tegra/phy-tegra194-p2u.ko
/tmp/insmod $M/pci/controller/dwc/pcie-tegra194.ko
/tmp/insmod $M/nvme/host/nvme-core.ko
/tmp/insmod $M/nvme/host/nvme.ko
```

- `pcie-tegra194.ko` 로드 시 커널 메시지가 많이 출력됨. 멈출 때까지 기다리고, 프롬프트가 안 보이면 Enter.
- `Phy link never came up` 메시지는 **SSD를 꽂지 않은 다른 빈 M.2 슬롯**을 검사한 결과라 무시해도 됨.
- `nvme.ko` 로드 후 `nvme0n1: p1 p2 ... p15`가 출력되면 성공.
- `GPT: Primary header thinks Alt. header is not at the end of the disk` 경고는 이미지가 SSD보다 작아서 뒤쪽이 비어 있다는 안내. 부팅에는 무관하며, 이후 초기 설정에서 정리됨.

확인:

```bash
ls /dev/nvme* /sys/block
```

`/dev/nvme0n1p1` ~ `/dev/nvme0n1p15`, `/sys/block/nvme0n1`이 보이면 성공.

### 4-5. 부팅 설정 파일 수정

SSD의 OS 파티션을 마운트:

```bash
mkdir -p /mnt
mount /dev/nvme0n1p1 /mnt
```

`EXT4-fs (nvme0n1p1): mounted filesystem with ordered data mode` 출력되면 성공.

현재 설정 확인:

```bash
grep root= /mnt/boot/extlinux/extlinux.conf
```

```
APPEND ${cbootargs} root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 ...
```

`mmcblk0p1` → `nvme0n1p1`로 변경:

```bash
sed -i 's/mmcblk0p1/nvme0n1p1/g' /mnt/boot/extlinux/extlinux.conf
```

변경 확인:

```bash
grep root= /mnt/boot/extlinux/extlinux.conf
```

```
APPEND ${cbootargs} root=/dev/nvme0n1p1 rw rootwait rootfstype=ext4 ...
```

### 4-6. 저장 후 재부팅

비상 셸에는 `sync` 명령이 없음 (`bash: sync: command not found`) → `busybox sync` 사용.

```bash
busybox sync
umount /mnt
reboot
```

- `umount`가 아무 메시지 없이 끝나면 기록이 완료된 것.
- `reboot`가 안 먹히면 `umount` 성공 후 전원 어댑터를 뽑았다 다시 꽂아도 됨.

### 결과

재부팅 후 **Ubuntu 초기 설정 화면(System Configuration)** 이 정상적으로 나타남.

---

## 5. Ubuntu 초기 설정 (oem-config)

화면 아래 동그라미 8개 = 8단계. 마우스가 없으면 `Tab`으로 이동, `Space`로 체크.

| 단계 | 선택 | 메모 |
|---|---|---|
| 1. 라이선스 | `I accept the terms of these licenses` 체크 → Continue | NVIDIA EULA |
| 2. 언어 | English | 에러 메시지 검색과 개발 도구 호환이 편함. 한글 입력은 나중에 입력기로 추가 |
| 3. 키보드 | Korean (101/104-key compatible) 또는 English (US) | 영문/숫자 배치는 동일. Korean은 한/영·한자 키를 따로 인식. 어느 쪽이든 한글 입력은 입력기(ibus-hangul 등) 설정이 별도로 필요 |
| 4. Wi-Fi | 연결 또는 건너뛰기 | |
| 5. 시간대 | Seoul | |
| 6. 사용자 계정 | 사용자 이름 `orin-nano`, 컴퓨터 이름 `orinnano-desktop` | 비밀번호는 `sudo`에 계속 사용 |
| 7. APP Partition Size | **기본값(최대 크기)** | SSD 전체 사용. GPT 경고도 이 단계에서 정리됨 |
| 8. 브라우저 설치 | 설치 안 함 | snap(Chromium) 기반이라 필요할 때 `sudo snap install chromium` |

### 컴퓨터 이름(hostname)

- 네트워크에서 이 Jetson을 부르는 이름. 터미널 프롬프트(`orin-nano@orinnano-desktop:~$`)와 SSH 접속(`ssh orin-nano@orinnano-desktop.local`)에 사용.
- 영어 소문자, 숫자, 하이픈만 사용.
- 나중에 변경: `sudo hostnamectl set-hostname 새이름`

### 첫 로그인 후 나오는 창

| 창 | 선택 |
|---|---|
| Online Accounts (Google 등 계정 연결) | Skip |
| Privacy - Location Services | 꺼진 상태 그대로 Next |
| 기타 Welcome to Ubuntu 안내 | Next / Done |
| Software Updater (약 1.4GB) | **Remind Me Later** → 터미널에서 직접 업데이트 (7장) |

> Software Updater 창으로 업데이트하지 않은 이유: 커널·부트로더·펌웨어 관련 패키지가 포함되어 있고, 설정 파일 덮어쓰기 질문이 잘 보이지 않을 수 있으며, 직접 수정한 SSD 부팅 설정이 유지되는지 터미널에서 확인하며 진행하기 위함.

---

## 6. SSD 부팅 확인

```bash
df -h /
```

- `Filesystem`이 **`/dev/nvme0n1p1`**, `Size`가 **234G** → SSD에서 부팅됐고 용량도 거의 전부 사용 중.

---

## 7. 시스템 업데이트

### 7-1. 패키지 목록 갱신 (필수)

```bash
sudo apt update
```

- 비밀번호 입력 시 화면에 표시되지 않는 것이 정상.
- `519 packages can be upgraded` 확인.
- 저장소 목록에 `/var/cudnn-local-tegra-repo-ubuntu2204-9.3.0`, `/var/l4t-cuda-tegra-repo-ubuntu2204-12-6-local`, `/var/nv-tensorrt-local-tegra-repo-ubuntu2204-10.3.0-cuda-12.5` 등 **로컬 저장소가 이미 포함**되어 있음 → CUDA/cuDNN/TensorRT를 인터넷에서 다시 받지 않고 설치 가능.

### 7-2. 업그레이드 (권장)

```bash
sudo apt upgrade
```

- `Do you want to continue? [Y/n]` → `Y`
- 10~30분 소요. 진행 중 터미널을 닫지 않음.
- 설정 파일(configuration file) 덮어쓰기 질문이 나오면 기본값 `N`(현재 파일 유지).
- 화면이 절전으로 꺼져도 업데이트는 계속 진행됨.

| 명령 | 하는 일 | 필수 여부 |
|---|---|---|
| `sudo apt update` | 설치 가능한 패키지 목록만 새로 받기 | 필수 (설치 전 항상) |
| `sudo apt upgrade` | 설치된 프로그램을 실제로 업데이트 | 권장 (CUDA/TensorRT 설치 전에 해 두면 순서상 깔끔) |

### 7-3. 재부팅 전 부팅 설정 확인

업데이트 중 커널·부트로더가 바뀌었을 수 있으므로 반드시 확인.

```bash
grep root= /boot/extlinux/extlinux.conf
```

- `root=/dev/nvme0n1p1` 유지됨 → 재부팅.
- (`mmcblk0p1`로 돌아가 있으면 재부팅하지 말고 다시 수정할 것)

```bash
sudo reboot
```

- 재부팅 시 로고 아래 흰색 게이지가 차면 펌웨어 업데이트 중 → 전원을 뽑지 않음.

### 7-4. 버전 확인

```bash
cat /etc/nv_tegra_release
```

- `R36 (release), REVISION: 4.x` 형태.
- jtop 표시 기준: **Jetpack 6.2.1 [L4T 36.4.7]**

### 화면 절전 끄기 (선택)

Ubuntu는 기본적으로 약 5분간 입력이 없으면 화면이 꺼지고 잠김 (작업은 계속 진행됨).

- 설정 앱: Settings → Power → Screen Blank → Never / Privacy → Screen Lock → Automatic Screen Lock 끄기
- 터미널:

```bash
gsettings set org.gnome.desktop.session idle-delay 0
```

(원래대로: `0` 대신 `300`)

---

## 8. CUDA / cuDNN / TensorRT 설치

JetPack SD 이미지에는 Jetson Linux(OS, 드라이버, 커널)만 들어 있고, AI 라이브러리는 별도 설치.

```bash
sudo apt install nvidia-jetpack
```

- `[Y/n]` → `Y`, 10~20분 소요.
- 마지막에 `Setting up nvidia-jetpack (6.2.1+b38)` 출력되면 성공.
- 함께 설치되는 것: CUDA, cuDNN, TensorRT, VPI, OpenCV, Nsight Systems / Graphics 등.

### 설치 확인 (두 명령을 반드시 따로 실행)

```bash
dpkg -l | grep nvidia-jetpack
```

```bash
/usr/local/cuda/bin/nvcc --version
```

- `release 12.6` 확인.

> 주의: `dpkg -l | grep nvidia-jetpack /usr/local/cuda/bin/nvcc --version`처럼 한 줄에 붙여 쓰면 `grep`이 `--version` 옵션을 가져가서 grep 버전이 출력됨.

### CUDA 경로 등록

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
nvcc --version
```

---

## 9. 전원 모드 MAXN SUPER 설정

- GUI: 화면 오른쪽 위 `25W` 클릭 → **MAXN SUPER** 선택 → 재부팅 안내가 나오면 재부팅.
- 확인:

```bash
sudo nvpmodel -q
```

| 모드 | 특징 |
|---|---|
| **MAXN SUPER** | 전력 상한 없음. GPU·CPU·메모리 클럭 최대 |
| 25W (기본값, jtop에서 `D` 표시) | 전력 25W 제한. 발열 적음 |
| 15W / 7W | 배터리 구동 등 전력이 중요할 때 |

- MAXN SUPER를 쓰는 이유: 최대 성능 확인(FPS 측정), TensorRT 변환·테스트 시간 단축, 어댑터 전원 사용 시 전력을 아낄 이유가 적음.
- 단점: 발열·팬 소음 증가, 부하가 크면 전력/온도 때문에 클럭이 자동으로 낮아질(쓰로틀링) 수 있음.
- 개발·측정은 MAXN SUPER, 실제 배포 환경(배터리 등)에 맞춰서는 해당 모드로 다시 측정.

---

## 10. PyTorch (GPU) 설치

`nvidia-jetpack`에는 PyTorch가 포함되어 있지 않음.

> **주의:** 일반 `pip install torch`로 설치하면 ARM용 **CPU 전용** 버전이 설치되어 GPU를 쓸 수 없음. NVIDIA Jetson용 패키지 저장소에서 받아야 함.

### 10-1. pip 설치

```bash
sudo apt install python3-pip
```

### 10-2. Jetson용 PyTorch / torchvision 설치

```bash
pip3 install torch torchvision --index-url https://pypi.jetson-ai-lab.io/jp6/cu126
```

- 다운로드 주소에 `pypi.jetson-ai-lab.io`가 보이면 정상.
- 설치 결과: `torch-2.11.0`, `torchvision-0.26.0`
- `WARNING: The script ... is installed in '/home/orin-nano/.local/bin' which is not on PATH` → 아래로 경로 등록:

```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc && source ~/.bashrc
```

### 10-3. GPU 확인

```bash
python3 -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

→ 이 단계에서 `libcudss.so.0` 에러 발생 (11장).

---

## 11. 트러블슈팅: `libcudss.so.0` 에러

```
ImportError: libcudss.so.0: cannot open shared object file: No such file or directory
```

### 원인

- GPU 버전 PyTorch는 정상 설치됨.
- 최근 Jetson용 PyTorch(2.8 이상)는 **cuDSS**(희소 행렬 연산 라이브러리)를 사용하도록 빌드되어 있으나, JetPack 6.2.1에는 cuDSS가 포함되어 있지 않음.
- NVIDIA 포럼 답변: cuDSS를 별도 설치.

### 11-1. NVIDIA CUDA 저장소 등록

설치 명령은 <https://developer.nvidia.com/cudss-downloads> 에서 `Linux / aarch64-jetson / Ubuntu / 22.04 / deb (network)` 선택 시 표시됨.

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/arm64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
```

### 11-2. CUDA 12용 cuDSS 패키지 확인

```bash
apt-cache search cudss
```

- `cudss`, `cudss-cuda-12`, `libcudss0-cuda-12`, `libcudss0-dev-cuda-12`, `libcudss0-static-cuda-12`, `cudss-cuda-13`, `libcudss0-cuda-13` ... 등이 있음.
- 공식 안내의 `sudo apt-get -y install cudss`는 **CUDA 13용까지 끌어올 수 있으므로 사용하지 않음.**
- PyTorch에 필요한 것은 실행용 라이브러리 `libcudss.so.0` → **`libcudss0-cuda-12`만 설치.**
  - `-dev`(헤더), `-static`(정적 라이브러리)은 C++로 cuDSS를 직접 쓸 때만 필요.

### 11-3. 설치

```bash
sudo apt-get install -y libcudss0-cuda-12
sudo ldconfig
```

- 설치 버전: `libcudss0-cuda-12 0.8.0.10-1`
- 그래도 같은 에러 발생 → 라이브러리가 기본 검색 경로 밖(하위 폴더)에 설치됨.

### 11-4. 라이브러리 경로 등록

위치 찾기:

```bash
find / -name "libcudss.so.0" 2>/dev/null
```

찾은 경로에서 파일 이름을 뺀 **폴더 경로**를 시스템 라이브러리 검색 경로에 등록 (예시는 `/usr/lib/aarch64-linux-gnu/libcudss/12`):

```bash
echo /usr/lib/aarch64-linux-gnu/libcudss/12 | sudo tee /etc/ld.so.conf.d/cudss.conf
sudo ldconfig
```

- `/etc/ld.so.conf.d/`에 경로 파일을 두면 재부팅 후에도 유지되며, `LD_LIBRARY_PATH`를 매번 설정하는 것보다 확실함.

### 11-5. 최종 확인

```bash
python3 -c "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

- `2.11.0` / `True` / `Orin` → **PyTorch에서 GPU 사용 가능.**

TensorRT 확인:

```bash
python3 -c "import tensorrt; print(tensorrt.__version__)"
```

- `10.3.0`

### 주의 사항

- **`pip install torch`를 다시 하지 않기.** ultralytics 등 다른 패키지 설치 시 PyTorch가 CPU 버전으로 바뀔 수 있으므로, 새 패키지 설치 후 `torch.cuda.is_available()`을 다시 확인.
- CUDA 저장소를 추가했으므로, 이후 `sudo apt upgrade` 시 `cuda` 관련 패키지가 대량으로 바뀐다고 나오면 진행 전에 확인.
- 다른 라이브러리에서 `libcusparseLt.so.0` 같은 에러가 나면 cuDSS와 같은 방식(CUDA 12용 패키지 설치 + 경로 확인)으로 해결.

---

## 12. VS Code Remote-SSH 원격 개발 환경

파일을 복사할 필요 없이 PC의 VS Code에서 **Jetson 안의 파일을 직접 열고 수정·실행**하는 방식.

| 방법 | 용도 |
|---|---|
| VS Code Remote-SSH | 평소 개발 (추천) |
| Git (GitHub) | 버전 관리, PC ↔ Jetson 코드 동기화 |
| `scp` | 데이터셋, 모델 파일(.pt) 등 큰 파일 복사 |

### 12-1. Jetson IP / SSH 서버 확인 (Jetson 터미널)

```bash
hostname -I
systemctl status ssh
```

- 첫 번째 주소(`192.168.x.x`)가 공유기에서 받은 IP.
- `active (running)` 확인 후 `q`로 빠져나옴.
- PC와 Jetson이 **같은 공유기(같은 네트워크)** 에 있어야 함. 학교·회사 Wi-Fi는 기기 간 접속을 막는 경우가 있음.

### 12-2. PC에서 SSH 접속 테스트 (Windows PowerShell)

```powershell
ssh orin-nano@<Jetson IP>
```

- 처음 접속 시 `Are you sure you want to continue connecting (yes/no)?` → `yes`
- 비밀번호 입력 → `orin-nano@orinnano-desktop:~$` 나오면 성공.
- 종료: `exit`

### 12-3. VS Code 연결

1. 확장 탭(`Ctrl+Shift+X`) → **Remote - SSH** (Microsoft) 설치.
2. 왼쪽 아래 `><` 아이콘 → **Connect to Host...** → **+ Add New SSH Host...**
3. `ssh orin-nano@<Jetson IP>` 입력.
4. 설정 파일 저장 위치 → 첫 번째 항목 (`C:\Users\사용자이름\.ssh\config`).
5. 다시 `><` → **Connect to Host...** → 추가한 IP 선택.
6. OS → **Linux**, 비밀번호 입력.
7. 처음 연결 시 Jetson에 VS Code 서버가 자동 설치됨 (1~2분). 왼쪽 아래 `SSH: 192.168.x.x` 표시되면 연결 완료.

### 12-4. 작업 폴더 / Python 설정

```bash
mkdir -p ~/projects/test
```

1. **File → Open Folder** → `/home/orin-nano/projects/test`
2. 확장 탭 → **Python** (Microsoft) → **Install in SSH: 192.168.x.x** (PC에 설치되어 있어도 Jetson 쪽에 별도 설치 필요)
3. `Ctrl+Shift+P` → **Python: Select Interpreter** → **`/usr/bin/python3` (3.10)** (PyTorch·TensorRT가 설치된 Python)

### 12-5. 연결 종료

- 연결만 끊기: 왼쪽 아래 `SSH: ...` 클릭 → **Close Remote Connection**
- 창 닫기: X 버튼 또는 File → Exit
- VS Code를 닫아도 Jetson은 켜져 있음. Jetson까지 끄려면 `sudo poweroff`.
- 재접속: `><` → Connect to Host → 저장된 IP, 폴더는 File → Open Recent.

### 12-6. 큰 파일 복사 (PowerShell)

```powershell
scp C:\경로\best.pt orin-nano@<Jetson IP>:~/
```

> IP가 바뀌면 다시 입력해야 하므로 공유기에서 Jetson에 **고정 IP(DHCP 예약)** 를 걸어 두면 편함. 환경에 따라 `orin-nano@orinnano-desktop.local`로도 접속 가능.

---

## 13. GPU 동작 테스트

`~/projects/test/gpu_test.py`

```python
import time
import torch
import tensorrt

print("torch:", torch.__version__, "| tensorrt:", tensorrt.__version__)
print("device:", torch.cuda.get_device_name(0))

x = torch.rand(4096, 4096, device="cuda")

# 첫 실행은 CUDA 초기화 시간이 섞이므로 한 번 돌리고 버림
torch.matmul(x, x)
torch.cuda.synchronize()

# GPU 연산은 비동기로 실행되므로 synchronize로 끝날 때까지 기다린 뒤 시간을 잼
start = time.time()
for _ in range(10):
    torch.matmul(x, x)
torch.cuda.synchronize()
print(f"4096x4096 matmul x10: {time.time() - start:.3f}s")
```

- `synchronize()`가 없으면 GPU에 명령만 보내고 바로 다음 줄로 넘어가서 실제보다 훨씬 빠른 시간이 측정됨.

### 실행

Ubuntu 22.04에는 `python3`만 있고 `python` 명령은 없음.

```bash
cd ~/projects/test
python3 gpu_test.py
```

(`python`으로도 실행하려면 `sudo apt install python-is-python3`)

### 측정 결과와 해석

- torch 2.11.0 / tensorrt 10.3.0 / device Orin / **4096x4096 matmul x10: 1.018s**
- 4096×4096 행렬곱 1회 = 2 × 4096³ ≈ 137 GFLOP → 10회 1.37 TFLOP / 1.018s ≈ **1.35 TFLOPS (FP32)**
- Orin Nano Super FP32 이론치 약 2 TFLOPS (CUDA 코어 1024개 × 약 1GHz × 2) 대비 약 65% → MAXN SUPER 정상 적용.
- FP16으로 바꾸면 Tensor Core를 사용해 훨씬 빨라짐:

```python
x = torch.rand(4096, 4096, device="cuda", dtype=torch.float16)
```

(YOLO를 TensorRT FP16/INT8로 변환하면 빨라지는 이유)

---

## 14. jtop 설치

Jetson 전용 실시간 모니터링 도구 (`jetson-stats`, 오픈소스). GPU/CPU 사용률·클럭, RAM(CPU·GPU 공유 8GB), 온도, 전력, 전원 모드, 팬, 설치된 CUDA/cuDNN/TensorRT/OpenCV 버전 확인 가능.

```bash
sudo pip3 install -U jetson-stats
```

처음 실행 시 서비스가 등록되지 않았다는 안내가 나옴:

```
The jtop.service is not active. Please run:
sudo jtop --install-service
```

```bash
sudo jtop --install-service
sudo reboot
```

- 서비스 등록 + 현재 사용자를 `jtop` 그룹에 추가 (그래야 `sudo` 없이 실행 가능). 그룹 변경은 재로그인이 필요하므로 재부팅.

```bash
jtop
```

- 숫자 키(1~7) 또는 마우스로 탭 이동, `q`로 종료.
- GPU 테스트 코드를 돌리면서 다른 터미널에서 jtop을 켜 두면 GPU 사용률 변화를 직접 확인 가능.
- Jetson 기본 내장 도구 `tegrastats`도 같은 정보를 텍스트로 출력함.

### 구조

| 구성 | 동작 |
|---|---|
| `jtop.service` | 부팅 시 자동 시작, 백그라운드에서 정보 수집 (자원 사용 매우 적음) |
| `jtop` 명령 | 실행할 때만 화면 표시, `q`로 닫아도 서비스는 계속 동작 |

서비스 끄기 / 켜기:

```bash
sudo systemctl disable --now jtop.service
sudo systemctl enable --now jtop.service
```

### 쉬는 상태 화면 읽기 (GUI 켜진 상태)

| 항목 | 값 | 의미 |
|---|---|---|
| CPU 1~6 | 0~6%, 729MHz | 쉬는 중이라 클럭을 낮춤 |
| GPU | 0%, 306MHz | 쉬는 중 |
| Mem | 1.4G / 7.4G | CPU·GPU 공유 메모리 |
| Swp | 0k / 3.7G | |
| Dsk | 23.6G / 233G | |
| 온도 | 약 56°C | 정상 |
| VDD_IN | 5.1W | 보드 전체 전력 |
| FAN | 35%, 2147RPM | quiet 프로필 |

- `cv0`, `cv1`, `cv2 Offline`: 비전 가속기 센서, 사용하지 않을 때 꺼져 있어 정상.
- `Jetson Clocks: inactive`: 클럭이 부하에 따라 자동 조절되는 상태. 추론 속도를 정확히 잴 때만 `sudo jetson_clocks`로 최대 고정 (평소에는 끄기, 켜 두면 쉬는 동안에도 온도가 오름).

---

## 15. GUI(데스크톱) 끄기

VS Code로 원격 작업만 한다면 데스크톱이 필요 없음. 끄면 메모리(약 0.5~1GB)·전력·발열 감소.

```bash
sudo systemctl set-default multi-user.target
sudo reboot
```

- 부팅하면 모니터에 글자 로그인(`orinnano-desktop login:`)이 나옴. 사용자 이름 `orin-nano`, 비밀번호 입력하면 바로 터미널 상태.
- 글자 화면에서는 한글이 깨져 보일 수 있고, 마우스는 사용 불가.
- **SSH / VS Code 원격 접속은 GUI와 무관하게 그대로 동작.**

| 하고 싶은 것 | 명령 | 재부팅 후 |
|---|---|---|
| GUI 기본 끄기 | `sudo systemctl set-default multi-user.target` | 글자 화면 |
| 지금만 GUI 켜기 | `sudo systemctl start gdm` | 다시 글자 화면 |
| GUI 기본 켜기 | `sudo systemctl set-default graphical.target` | GUI 화면 |

- GUI를 끈 뒤 메모리는 확실히 줄었지만 온도는 57°C로 비슷 → 팬(quiet)이 발열이 줄어든 만큼 느려지기 때문 (16장에서 해결).

---

## 16. 팬 프로필 cool 설정

### 배경

- Orin은 CPU와 GPU가 **하나의 칩(SoC)** 에 있어서 cpu/gpu/soc 온도가 거의 같게 나오고, 따로 식힐 수 없음.
- 기본 팬 프로필 **quiet**는 온도가 일정 범위 안이면 팬을 천천히 돌려 소음을 줄이는 방식 → 쉬는 상태 온도를 낮추려면 팬 프로필 변경이 가장 확실.

### jtop에서 설정

- jtop → **CTRL 탭(5번)**
- 전원 모드는 `NVP modes: [-] 2 [+]`처럼 **키보드 `-` / `+`** 로 변경 가능.
- 팬은 `[quiet] [cool] [manual]` 및 `Speed [-] [+]`가 **버튼으로만** 되어 있어 **마우스 클릭 필요**.
  - GUI를 끈 글자 화면에서는 마우스가 안 되므로, `sudo systemctl start gdm`으로 GUI를 잠깐 켜거나 PC의 VS Code 터미널 / PowerShell SSH에서 jtop을 실행해 클릭.
  - `Speed [-] [+]`는 `[manual]` 상태에서만 동작.
- **`[cool]` 클릭.**

### 유지 여부 확인

```bash
grep FAN_DEFAULT_PROFILE /etc/nvfancontrol.conf
```

- `FAN_DEFAULT_PROFILE cool` → 설정 파일에 저장됨, **재부팅해도 유지.**

(jtop 설정이 파일에 저장되지 않는 경우 수동 고정 방법)

```bash
sudo systemctl stop nvfancontrol
sudo sed -i 's/FAN_DEFAULT_PROFILE quiet/FAN_DEFAULT_PROFILE cool/' /etc/nvfancontrol.conf
sudo rm /var/lib/nvfancontrol/status
sudo systemctl start nvfancontrol
```

- `status` 파일에 이전 팬 설정이 저장되어 있어 지우지 않으면 바뀐 설정이 적용되지 않음.

### 결과

- 쉬는 상태 온도 **약 58°C → 51~52°C** (6~7°C 하락).
- 소음이 거슬리면 jtop에서 다시 `[quiet]` 선택.

### 기타 온도 관리

- 케이스 사용 시 팬 위 통풍구가 막히지 않게 둘 것.
- 천 패드보다 딱딱하고 통풍되는 곳에 둘 것.
- 실제로 신경 쓸 시점은 모델 추론 등 부하 상태에서 80°C 이상이 계속 유지될 때.

---

## 17. 기타 메모

### 종료

```bash
sudo poweroff
```

- 화면이 꺼지고 팬이 멈추거나 LED가 꺼진 뒤 전원 어댑터를 뽑음 (OS가 SSD에 있으므로 바로 뽑으면 파일 손상 가능).
- 다시 켤 때는 어댑터를 꽂으면 바로 켜짐.

### 케이스 전원 버튼 연결 (버튼 헤더 J14)

- 팬 아래쪽 12핀 버튼 헤더에 연결. **보드에 인쇄된 글자를 보고 맞추는 것이 가장 정확.**
- 공식 문서로 확인된 핀: **7·8번 = RST(리셋)**, **9·10번 = FC REC(리커버리)** → 평소에는 비워 둘 것.
- 비공식 자료: 1·2번 = PWR BTN / GND, 5·6번 = 자동 전원 켜짐 해제(DIS AUTO ON).

| 케이스 선 | 보드 헤더 | 방향 |
|---|---|---|
| POWER SW | PWR BTN + GND | 상관없음 |
| RESET SW | RST + GND (7·8번) | 상관없음 |
| POWER LED | LED+ / LED- | +/- 맞춰야 함 |

- LED 선과 PWR 선을 바꿔 꽂아 전원이 안 켜졌다는 포럼 사례 있음.
- DIS AUTO ON 점퍼를 끼우지 않으면 어댑터를 꽂을 때 바로 켜지는 동작 유지.
- 케이스 조립은 OS 설치와 정상 부팅을 확인한 뒤에 하는 것이 좋음 (설치 중 SSD 재장착, 리커버리 점퍼 연결이 필요할 수 있음).

### SSD 교체 시 내용 유지 (클론)

- **방법 1 (Etcher Clone drive):** 기존 SSD와 새 SSD를 각각 USB-NVMe 외장 케이스에 넣어 PC에 연결 → Etcher → **Clone drive** → 원본: 기존 SSD, 대상: 새 SSD → Flash. (케이스 2개 필요)
- 케이스가 하나뿐이면 Win32 Disk Imager 등으로 기존 SSD를 이미지 파일로 저장 → Etcher로 새 SSD에 굽기 (PC에 256GB 빈 공간 필요).
- 같은 슬롯에 꽂으면 장치 이름이 `nvme0n1p1`로 동일하여 부팅 설정 그대로 동작.
- 새 SSD는 기존(256GB)보다 같거나 커야 함. 더 큰 SSD로 옮기면 복제 후 파티션 확장 작업이 별도로 필요.
- 대안: 새 SSD에 처음부터 다시 설치하고 작업 파일만 옮기기. 평소 코드를 Git에 올려 두면 안전.

### 관련 하드웨어 메모

- 보드 뒷면(캐리어 보드 아래쪽) M.2 Key E 슬롯의 카드 = **Wi-Fi/블루투스 모듈** (안테나 선 2가닥). 빼지 말 것.
- M.2 Key M 슬롯 2개: SSD용. 2280 SSD는 긴 슬롯(PCIe x4) 권장.
- microSD 슬롯은 캐리어 보드가 아니라 Jetson 모듈 아래쪽에 있음. 개발자 키트에 microSD 카드는 기본 포함되지 않음.
- 노트북은 Jetson의 전원·모니터 역할을 할 수 없음 (전원은 DC 어댑터 전용, 노트북 HDMI는 출력 전용).
- 모니터 하나로 PC와 번갈아 보기: PC → 모니터 HDMI, Jetson → 모니터 DP, 모니터 입력 전환 버튼으로 전환.

---

## 18. ssh 서버 설치 및 종료

```
1. 설치 여부 확인
Get-WindowsCapability -Online -Name OpenSSH.Server*

2. 서비스 시작과 자동 실행 설정
Start-Service sshd
```

## 참고 자료

- [Jetson Orin Nano Developer Kit Quick Start Guide](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/quick_start.html)
- [Jetson Orin Nano Developer Kit How-to Guides](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/howto.html)
- [JetPack Archive](https://developer.nvidia.com/embedded/jetpack-archive)
- [JetPack 6.2.2 (6.2.1 SD 카드 이미지 링크)](https://developer.nvidia.com/embedded/jetpack-sdk-622)
- [JetPack 7.2 / r39.2 on Orin Nano — Getting Started and feedback thread](https://forums.developer.nvidia.com/t/jetpack-7-2-jetson-linux-r39-2-on-jetson-orin-nano-developer-kit-getting-started-and-feedback-thread/372151)
- [AGX Orin black screen after JetPack 7.2 ISO install](https://forums.developer.nvidia.com/t/jetson-agx-orin-64g-after-flashing-to-jetpack-7-2-in-iso-mode-the-machine-goes-black/374511)
- [Looking for PyTorch compatible with CUDA and JetPack 6.2](https://forums.developer.nvidia.com/t/looking-for-pytorch-compatible-with-cuda-and-jetpack-6-2/348897)
- [PyTorch 2.8.0 on Jetson Orin Nano: ImportError: libcudss.so.0 not found](https://forums.developer.nvidia.com/t/pytorch-2-8-0-on-jetson-orin-nano-importerror-libcudss-so-0-not-found/346195)
- [Lack of libcudss.so.0](https://forums.developer.nvidia.com/t/lack-of-libcudss-so-0/346835)
- [cuDSS Downloads](https://developer.nvidia.com/cudss-downloads)
- [Power Button on Jetson Orin Nano](https://forums.developer.nvidia.com/t/power-button-on-jetson-orin-nano/294527)
