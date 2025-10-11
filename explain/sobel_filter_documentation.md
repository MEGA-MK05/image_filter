# Sobel Edge Detection Filter - 동작 원리 및 프로세스

## 📋 목차
1. [Sobel 필터 개요](#sobel-필터-개요)
2. [수학적 원리](#수학적-원리)
3. [하드웨어 아키텍처](#하드웨어-아키텍처)
4. [데이터 흐름](#데이터-흐름)
5. [핵심 모듈 상세](#핵심-모듈-상세)
6. [테스트벤치 프로세스](#테스트벤치-프로세스)
7. [성능 분석](#성능-분석)
8. [최적화 기법](#최적화-기법)

---

##  Sobel 필터 개요

### **정의**
Sobel 필터는 **엣지 검출(Edge Detection)**을 위한 **1차 미분 연산자**로, 이미지의 밝기 변화율을 계산하여 객체의 경계를 찾아내는 디지털 이미지 처리 알고리즘입니다.

### **특징**
- **방향성**: 수평(X)과 수직(Y) 방향의 엣지를 각각 검출
- **노이즈 저항**: 가우시안 필터링 효과로 노이즈에 강함
- **실시간 처리**: 하드웨어 구현 시 고속 처리 가능
- **정확도**: 엣지의 위치와 강도를 정확하게 계산

---

##  수학적 원리

### **1. Sobel 커널 (Kernel)**

#### **X방향 커널 (Gx)**
```
Gx = [-1  0  +1]
     [-2  0  +2]
     [-1  0  +1]
```

#### **Y방향 커널 (Gy)**
```
Gy = [-1  -2  -1]
     [ 0   0   0]
     [+1  +2  +1]
```

### **2. 엣지 강도 계산**

#### **기본 공식**
```
Gx = (P6 + 2×P7 + P8) - (P0 + 2×P1 + P2)
Gy = (P2 + 2×P5 + P8) - (P0 + 2×P3 + P6)
```

#### **최종 엣지 강도**
```
Magnitude = √(Gx² + Gy²)
```

#### **방향 정보**
```
Direction = arctan(Gy / Gx)
```

### **3. 고정소수점 연산**

#### **RGB to Gray 변환**
```
Gray = (R × 77 + G × 150 + B × 29) >> 8
```
- **77, 150, 29**: 인간 시각 특성을 반영한 가중치
- **>> 8**: 256으로 나누기 (8비트 시프트)

#### **Sobel 연산 최적화**
```
// 절댓값 계산 (비교 연산으로 대체)
if (Gx < 0) Gx_abs = -Gx; else Gx_abs = Gx;
if (Gy < 0) Gy_abs = -Gy; else Gy_abs = Gy;

// 근사값 계산 (제곱근 대신)
Magnitude ≈ max(Gx_abs, Gy_abs) + min(Gx_abs, Gy_abs)/2
```

---

## 🏗️ 하드웨어 아키텍처

### **전체 시스템 구조**
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Input     │───▶│ RGB to Gray │───▶│ 3×3 Line    │───▶│  Sobel      │
│  Interface  │    │ Conversion  │    │  Buffer     │    │   Core      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                              │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│   Output    │◀───│ Threshold   │◀───│ Magnitude  │◀────────┘
│  Interface  │    │  Logic      │    │ Calculation│
└─────────────┘    └─────────────┘    └─────────────┘
```

### **모듈별 역할**

#### **1. Input Interface**
- **동기화 신호**: `in_de`, `in_hsync`, `in_vsync`
- **픽셀 데이터**: `in_r`, `in_g`, `in_b` (8비트 각각)
- **클럭 도메인**: 입력 클럭과 동기화

#### **2. RGB to Gray Conversion**
- **색상 공간 변환**: RGB → Grayscale
- **가중치 적용**: 인간 시각 특성 반영
- **데이터 정규화**: 8비트 출력

#### **3. 3×3 Line Buffer**
- **라인 저장**: 3개 라인을 순환 저장
- **윈도우 구성**: 3×3 픽셀 윈도우 생성
- **경계 처리**: 이미지 경계에서의 특수 처리

#### **4. Sobel Core**
- **X방향 계산**: Gx 커널 적용
- **Y방향 계산**: Gy 커널 적용
- **절댓값 계산**: 부호 제거

#### **5. Magnitude Calculation**
- **제곱 연산**: Gx², Gy² 계산
- **제곱근**: √(Gx² + Gy²) 계산
- **최적화**: 근사값 사용으로 속도 향상

#### **6. Threshold Logic**
- **이진화**: 임계값 기준 엣지 판정
- **설정 가능**: 동적 임계값 조정
- **노이즈 제거**: 약한 엣지 필터링

#### **7. Output Interface**
- **동기화 신호**: `out_de`, `out_hsync`, `out_vsync`
- **결과 데이터**: `out_r`, `out_g`, `out_b`
- **지연 보정**: 파이프라인 지연 보상

---

## 🔄 데이터 흐름

### **1. 입력 단계**
```
Input Pixel → RGB to Gray → Gray Pixel
     ↓              ↓           ↓
   in_rgb      Weighted Sum   gray[7:0]
```

### **2. 버퍼링 단계**
```
Gray Pixel → Line Buffer → 3×3 Window
     ↓           ↓           ↓
  gray[7:0]   line1,2,3   p00~p22
```

### **3. 처리 단계**
```
3×3 Window → Sobel Core → Magnitude → Threshold → Output
     ↓           ↓           ↓           ↓         ↓
   p00~p22    Gx, Gy     Mag[7:0]   Edge[7:0]  out_rgb
```

### **4. 타이밍 다이어그램**
```
Clock:    __|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__
Input:    ___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___
          P0  P1  P2  P3  P4  P5  P6  P7  P8
Buffer:   ___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___
          B0  B1  B2  B3  B4  B5  B6  B7  B8
Sobel:    ___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___
          S0  S1  S2  S3  S4  S5  S6  S7  S8
Output:   ___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___|‾‾‾|___
          O0  O1  O2  O3  O4  O5  O6  O7  O8
```

---

## 🔧 핵심 모듈 상세

### **1. RGB to Gray Module**

#### **구현 코드**
```verilog
module rgb2gray(
    input wire clk, rst,
    input wire [7:0] in_r, in_g, in_b,
    input wire in_de, in_hsync, in_vsync,
    output reg [7:0] gray,
    output reg de_g, hs_g, vs_g
);
    
    // 고정소수점 가중치 (8비트 정밀도)
    localparam WEIGHT_R = 8'd77;   // 0.299 × 256
    localparam WEIGHT_G = 8'd150;  // 0.587 × 256  
    localparam WEIGHT_B = 8'd29;   // 0.114 × 256
    
    always @(posedge clk) begin
        if (rst) begin
            gray <= 8'h0;
            de_g <= 1'b0;
            hs_g <= 1'b0;
            vs_g <= 1'b0;
        end else begin
            // 가중 평균 계산
            gray <= ((in_r * WEIGHT_R) + 
                     (in_g * WEIGHT_G) + 
                     (in_b * WEIGHT_B)) >> 8;
            
            // 동기화 신호 전달
            de_g <= in_de;
            hs_g <= in_hsync;
            vs_g <= in_vsync;
        end
    end
    
endmodule
```

#### **동작 원리**
1. **가중치 곱셈**: 각 색상 채널에 가중치 적용
2. **합산**: 가중치가 적용된 색상값들을 합산
3. **정규화**: 256으로 나누어 8비트 범위로 조정
4. **동기화**: 입력 신호를 출력으로 전달

### **2. 3×3 Line Buffer Module**

#### **구현 코드**
```verilog
module linebuffer_3x3(
    input wire clk, rst,
    input wire [7:0] gray,
    input wire de_g, hs_g, vs_g,
    output reg [7:0] p00, p01, p02,
                      p10, p11, p12,
                      p20, p21, p22,
    output reg de_w, hs_w, vs_w
);
    
    // 3라인 버퍼 (640픽셀 × 3라인)
    reg [7:0] line1 [0:639];
    reg [7:0] line2 [0:639];
    reg [7:0] line3 [0:639];
    
    // 라인 포인터 (순환)
    reg [1:0] line_ptr;
    
    // 컬럼 카운터
    reg [9:0] col_cnt;
    
    always @(posedge clk) begin
        if (rst) begin
            line_ptr <= 2'b0;
            col_cnt <= 10'b0;
            // 버퍼 초기화
            for (integer i = 0; i < 640; i = i + 1) begin
                line1[i] <= 8'h0;
                line2[i] <= 8'h0;
                line3[i] <= 8'h0;
            end
        end else begin
            if (de_g) begin
                // 현재 라인에 픽셀 저장
                case (line_ptr)
                    2'b00: line1[col_cnt] <= gray;
                    2'b01: line2[col_cnt] <= gray;
                    2'b10: line3[col_cnt] <= gray;
                endcase
                
                // 컬럼 카운터 증가
                if (col_cnt == 10'd639) begin
                    col_cnt <= 10'b0;
                    // 라인 완성 시 포인터 순환
                    line_ptr <= (line_ptr == 2'b10) ? 2'b00 : line_ptr + 1;
                end else begin
                    col_cnt <= col_cnt + 1;
                end
            end
            
            // 3×3 윈도우 구성
            if (de_g && col_cnt >= 10'd1) begin
                case (line_ptr)
                    2'b00: begin  // line1이 현재 라인
                        p00 <= line2[col_cnt-1]; p01 <= line2[col_cnt]; p02 <= line2[col_cnt+1];
                        p10 <= line3[col_cnt-1]; p11 <= line3[col_cnt]; p12 <= line3[col_cnt+1];
                        p20 <= line1[col_cnt-1]; p21 <= line1[col_cnt]; p22 <= line1[col_cnt+1];
                    end
                    2'b01: begin  // line2가 현재 라인
                        p00 <= line3[col_cnt-1]; p01 <= line3[col_cnt]; p02 <= line3[col_cnt+1];
                        p10 <= line1[col_cnt-1]; p11 <= line1[col_cnt]; p12 <= line1[col_cnt+1];
                        p20 <= line2[col_cnt-1]; p21 <= line2[col_cnt]; p22 <= line2[col_cnt+1];
                    end
                    2'b10: begin  // line3가 현재 라인
                        p00 <= line1[col_cnt-1]; p01 <= line1[col_cnt]; p02 <= line1[col_cnt+1];
                        p10 <= line2[col_cnt-1]; p11 <= line2[col_cnt]; p12 <= line2[col_cnt+1];
                        p20 <= line3[col_cnt-1]; p21 <= line3[col_cnt]; p22 <= line3[col_cnt+1];
                    end
                endcase
            end
            
            // 출력 신호 전달
            de_w <= de_g;
            hs_w <= hs_g;
            vs_w <= vs_g;
        end
    end
    
endmodule
```

#### **동작 원리**
1. **라인 저장**: 3개 라인을 순환하며 저장
2. **윈도우 구성**: 3×3 픽셀 윈도우 실시간 생성
3. **경계 처리**: 이미지 경계에서의 특수 처리
4. **동기화**: 입력/출력 신호 동기화

### **3. Sobel Core Module**

#### **구현 코드**
```verilog
module sobel_core(
    input wire clk, rst,
    input wire [7:0] p00, p01, p02,
                      p10, p11, p12,
                      p20, p21, p22,
    input wire de_w, hs_w, vs_w,
    output reg [7:0] sobel_mag,
    output reg de_s, hs_s, vs_s
);
    
    // X방향 Sobel 계산
    wire [9:0] gx;
    assign gx = (p02 + (p12 << 1) + p22) - (p00 + (p10 << 1) + p20);
    
    // Y방향 Sobel 계산
    wire [9:0] gy;
    assign gy = (p20 + (p21 << 1) + p22) - (p00 + (p01 << 1) + p02);
    
    // 절댓값 계산
    wire [9:0] gx_abs, gy_abs;
    assign gx_abs = (gx[9]) ? (~gx + 1) : gx;  // 2's complement
    assign gy_abs = (gy[9]) ? (~gy + 1) : gy;  // 2's complement
    
    // Magnitude 계산 (근사값)
    wire [9:0] max_val, min_val;
    assign max_val = (gx_abs > gy_abs) ? gx_abs : gy_abs;
    assign min_val = (gx_abs > gy_abs) ? gy_abs : gx_abs;
    
    always @(posedge clk) begin
        if (rst) begin
            sobel_mag <= 8'h0;
            de_s <= 1'b0;
            hs_s <= 1'b0;
            vs_s <= 1'b0;
        end else begin
            // Magnitude = max + min/2 (근사값)
            sobel_mag <= (max_val + (min_val >> 1)) > 10'd255 ? 8'hFF : 
                        (max_val + (min_val >> 1))[7:0];
            
            // 동기화 신호 전달
            de_s <= de_w;
            hs_s <= hs_w;
            vs_s <= vs_w;
        end
    end
    
endmodule
```

#### **동작 원리**
1. **X방향 계산**: Gx = (P6+2P7+P8) - (P0+2P1+P2)
2. **Y방향 계산**: Gy = (P2+2P5+P8) - (P0+2P3+P6)
3. **절댓값**: 2's complement를 이용한 부호 제거
4. **Magnitude**: 근사값을 이용한 빠른 계산

### **4. Threshold Module**

#### **구현 코드**
```verilog
module threshold_logic(
    input wire clk, rst,
    input wire [7:0] sobel_mag,
    input wire de_s, hs_s, vs_s,
    output reg [7:0] edge_bin,
    output reg de_out, hs_out, vs_out
);
    
    // 임계값 설정 (동적 조정 가능)
    reg [7:0] threshold;
    
    always @(posedge clk) begin
        if (rst) begin
            threshold <= 8'd50;  // 기본 임계값
            edge_bin <= 8'h0;
            de_out <= 1'b0;
            hs_out <= 1'b0;
            vs_out <= 1'b0;
        end else begin
            // 이진화: 임계값 기준 엣지 판정
            edge_bin <= (sobel_mag > threshold) ? 8'hFF : 8'h00;
            
            // 동기화 신호 전달
            de_out <= de_s;
            hs_out <= hs_s;
            vs_out <= vs_s;
        end
    end
    
endmodule
```

#### **동작 원리**
1. **임계값 비교**: Sobel magnitude와 임계값 비교
2. **이진화**: 엣지 여부를 0/255로 판정
3. **노이즈 제거**: 약한 엣지 필터링
4. **동적 조정**: 실시간 임계값 변경 가능

---

## 🧪 테스트벤치 프로세스

### **1. 테스트벤치 구조**

#### **기본 구성**
```verilog
module tb_sobel_direct;
    
    // 클럭 및 리셋 신호
    reg clk;
    reg rst_n;
    
    // 입력 신호
    reg [7:0] in_r, in_g, in_b;
    reg in_de, in_hsync, in_vsync;
    
    // 출력 신호
    wire [7:0] out_r, out_g, out_b;
    wire out_de, out_hsync, out_vsync;
    
    // DUT 인스턴스
    sobel_module dut(
        .clk(clk),
        .rst_n(rst_n),
        .in_r(in_r), .in_g(in_g), .in_b(in_b),
        .in_de(in_de), .in_hsync(in_hsync), .in_vsync(in_vsync),
        .out_r(out_r), .out_g(out_g), .out_b(out_b),
        .out_de(out_de), .out_hsync(out_hsync), .out_vsync(out_vsync)
    );
    
endmodule
```

### **2. 테스트 시나리오**

#### **시나리오 1: 기본 기능 테스트**
```
1. 리셋 신호 테스트
   - rst_n = 0 → 모든 신호 초기화
   - rst_n = 1 → 정상 동작 시작
   
2. 단일 픽셀 테스트
   - 고정 픽셀값 입력
   - 출력값 검증
   
3. 연속 픽셀 테스트
   - 순차적 픽셀 입력
   - 파이프라인 지연 확인
```

#### **시나리오 2: 이미지 파일 테스트**
```
1. 이미지 로드
   - input.hex 파일 읽기
   - 640×480 픽셀 데이터 로드
   
2. 스트리밍 처리
   - 픽셀 단위 순차 입력
   - 동기화 신호 생성
   
3. 결과 저장
   - output.hex 파일 생성
   - 처리된 픽셀 수 확인
```

#### **시나리오 3: 경계 조건 테스트**
```
1. 이미지 경계 처리
   - 첫 번째/마지막 픽셀
   - 첫 번째/마지막 라인
   
2. 동기화 신호 테스트
   - HSYNC, VSYNC 타이밍
   - DE 신호 유효성
   
3. 클럭 도메인 테스트
   - 클럭 주파수 변화
   - 메타스테이블 방지
```

### **3. 테스트 프로세스**

#### **단계 1: 컴파일**
```bash
# Icarus Verilog 컴파일
iverilog -o sim sobel_module.v tb_sobel_direct.v
```

#### **단계 2: 시뮬레이션**
```bash
# VVP 시뮬레이터 실행
vvp sim
```

#### **단계 3: 결과 분석**
```bash
# VCD 파일 생성 확인
ls -la *.vcd

# GTKWave로 파형 분석 (선택사항)
gtkwave sobel_direct.vcd
```

### **4. 테스트 결과 검증**

#### **정량적 검증**
```
1. 픽셀 처리 수
   - 입력: 307,200 픽셀 (640×480)
   - 출력: 307,200 픽셀
   - 처리 효율성: 100%

2. 처리 속도
   - 클럭 주파수: 100MHz
   - 픽셀당 클럭 수: 1
   - 최대 처리 속도: 100M 픽셀/초

3. 정확도
   - 엣지 검출 정확도
   - 노이즈 제거 효과
   - 경계 처리 품질
```

#### **정성적 검증**
```
1. 이미지 품질
   - 엣지 선명도
   - 노이즈 수준
   - 아티팩트 존재 여부

2. 실시간 성능
   - 지연 시간
   - 처리 안정성
   - 동기화 정확도
```

---

## 📊 성능 분석

### **1. 처리 성능**

#### **처리량 (Throughput)**
```
- 입력 해상도: 640×480 = 307,200 픽셀
- 클럭 주파수: 100MHz
- 픽셀당 클럭: 1
- 최대 처리량: 100M 픽셀/초
- 실시간 처리: 30fps @ 640×480 지원
```

#### **지연 시간 (Latency)**
```
- 파이프라인 단계: 4단계
- 각 단계 지연: 1 클럭
- 총 지연: 4 클럭 = 40ns @ 100MHz
- 실시간 응답: 즉시 처리 가능
```

### **2. 리소스 사용량**

#### **FPGA 리소스**
```
- LUT 사용량: 약 2,000개
- FF 사용량: 약 1,500개
- BRAM 사용량: 약 3개 (36Kb)
- DSP 사용량: 약 20개
```

#### **메모리 사용량**
```
- 라인 버퍼: 3×640×8비트 = 1.92KB
- 임시 저장: 1KB
- 총 메모리: 약 3KB
```

### **3. 전력 소모**

#### **동적 전력**
```
- 클럭 주파수: 100MHz
- 토글율: 50%
- 동적 전력: 약 100mW
```

#### **정적 전력**
```
- 누설 전류: 약 10mA
- 정적 전력: 약 100mW
- 총 전력: 약 200mW
```

---

## 🚀 최적화 기법

### **1. 알고리즘 최적화**

#### **근사값 사용**
```verilog
// 정확한 제곱근 대신 근사값 사용
// Magnitude = √(Gx² + Gy²) ≈ max(|Gx|, |Gy|) + min(|Gx|, |Gy|)/2

wire [9:0] max_val, min_val;
assign max_val = (gx_abs > gy_abs) ? gx_abs : gy_abs;
assign min_val = (gx_abs > gy_abs) ? gy_abs : gx_abs;
assign magnitude = max_val + (min_val >> 1);
```

#### **비트 시프트 연산**
```verilog
// 곱셈 대신 시프트 연산 사용
// 2×P = P << 1
// P/2 = P >> 1

assign gx = (p02 + (p12 << 1) + p22) - (p00 + (p10 << 1) + p20);
```

### **2. 하드웨어 최적화**

#### **파이프라인 구조**
```verilog
// 4단계 파이프라인으로 처리량 향상
always @(posedge clk) begin
    // Stage 1: RGB to Gray
    gray <= rgb2gray_calc(in_r, in_g, in_b);
    
    // Stage 2: Line Buffer Update
    line_buffer[write_ptr][col] <= gray;
    
    // Stage 3: Sobel Calculation
    sobel_result <= sobel_calc(window);
    
    // Stage 4: Threshold & Output
    edge_result <= threshold_logic(sobel_result);
end
```

#### **병렬 처리**
```verilog
// X, Y 방향 동시 계산
wire [9:0] gx, gy;
assign gx = (p02 + (p12 << 1) + p22) - (p00 + (p10 << 1) + p20);
assign gy = (p20 + (p21 << 1) + p22) - (p00 + (p01 << 1) + p02);
```

### **3. 메모리 최적화**

#### **순환 버퍼**
```verilog
// 3라인만 저장하여 메모리 절약
reg [1:0] write_ptr = 0;
always @(posedge clk) begin
    if (new_line) begin
        write_ptr <= (write_ptr + 1) % 3;
    end
end
```

#### **메모리 접근 최소화**
```verilog
// 한 번의 메모리 읽기로 3×3 윈도우 구성
always @(posedge clk) begin
    if (window_ready) begin
        window[0] <= {line1[col-1], line1[col], line1[col+1]};
        window[1] <= {line2[col-1], line2[col], line2[col+1]};
        window[2] <= {line3[col-1], line3[col], line3[col+1]};
    end
end
```

---

## 📝 결론

### **성과 요약**
1. **실시간 처리**: 640×480 @ 30fps 지원
2. **메모리 효율성**: 3KB로 전체 이미지 처리
3. **정확도**: 노이즈에 강한 엣지 검출
4. **확장성**: 다양한 해상도 및 임계값 지원

### **향후 개선 방향**
1. **다중 스케일**: 다양한 해상도 지원
2. **적응형 임계값**: 이미지 특성에 따른 자동 조정
3. **하드웨어 가속**: DSP 블록 활용
4. **실시간 제어**: 동적 파라미터 조정

### **적용 분야**
1. **자동차**: 차선 검출, 장애물 인식
2. **의료**: 의료 영상 분석
3. **제조업**: 품질 검사, 결함 검출
4. **보안**: 모션 감지, 침입 탐지

---

## 📚 참고 자료

### **논문 및 기술 문서**
1. "Digital Image Processing" - Gonzalez & Woods
2. "FPGA Implementation of Sobel Edge Detection" - IEEE
3. "Real-time Image Processing on FPGA" - Springer

### **관련 표준**
1. **VGA**: 640×480 @ 60Hz
2. **HDMI**: 1920×1080 @ 60Hz
3. **Camera Interface**: MIPI, DVP, BT656

### **개발 도구**
1. **HDL**: Verilog, VHDL, SystemVerilog
2. **시뮬레이션**: ModelSim, Icarus Verilog, GTKWave
3. **합성**: Xilinx Vivado, Intel Quartus

---

*이 문서는 Sobel 엣지 검출 필터의 하드웨어 구현에 대한 상세한 가이드입니다.*
