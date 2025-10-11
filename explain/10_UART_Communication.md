# UART 통신 시스템 - 원격 제어

## 📋 개요
UART 통신 시스템은 AI Fourcut 프로젝트의 원격 제어 기능을 제공하며, 사용자가 PC나 모바일 기기로 카메라를 제어할 수 있게 합니다.

## 🔧 UART 모듈 구조

### **인터페이스 정의**
```systemverilog
module UART (
    input  logic clk,              // 시스템 클럭
    input  logic reset,            // 리셋 신호
    input  logic rx,               // 수신 신호
    output logic tx,                // 송신 신호
    
    // 로컬 제어 입력
    input logic [3:0] sel,         // 필터 선택
    input logic [1:0] vga_sw,      // VGA 모드
    input logic btn_sticker,       // 스티커 버튼
    input logic btn_left,          // 왼쪽 버튼
    input logic btn_right,         // 오른쪽 버튼
    
    // 최종 제어 출력
    output logic [3:0] sel_final,        // 최종 필터 선택
    output logic [1:0] vga_sw_final,      // 최종 VGA 모드
    output logic btn_sticker_final,       // 최종 스티커 버튼
    output logic btn_right_final,        // 최종 오른쪽 버튼
    output logic btn_left_final          // 최종 왼쪽 버튼
);
```

## 🔄 통신 프로토콜

### **UART 설정**
- **보드레이트**: 9600 bps
- **데이터 비트**: 8비트
- **패리티**: 없음
- **스톱 비트**: 1비트
- **플로우 제어**: 없음

### **명령어 구조**
```
┌─────────┬─────────┬─────────┐
│  START  │  DATA   │  STOP   │
│  (1bit) │ (8bit)  │ (1bit)  │
└─────────┴─────────┴─────────┘
```

## 🎮 제어 명령어

### **필터 선택 명령어**
```systemverilog
always_ff @(posedge clk or posedge reset) begin
    if (reset) begin
        uart_btn_code <= 4'b0000;
    end else if (rx_done) begin
        case (rx_data)
            "0": uart_btn_code <= 4'b0000;  // 원본
            "1": uart_btn_code <= 4'b0001;  // Gaussian
            "2": uart_btn_code <= 4'b0010;  // Sharpen
            "3": uart_btn_code <= 4'b0011;  // Sobel
            "4": uart_btn_code <= 4'b0100;  // Warm
            "5": uart_btn_code <= 4'b0101;  // Cool
            "6": uart_btn_code <= 4'b0110;  // Graduation
            "7": uart_btn_code <= 4'b0111;  // Cartoon
            "8": uart_btn_code <= 4'b1000;  // Ghost
            "9": uart_btn_code <= 4'b1001;  // Mirror
            ".": uart_btn_code <= 4'b1010;  // Sticker
            default: uart_btn_code <= uart_btn_code;
        endcase
    end
end
```

### **디스플레이 모드 명령어**
```systemverilog
always_ff @(posedge clk or posedge reset) begin
    if (reset) begin
        uart_sw1415_code <= 2'b00;
    end else if (rx_done) begin
        case (rx_data)
            "z": uart_sw1415_code <= 2'b00;  // 원본
            "x": uart_sw1415_code <= 2'b01;  // 풀스크린
            "c": uart_sw1415_code <= 2'b10;  // 4컷
            default: uart_sw1415_code <= uart_sw1415_code;
        endcase
    end
end
```

### **버튼 제어 명령어**
```systemverilog
always_ff @(posedge clk or posedge reset) begin
    if (reset) begin
        uart_button_code <= 3'b100;  // idle
        hold_cnt <= 0;
    end else if (rx_done) begin
        case (rx_data)
            "s": uart_button_code <= 3'b000;  // 리셋
            "w": uart_button_code <= 3'b001;  // 스티커
            "a": uart_button_code <= 3'b010;  // 왼쪽
            "d": uart_button_code <= 3'b011;  // 오른쪽
        endcase
        hold_cnt <= 4'd7;  // 8클럭 정도 유지
    end else if (hold_cnt != 0) begin
        hold_cnt <= hold_cnt - 1;
        uart_button_code <= uart_button_code;  // 값 유지
    end else begin
        uart_button_code <= 3'b100;  // idle 복귀
    end
end
```

## 🔧 UART 하드웨어 구현

### **보드레이트 생성기**
```systemverilog
module baudrate_gen (
    input  logic clk,
    input  logic reset,
    output logic br_tick
);

logic [$clog2(100_000_000/9600/16)-1:0] br_counter;

always_ff @(posedge clk, posedge reset) begin
    if (reset) begin
        br_counter <= 0;
        br_tick <= 0;
    end else begin
        if (br_counter == 100_000_000 / 9600 / 16 - 1) begin
            br_counter <= 0;
            br_tick <= 1;
        end else begin
            br_counter <= br_counter + 1;
            br_tick <= 0;
        end
    end
end
```

### **수신기 (Receiver)**
```systemverilog
module receiver (
    input clk,
    input rst,
    input tick,
    input rx,
    output rx_done,
    output [7:0] rx_data
);

typedef enum {
    IDLE,
    START,
    DATA,
    STOP
} rx_state_e;

rx_state_e rx_state, rx_next_state;

// 상태 머신 구현
always_comb begin
    case (rx_state)
        IDLE: begin
            if (rx == 1'b0) begin
                rx_next_state = START;
            end
        end
        START: begin
            if (tick == 1'b1) begin
                if (tick_count_reg == 7) begin
                    rx_next_state = DATA;
                end
            end
        end
        DATA: begin
            if (tick == 1'b1) begin
                if (tick_count_reg == 15) begin
                    rx_data_next = {rx, rx_data_reg[7:1]};
                    if (bit_count_reg == 7) begin
                        rx_next_state = STOP;
                    end
                end
            end
        end
        STOP: begin
            if (tick == 1'b1) begin
                if (tick_count_reg == 23) begin
                    rx_done_next = 1'b1;
                    rx_next_state = IDLE;
                end
            end
        end
    endcase
end
```

### **송신기 (Transmitter)**
```systemverilog
module transmitter (
    input  logic       clk,
    input  logic       reset,
    input  logic       br_tick,
    input  logic [7:0] tx_data,
    input  logic       start,
    output logic       tx_busy,
    output logic       tx_done,
    output logic       tx
);

typedef enum {
    IDLE,
    START,
    DATA,
    STOP
} tx_state_e;

// 상태 머신 구현
always_comb begin
    case (tx_state)
        IDLE: begin
            if (start) begin
                tx_next_state = START;
                temp_data_next = tx_data;
            end
        end
        START: begin
            tx_next = 0;
            if (br_tick) begin
                if (tick_cnt_reg == 15) begin
                    tx_next_state = DATA;
                end
            end
        end
        DATA: begin
            tx_next = temp_data_reg[0];
            if (br_tick) begin
                if (tick_cnt_reg == 15) begin
                    if (bit_cnt_reg == 7) begin
                        tx_next_state = STOP;
                    end else begin
                        temp_data_next = {1'b0, temp_data_reg[7:1]};
                        bit_cnt_next = bit_cnt_reg + 1;
                    end
                end
            end
        end
        STOP: begin
            tx_next = 1;
            if (br_tick) begin
                if (tick_cnt_reg == 15) begin
                    tx_next_state = IDLE;
                    tx_done_next = 1;
                end
            end
        end
    endcase
end
```

## 🔄 제어 신호 통합

### **Bridge 모듈**
```systemverilog
module bridge (
    input logic [3:0] sel,         // 로컬 필터 선택
    input logic [3:0] filter_sel, // UART 필터 선택
    input logic [1:0] vga_sw,     // 로컬 VGA 모드
    input logic [1:0] cut_sel,     // UART VGA 모드
    input logic [2:0] btn_sel,    // UART 버튼
    input logic btn_sticker,      // 로컬 스티커 버튼
    input logic btn_left,         // 로컬 왼쪽 버튼
    input logic btn_right,        // 로컬 오른쪽 버튼
    
    output logic [3:0] sel_final,        // 최종 필터 선택
    output logic [1:0] vga_sw_final,      // 최종 VGA 모드
    output logic btn_sticker_final,       // 최종 스티커 버튼
    output logic btn_right_final,        // 최종 오른쪽 버튼
    output logic btn_left_final          // 최종 왼쪽 버튼
);

// UART 버튼 디코딩
logic btn_reset_uart = (btn_sel == 3'b000);    // "s"
logic btn_sticker_uart = (btn_sel == 3'b001);  // "w"
logic btn_left_uart = (btn_sel == 3'b010);     // "a"
logic btn_right_uart = (btn_sel == 3'b011);    // "d"

// 최종 제어 신호 생성
assign btn_sticker_final = btn_sticker | btn_sticker_uart;
assign btn_left_final = btn_left_uart | btn_left;
assign btn_right_final = btn_right_uart | btn_right;
assign sel_final = (sel != 4'b0000) ? sel : filter_sel;
assign vga_sw_final = (vga_sw != 2'b00) ? vga_sw : cut_sel;
```

## ⚡ 성능 특성

### **통신 속도**
- **보드레이트**: 9600 bps
- **비트 시간**: 104.17μs
- **바이트 시간**: 1.04ms
- **응답 시간**: 실시간

### **제어 지연**
- **UART 수신**: 1.04ms
- **명령어 처리**: 1클럭
- **제어 적용**: 즉시
- **전체 지연**: 1.04ms

## 🎯 학습 포인트

1. **UART 프로토콜**: 시리얼 통신 이해
2. **보드레이트 생성**: 정확한 클럭 생성
3. **상태 머신**: 복잡한 통신 제어
4. **제어 통합**: 다중 입력 소스 통합
5. **실시간 처리**: 하드웨어 기반 제어

## 🔧 구현 도전과제

### **1. 타이밍 정확성**
- **제약**: 정확한 보드레이트 생성
- **해결**: 하드웨어 기반 클럭 생성
- **최적화**: 정밀한 분주기

### **2. 제어 통합**
- **제약**: 다중 입력 소스
- **해결**: 우선순위 기반 선택
- **최적화**: 효율적인 제어 로직

### **3. 실시간 처리**
- **제약**: 지연 시간 최소화
- **해결**: 하드웨어 기반 처리
- **최적화**: 병렬 처리

## 📝 다음 단계
- `11_VGA_Display_System.md`: VGA 디스플레이 시스템
- `12_Display_Modes.md`: 다양한 디스플레이 모드
- `01_ISP_Top_Overview.md`: 전체 시스템 아키텍처
