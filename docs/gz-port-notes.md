# GenieZone 이식 메모 (Gunyah 가정 목록 + PAC 테스트)

출처: /tmp 에 shallow clone 한 UniXNU / macOS-DroidVM / DroidVM (읽기만 함, 장치에서 실행한 것 없음).
주의: 사용자가 말한 `device/UniXNU-droidvm` 는 현재 macOS-DroidVM 트리에 없음 (`ls device` 실패, REPO-LAYOUT.md 도 "device/ 더 이상 없음"). 대응물은 `overlay/droidvm/`, `overlay/edk2/GunyahBootPkg/`, `scripts/run-macos-vm.py`.

## A. Gunyah 전제 → GenieZone에서 바뀌어야 할 것

| # | Gunyah 전제 (근거) | GenieZone에서의 문제 |
|---|---|---|
| 1 | `--hypervisor gunyah` + `--protected-vm-pseudo-unprotected` (scripts/run-macos-vm.py:78-79) | DroidVM이 직접 "Pseudo-unprotected requires Gunyah; using normal mode" 로 낮춤 (CrosvmBackendInstance.java:419). gzvm은 `--hypervisor geniezone`, 일반 VM 모드. |
| 2 | `--hugepages --prepare-lend-mthp-mode single` (run-macos-vm.py:80) | Gunyah 메모리 lend 풀 전용. PoolPreflight.java:121: "KVM and GenieZone hand over nothing". 처음엔 제거하고 시작. (DroidVM 자신은 gzvm에서 `chunked` 사용 중이라 crosvm이 옵션은 받음 — 의미는 unverified.) |
| 3 | EDK2 `Platform.c` 가 FDT에 하드 어서션: `restricted-dma-pool` 있으면 EFI_UNSUPPORTED; stdout-path=ns16550a; `arm,gic-v3`; PSCI `arm,psci-0.2` method=`hvc`; 타이머 PPI 3번째=가상 타이머; simple-framebuffer a8r8g8b8; RAM base≥0x80000000 & 크기≤1 GiB; 장치 주소<0x40000000 | 사용자 DroidVM 명령줄은 `--swiotlb 256` → restricted-dma-pool 노드 생성 → **Platform.c가 거부**. gzvm FDT의 PSCI method / RAM base / 크기 / PPI 번호는 unverified (`--dump-device-tree-blob` 로 덤프해 확인). |
| 4 | `--swiotlb` 불필요 가정 (pseudo-unprotected라 호스트가 게스트 RAM 접근 가능, CrosvmBackendInstance.java 주석) | gzvm 일반 VM도 호스트가 접근 가능하므로 swiotlb 없이 시도. 단, 사용자 gotcha("큰 쓰기에서 crosvm 크래시, swiotlb 올림")와 충돌 — 실험으로 판단. unverified. |
| 5 | WFI는 트랩(vCPU 블록), WFE는 게스트에서 실행 (xnu-12377/0020, ArmCpu-park-wfi-latency.patch) | gzvm의 WFI/WFE 처리 unverified. 패치는 보드 전체 WFI 종료라 gzvm에서도 무해할 가능성이 높지만, `parkCore()`/타임아웃 값이 다를 수 있음. |
| 6 | 코어가 첫 IPI를 받아야 `SIGPdisabled` 해제 → ArmCpu가 SGI 0 자기 전송 (TODO.md:82-85). ArmCpu 주석: "ICC_SGI1R_EL1 generates nothing on this board" → `armcpu-spi-ipi=1` 대체 경로 존재 | gzvm GICv3의 SGI/redistributor 동작 unverified. 멀티 vCPU 자체가 crosvm geniezone 백엔드에서 미확인(질문 #2). |
| 7 | 카운터: "Gunyah passes the host's counter through" (xnu-12377/0018), `droidvm-timebase-hz=24000000`, 커널이 호스트 카운터(주석상 19.2 MHz)를 24 MHz로 환산 (droidvm-app.py REQUIRED_ARGS) | MT6989의 CNTFRQ는 다를 수 있음 (값 unverified — 게스트에서 `cntfrq_el0` 읽어 확인). EDK2 Platform.c는 `cntfrq_el0` 를 그대로 읽어 ADT로 전달하므로 환산 인자 재계산 필요. |
| 8 | MMIO는 단순 ldr/str만 (NativeVirtioPci.h: "KVM and Gunyah cannot emulate writeback forms (ISV=0)") | gzvm도 KVM 계열이라 같을 가능성 높음, unverified. 같은 규칙을 그대로 유지하면 안전. |
| 9 | 전용 crosvm 포크 `bin/crosvm-mmio` (`--virtio-mmio-*`, `--dump-device-tree-blob`, `--droidvm-gpu`, `--prepare-lend-mthp-mode`) (run-macos-vm.py:29) | 사용자 장치의 `~/crosvm`(Google 업스트림)·DroidVM 번들 crosvm에 이 옵션들이 있는지 unverified. `--help` 로 확인. 포크 소스는 macOS-DroidVM 트리에 없음(external/crosvm, 별도 저장소). |
| 10 | GPU 경로 전체가 Qualcomm 전용: Adreno/KGSL, Turnip, `drm2kgsl`, `gunyah-pvm=true` (CrosvmBackendInstance.java:821), DroidVMGpu | Mali에는 해당 없음 → 그래픽은 전부 재설계 (계획 5단계, 지금은 무관). simplefb(GOP)만 사용. |
| 11 | CPU 토폴로지/어피니티: 8 vCPU, 큰코어 6/7 capacity 1024, `MACOS_HOST_CORES=8`, `--cpu-affinity` (droidvm-app.py:84-97, TODO.md:45-46) | MT6989는 1+3+4 구성, 값 전부 재설정. 먼저 `--cpus 1`. |
| 12 | 락 타임아웃 튜닝: `ml-timeout-lock-scale=40`, `ml-timeout-debug-ack-scale=100` (Gunyah에서 vCPU가 오래 안 돌 때 panic 방지) | gzvm 스케줄링 특성에 따라 재조정. 시작점은 그대로. |
| 13 | 파일/문서 명칭 `edk2-gunyah.fd`, `GunyahBootPkg` | 이름뿐. DroidVM이 gzvm에서 `edk2-gunyah.fd` 를 이미 부팅함(사용자 로그) → 펌웨어 자체는 재사용 가능. |

PAC 관련 간접 증거 (verified from source text only, 실행 아님): macOS-DroidVM은 Gunyah 위에서 arm64e 커널로 launchd/WindowServer까지 도달 (TODO.md:7-34), docs/xnu-12377-port.md:17 "PAC 키는 명시적 프로그래밍 없음". 즉 Gunyah는 EL1의 PAC 명령/키 레지스터를 트랩하지 않음. gzvm 동일 여부는 unverified → 아래 테스트.

## B. PAC 가용성 테스트 (파괴적 동작 없음, 전부 복사본 디스크 사용)

원칙: 게스트로 Linux(Alpine)를 쓴다. Linux는 `CONFIG_ARM64_PTR_AUTH(_KERNEL)` 일 때 EL1에서 APIA/APIB/APDA/APDB/APGA 키를 부팅 중 쓴다. 키 쓰기가 트랩되거나 PAC 미지원이면 dmesg에 해당 기능 감지 줄이 없거나 부팅이 깨진다. 이 로그가 곧 증거.

### B0. crosvm 옵션 확인 (읽기 전용)
```bash
CV=/data/data/cn.classfun.droidvm/usr/bin/crosvm
su -c "$CV --version; $CV run --help" 2>&1 | grep -n -i -E 'hypervisor|serial|socket|--mem|--cpus|swiotlb|protected|dump-device-tree|pflash|hugepages|lend'
```
(실행 권한/라이브러리 오류가 나면 `LD_LIBRARY_PATH=/data/data/cn.classfun.droidvm/usr/lib` 를 `su -c` 안에 추가 — unverified.)

### B1. 앱 VM 정지 확인 + 디스크 복사
```bash
su -c 'ps -A | grep -i crosvm'
```
출력이 비어야 함. 앱이 돌고 있으면 앱에서 먼저 정지.
```bash
su -c 'ls -l /storage/emulated/0/DroidVM /data/data/cn.classfun.droidvm/ 2>&1 | head -40'
```
Alpine 디스크 경로를 확인한 뒤(아래 `SRC`), 복사:
```bash
SRC=<위에서 확인한 Alpine-edge qcow2 경로>
su -c "cp $SRC /data/local/tmp/gz-test.qcow2 && ls -l /data/local/tmp/gz-test.qcow2"
```

### B2. 수동 crosvm 실행 (시리얼을 터미널로)
```bash
D=/data/data/cn.classfun.droidvm
su -c "$D/usr/bin/crosvm --extended-status run --name gz-pac --mem 512 --cpus 1 --hypervisor geniezone --no-balloon --disable-sandbox --socket /data/local/tmp/gz.sock --block /data/local/tmp/gz-test.qcow2,lock=false --serial type=stdout,hardware=serial,num=1,console,stdin,earlycon $D/usr/share/droidvm/edk2-gunyah.fd --pflash path=$D/usr/share/droidvm/edk2-gunyah.vars.fd,block_size=262144"
```
- `--hugepages/--prepare-lend-mthp-mode/--swiotlb/--net/--simplefb/--input/--virtio-snd` 는 의도적으로 뺌. 부팅이 안 되면 하나씩 되돌려 추가.
- `--pflash` 의 vars 파일은 앱 소유라 사본 사용 권장: `cp $D/usr/share/droidvm/edk2-gunyah.vars.fd /data/local/tmp/gz.vars.fd` 후 경로 교체.
- 시리얼 키 이름(`type=stdout`, `stdin`, `earlycon`)은 B0 출력과 대조해서 맞출 것 (unverified).
- 종료: 다른 터미널에서 `su -c "$D/usr/bin/crosvm stop /data/local/tmp/gz.sock"`.

### B3. 판독 (게스트 시리얼 로그에서)
부팅 로그 또는 로그인 후:
```bash
dmesg | grep -i -E 'address authentication|generic authentication|pointer auth|CPU features|Booted|psci|GICv3|arch_timer'
grep -m1 Features /proc/cpuinfo
```
- 합격 신호: `CPU features: detected: Address authentication (architected QARMA5 algorithm)` 와 `Generic authentication` 줄, cpuinfo Features 에 `paca pacg`, 부팅 정상 완료.
- 불합격 신호: 위 줄 없음(= 게스트에 FEAT_PAuth 미노출, ID_AA64ISAR1/ISAR2 가 0), 또는 부팅 중 EL1 키 쓰기에서 hang/exception (시리얼 마지막 줄 확인).
- 부가 정보: `cat /sys/devices/system/clocksource/clocksource0/current_clocksource`, `dmesg | grep -i 'arch_timer: cp15 timer'` → 게스트 CNTFRQ 확인(A표 #7).

### B4. (B3 합격 후) 사용자 공간 PAC 명령 테스트
Alpine에서 네트워크가 있으면:
```bash
apk add gcc musl-dev
cat > /tmp/pac.c <<'EOF'
#include <stdio.h>
int main(void){
  unsigned long p=(unsigned long)&p, q=p;
  __asm__ volatile("paciza %0" : "+r"(q));
  printf("ptr=%lx signed=%lx %s\n", p, q, q!=p ? "PAC bits set" : "NO PAC bits");
  __asm__ volatile("autiza %0" : "+r"(q));
  printf("auth round-trip: %s\n", q==p ? "ok" : "mismatch");
  return 0;
}
EOF
gcc -march=armv8.3-a -o /tmp/pac /tmp/pac.c && /tmp/pac
```
`SIGILL` = PAC 명령이 EL0에서 트랩/미구현. 네트워크가 없으면 이 단계는 건너뛰고 B3 결과만 기록.

### B5. EL1 키 레지스터 직접 쓰기 (XNU 요구사항 정확 검증, B3 합격 후에도 필요 시)
Linux 부팅이 키 쓰기를 이미 간접 증명하지만, 5개 키(APIA/APIB/APDA/APDB/APGA)를 명시적으로 쓰는 EFI 앱 또는 커널 모듈이 필요. 지금은 작성하지 않음 — B3 결과 후 결정. 또한 HCR_EL2.APK/API 가 게스트에 대해 설정된 상태인지는 게스트에서 읽을 수 없으므로(EL2 레지스터) 키 쓰기 성공 여부로만 추정.

## 한계
- 위 명령은 전부 이 컨테이너에서 실행하지 않았고 장치 출력도 없음 → 모든 옵션 이름/경로는 unverified.
- crosvm `--help` 출력은 장치에서만 확인 가능(B0).
