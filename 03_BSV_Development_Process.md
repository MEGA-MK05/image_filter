# BSV 개발 프로세스 가이드

## 1. APB → BSV 변환 단계별 프로세스

### 1단계: 인터페이스 설계
**목표**: SystemVerilog 신호를 BSV 메서드로 추상화

#### 1.1 APB Master 인터페이스 설계
```bsv
// SystemVerilog 신호 분석
// input  logic        transfer, ready, write
// input  logic [31:0] addr, wdata
// output logic [31:0] rdata

// BSV 인터페이스로 변환
interface APBMasterIfc;
    // 트랜잭션 시작
    method Action start_transaction(Bit#(32) addr, Bit#(32) data, Bool write);
    
    // 응답 받기
    method ActionValue#(Bit#(32)) get_response();
    
    // 상태 확인
    method Bool is_ready();
    method Bool is_done();
endinterface
```

#### 1.2 APB Slave 인터페이스 설계
```bsv
// SystemVerilog 신호 분석
// input  logic [3:0]  PADDR
// input  logic        PWRITE, PENABLE, PSEL
// input  logic [31:0] PWDATA
// output logic [31:0] PRDATA
// output logic        PREADY

// BSV 인터페이스로 변환
interface APBSlaveIfc;
    // 요청 받기
    method Action put_request(Bit#(4) addr, Bit#(32) data, Bool write, Bool enable, Bool select);
    
    // 응답 보내기
    method ActionValue#(Bit#(32)) get_data();
    
    // 준비 상태
    method Bool is_ready();
endinterface
```

### 2단계: 타입 정의
**목표**: 하드웨어 데이터 구조를 BSV 타입으로 정의

```bsv
// APB 상태 정의
typedef enum {
    Idle,
    Setup,
    Access
} APBState deriving (Bits, Eq, FShow);

// APB 트랜잭션 정의
typedef struct {
    Bit#(32) addr;
    Bit#(32) data;
    Bool     write;
} APBTransaction deriving (Bits, Eq, FShow);

// APB 응답 정의
typedef struct {
    Bit#(32) data;
    Bool     error;
} APBResponse deriving (Bits, Eq, FShow);
```

### 3단계: 상태 머신을 규칙으로 변환
**목표**: SystemVerilog 상태 머신을 BSV 규칙으로 변환

#### 3.1 상태 레지스터 정의
```bsv
// SystemVerilog: reg [1:0] state, state_next;
// BSV 변환:
Reg#(APBState) state_reg <- mkReg(Idle);
Reg#(Bit#(32)) addr_reg <- mkReg(0);
Reg#(Bit#(32)) data_reg <- mkReg(0);
Reg#(Bool) write_reg <- mkReg(False);
```

#### 3.2 상태 전환 규칙
```bsv
// SystemVerilog always_comb 블록을 BSV 규칙으로 변환
rule handle_idle_to_setup (state_reg == Idle && start_req);
    state_reg <= Setup;
    addr_reg <= req_addr;
    data_reg <= req_data;
    write_reg <= req_write;
endrule

rule handle_setup_to_access (state_reg == Setup);
    state_reg <= Access;
endrule

rule handle_access_to_idle (state_reg == Access && slave_ready);
    state_reg <= Idle;
    if (!write_reg) begin
        resp_data <= slave_data;
    end
endrule
```

### 4단계: 메서드 구현
**목표**: 인터페이스 메서드를 구체적으로 구현

```bsv
// 트랜잭션 시작 메서드
method Action start_transaction(Bit#(32) addr, Bit#(32) data, Bool write) if (state_reg == Idle);
    req_addr <= addr;
    req_data <= data;
    req_write <= write;
    start_req <= True;
endmethod

// 응답 받기 메서드
method ActionValue#(Bit#(32)) get_response() if (state_reg == Idle && resp_valid);
    resp_valid <= False;
    return resp_data;
endmethod

// 상태 확인 메서드
method Bool is_ready();
    return (state_reg == Idle);
endmethod
```

### 5단계: 디코더와 멀티플렉서 구현
**목표**: 주소 디코딩과 데이터 멀티플렉싱을 BSV로 구현

#### 5.1 주소 디코더
```bsv
// SystemVerilog casex 문을 BSV로 변환
function Bit#(5) decode_address(Bit#(32) addr);
    case (addr[15:12]) matches
        4'h0: return 5'b00001;  // 슬레이브 0
        4'h1: return 5'b00010;  // 슬레이브 1
        4'h2: return 5'b00100;  // 슬레이브 2
        4'h3: return 5'b01000;  // 슬레이브 3
        4'h4: return 5'b10000;  // 슬레이브 4
        default: return 5'b00000;
    endcase
endfunction
```

#### 5.2 데이터 멀티플렉서
```bsv
// SystemVerilog case 문을 BSV로 변환
function Bit#(32) mux_data(Bit#(2) sel, Vector#(5, Bit#(32)) data_vec);
    return data_vec[sel];
endfunction
```

### 6단계: 레지스터 뱅크 구현
**목표**: 슬레이브의 레지스터 뱅크를 BSV로 구현

```bsv
// SystemVerilog reg [31:0] slv_reg0, slv_reg1, slv_reg2, slv_reg3;
// BSV 변환:
Vector#(4, Reg#(Bit#(32))) reg_bank <- replicateM(mkReg(0));

// 레지스터 접근 규칙
rule handle_register_access (psel && penable);
    Bit#(2) reg_addr = paddr[3:2];
    
    if (pwrite) begin
        // 쓰기 동작
        reg_bank[reg_addr] <= pwdata;
    end else begin
        // 읽기 동작
        prdata <= reg_bank[reg_addr];
    end
    
    pready <= True;
endrule
```

## 2. 개발 순서 및 체크리스트

### Phase 1: 기본 구조 설정
- [ ] 인터페이스 정의
- [ ] 타입 정의
- [ ] 기본 모듈 구조 생성

### Phase 2: 상태 머신 구현
- [ ] 상태 레지스터 정의
- [ ] 상태 전환 규칙 작성
- [ ] 조건부 실행 로직 구현

### Phase 3: 메서드 구현
- [ ] 입력 메서드 구현
- [ ] 출력 메서드 구현
- [ ] 상태 확인 메서드 구현

### Phase 4: 하드웨어 로직 구현
- [ ] 디코더 구현
- [ ] 멀티플렉서 구현
- [ ] 레지스터 뱅크 구현

### Phase 5: 테스트 및 검증
- [ ] 단위 테스트 작성
- [ ] 통합 테스트 작성
- [ ] 시뮬레이션 실행

## 3. 주의사항 및 팁

### 3.1 규칙 충돌 방지
```bsv
// 상호 배타적 조건 사용
rule rule1 (state == Idle && condition1);
    // 처리
endrule

rule rule2 (state == Setup && condition2);
    // 처리
endrule
```

### 3.2 메서드 조건
```bsv
// 메서드 실행 조건 명시
method Action put_data(Bit#(32) data) if (fifo.notFull);
    fifo.enq(data);
endmethod
```

### 3.3 타입 안전성
```bsv
// 명시적 타입 변환
Bit#(32) data = zeroExtend(byte_data);
Integer count = unpack(counter);
```

### 3.4 디버깅
```bsv
// FShow 인스턴스 활용
$display("State: %0s", fshow(current_state));
$display("Transaction: %0s", fshow(transaction));
```

## 4. 컴파일 및 시뮬레이션

### 4.1 컴파일 명령어
```bash
# BSV 컴파일
bsc -u -sim APBMaster.bsv
bsc -u -sim APBSlave.bsv

# 실행 파일 생성
bsc -sim -e mkTestBench -o testbench
```

### 4.2 시뮬레이션 실행
```bash
# 시뮬레이션 실행
./testbench

# VCD 파일 생성 (디버깅용)
bsc -sim -D VCD -e mkTestBench -o testbench_vcd
```

이 프로세스를 따라하면 SystemVerilog APB 설계를 BSV로 성공적으로 변환할 수 있습니다.
