# VGA 디스플레이 시스템 - 타이밍과 출력

## 📋 개요
VGA 디스플레이 시스템은 AI Fourcut 프로젝트의 최종 출력 단계로, 메모리에서 읽은 이미지 데이터를 VGA 모니터에 표시합니다.

## 🏗️ VGA 시스템 구조

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Frame_Buffer  │    │   VGA_          │    │   VGA_          │
│   (Memory)      │───▶│   MemController │───▶│   Decoder       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       ▼
         │                       │              ┌─────────────────┐
         │                       │              │   VGA 출력      │
         │                       │              │   (RGB444)      │
         │                       │              └─────────────────┘
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   메모리        │    │   픽셀          │
│   읽기          │    │   데이터        │
└─────────────────┘    └─────────────────┘
```

## 🔧 VGA_Decoder 모듈

### **인터페이스 정의**
```systemverilog
module VGA_Decoder (
    input  logic       clk,        // 시스템 클럭
    input  logic       reset,      // 리셋 신호
    output logic       pclk,       // 픽셀 클럭
    output logic       h_sync,     // 수평 동기
    output logic       v_sync,     // 수직 동기
    output logic [9:0] x_pixel,    // X 좌표
    output logic [9:0] y_pixel,    // Y 좌표
    output logic       DE          // Data Enable
);
```

### **픽셀 클럭 생성기**
```systemverilog
module Pixel_clk_gen (
    input  logic clk,      // 100MHz 시스템 클럭
    input  logic reset,
    output logic pclk      // 25MHz 픽셀 클럭
);

logic [1:0] p_counter;

always_ff @(posedge clk, posedge reset) begin
    if (reset) begin
        p_counter <= 0;
    end else begin
        if (p_counter == 3) begin
            p_counter <= 0;
            pclk <= 1'b1;
        end else begin
            p_counter <= p_counter + 1;
            pclk <= 1'b0;
        end
    end
end
```

### **픽셀 카운터**
```systemverilog
module pixel_counter (
    input  logic       pclk,       // 픽셀 클럭
    input  logic       reset,
    output logic [9:0] h_counter,  // 수평 카운터
    output logic [9:0] v_counter   // 수직 카운터
);

localparam H_MAX = 800, V_MAX = 525;  // VGA 표준

// 수평 카운터
always_ff @(negedge pclk, posedge reset) begin
    if (reset) begin
        h_counter <= 0;
    end else begin
        if (h_counter == H_MAX) begin
            h_counter <= 0;
        end else begin
            h_counter <= h_counter + 1;
        end
    end
end

// 수직 카운터
always_ff @(negedge pclk, posedge reset) begin
    if (reset) begin
        v_counter <= 0;
    end else begin
        if (h_counter == H_MAX - 1) begin
            if (v_counter == V_MAX - 1) begin
                v_counter <= 0;
            end else begin
                v_counter <= v_counter + 1;
            end
        end
    end
end
```

## 📊 VGA 타이밍 특성

### **VGA 표준 (640×480@60Hz)**
```
┌─────────────────────────────────────────────────────────┐
│                    수평 타이밍                          │
├─────────┬─────────┬─────────┬─────────┬─────────────────┤
│ Visible │ Front   │ Sync    │ Back    │ Total           │
│ 640     │ 16      │ 96      │ 48      │ 800             │
└─────────┴─────────┴─────────┴─────────┴─────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    수직 타이밍                          │
├─────────┬─────────┬─────────┬─────────┬─────────────────┤
│ Visible │ Front   │ Sync    │ Back    │ Total           │
│ 480     │ 10      │ 2       │ 33      │ 525             │
└─────────┴─────────┴─────────┴─────────┴─────────────────┘
```

### **동기 신호 생성**
```systemverilog
module vga_counter (
    input  logic [9:0] h_counter,
    input  logic [9:0] v_counter,
    output logic       h_sync,
    output logic       v_sync,
    output logic [9:0] x_pixel,
    output logic [9:0] y_pixel,
    output logic       DE
);

// VGA 타이밍 파라미터
localparam H_Visible_area = 640;
localparam H_Front_porch = 14;
localparam H_Sync_pulse = 96;
localparam H_Back_porch = 48;
localparam H_whole_line = 800;

localparam V_Visible_area = 480;
localparam V_Front_porch = 10;
localparam V_Sync_pulse = 2;
localparam V_Back_porch = 33;
localparam V_whole_frame = 525;

// 동기 신호 생성
assign h_sync = !((h_counter >= (H_Visible_area + H_Front_porch)) && 
                  (h_counter < (H_Visible_area + H_Front_porch + H_Sync_pulse)));
assign v_sync = !((v_counter >= (V_Visible_area + V_Front_porch)) && 
                  (v_counter < (V_Visible_area + V_Front_porch + V_Sync_pulse)));

// Data Enable 신호
assign DE = ((h_counter < H_Visible_area) && (v_counter < V_Visible_area));

// 픽셀 좌표
assign x_pixel = h_counter;
assign y_pixel = v_counter;
```

## 🎨 VGA_MemController 분석

### **3가지 디스플레이 모드**

#### **모드 0: 원본 (320×240)**
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

// 원본 크기 표시
assign den = DE && x_pixel < 320 && y_pixel < 240;
assign rAddr = den ? (y_pixel * 320 + x_pixel) : 17'bz;

// RGB565 → RGB444 변환
assign {r_port, g_port, b_port} = den ? 
    {rData[15:12], rData[10:7], rData[4:1]} : 12'b0;
```

#### **모드 1: 풀스크린 (640×480)**
```systemverilog
module VGA_MemController_FullScreen #(
    parameter int IMG_W = 320,
    parameter int IMG_H = 240
) (
    // ... 인터페이스 ...
);

localparam int OUT_W = IMG_W * 2;  // 640
localparam int OUT_H = IMG_H * 2;  // 480

wire in_active = (x_pixel < OUT_W) && (y_pixel < OUT_H);
assign den = DE && in_active;

// 2배 스케일링
wire [8:0] src_x = x_pixel[9:1];  // >>1
wire [7:0] src_y = y_pixel[8:1];  // >>1

wire [16:0] base = ({9'd0, src_y} << 8)  // *256
                 + ({11'd0, src_y} << 6);  // *64
assign rAddr = den ? (base + src_x) : 17'd0;

// RGB565 → RGB444 변환
wire [3:0] R4 = rData[15:12];
wire [3:0] G4 = rData[10:7];
wire [3:0] B4 = rData[4:1];

assign {r_port, g_port, b_port} = den ? {R4, G4, B4} : 12'h000;
```

#### **모드 2: 4컷 (640×480)**
```systemverilog
module VGA_MemController_SplitScreen #(
    parameter int IMG_W = 320,
    parameter int IMG_H = 240
) (
    // ... 인터페이스 ...
);

localparam int ACT_W = IMG_W * 2;  // 640
localparam int ACT_H = IMG_H * 2;  // 480

wire in_active = (x_pixel < ACT_W) && (y_pixel < ACT_H);
assign den = DE && in_active;

// 2x2 타일링
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

assign rAddr = (den) ? (src_y * IMG_W + src_x) : 17'd0;
```

## ⚡ 성능 특성

### **클럭 도메인**
- **시스템 클럭**: 100MHz
- **픽셀 클럭**: 25MHz
- **VGA 주파수**: 60Hz

### **메모리 대역폭**
- **읽기 속도**: 640×480×60fps = 18.4M 픽셀/초
- **메모리 접근**: 순차적, 연속적
- **지연 시간**: 1클럭

### **디스플레이 품질**
- **해상도**: 640×480
- **색상 깊이**: 12비트 (RGB444)
- **프레임율**: 60fps
- **동기화**: 하드웨어 기반

## 🔧 최적화 기법

### **1. 메모리 접근 최적화**
```systemverilog
// 순차적 메모리 접근
// 캐시 친화적 패턴
// 버스트 전송 활용
```

### **2. 타이밍 최적화**
```systemverilog
// 정확한 클럭 생성
// 동기화 처리
// 지연 시간 최소화
```

### **3. 디스플레이 최적화**
```systemverilog
// 다양한 해상도 지원
// 스케일링 최적화
// 색상 변환 최적화
```

## 🎯 학습 포인트

1. **VGA 타이밍**: 표준 VGA 타이밍 이해
2. **클럭 생성**: 정확한 픽셀 클럭 생성
3. **메모리 접근**: 효율적인 메모리 사용
4. **디스플레이 모드**: 다양한 해상도 지원
5. **색상 변환**: RGB565 → RGB444 변환

## 🔧 구현 도전과제

### **1. 타이밍 정확성**
- **제약**: 정확한 VGA 타이밍
- **해결**: 하드웨어 기반 클럭 생성
- **최적화**: 정밀한 타이밍 제어

### **2. 메모리 대역폭**
- **제약**: 제한된 메모리 대역폭
- **해결**: 효율적인 메모리 접근
- **최적화**: 순차적 접근 패턴

### **3. 디스플레이 품질**
- **제약**: 높은 품질 요구
- **해결**: 하드웨어 기반 처리
- **최적화**: 실시간 처리

## 📝 다음 단계
- `12_Display_Modes.md`: 다양한 디스플레이 모드
- `01_ISP_Top_Overview.md`: 전체 시스템 아키텍처
- `02_Filter_Top_Architecture.md`: 필터 시스템 구조
