# Filter_Top 아키텍처 - 이미지 필터 시스템

## 📋 개요
Filter_Top은 11가지 이미지 필터를 통합 관리하는 핵심 모듈입니다. 입력 이미지에 대해 선택된 필터를 적용하고 결과를 출력합니다.

## 🏗️ 시스템 구조

```
입력 이미지 (320x240 RGB565)
         │
         ▼
┌─────────────────────────────────────────┐
│              Filter_Top                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐      │
│  │  0  │ │  1  │ │  2  │ │  3  │ ...  │
│  │BYPASS│Gaussian│Sharpen│ Sobel │      │
│  └─────┘ └─────┘ └─────┘ └─────┘      │
│     │       │       │       │         │
│     └───────┼───────┼───────┘         │
│             │       │                 │
│  ┌─────────────────────────────────┐   │
│  │        filter_selector          │   │
│  │     (sel에 따른 선택)            │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
         │
         ▼
    출력 이미지
```

## 🔧 주요 구성 요소

### **1. 필터 모듈들 (11개)**

| 인덱스 | 필터명 | 기능 | 특징 |
|--------|--------|------|------|
| 0 | BYPASS | 원본 통과 | 필터 적용 없음 |
| 1 | Gaussian | 블러 효과 | 노이즈 제거, 부드러운 효과 |
| 2 | Sharpen | 선명화 | 엣지 강조 |
| 3 | Sobel | 엣지 검출 | 윤곽선 추출 |
| 4 | Warm | 따뜻한 톤 | 색온도 조정 |
| 5 | Cool | 차가운 톤 | 색온도 조정 |
| 6 | Graduation | 졸업사진 효과 | 특수 효과 |
| 7 | Cartoon | 만화 스타일 | 포스터라이제이션 + 엣지 |
| 8 | Ghost | 유령 효과 | 투명도 처리 |
| 9 | Mirror | 거울 효과 | 좌우 반전 |
| 10 | Sticker | 스티커 효과 | 객체 인식 + 오버레이 |

### **2. 필터 선택기 (filter_selector)**

```systemverilog
module filter_selector #(
    parameter int N = 16
) (
    input  logic [     3:0] sel,        // 선택 신호 (0-10)
    input  logic [   N-1:0] we_bus,      // Write Enable 버스
    input  logic [N*17-1:0] addr_bus,    // Address 버스
    input  logic [N*16-1:0] data_bus,    // Data 버스
    output logic            we_out,      // 선택된 Write Enable
    output logic [    16:0] wAddr_out,   // 선택된 Address
    output logic [    15:0] wData_out    // 선택된 Data
);
```

## 📊 데이터 흐름

### **입력 인터페이스**
```systemverilog
input  logic        we_in,      // Write Enable
input  logic [16:0]  wAddr_in,   // Write Address (0~76799)
input  logic [15:0]  wData_in    // Write Data (RGB565)
```

### **출력 인터페이스**
```systemverilog
output logic        o_we_out,   // Output Write Enable
output logic [16:0]  o_wAddr,    // Output Address
output logic [15:0]  o_rData     // Output Data
```

### **내부 버스 구조**
```systemverilog
logic [15:0] we[16];           // 16개 필터의 Write Enable
logic [16:0] addr[16];         // 16개 필터의 Address
logic [15:0] data[16];         // 16개 필터의 Data

// 버스 변환
logic [15:0] we_bus;           // 16비트 Write Enable 버스
logic [271:0] addr_bus;        // 272비트 Address 버스 (16×17)
logic [255:0] data_bus;        // 256비트 Data 버스 (16×16)
```

## 🎯 필터별 상세 분석

### **1. Gaussian Filter**
- **목적**: 이미지 블러링, 노이즈 제거
- **커널**: 3×3 가우시안 커널 (1 2 1; 2 4 2; 1 2 1)/16
- **특징**: 부드러운 효과, 노이즈 감소

### **2. Sobel Filter**
- **목적**: 엣지 검출
- **커널**: 
  - Gx: (-1 0 1; -2 0 2; -1 0 1)
  - Gy: (-1 -2 -1; 0 0 0; 1 2 1)
- **특징**: 윤곽선 강조, 이진화 출력

### **3. Cartoon Filter**
- **목적**: 만화 스타일 효과
- **처리 과정**:
  1. 가우시안 블러 적용
  2. 포스터라이제이션 (색상 단계 감소)
  3. 소벨 엣지 검출
  4. 엣지와 색상 합성

## ⚡ 성능 특성

### **처리 지연**
- **파이프라인 지연**: 3-4 클럭 사이클
- **메모리 접근**: 각 필터마다 독립적
- **실시간 처리**: 320×240@30fps 지원

### **리소스 사용량**
- **메모리**: 각 필터마다 라인 버퍼 필요
- **로직**: 필터별로 최적화된 하드웨어
- **파이프라인**: 병렬 처리로 성능 향상

## 🔍 설계 원리

### **1. 모듈화 설계**
- 각 필터가 독립적인 모듈
- 표준화된 인터페이스
- 재사용 가능한 구조

### **2. 파이프라인 처리**
- 입력과 출력이 독립적
- 실시간 처리 가능
- 지연 시간 최소화

### **3. 선택적 처리**
- sel 신호에 따른 동적 선택
- 불필요한 필터 비활성화
- 전력 소모 최적화

## 🎯 학습 포인트

1. **모듈화 설계**: 재사용 가능한 필터 모듈
2. **버스 구조**: 다중 신호의 효율적 관리
3. **선택 로직**: 동적 필터 선택 메커니즘
4. **파이프라인**: 실시간 이미지 처리
5. **인터페이스**: 표준화된 모듈 간 통신

## 📝 다음 단계
- `03_Image_Filters_Basic.md`: 기본 필터들 상세 분석
- `04_Color_Filters.md`: 색상 필터들 분석
- `06_Sticker_Filter.md`: 스티커 필터 특별 분석
