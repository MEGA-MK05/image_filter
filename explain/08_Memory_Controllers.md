# 메모리 컨트롤러들 - OV7670, VGA

## 📋 개요
이 문서는 AI Fourcut 프로젝트의 메모리 컨트롤러들(OV7670_MemController, VGA_MemController)의 원리와 구현을 상세히 분석합니다.

## 📷 OV7670_MemController

### **목적과 기능**
- **목적**: OV7670 카메라 데이터를 메모리에 저장
- **기능**: 8비트 스트림을 16비트 픽셀로 변환
- **특징**: 실시간 데이터 변환

### **인터페이스 정의**
```systemverilog
module OV7670_MemController #(
    parameter int IMG_W = 320,
    parameter int IMG_H = 240
) (
    input  logic        clk,         // 카메라 픽셀 클럭
    input  logic        reset,       // 리셋 신호
    input  logic        href,        // 수평 참조 신호
    input  logic        vsync,      // 수직 동기 신호
    input  logic [ 7:0]  ov7670_data, // 8비트 픽셀 데이터
    output logic        we,          // Write Enable
    output logic [16:0] wAddr,       // Write Address
    output logic [15:0] wData        // Write Data (RGB565)
);
```

### **데이터 변환 과정**

#### **1단계: 8비트 → 16비트 변환**
```systemverilog
// 내부 카운터
logic [ 9:0] h_counter;  // 0~639 (짝수/홀수 byte)
logic [ 7:0] v_counter;  // 0~239
logic [15:0] pixel_data;

// 픽셀 조립
always_ff @(posedge clk) begin
    if (href) begin
        h_counter <= h_counter + 1;
        
        if (h_counter[0] == 0) begin
            // 짝수 byte = MSB 먼저
            pixel_data[15:8] <= ov7670_data;
            we <= 1'b0;  // 아직 픽셀 미완성
        end else begin
            // 홀수 byte = LSB, 픽셀 완성
            pixel_data[7:0] <= ov7670_data;
            wData <= {pixel_data[15:8], ov7670_data};
            we <= 1'b1;  // Write Enable
        end
    end
end
```

#### **2단계: 주소 계산**
```systemverilog
// 주소 계산
wire [16:0] addr_next = v_counter * IMG_W + h_counter[9:1];

// 세로 카운터
always_ff @(posedge clk) begin
    if (reset) begin
        v_counter <= 0;
    end else begin
        if (vsync) begin
            v_counter <= 0;  // 프레임 시작
        end else begin
            if (h_counter == (IMG_W * 2 - 1)) begin
                v_counter <= v_counter + 1;
            end
        end
    end
end
```

### **타이밍 다이어그램**
```
vsync:  ┌─────┐           ┌─────┐
        │     │           │     │
        └─────┴───────────┴─────┴───

href:   ┌─┐ ┌─┐ ┌─┐     ┌─┐ ┌─┐ ┌─┐
        │ │ │ │ │ │     │ │ │ │ │ │
        └─┘ └─┘ └─┘     └─┘ └─┘ └─┘

data:   MSB LSB MSB LSB MSB LSB
        │   │   │   │   │   │
        └───┴───┴───┴───┴───┴───
```

## 🖥️ VGA_MemController

### **목적과 기능**
- **목적**: 메모리에서 VGA 출력용 데이터 읽기
- **기능**: 3가지 디스플레이 모드 지원
- **특징**: 다양한 해상도 지원

### **인터페이스 정의**
```systemverilog
module VGA_MemController (
    input  logic [ 1:0] vga_sw,      // 디스플레이 모드 선택
    input  logic        DE,           // Data Enable
    input  logic [ 9:0] x_pixel,      // VGA X 좌표
    input  logic [ 9:0] y_pixel,      // VGA Y 좌표
    output logic        den,          // Data Enable
    output logic [16:0] rAddr,       // Read Address
    input  logic [15:0] rData,        // Read Data
    output logic [ 3:0] r_port,       // Red Output
    output logic [ 3:0] g_port,       // Green Output
    output logic [ 3:0] b_port        // Blue Output
);
```

### **디스플레이 모드**

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

assign den = DE && x_pixel < 320 && y_pixel < 240;
assign rAddr = den ? (y_pixel * 320 + x_pixel) : 17'bz;
assign {r_port, g_port, b_port} = den ? {rData[15:12], rData[10:7], rData[4:1]} : 12'b0;
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

## 🔄 데이터 흐름

### **카메라 → 메모리**
```
OV7670 → OV7670_MemController → Frame_Buffer
```

### **메모리 → VGA**
```
Frame_Buffer → VGA_MemController → VGA 출력
```

### **이미지 처리**
```
카메라 데이터 → Filter_Top → 필터 적용 → Frame_Buffer
```

## ⚡ 성능 특성

### **OV7670_MemController**
- **입력 속도**: 320×240×30fps = 2.3M 픽셀/초
- **데이터 변환**: 8비트 → 16비트
- **지연 시간**: 1클럭

### **VGA_MemController**
- **출력 속도**: 640×480×60fps = 18.4M 픽셀/초
- **디스플레이 모드**: 3가지 모드 지원
- **지연 시간**: 1클럭

## 🔧 최적화 기법

### **1. 메모리 접근 최적화**
```systemverilog
// 순차적 메모리 접근
// 캐시 친화적 패턴
// 버스트 전송 활용
```

### **2. 데이터 변환 최적화**
```systemverilog
// 하드웨어 기반 변환
// 파이프라인 처리
// 병렬 연산
```

### **3. 타이밍 최적화**
```systemverilog
// 클럭 도메인 분리
// 지연 시간 최소화
// 실시간 처리
```

## 🎯 학습 포인트

1. **데이터 변환**: 8비트 → 16비트 변환
2. **주소 계산**: 2D 좌표 → 1D 주소
3. **디스플레이 모드**: 다양한 해상도 지원
4. **메모리 접근**: 효율적인 메모리 사용
5. **실시간 처리**: 하드웨어 기반 처리

## 🔧 구현 도전과제

### **1. 데이터 동기화**
- **제약**: 카메라와 VGA 간의 속도 차이
- **해결**: 중간 버퍼 사용
- **최적화**: 파이프라인 구조

### **2. 메모리 대역폭**
- **제약**: 제한된 메모리 대역폭
- **해결**: 효율적인 메모리 접근
- **최적화**: 순차적 접근 패턴

### **3. 실시간 처리**
- **제약**: 지연 시간 최소화
- **해결**: 하드웨어 기반 처리
- **최적화**: 병렬 처리

## 📝 다음 단계
- `09_SCCB_Protocol.md`: 카메라 통신 프로토콜
- `10_UART_Communication.md`: UART 통신 시스템
- `11_VGA_Display_System.md`: VGA 디스플레이 시스템
