# 모두의 골프 포터블 2 (みんなのGOLF ポータブル２) 한글화 패치

소니 2007년작 PSP 골프 게임 **모두의 골프 포터블 2 (Everybody's Golf Portable 2)** 의 한글화 작업입니다.

> 🔬 **분석 중** — 아직 번역을 시작하지 않았습니다. 컨테이너 포맷 해독이 진행 중입니다.

## 다운로드

**준비 중**

## 원본 정보

```
Minna no Golf Portable 2 (Japan, Korea) (v1.09).iso
크기: 922,025,984 bytes
MD5:  7c94ccb94b3bb336095af3de0fd912cb
```

| 항목 | 값 |
|---|---|
| DISC_ID | UCJS-10075 |
| TITLE | みんなのGOLF ポータブル２ |
| DISC_VERSION | 1.09 |
| PSP_SYSTEM_VER | 3.72 |

「Japan, Korea」는 같은 UMD가 두 지역에 유통됐다는 뜻이고, **내용은 일본어판**입니다.

## 진행 상황

| 항목 | 상태 |
|------|------|
| ISO9660 읽기/쓰기 도구 | ✅ 완료 |
| UMD 구성 파악 (2,883 파일) | ✅ 완료 |
| 실행 파일 분석 (MIPS 디스어셈블러) | ✅ 완료 |
| 리소스 시스템·경로 규칙 | ✅ 완료 |
| `.xb` 컨테이너 구조 | 🔶 대부분 확인 |
| **`.xb` 압축 코덱** | ❌ **미해결 — 현재 최대 관문** |
| 텍스트 추출 | ⏸ 코덱 대기 |
| 폰트 (한글 글리프) | ⏸ 코덱 대기 |
| 번역 | ⏸ |

## 구성

```
PSP_GAME/
  SYSDIR/EBOOT.BIN            3,361,056B  서명된 부트 모듈
  SYSDIR/BOOT.BIN             3,360,708B  비암호화 (= mgp_rel_umd.elf)
  SYSDIR/UPDATE/              펌웨어 업데이터 (건드릴 일 없음)
  USRDIR/mgp_rel_umd.elf      3,360,708B  본체 (MIPS ELF, 비암호화)
  USRDIR/mgp.ufl                354,422B  UMD 레이아웃 매니페스트 (평문 CSV)
  USRDIR/*.ovl               13개         화면별 오버레이 코드
  USRDIR/module/*.prx         14개         시스템 모듈
  USRDIR/movie/*.pmf          12개         동영상
  USRDIR/xbdata/**/*.xb    2,806개  695MB  게임 데이터 전부
```

`mgp.ufl` 은 `"경로", "원본경로", LBA, 크기, 플래그` 형식의 **평문 CSV**라 ISO 재빌드에 그대로 쓸 수 있습니다.

## 기술 개요

### 실행 파일

`mgp_rel_umd.elf` 는 **암호화되어 있지 않습니다**. `.text` = VA `0x08804000` (2.9MB),
`.data` = VA `0x08ACDBDC` (434KB). 임포트에 `sceLibFont` 가 있어 PSP 시스템 폰트(PGF)를
쓰고, 동시에 자체 GIM 비트맵 폰트(`font10c.gim`, `flag_fonts`, `Lmsg.gim`)도 있습니다.
엔진은 소니 i3d (`.data` 의 `i3d_package_version 050121-stable-060426`).

`.data` 에서 찾은 일본어는 전부 엔진 디버그 메시지이고, **게임 텍스트는 `.xb` 안**에 있습니다.

### 리소스 시스템

`disc0:/PSP_GAME/USRDIR/xbdata/%s` 로 열고 `crs/%02d/hol%02d.xb`, `kuwa/pc/face/face%02d%s.xb`,
`yumo/loadmes/A%02d.xb` 같은 서식으로 참조합니다. `.xb` 안의 리소스 이름은 **개발 당시 경로**
(`..\data\hatsuyama\common\font10c.gim`)라서 무엇이 무엇인지 바로 읽힙니다.

### `.xb` 컨테이너

```
0x00  'x' 'e' 0x00 0x01
0x04  u32 count
0x08  count × { u32 dec_size, u32 flags_off }   상위 4비트 = 코덱 플래그(0·1·2·3)
+     u32 name_dec, u32 name_comp               name_comp==0 이면 이름표 무압축
+     [이름표]
+     리소스별 [ u32 dec, u32 comp ][스트림] … 청크 반복
```

2,806개 중 581개는 이름표가 무압축이라 리소스 목록을 바로 읽을 수 있습니다.

### 남은 관문 — 압축 코덱

청크 스트림이 두 종류입니다.

- **LZ 단독** — 사전이 비어 있어 시작부 리터럴이 평문으로 보입니다(`MIG.00.1PSP`, `TIM2`,
  `SGXD` 매직과 경로 문자열이 그대로 노출). 제어바이트 구조 확정이 필요합니다.
- **허프만+LZ** — 스트림이 `0b 00 00 01 …` 형태의 정규 허프만 표로 시작합니다.
  `[개수][심볼]` 반복으로 256심볼을 채우는지 시험했으나 실패해 표 형식이 아직 미확정입니다.

검증용 기지 평문(무압축 리소스의 매직, 이름표 경로)이 충분히 있어 비트 단위 역해석이
가능하고, `mgp_rel_umd.elf` 가 비암호화라 압축 해제 루틴을 직접 읽는 길도 열려 있습니다.

### 패치 배포 방향

텍스트가 `.xb` 데이터에만 있으면 **코드를 건드리지 않아** EBOOT 재서명 문제가 없습니다.
`isolib.write()` 는 같은 크기 in-place 교체만 허용하므로 ISO 크기·LBA가 바뀌지 않습니다.
크기가 늘어나야 하면 `mgp.ufl` 기준으로 ISO를 재빌드합니다.

## 개발 도구 (`src/tools/`)

| 파일 | 역할 |
|---|---|
| `isolib.py` | ISO9660 순회·읽기·in-place 쓰기 |
| `xb.py` | `.xb` 컨테이너 파서 (코덱 미구현) |
| `sjis.py` | Shift-JIS 문자열 스캐너 |
| `mips.py` | PSP MIPS 최소 디스어셈블러 (상수·주소 참조 추적) |

## 변경 이력

### (분석 중) 2026-07-30
- 프로젝트 개설, ISO 도구·MIPS 디스어셈블러 작성
- UMD 구성·리소스 시스템·`.xb` 컨테이너 구조 파악
- 실행 파일이 비암호화임을 확인 (코드 역해석 가능)
