# BlueSpec SystemVerilog (BSV) 문법 가이드

## 1. BSV 개요

BlueSpec SystemVerilog는 하드웨어 설계를 위한 고수준 언어로, **규칙 기반 설계**와 **강력한 타입 시스템**을 특징으로 합니다.

### BSV의 핵심 개념
- **Rules**: 하드웨어 동작을 규칙으로 표현
- **Methods**: 인터페이스 메서드로 통신
- **Types**: 강력한 타입 시스템
- **Interfaces**: 모듈 간 통신 인터페이스

## 2. 기본 문법

### 2.1 모듈 선언
```bsv
// SystemVerilog 스타일
module mkAPBMaster(APBMaster);
    // 모듈 내용
endmodule

// BSV 스타일 (권장)
module mkAPBMaster#(parameter Integer addr_width = 32)(APBMaster);
    // 모듈 내용
endmodule
```

### 2.2 타입 정의
```bsv
// 열거형 (Enumeration)
typedef enum {
    Idle,
    Setup,
    Access
} APBState deriving (Bits, Eq, FShow);

// 구조체 (Structure)
typedef struct {
    Bit#(32) addr;
    Bit#(32) data;
    Bool     write;
} APBTransaction deriving (Bits, Eq, FShow);

// 유니온 타입
typedef union tagged {
    void     Idle;
    Bit#(32) Setup;
    Bit#(32) Access;
} APBStateUnion deriving (Bits, Eq, FShow);
```

### 2.3 인터페이스 정의
```bsv
// APB 마스터 인터페이스
interface APBMaster;
    method Action start_transaction(APBTransaction trans);
    method ActionValue#(Bit#(32)) get_response();
    method Bool is_ready();
endinterface

// APB 슬레이브 인터페이스
interface APBSlave;
    method Action put_request(APBTransaction trans);
    method ActionValue#(Bit#(32)) get_data();
    method Bool is_ready();
endinterface
```

## 3. 규칙 (Rules) 기반 설계

### 3.1 규칙 기본 문법
```bsv
// 규칙 정의
rule rule_name (condition);
    // 규칙 내용
endrule

// 예시: APB 상태 전환 규칙
rule handle_idle_to_setup (state == Idle && start_req);
    state <= Setup;
    addr_reg <= req.addr;
    data_reg <= req.data;
    write_reg <= req.write;
endrule
```

### 3.2 규칙 우선순위
```bsv
// 우선순위 지정
(* descending_urgency = "rule1, rule2, rule3" *)

rule rule1 (condition1);
    // 가장 높은 우선순위
endrule

rule rule2 (condition2);
    // 두 번째 우선순위
endrule
```

### 3.3 규칙 충돌 방지
```bsv
// 상호 배타적 조건
rule handle_setup (state == Setup && !setup_done);
    // SETUP 상태 처리
endrule

rule handle_access (state == Access && !access_done);
    // ACCESS 상태 처리
endrule
```

## 4. 메서드 (Methods)

### 4.1 메서드 타입
```bsv
// Action 메서드 (입력만)
method Action put_data(Bit#(32) data);

// Value 메서드 (출력만)
method Bit#(32) get_data();

// ActionValue 메서드 (입력/출력)
method ActionValue#(Bit#(32)) read_data(Bit#(32) addr);

// ActionValue#(void) 메서드 (입력만, 완료 신호)
method ActionValue#(void) write_data(Bit#(32) addr, Bit#(32) data);
```

### 4.2 메서드 구현
```bsv
// 메서드 구현 예시
method Action start_transaction(APBTransaction trans) if (state == Idle);
    state <= Setup;
    addr_reg <= trans.addr;
    data_reg <= trans.data;
    write_reg <= trans.write;
endmethod

method ActionValue#(Bit#(32)) get_response() if (state == Access && ready);
    state <= Idle;
    return data_reg;
endmethod
```

## 5. 레지스터와 메모리

### 5.1 레지스터
```bsv
// 단일 레지스터
Reg#(Bit#(32)) addr_reg <- mkReg(0);
Reg#(APBState) state_reg <- mkReg(Idle);

// 조건부 레지스터
Reg#(Bit#(32)) data_reg <- mkReg(0, clocked_by clk, reset_by rstn);

// 레지스터 배열
Vector#(4, Reg#(Bit#(32))) reg_bank <- replicateM(mkReg(0));
```

### 5.2 메모리
```bsv
// BRAM (Block RAM)
BRAM_Configure cfg = defaultValue;
BRAM1Port#(Bit#(10), Bit#(32)) memory <- mkBRAM1Server(cfg);

// 레지스터 파일
RegFile#(Bit#(4), Bit#(32)) regfile <- mkRegFileFull;
```

## 6. 조건부 실행

### 6.1 if 조건
```bsv
// 메서드 내 조건
method Action put_data(Bit#(32) data) if (state == Idle);
    if (data[31] == 1'b1) begin
        state <= Setup;
    end else begin
        state <= Error;
    end
endmethod
```

### 6.2 when 조건
```bsv
// 규칙 내 조건
rule handle_setup (state == Setup);
    when (addr_valid, noAction);
    state <= Access;
endrule
```

## 7. 파이프라인과 FIFO

### 7.1 FIFO
```bsv
// FIFO 생성
FIFO#(APBTransaction) req_fifo <- mkFIFO;
FIFO#(Bit#(32)) resp_fifo <- mkFIFO;

// FIFO 사용
rule enqueue_request (req_fifo.notFull);
    req_fifo.enq(transaction);
endrule

rule dequeue_response (resp_fifo.notEmpty);
    let data = resp_fifo.first;
    resp_fifo.deq;
endrule
```

### 7.2 파이프라인
```bsv
// 파이프라인 레지스터
Reg#(Bit#(32)) stage1 <- mkReg(0);
Reg#(Bit#(32)) stage2 <- mkReg(0);

rule stage1_rule (input_valid);
    stage1 <= input_data;
endrule

rule stage2_rule (stage1_valid);
    stage2 <= process(stage1);
endrule
```

## 8. 타입 변환과 패턴 매칭

### 8.1 타입 변환
```bsv
// 비트 벡터 변환
Bit#(32) data = 32'h12345678;
Bit#(8) byte_data = data[7:0];

// 정수 변환
Integer addr_int = unpack(addr_bits);
Bit#(32) addr_bits = pack(addr_int);
```

### 8.2 패턴 매칭
```bsv
// tagged union 패턴 매칭
case (state_union) matches
    tagged Idle: begin
        // Idle 상태 처리
    end
    tagged Setup .addr: begin
        // Setup 상태 처리, addr 값 사용
    end
    tagged Access .data: begin
        // Access 상태 처리, data 값 사용
    end
endcase
```

## 9. 컴파일 지시어

### 9.1 속성 지정
```bsv
// 규칙 속성
(* fire_when_enabled *)
rule always_fire;
    // 항상 실행되는 규칙
endrule

(* no_implicit_conditions *)
rule no_conditions;
    // 암시적 조건 없음
endrule
```

### 9.2 최적화 힌트
```bsv
// 병렬 실행 가능
(* mutually_exclusive = "rule1, rule2" *)
rule rule1 (condition1);
endrule

rule rule2 (condition2);
endrule
```

## 10. 디버깅과 시뮬레이션

### 10.1 디버그 출력
```bsv
// FShow 인스턴스 사용
$display("Current state: %0d", fshow(state));

// 조건부 디버그
`ifdef DEBUG
    $display("Debug: addr = %h", addr_reg);
`endif
```

### 10.2 시뮬레이션 제어
```bsv
// 시뮬레이션 종료
rule finish_simulation (cycle_count > 1000);
    $finish;
endrule
```

이 가이드를 통해 APB 설계를 BSV로 변환할 때 필요한 핵심 문법들을 익힐 수 있습니다.
