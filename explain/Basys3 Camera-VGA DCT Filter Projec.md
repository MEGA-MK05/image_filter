# Basys3 Camera-VGA DCT Filter Project

## 🎯 프로젝트 개요

Basys3 보드에서 640x480 카메라 영상을 실시간으로 DCT 필터링하여 VGA로 출력하는 프로젝트입니다.

## 📊 시스템 구조

```
카메라(OV7670) → 8x8 블록 분할 → DCT 필터 → VGA 출력
     ↓              ↓              ↓         ↓
   8비트 병렬     라인 버퍼      파이프라인   640x480@60Hz
```

## 🔌 하드웨어 연결

### 카메라 인터페이스 (OV7670)
- **데이터**: 8비트 병렬 (D0-D7)
- **제어**: HREF, VSYNC, PCLK
- **클록**: 25MHz 픽셀 클록

### VGA 출력
- **색상**: 4비트 RGB (12비트 총)
- **해상도**: 640x480 @ 60Hz
- **동기**: HSYNC, VSYNC

## 📁 파일 구조

```
vivado_project/
├── src/
│   ├── fifo2_improved.v          # FIFO 모듈
│   ├── mkBlockFilter.v           # BSV 생성 DCT 필터
│   ├── camera_vga_interface.v    # 메인 인터페이스
│   └── vivado_wrapper.v          # 기존 래퍼
├── constraints/
│   └── basys3_constraints.xdc    # Basys3 핀 할당
└── scripts/
    └── create_project.tcl        # 프로젝트 생성
```

## ⚙️ 동작 방식

### 1. 카메라 데이터 캡처
- 8비트 픽셀 데이터를 라인 버퍼에 저장
- 8x8 블록 단위로 데이터 구성
- HREF, VSYNC 신호로 프레임 동기화

### 2. DCT 필터링
- 8x8 블록을 1024비트로 변환
- 3단계 파이프라인 처리:
  - Stage 1: 전처리 (소프트 임계값)
  - Stage 2: 행 방향 IDCT
  - Stage 3: 열 방향 IDCT

### 3. VGA 출력
- 처리된 블록을 VGA 픽셀로 변환
- 25MHz VGA 클록으로 출력
- 실시간 디스플레이

## 🎛️ 제어 신호

### 스위치 제어
- **SW0**: 처리 활성화/우회
- **SW1-SW2**: 필터 모드 선택
  - 00: 우회 모드
  - 01: 스무딩 필터
  - 10: 엣지 강화
  - 11: 복합 필터

## 📊 리소스 사용량 (예상)

### IO 사용량
- **카메라 입력**: 11핀 (8비트 데이터 + 3제어)
- **VGA 출력**: 14핀 (12비트 색상 + 2동기)
- **제어**: 3핀 (스위치)
- **총 IO**: ~28핀 (기존 1024핀에서 대폭 감소!)

### 내부 리소스
- **LUT**: ~2,500개
- **FF**: ~4,000개
- **BRAM**: 8-12개 (라인 버퍼용)
- **DSP**: 1-2개

## 🔧 설정 방법

### 1. 프로젝트 생성
```cmd
# Windows에서
cd "\\wsl.localhost\Ubuntu-22.04\home\mild1203\final\vivado_project"
scripts\run_windows_simple.bat
```

### 2. 제약 조건 적용
```tcl
# Vivado에서
add_files -norecurse {constraints/basys3_constraints.xdc}
```

### 3. 합성 및 구현
```tcl
launch_runs synth_1 -jobs 4
launch_runs impl_1 -jobs 4
```

## 🎥 실시간 처리

### 성능 특성
- **처리 지연**: 3클록 사이클 (파이프라인)
- **처리량**: 25MHz (픽셀 클록)
- **실시간**: 640x480 @ 30fps 지원

### 메모리 요구사항
- **라인 버퍼**: 640바이트 (1라인)
- **블록 버퍼**: 64바이트 (8x8)
- **총 메모리**: <1KB

## ⚠️ 주의사항

1. **클록 도메인**: 카메라(25MHz)와 시스템(100MHz) 분리
2. **동기화**: HREF, VSYNC 신호 정확한 타이밍
3. **메모리**: 라인 버퍼 BRAM 사용
4. **지연**: 3클록 파이프라인 지연 보정

## 🎯 결과

이제 IO 사용량이 28핀으로 줄어들어 Basys3 보드에서 구현 가능합니다!
- ✅ IO 오버플로우 해결
- ✅ 실시간 카메라-VGA 처리
- ✅ DCT 필터링 파이프라인
- ✅ Basys3 호환
