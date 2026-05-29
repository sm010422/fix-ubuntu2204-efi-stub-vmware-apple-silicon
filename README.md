# VMware Fusion + Apple Silicon (M2) 에서 Ubuntu 22.04 ARM 부팅 불가 문제 해결

> **환경**: MacBook (Apple M2) + VMware Fusion Pro 26H1 + Ubuntu 22.04.5 ARM64 Server  
> **증상**: 설치 후 부팅 시 `EFI stub: Exiting boot services...` 에서 영구 정지

---

## 목차

1. [문제 설명](#1-문제-설명)
2. [원인 분석](#2-원인-분석)
3. [실패한 시도들](#3-실패한-시도들)
4. [최종 해결 방법](#4-최종-해결-방법)
5. [영구 적용 (HWE 커널 설치)](#5-영구-적용-hwe-커널-설치)
6. [핵심 요약](#6-핵심-요약)

---

## 1. 문제 설명

Ubuntu 22.04.5 ARM64 Server ISO로 VMware Fusion에서 정상적으로 설치를 완료한 후, 재부팅 시 아래 메시지에서 **무한 정지**하는 현상이 발생한다.

```
EFI stub: Booting Linux Kernel...
EFI stub: EFI_RNG_PROTOCOL unavailable
EFI stub: Using DTB from configuration table
EFI stub: Exiting boot services...
```

이후 아무런 출력 없이 검은 화면이 유지되며, 아무리 기다려도 부팅이 진행되지 않는다.

---

## 2. 원인 분석

Ubuntu 22.04의 기본 커널인 **`5.15.x`** 계열이 VMware Fusion의 Apple Silicon 환경에서 부팅에 실패하는 알려진 버그다.

- EFI stub이 부트 서비스를 종료한 후 커널이 초기화를 시작해야 하는데, 이 단계에서 멈춤
- `EFI_RNG_PROTOCOL unavailable`은 경고일 뿐 원인이 아님
- Recovery mode도 동일한 커널을 사용하므로 마찬가지로 실패
- **HWE(Hardware Enablement) 커널 (`5.19+`)** 을 사용하면 정상 부팅됨

---

## 3. 실패한 시도들

아래 방법들을 모두 시도했으나 동일하게 실패했다.

### 3-1. GRUB에서 `acpi=force` 추가

GRUB 편집 화면(`e` 키)에서 `linux` 줄 끝에 추가:
```
acpi=force
```
→ **실패**: 동일하게 EFI stub에서 정지

### 3-2. GRUB에서 `earlycon` + `console` 추가

```
acpi=force earlycon=pl011,0x9000000 console=ttyAMA0
```
→ **실패**: 커널 로그조차 출력되지 않고 정지

### 3-3. `.vmx` 파일에 소프트웨어 MMU 옵션 추가

```
monitor.virtual_mmu = "software"
monitor.virtual_exec = "software"
```
→ **실패**: 동일 증상

### 3-4. `.nvram` 파일 삭제 (EFI 설정 초기화)

```bash
rm "Ubuntu 64-bit Arm Server 22.04.5 4.nvram"
```
→ **실패**: 동일 증상

### 3-5. Recovery mode 부팅

GRUB → Advanced options → recovery mode 선택  
→ **실패**: 동일한 커널(5.15)을 사용하므로 동일하게 정지

### 3-6. GRUB 쉘에서 `acpi=off`로 수동 부팅

```
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-5.15.0-179-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro acpi=off
grub> initrd /initrd.img-5.15.0-179-generic
grub> boot
```
→ **실패**: 5.15 커널 자체가 문제이므로 파라미터와 무관하게 실패

---

## 4. 최종 해결 방법

### 핵심 아이디어

Ubuntu 22.04 설치 ISO의 `/casper/` 디렉토리에는 **HWE 커널** (`hwe-vmlinuz`, `hwe-initrd`)이 포함되어 있다. GRUB 쉘에서 ISO에 있는 HWE 커널로 직접 부팅하면 정상 작동한다.

### 단계별 절차

#### Step 1. GRUB 쉘 진입

VM 시작 직후 즉시 VMware 창을 클릭하여 키보드 입력을 잡고, **`c`** 키를 눌러 GRUB 커맨드라인(쉘)으로 진입한다.

> **팁**: GRUB 메뉴가 안 보이면 VM 시작 직후 **ESC** 키를 연타하여 Boot Manager → ubuntu → GRUB 메뉴 순서로 접근한다.

```
GNU GRUB  version 2.06

grub> _
```

#### Step 2. 디바이스 목록 확인

```
grub> ls
```

출력 예시:
```
(proc) (memdisk) (hd0) (hd0,gpt3) (hd0,gpt2) (hd0,gpt1) (cd0) (cd0,msdos2) (cd0,msdos1) (lvm/ubuntu--vg-ubuntu--lv)
```

`(cd0)` 가 Ubuntu ISO가 마운트된 CDROM 드라이브다.

#### Step 3. ISO의 casper 디렉토리 확인

```
grub> ls (cd0)/casper/
```

출력에서 `hwe-vmlinuz` 와 `hwe-initrd` 가 있는지 확인한다.

```
hwe-initrd  hwe-vmlinuz  initrd  vmlinuz  ...
```

#### Step 4. HWE 커널로 부팅

```
grub> linux (cd0)/casper/hwe-vmlinuz
grub> initrd (cd0)/casper/hwe-initrd
grub> boot
```

정상적으로 Ubuntu 설치 환경으로 부팅된다.

---

## 5. 영구 적용 (HWE 커널 설치)

ISO로 부팅한 후 설치 환경 쉘에 접근하거나, 설치된 시스템에 chroot로 진입하여 HWE 커널을 설치한다.

### 방법 A: ISO 부팅 후 chroot로 진입하여 설치

```bash
# LVM 볼륨 활성화
vgchange -ay

# 루트 파티션 마운트
mount /dev/mapper/ubuntu--vg-ubuntu--lv /mnt

# 필수 디렉토리 바인드 마운트
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

# chroot 진입
chroot /mnt

# HWE 커널 설치
apt update
apt install linux-generic-hwe-22.04 -y

# 완료 후 재부팅
exit
reboot
```

### 방법 B: 정상 부팅 후 바로 설치 (방법 A로 부팅에 성공한 경우)

로그인 후 바로 실행:

```bash
sudo apt update
sudo apt install linux-generic-hwe-22.04 -y
sudo reboot
```

재부팅 후 ISO 없이 정상 부팅되는지 확인한다.

---

## 6. 핵심 요약

| 항목 | 내용 |
|------|------|
| **문제 커널** | `linux-generic` (5.15.x) |
| **정상 커널** | `linux-generic-hwe-22.04` (5.19+) |
| **해결 핵심** | ISO 내장 HWE 커널(`hwe-vmlinuz`)로 GRUB 쉘에서 직접 부팅 |
| **영구 해결** | `apt install linux-generic-hwe-22.04` |
| **영향 환경** | VMware Fusion + Apple Silicon (M1/M2) + Ubuntu 22.04 |

---

## 참고

- 이 문제는 VMware Fusion 13.x ~ 26H1 전 버전에서 동일하게 발생한다.
- Ubuntu 22.04.2 LTS 이상에서 HWE 커널을 사용하면 VMware Fusion ARM 환경에서 정상 동작한다.
- Ubuntu 24.04 LTS는 기본 커널(6.x)부터 이 문제가 없으므로, 새로 설치할 경우 24.04를 권장한다.

---

*이 문서는 실제 트러블슈팅 경험을 바탕으로 작성되었습니다.*
