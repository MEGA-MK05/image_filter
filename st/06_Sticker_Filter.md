# Sticker Filter - 스티커 효과 필터

## 📋 개요
Sticker Filter는 AI Fourcut 프로젝트의 가장 복잡한 필터로, 객체 인식과 스티커 오버레이 기능을 제공합니다.

## 🎯 주요 기능

### **1. 객체 인식**
- **목적**: 특정 색상 영역 감지
- **원리**: 색상 기반 세그멘테이션
- **응용**: 얼굴, 손, 특정 객체 감지

### **2. 스티커 오버레이**
- **목적**: 감지된 객체에 스티커 적용
- **원리**: 알파 블렌딩과 좌표 변환
- **효과**: 실시간 스티커 효과

### **3. 사용자 제어**
- **버튼 제어**: 스티커 선택, 위치 조정
- **실시간 피드백**: 즉시 효과 확인
- **다양한 스티커**: 여러 스티커 옵션

## 🔍 객체 인식 알고리즘

### **색상 기반 세그멘테이션**
```systemverilog
// RGB565 → HSV 변환
logic [7:0] h, s, v;
rgb_to_hsv(wData_in, h, s, v);

// 피부색 감지 (HSV 범위)
logic skin_detect;
skin_detect = (h >= 8'd10) && (h <= 8'd25) &&    // 색상 범위
              (s >= 8'd50) && (s <= 8'd255) &&    // 채도 범위
              (v >= 8'd50) && (v <= 8'd255);     // 명도 범위

// 객체 영역 표시
if (skin_detect) begin
    object_pixel <= 16'hFFFF;  // 흰색으로 표시
end else begin
    object_pixel <= 16'h0000;  // 검은색으로 표시
end
```

### **연결된 구성 요소 분석**
```systemverilog
// 3×3 윈도우에서 연결성 분석
logic [8:0] connectivity;
connectivity = {p00, p01, p02, p10, p11, p12, p20, p21, p22};

// 연결된 픽셀 수 계산
logic [3:0] connected_count;
connected_count = p00 + p01 + p02 + p10 + p11 + p12 + p20 + p21 + p22;

// 객체 영역 판정
logic is_object;
is_object = (connected_count >= 4) && (p11 == 1'b1);
```

## 🎨 스티커 오버레이 시스템

### **스티커 데이터 구조**
```systemverilog
// 스티커 타입 정의
typedef enum {
    STICKER_NONE,
    STICKER_HEART,
    STICKER_STAR,
    STICKER_CAT,
    STICKER_DOG
} sticker_type_e;

// 스티커 파라미터
logic [7:0] sticker_x, sticker_y;  // 스티커 위치
logic [7:0] sticker_size;         // 스티커 크기
sticker_type_e current_sticker;   // 현재 스티커
```

### **스티커 렌더링**
```systemverilog
// 스티커 영역 감지
logic in_sticker_area;
in_sticker_area = (x >= sticker_x) && (x < (sticker_x + sticker_size)) &&
                  (y >= sticker_y) && (y < (sticker_y + sticker_size));

// 스티커 픽셀 계산
logic [15:0] sticker_pixel;
logic [7:0] sticker_x_local = x - sticker_x;
logic [7:0] sticker_y_local = y - sticker_y;

// 스티커 데이터 읽기
case (current_sticker)
    STICKER_HEART: sticker_pixel <= heart_data[sticker_y_local][sticker_x_local];
    STICKER_STAR:  sticker_pixel <= star_data[sticker_y_local][sticker_x_local];
    STICKER_CAT:   sticker_pixel <= cat_data[sticker_y_local][sticker_x_local];
    default:       sticker_pixel <= 16'h0000;
endcase
```

### **알파 블렌딩**
```systemverilog
// 스티커 투명도 처리
logic [4:0] alpha;
alpha = sticker_pixel[15:11];  // R 채널을 알파로 사용

// 알파 블렌딩
logic [4:0] blended_r, blended_g, blended_b;
blended_r = (sticker_pixel[15:11] * alpha + r5 * (5'd31 - alpha)) >> 5;
blended_g = (sticker_pixel[10:5] * alpha + g6 * (6'd63 - alpha)) >> 5;
blended_b = (sticker_pixel[4:0] * alpha + b5 * (5'd31 - alpha)) >> 5;
```

## 🎮 사용자 제어 시스템

### **버튼 인터페이스**
```systemverilog
// 버튼 입력
input logic btn_sticker;  // 스티커 토글
input logic btn_left;     // 왼쪽 이동
input logic btn_right;    // 오른쪽 이동

// 스티커 상태 관리
logic [2:0] sticker_index;
logic [7:0] sticker_x, sticker_y;

always_ff @(posedge clk) begin
    if (reset) begin
        sticker_index <= 3'b000;
        sticker_x <= 8'd160;  // 중앙
        sticker_y <= 8'd120;
    end else begin
        if (btn_sticker) begin
            sticker_index <= sticker_index + 1'b1;
        end
        if (btn_left) begin
            sticker_x <= (sticker_x > 8'd10) ? sticker_x - 8'd10 : 8'd0;
        end
        if (btn_right) begin
            sticker_x <= (sticker_x < 8'd250) ? sticker_x + 8'd10 : 8'd250;
        end
    end
end
```

### **스티커 선택**
```systemverilog
// 스티커 타입 선택
always_comb begin
    case (sticker_index)
        3'b000: current_sticker = STICKER_NONE;
        3'b001: current_sticker = STICKER_HEART;
        3'b010: current_sticker = STICKER_STAR;
        3'b011: current_sticker = STICKER_CAT;
        3'b100: current_sticker = STICKER_DOG;
        default: current_sticker = STICKER_NONE;
    endcase
end
```

## 📊 성능 최적화

### **메모리 최적화**
```systemverilog
// 스티커 데이터 압축
logic [7:0] sticker_rom[0:63][0:63];  // 64×64 스티커 데이터

// 압축된 스티커 데이터
always_comb begin
    case (sticker_pixel[7:0])
        8'h00: sticker_color = 16'h0000;  // 투명
        8'h01: sticker_color = 16'hF800;  // 빨간색
        8'h02: sticker_color = 16'h07E0;  // 녹색
        8'h03: sticker_color = 16'h001F;  // 파란색
        default: sticker_color = 16'h0000;
    endcase
end
```

### **파이프라인 처리**
```systemverilog
// 3단계 파이프라인
logic [15:0] pixel_d1, pixel_d2, pixel_d3;
logic [7:0] x_d1, x_d2, x_d3;
logic [7:0] y_d1, y_d2, y_d3;

always_ff @(posedge clk) begin
    // 1단계: 객체 감지
    pixel_d1 <= wData_in;
    x_d1 <= wAddr_in % IMG_WIDTH;
    y_d1 <= wAddr_in / IMG_WIDTH;
    
    // 2단계: 스티커 적용
    pixel_d2 <= pixel_d1;
    x_d2 <= x_d1;
    y_d2 <= y_d1;
    
    // 3단계: 최종 출력
    pixel_d3 <= pixel_d2;
    x_d3 <= x_d2;
    y_d3 <= y_d2;
end
```

## 🎯 학습 포인트

1. **객체 인식**: 색상 기반 세그멘테이션
2. **스티커 시스템**: 오버레이와 알파 블렌딩
3. **사용자 인터페이스**: 실시간 제어
4. **메모리 관리**: 스티커 데이터 저장
5. **성능 최적화**: 파이프라인 처리

## 🔧 구현 도전과제

### **1. 실시간 처리**
- 객체 인식과 스티커 적용을 동시에 처리
- 지연 시간 최소화

### **2. 메모리 효율성**
- 스티커 데이터 압축
- ROM 크기 최적화

### **3. 사용자 경험**
- 반응성 있는 버튼 제어
- 부드러운 스티커 이동

## 📝 다음 단계
- `07_Frame_Buffer_System.md`: 메모리 시스템 분석
- `08_Memory_Controllers.md`: 메모리 컨트롤러 분석
- `09_SCCB_Protocol.md`: 카메라 통신 프로토콜
