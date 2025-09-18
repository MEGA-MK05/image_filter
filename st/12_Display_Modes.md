# 디스플레이 모드들 - 다양한 출력 방식

## 📋 개요
AI Fourcut 프로젝트는 3가지 디스플레이 모드를 지원하여 사용자에게 다양한 시각적 경험을 제공합니다.

## 🎨 디스플레이 모드 개요

### **모드 0: 원본 (Original)**
- **해상도**: 320×240
- **크기**: 1:1 비율
- **용도**: 원본 이미지 확인
- **특징**: 최고 품질, 작은 화면

### **모드 1: 풀스크린 (FullScreen)**
- **해상도**: 640×480
- **크기**: 2배 확대
- **용도**: 전체 화면 표시
- **특징**: 큰 화면, 스케일링

### **모드 2: 4컷 (SplitScreen)**
- **해상도**: 640×480
- **크기**: 2×2 타일링
- **용도**: 4컷 사진 스타일
- **특징**: 4개 동일 이미지

## 🔧 모드별 구현 분석

### **모드 0: 원본 모드**

#### **특징**
- **1:1 매핑**: 원본 픽셀을 그대로 표시
- **최고 품질**: 스케일링 없음
- **작은 화면**: 320×240 영역만 사용

#### **구현**
```systemverilog
module VGA_MemController_Original (
    // VGA side
    input  logic        DE,
    input  logic [ 9:0] x_pixel,
    input  logic [ 9:0] y_pixel,
    // frame buffer side
    output logic        den,
    output logic [16:0] rAddr,
    input  logic [15:0] rData,
    // export side
    output logic [ 3:0] r_port,
    output logic [ 3:0] g_port,
    output logic [ 3:0] b_port
);

// 원본 크기 영역만 활성화
assign den = DE && x_pixel < 320 && y_pixel < 240;

// 1:1 주소 매핑
assign rAddr = den ? (y_pixel * 320 + x_pixel) : 17'bz;

// RGB565 → RGB444 변환
assign {r_port, g_port, b_port} = den ? 
    {rData[15:12], rData[10:7], rData[4:1]} : 12'b0;
```

#### **장단점**
- **장점**: 최고 품질, 빠른 처리
- **단점**: 작은 화면, 화면 공간 낭비

### **모드 1: 풀스크린 모드**

#### **특징**
- **2배 스케일링**: 320×240 → 640×480
- **전체 화면**: VGA 전체 영역 사용
- **픽셀 복제**: 각 픽셀을 2×2로 확대

#### **구현**
```systemverilog
module VGA_MemController_FullScreen #(
    parameter int IMG_W = 320,
    parameter int IMG_H = 240
) (
    // ... 인터페이스 ...
);

localparam int OUT_W = IMG_W * 2;  // 640
localparam int OUT_H = IMG_H * 2;  // 480

// 활성 영역 확인
wire in_active = (x_pixel < OUT_W) && (y_pixel < OUT_H);
assign den = DE && in_active;

// 2배 스케일링 (>>1)
wire [8:0] src_x = x_pixel[9:1];  // 상위 비트 제거
wire [7:0] src_y = y_pixel[8:1];  // 상위 비트 제거

// 주소 계산
wire [16:0] base = ({9'd0, src_y} << 8)  // *256
                 + ({11'd0, src_y} << 6);  // *64
assign rAddr = den ? (base + src_x) : 17'd0;

// RGB565 → RGB444 변환
wire [3:0] R4 = rData[15:12];
wire [3:0] G4 = rData[10:7];
wire [3:0] B4 = rData[4:1];

assign {r_port, g_port, b_port} = den ? {R4, G4, B4} : 12'h000;
```

#### **스케일링 알고리즘**
```
원본 픽셀 (x, y) → 출력 픽셀 (2x, 2y), (2x+1, 2y), (2x, 2y+1), (2x+1, 2y+1)
```

#### **장단점**
- **장점**: 큰 화면, 전체 화면 활용
- **단점**: 품질 저하, 픽셀 복제

### **모드 2: 4컷 모드**

#### **특징**
- **2×2 타일링**: 동일 이미지를 4개 표시
- **4컷 사진**: 인스타그램 스타일
- **대칭 레이아웃**: 좌상, 우상, 좌하, 우하

#### **구현**
```systemverilog
module VGA_MemController_SplitScreen #(
    parameter int IMG_W = 320,
    parameter int IMG_H = 240
) (
    // ... 인터페이스 ...
);

localparam int ACT_W = IMG_W * 2;  // 640
localparam int ACT_H = IMG_H * 2;  // 480

// 활성 영역 확인
wire in_active = (x_pixel < ACT_W) && (y_pixel < ACT_H);
assign den = DE && in_active;

// 2x2 타일링 좌표 변환
logic [8:0] src_x;  // 0..319
logic [7:0] src_y;  // 0..239

always_comb begin
    // x mod 320
    if (x_pixel < IMG_W) src_x = x_pixel[8:0];
    else src_x = (x_pixel - IMG_W);
    
    // y mod 240
    if (y_pixel < IMG_H) src_y = y_pixel[7:0];
    else src_y = (y_pixel - IMG_H);
end

// 주소 계산
assign rAddr = (den) ? (src_y * IMG_W + src_x) : 17'd0;

// RGB565 → RGB444 변환
wire [3:0] R4 = rData[15:12];
wire [3:0] G4 = rData[10:7];
wire [3:0] B4 = rData[4:1];

assign {r_port, g_port, b_port} = (den) ? {R4, G4, B4} : 12'h000;
```

#### **타일링 알고리즘**
```
VGA 좌표 (x, y) → 타일 좌표 (x%320, y%240)
```

#### **장단점**
- **장점**: 4컷 사진 스타일, 대칭 레이아웃
- **단점**: 동일 이미지 반복, 창의성 제한

## 🔄 모드 전환 시스템

### **VGA_MemController 통합**
```systemverilog
module VGA_MemController (
    input  logic [ 1:0] vga_sw,      // 모드 선택
    // VGA side
    input  logic        DE,
    input  logic [ 9:0] x_pixel,
    input  logic [ 9:0] y_pixel,
    // frame buffer side
    output logic        den,
    output logic [16:0] rAddr,
    input  logic [15:0] rData,
    // export side
    output logic [ 3:0] r_port,
    output logic [ 3:0] g_port,
    output logic [ 3:0] b_port
);

// 각 모드별 컨트롤러 인스턴스
VGA_MemController_Original U_Original (...);
VGA_MemController_FullScreen U_FullScreen (...);
VGA_MemController_SplitScreen U_SplitScreen (...);

// 모드 선택 MUX
vga_mux U_VGA_MUX (
    .vga_sw(vga_sw),
    .den_original(den_original),
    .rAddr_original(rAddr_original),
    .r_port_original(r_port_original),
    .g_port_original(g_port_original),
    .b_port_original(b_port_original),
    .den_fullscreen(den_fullscreen),
    .rAddr_fullscreen(rAddr_fullscreen),
    .r_port_fullscreen(r_port_fullscreen),
    .g_port_fullscreen(g_port_fullscreen),
    .b_port_fullscreen(b_port_fullscreen),
    .den_splitscreen(den_splitscreen),
    .rAddr_splitscreen(rAddr_splitscreen),
    .r_port_splitscreen(r_port_splitscreen),
    .g_port_splitscreen(g_port_splitscreen),
    .b_port_splitscreen(b_port_splitscreen),
    .den_out(den),
    .rAddr_out(rAddr),
    .r_port_out(r_port),
    .g_port_out(g_port),
    .b_port_out(b_port)
);
```

### **모드 선택 MUX**
```systemverilog
module vga_mux (
    input logic [1:0] vga_sw,
    
    // 원본 모드
    input logic        den_original,
    input logic [16:0] rAddr_original,
    input logic [ 3:0] r_port_original,
    input logic [ 3:0] g_port_original,
    input logic [ 3:0] b_port_original,
    
    // 풀스크린 모드
    input logic        den_fullscreen,
    input logic [16:0] rAddr_fullscreen,
    input logic [ 3:0] r_port_fullscreen,
    input logic [ 3:0] g_port_fullscreen,
    input logic [ 3:0] b_port_fullscreen,
    
    // 4컷 모드
    input logic        den_splitscreen,
    input logic [16:0] rAddr_splitscreen,
    input logic [ 3:0] r_port_splitscreen,
    input logic [ 3:0] g_port_splitscreen,
    input logic [ 3:0] b_port_splitscreen,
    
    // 출력
    output logic        den_out,
    output logic [16:0] rAddr_out,
    output logic [ 3:0] r_port_out,
    output logic [ 3:0] g_port_out,
    output logic [ 3:0] b_port_out
);

always_comb begin
    case (vga_sw)
        2'b00: begin  // 원본 모드
            den_out = den_original;
            rAddr_out = rAddr_original;
            r_port_out = r_port_original;
            g_port_out = g_port_original;
            b_port_out = b_port_original;
        end
        2'b01: begin  // 풀스크린 모드
            den_out = den_fullscreen;
            rAddr_out = rAddr_fullscreen;
            r_port_out = r_port_fullscreen;
            g_port_out = g_port_fullscreen;
            b_port_out = b_port_fullscreen;
        end
        2'b10: begin  // 4컷 모드
            den_out = den_splitscreen;
            rAddr_out = rAddr_splitscreen;
            r_port_out = r_port_splitscreen;
            g_port_out = g_port_splitscreen;
            b_port_out = b_port_splitscreen;
        end
        default: begin  // 기본값
            den_out = 0;
            rAddr_out = 0;
            r_port_out = 0;
            g_port_out = 0;
            b_port_out = 0;
        end
    endcase
end
```

## ⚡ 성능 비교

| 모드 | 해상도 | 품질 | 화면 크기 | 처리 복잡도 | 용도 |
|------|--------|------|-----------|-------------|------|
| 원본 | 320×240 | 최고 | 작음 | 낮음 | 원본 확인 |
| 풀스크린 | 640×480 | 중간 | 큼 | 중간 | 전체 화면 |
| 4컷 | 640×480 | 중간 | 큼 | 중간 | 4컷 사진 |

## 🎯 학습 포인트

1. **디스플레이 모드**: 다양한 출력 방식
2. **스케일링**: 이미지 크기 조정
3. **타일링**: 반복 패턴 생성
4. **모드 전환**: 동적 모드 선택
5. **사용자 경험**: 다양한 시각적 효과

## 🔧 구현 도전과제

### **1. 스케일링 품질**
- **제약**: 픽셀 복제로 인한 품질 저하
- **해결**: 하드웨어 기반 스케일링
- **최적화**: 고품질 스케일링 알고리즘

### **2. 메모리 효율성**
- **제약**: 제한된 메모리 대역폭
- **해결**: 효율적인 메모리 접근
- **최적화**: 순차적 접근 패턴

### **3. 실시간 처리**
- **제약**: 지연 시간 최소화
- **해결**: 하드웨어 기반 처리
- **최적화**: 병렬 처리

## 📝 다음 단계
- `01_ISP_Top_Overview.md`: 전체 시스템 아키텍처
- `02_Filter_Top_Architecture.md`: 필터 시스템 구조
- `03_Image_Filters_Basic.md`: 기본 이미지 필터들
