# 특수 필터들 - Ghost, Mirror, Graduation

## 📋 개요
이 문서는 AI Fourcut 프로젝트의 특수 효과 필터들(Ghost, Mirror, Graduation)의 원리와 구현을 상세히 분석합니다.

## 👻 Ghost Filter (유령 효과 필터)

### **목적과 원리**
- **목적**: 투명도 기반 유령 효과
- **원리**: 알파 블렌딩과 색상 조정
- **효과**: 반투명한 유령 같은 이미지

### **처리 과정**

#### **1단계: 투명도 계산**
```systemverilog
// 픽셀 밝기 기반 투명도 계산
logic [7:0] brightness;
brightness = (r5 << 2) + (g6 << 1) + b5;  // 가중치 합

// 투명도 매핑 (밝을수록 투명)
logic [4:0] alpha;
alpha = (brightness > 8'd200) ? 5'd0 :     // 매우 밝음 → 완전 투명
        (brightness > 8'd150) ? 5'd8 :     // 밝음 → 반투명
        (brightness > 8'd100) ? 5'd16 :    // 중간 → 반투명
        5'd31;                             // 어두움 → 불투명
```

#### **2단계: 알파 블렌딩**
```systemverilog
// 배경색과 블렌딩 (회색 배경)
logic [4:0] bg_r = 5'd16;  // 중간 회색
logic [5:0] bg_g = 6'd32;
logic [4:0] bg_b = 5'd16;

// 알파 블렌딩
ghost_r = (r5 * alpha + bg_r * (5'd31 - alpha)) >> 5;
ghost_g = (g6 * alpha + bg_g * (6'd63 - alpha)) >> 5;
ghost_b = (b5 * alpha + bg_b * (5'd31 - alpha)) >> 5;
```

### **특징**
- **투명도 효과**: 밝기 기반 알파 계산
- **부드러운 전환**: 그라데이션 투명도
- **실시간 처리**: 하드웨어 최적화

## 🪞 Mirror Filter (거울 효과 필터)

### **목적과 원리**
- **목적**: 좌우 반전 효과
- **원리**: 픽셀 좌표 변환
- **효과**: 거울에 비친 것 같은 이미지

### **좌표 변환**
```systemverilog
// 원본 좌표
logic [8:0] orig_x = wAddr_in % IMG_WIDTH;
logic [7:0] orig_y = wAddr_in / IMG_WIDTH;

// 거울 좌표 변환
logic [8:0] mirror_x;
logic [7:0] mirror_y;

always_comb begin
    case (MODE)
        2'b00: begin  // 수평 반전
            mirror_x = IMG_WIDTH - 1 - orig_x;
            mirror_y = orig_y;
        end
        2'b01: begin  // 수직 반전
            mirror_x = orig_x;
            mirror_y = IMG_HEIGHT - 1 - orig_y;
        end
        2'b10: begin  // 대각선 반전
            mirror_x = IMG_WIDTH - 1 - orig_x;
            mirror_y = IMG_HEIGHT - 1 - orig_y;
        end
        default: begin  // 원본
            mirror_x = orig_x;
            mirror_y = orig_y;
        end
    endcase
end
```

### **메모리 접근**
```systemverilog
// 거울 주소 계산
logic [16:0] mirror_addr;
mirror_addr = mirror_y * IMG_WIDTH + mirror_x;

// 거울 픽셀 읽기
logic [15:0] mirror_pixel;
always_ff @(posedge clk) begin
    if (we_in) begin
        // 원본 픽셀을 거울 위치에 저장
        mirror_pixel <= wData_in;
        wAddr_out <= mirror_addr;
        wData_out <= mirror_pixel;
    end
end
```

### **특징**
- **좌표 변환**: 수학적 좌표 변환
- **메모리 효율성**: 추가 메모리 불필요
- **다양한 모드**: 수평/수직/대각선 반전

## 🎓 Graduation Filter (졸업사진 효과 필터)

### **목적과 원리**
- **목적**: 졸업사진 스타일 효과
- **원리**: 스케일링 + 텍스트 오버레이
- **효과**: 4컷 사진 스타일

### **처리 과정**

#### **1단계: 이미지 스케일링**
```systemverilog
// 스케일링 파라미터
parameter SCALE_NUM = 3;  // 분자
parameter SCALE_DEN = 2;  // 분모
parameter TOP_MARGIN = 6;
parameter BOTTOM_MARGIN = 6;
parameter MID_GAP = 8;

// 스케일링 계산
logic [8:0] scaled_x = (orig_x * SCALE_NUM) / SCALE_DEN;
logic [7:0] scaled_y = (orig_y * SCALE_NUM) / SCALE_DEN;
```

#### **2단계: 4컷 레이아웃**
```systemverilog
// 4컷 영역 계산
logic [8:0] cut_width = (IMG_WIDTH - MID_GAP) >> 1;
logic [7:0] cut_height = (IMG_HEIGHT - TOP_MARGIN - BOTTOM_MARGIN) >> 1;

// 4컷 위치 결정
logic [1:0] cut_x, cut_y;
cut_x = (scaled_x >= cut_width) ? 2'b01 : 2'b00;
cut_y = (scaled_y >= cut_height) ? 2'b01 : 2'b00;

// 4컷 내부 좌표
logic [8:0] local_x = scaled_x % cut_width;
logic [7:0] local_y = scaled_y % cut_height;
```

#### **3단계: 텍스트 오버레이**
```systemverilog
// 텍스트 영역 감지
logic text_area;
text_area = (local_y >= (cut_height - 16)) &&  // 하단 16픽셀
           (local_x >= 8) && (local_x < (cut_width - 8));  // 좌우 여백

// 텍스트 색상
logic [4:0] text_r = 5'd31;  // 흰색
logic [5:0] text_g = 6'd63;
logic [4:0] text_b = 5'd31;

// 텍스트 오버레이
if (text_area) begin
    out_r5 <= text_r;
    out_g6 <= text_g;
    out_b5 <= text_b;
end else begin
    out_r5 <= scaled_r;
    out_g6 <= scaled_g;
    out_b5 <= scaled_b;
end
```

### **특징**
- **레이아웃 처리**: 4컷 사진 스타일
- **텍스트 오버레이**: 하단 텍스트 영역
- **스케일링**: 이미지 크기 조정

## ⚡ 성능 비교

| 필터 | 처리 복잡도 | 메모리 사용량 | 특수 기능 |
|------|-------------|---------------|-----------|
| Ghost | 중간 | 라인 버퍼 | 투명도 처리 |
| Mirror | 낮음 | 없음 | 좌표 변환 |
| Graduation | 높음 | 프레임 버퍼 | 레이아웃 처리 |

## 🔧 공통 설계 패턴

### **1. 좌표 변환**
```systemverilog
// 1차원 주소 → 2차원 좌표
logic [8:0] x = addr % IMG_WIDTH;
logic [7:0] y = addr / IMG_WIDTH;

// 2차원 좌표 → 1차원 주소
logic [16:0] addr = y * IMG_WIDTH + x;
```

### **2. 조건부 처리**
```systemverilog
// 영역별 처리
if (in_text_area) begin
    // 텍스트 오버레이
    pixel_out <= text_color;
end else if (in_mirror_area) begin
    // 거울 효과
    pixel_out <= mirror_pixel;
end else begin
    // 원본 픽셀
    pixel_out <= original_pixel;
end
```

### **3. 파라미터화**
```systemverilog
// 필터별 파라미터
parameter int MODE = 2;           // 거울 모드
parameter int SCALE_NUM = 3;       // 스케일링 분자
parameter int SCALE_DEN = 2;       // 스케일링 분모
parameter int TEXT_COLOR = 16'hFFFF;  // 텍스트 색상
```

## 🎯 학습 포인트

1. **좌표 변환**: 2D 좌표 시스템 이해
2. **투명도 처리**: 알파 블렌딩 원리
3. **레이아웃 처리**: 4컷 사진 스타일
4. **조건부 처리**: 영역별 다른 효과
5. **파라미터화**: 재사용 가능한 설계

## 📝 다음 단계
- `06_Sticker_Filter.md`: 스티커 필터 특별 분석
- `07_Frame_Buffer_System.md`: 메모리 시스템 분석
- `08_Memory_Controllers.md`: 메모리 컨트롤러 분석
