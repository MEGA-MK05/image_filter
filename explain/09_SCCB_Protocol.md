# SCCB 프로토콜 - 카메라 통신 시스템

## 📋 개요
SCCB(Serial Camera Control Bus)는 OV7670 카메라와의 통신을 위한 프로토콜로, I2C 기반의 시리얼 통신 방식을 사용합니다.

## 🔧 SCCB 프로토콜 구조

### **기본 구조**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   SCCB          │    │   I2C           │    │   OV7670        │
│   Controller    │◄──►│   Protocol      │◄──►│   Camera        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### **통신 방식**
- **프로토콜**: I2C 기반
- **클럭 속도**: 400kHz
- **데이터 폭**: 8비트
- **주소**: 7비트 (0x42)

## 🔍 SCCB 모듈 분석

### **인터페이스 정의**
```systemverilog
module SCCB (
    input  logic clk,      // 시스템 클럭
    input  logic reset,    // 리셋 신호
    output logic SCL,      // 시리얼 클럭
    output logic SDA       // 시리얼 데이터
);
```

### **주요 구성 요소**

#### **1. I2C 클럭 생성기**
```systemverilog
module I2C_clk_gen (
    input  logic clk,
    input  logic reset,
    input  logic I2C_clk_en,
    output logic I2C_clk_400khz
);

logic [$clog2(250):0] counter;

always_ff @(posedge clk) begin
    if (reset) begin
        I2C_clk_400khz <= 0;
        counter <= 0;
    end else begin
        if (I2C_clk_en) begin
            if (counter == 250) begin
                I2C_clk_400khz <= 1;
                counter <= 0;
            end else begin
                I2C_clk_400khz <= 0;
                counter <= counter + 1;
            end
        end else begin
            I2C_clk_400khz <= 0;
            counter <= 0;
        end
    end
end
```

#### **2. SCCB 제어 유닛**
```systemverilog
module SCCB_controlUnit (
    input  logic        clk,
    input  logic        reset,
    input  logic        I2C_clk_400khz,
    input  logic [23:0] initData,    // 24비트 데이터
    input  logic        startSig,    // 시작 신호
    output logic        SCL,         // 시리얼 클럭
    output logic        SDA,         // 시리얼 데이터
    output logic        I2C_clk_en,  // 클럭 활성화
    output logic [ 7:0] addr          // 주소
);
```

## 🔄 통신 프로토콜

### **I2C 프레임 구조**
```
┌─────────┬─────────┬─────────┬─────────┐
│  START  │  ADDR   │  DATA   │  STOP   │
│  (1bit) │ (8bit)  │ (8bit)  │ (1bit)  │
└─────────┴─────────┴─────────┴─────────┘
```

### **상태 머신**

#### **SCL 상태 머신**
```systemverilog
typedef enum {
    SCL_IDLE,    // 대기
    SCL_START,   // 시작
    SCL_H2L,     // High to Low
    SCL_L2L,     // Low to Low
    SCL_L2H,     // Low to High
    SCL_H2H,     // High to High
    SCL_STOP     // 정지
} scl_e;
```

#### **SDA 상태 머신**
```systemverilog
typedef enum {
    SDA_IDLE,        // 대기
    SDA_START,       // 시작
    DEVICE_ID,       // 디바이스 ID
    ADDRESS_REG,     // 주소 레지스터
    DATA_REG,        // 데이터 레지스터
    SDA_STOP         // 정지
} sda_e;
```

### **통신 시퀀스**
```systemverilog
// 1. START 조건
if (sda_state == SDA_START) begin
    r_I2C_clk_en <= 1'b1;
    scl_state <= SCL_START;
end

// 2. 디바이스 ID 전송
if (scl_state == SCL_H2L) begin
    if (I2C_clk_400khz) begin
        r_sda_next = initData[dataBit];
        dataBit_next = dataBit - 1;
    end
end

// 3. 주소 레지스터 전송
if (sda_state == ADDRESS_REG) begin
    r_sda_next = initData[dataBit];
    dataBit_next = dataBit - 1;
end

// 4. 데이터 레지스터 전송
if (sda_state == DATA_REG) begin
    r_sda_next = initData[dataBit];
    dataBit_next = dataBit - 1;
end
```

## 📊 OV7670 설정 데이터

### **설정 레지스터**
```systemverilog
module OV7670_config_rom (
    input logic clk,
    input logic [7:0] addr,
    output logic [15:0] dout
);

always @(posedge clk) begin
    case (addr)
        0: dout <= 16'h12_80;  // reset
        1: dout <= 16'hFF_F0;  // delay
        2: dout <= 16'h12_14;  // COM7, RGB color output, QVGA
        3: dout <= 16'h11_80;  // CLKRC, internal PLL
        4: dout <= 16'h0C_04;  // COM3, default settings
        5: dout <= 16'h3E_19;  // COM14, no scaling
        6: dout <= 16'h04_00;  // COM1, disable CCIR656
        7: dout <= 16'h40_d0;  // COM15, RGB565, full range
        8: dout <= 16'h3a_04;  // TSLB
        9: dout <= 16'h14_18;  // COM9, MAX AGC value x4
        10: dout <= 16'h4F_B3; // MTX1
        // ... 더 많은 설정들
        default: dout <= 16'hFF_FF;  // end of ROM
    endcase
end
```

### **주요 설정 파라미터**

| 레지스터 | 값 | 기능 |
|----------|-----|------|
| 0x12 | 0x80 | 리셋 |
| 0x12 | 0x14 | RGB 출력, QVGA |
| 0x11 | 0x80 | 내부 PLL |
| 0x40 | 0xd0 | RGB565, 전체 범위 |
| 0x3a | 0x04 | TSLB 설정 |

## ⚡ 타이밍 특성

### **클럭 생성**
```systemverilog
// 100MHz → 400kHz
// 분주비: 250
// 주기: 2.5μs
```

### **통신 속도**
- **클럭 주기**: 2.5μs
- **비트 전송**: 8비트
- **레지스터 설정**: 24비트
- **전체 시간**: 60μs per register

### **초기화 시간**
- **총 레지스터**: 75개
- **초기화 시간**: 4.5ms
- **실시간 처리**: 가능

## 🔧 최적화 기법

### **1. 하드웨어 최적화**
```systemverilog
// 상태 머신 기반 처리
// 병렬 데이터 처리
// 파이프라인 구조
```

### **2. 메모리 최적화**
```systemverilog
// ROM 기반 설정 데이터
// 압축된 데이터 형식
// 효율적인 주소 매핑
```

### **3. 타이밍 최적화**
```systemverilog
// 정확한 클럭 생성
// 동기화 처리
// 지연 시간 최소화
```

## 🎯 학습 포인트

1. **I2C 프로토콜**: 시리얼 통신 이해
2. **상태 머신**: 복잡한 통신 제어
3. **클럭 생성**: 정확한 타이밍
4. **데이터 설정**: 카메라 초기화
5. **하드웨어 최적화**: 효율적인 구현

## 🔧 구현 도전과제

### **1. 타이밍 정확성**
- **제약**: 정확한 클럭 생성
- **해결**: 하드웨어 기반 클럭 생성
- **최적화**: 정밀한 분주기

### **2. 상태 동기화**
- **제약**: 복잡한 상태 전환
- **해결**: 상태 머신 설계
- **최적화**: 병렬 처리

### **3. 데이터 무결성**
- **제약**: 통신 오류 방지
- **해결**: 체크섬 검증
- **최적화**: 재전송 메커니즘

## 📝 다음 단계
- `10_UART_Communication.md`: UART 통신 시스템
- `11_VGA_Display_System.md`: VGA 디스플레이 시스템
- `12_Display_Modes.md`: 다양한 디스플레이 모드
