# APB Master/Slave 구조 분석

## 1. APB (Advanced Peripheral Bus) 개요

APB는 ARM에서 정의한 간단한 메모리 매핑 인터페이스로, 저전력과 낮은 복잡성을 특징으로 합니다.

### APB의 특징
- **단순한 2-phase 프로토콜**: SETUP → ACCESS
- **동기식 인터페이스**: 모든 신호가 클럭에 동기화
- **낮은 전력 소모**: 복잡한 핸드셰이킹 없음
- **제한된 성능**: 한 번에 하나의 트랜잭션만 처리

## 2. APB Master 구조 분석

### 2.1 신호 인터페이스
```verilog
// Global Signals
input  logic        PCLK,    // APB 클럭
input  logic        PRESET,  // APB 리셋 (active high)

// APB Master Outputs
output logic [31:0] PADDR,   // 주소 버스
output logic        PWRITE,  // 쓰기 신호 (1: write, 0: read)
output logic        PENABLE, // 활성화 신호
output logic [31:0] PWDATA,  // 쓰기 데이터
output logic [4:0]  PSELx,   // 슬레이브 선택 신호 (5개 슬레이브)

// APB Master Inputs
input  logic [31:0] PRDATAx, // 읽기 데이터 (5개 슬레이브)
input  logic [4:0]  PREADYx, // 준비 신호 (5개 슬레이브)
```

### 2.2 상태 머신
```verilog
typedef enum {
    IDLE,   // 대기 상태
    SETUP,  // 설정 상태 (PSEL=1, PENABLE=0)
    ACCESS  // 접근 상태 (PSEL=1, PENABLE=1)
} apb_state_e;
```

### 2.3 주요 모듈 구성
1. **APB_Master**: 메인 상태 머신과 제어 로직
2. **APB_Decoder**: 주소 디코딩 (어떤 슬레이브 선택할지)
3. **APB_Mux**: 다중화기 (슬레이브 응답 선택)

## 3. APB Slave 구조 분석

### 3.1 신호 인터페이스
```verilog
// Global Signals
input  logic        PCLK,    // APB 클럭
input  logic        PRESET,  // APB 리셋 (active high)

// APB Slave Inputs
input  logic [3:0]  PADDR,   // 주소 버스 (4비트)
input  logic        PWRITE,  // 쓰기 신호
input  logic        PENABLE, // 활성화 신호
input  logic [31:0] PWDATA,  // 쓰기 데이터
input  logic        PSEL,    // 선택 신호

// APB Slave Outputs
output logic [31:0] PRDATA,  // 읽기 데이터
output logic        PREADY   // 준비 신호
```

### 3.2 레지스터 뱅크
```verilog
logic [31:0] slv_reg0, slv_reg1, slv_reg2, slv_reg3;
```

### 3.3 주소 매핑
- `PADDR[3:2]`로 레지스터 선택
- 4개의 32비트 레지스터 지원

## 4. APB 프로토콜 타이밍

### 4.1 쓰기 트랜잭션
```
클럭:  |  1  |  2  |  3  |
PSEL:  |  0  |  1  |  1  |
PENABLE:|  0  |  0  |  1  |
PWRITE: |  X  |  1  |  1  |
PADDR:  |  X  | ADDR| ADDR|
PWDATA: |  X  | DATA| DATA|
PREADY: |  X  |  X  |  1  |
```

### 4.2 읽기 트랜잭션
```
클럭:  |  1  |  2  |  3  |
PSEL:  |  0  |  1  |  1  |
PENABLE:|  0  |  0  |  1  |
PWRITE: |  X  |  0  |  0  |
PADDR:  |  X  | ADDR| ADDR|
PRDATA: |  X  |  X  | DATA|
PREADY: |  X  |  X  |  1  |
```

## 5. 메모리 맵핑

### 5.1 주소 디코딩
```verilog
casex (sel)
    32'h1000_0xxx: y = 4'b0001;  // 슬레이브 0
    32'h1000_1xxx: y = 4'b0010;  // 슬레이브 1
    32'h1000_2xxx: y = 4'b0100;  // 슬레이브 2
    32'h1000_3xxx: y = 4'b1000;  // 슬레이브 3
endcase
```

### 5.2 주소 공간
- 각 슬레이브는 4KB 주소 공간 할당
- 슬레이브 내부는 4바이트 단위로 레지스터 접근

## 6. BSV로 변환 시 고려사항

### 6.1 규칙 기반 설계
- APB 상태 머신을 BSV 규칙으로 변환
- 각 상태별로 별도 규칙 정의

### 6.2 메서드 인터페이스
- APB 신호들을 메서드로 추상화
- 읽기/쓰기 메서드 분리

### 6.3 타입 시스템 활용
- APB 상태를 열거형으로 정의
- 주소와 데이터 타입 명시적 정의
