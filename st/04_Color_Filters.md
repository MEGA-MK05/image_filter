# 색상 필터들 - Warm, Cool, Cartoon

## 📋 개요
이 문서는 AI Fourcut 프로젝트의 색상 기반 필터들(Warm, Cool, Cartoon)의 원리와 구현을 상세히 분석합니다.

## 🔥 Warm Filter (따뜻한 톤 필터)

### **목적과 원리**
- **목적**: 따뜻한 색감 조정
- **원리**: 빨간색/노란색 채널 강화
- **효과**: 따뜻하고 부드러운 이미지

### **색상 변환 공식**
```
R_out = R_in + α × R_in
G_out = G_in + β × G_in  
B_out = B_in - γ × B_in

여기서 α > β > γ (빨간색 강화, 파란색 감소)
```

### **하드웨어 구현**
```systemverilog
// RGB565 분리
logic [4:0] in_r5 = wData_in[15:11];
logic [5:0] in_g6 = wData_in[10:5];
logic [4:0] in_b5 = wData_in[4:0];

// 따뜻한 톤 변환 (가중치 적용)
warm_r = in_r5 + (in_r5 >> 2);  // R 채널 25% 증가
warm_g = in_g6 + (in_g6 >> 3);  // G 채널 12.5% 증가
warm_b = in_b5 - (in_b5 >> 2);  // B 채널 25% 감소

// 클리핑 처리
if (warm_r > 5'd31) warm_r = 5'd31;
if (warm_g > 6'd63) warm_g = 6'd63;
if (warm_b < 5'd0) warm_b = 5'd0;
```

### **특징**
- **색온도 조정**: 따뜻한 색감
- **자연스러운 효과**: 과도하지 않은 조정
- **실시간 처리**: 하드웨어 최적화

## ❄️ Cool Filter (차가운 톤 필터)

### **목적과 원리**
- **목적**: 차가운 색감 조정
- **원리**: 파란색/청록색 채널 강화
- **효과**: 차갑고 선명한 이미지

### **색상 변환 공식**
```
R_out = R_in - α × R_in
G_out = G_in + β × G_in
B_out = B_in + γ × B_in

여기서 γ > β > α (파란색 강화, 빨간색 감소)
```

### **하드웨어 구현**
```systemverilog
// 차가운 톤 변환
cool_r = in_r5 - (in_r5 >> 2);  // R 채널 25% 감소
cool_g = in_g6 + (in_g6 >> 4);  // G 채널 6.25% 증가
cool_b = in_b5 + (in_b5 >> 1);  // B 채널 50% 증가

// 클리핑 처리
if (cool_r < 5'd0) cool_r = 5'd0;
if (cool_g > 6'd63) cool_g = 6'd63;
if (cool_b > 5'd31) cool_b = 5'd31;
```

### **특징**
- **색온도 조정**: 차가운 색감
- **선명도 향상**: 파란색 강조
- **대비 효과**: 색상 대비 증가

## 🎨 Cartoon Filter (만화 스타일 필터)

### **목적과 원리**
- **목적**: 만화/애니메이션 스타일 효과
- **원리**: 블러 + 포스터라이제이션 + 엣지 검출
- **효과**: 평면적이고 선명한 만화 스타일

### **처리 과정**

#### **1단계: 가우시안 블러**
```systemverilog
// 3×3 가우시안 커널 적용
sum_r <= (r00 + (r01<<1) + r02) + ((r10<<1) + (r11<<2) + (r12<<1)) + (r20 + (r21<<1) + r22);
blur_r5 <= (sum_r[11:4] > 12'd31) ? 5'd31 : sum_r[8:4];
```

#### **2단계: 포스터라이제이션**
```systemverilog
// MSB만 유지하여 색상 단계 감소
generate
    if (KEEP_RB_MSBS == 2) begin
        assign post_r5_w = {blur_r5[4:3], 3'b000};  // 상위 2비트만 유지
        assign post_b5_w = {blur_b5[4:3], 3'b000};
    end
endgenerate
```

#### **3단계: 소벨 엣지 검출**
```systemverilog
// G 채널에서 엣지 검출
gx <= $signed({1'b0, g02}) + $signed({1'b0, (g12 << 1)}) + $signed({1'b0, g22}) - 
      ($signed({1'b0, g00}) + $signed({1'b0, (g10 << 1)}) + $signed({1'b0, g20}));

gy <= $signed({1'b0, g20}) + $signed({1'b0, (g21 << 1)}) + $signed({1'b0, g22}) - 
      ($signed({1'b0, g00}) + $signed({1'b0, (g01 << 1)}) + $signed({1'b0, g02}));

edge_mag <= abs_gx + abs_gy;
```

#### **4단계: 엣지와 색상 합성**
```systemverilog
if (edge_mag >= EDGE_THR) begin
    out_r5 <= 5'd0;   // 검은색 엣지
    out_g6 <= 6'd0;
    out_b5 <= 5'd0;
end else begin
    out_r5 <= post_r5;  // 포스터라이즈된 색상
    out_g6 <= post_g6;
    out_b5 <= post_b5;
end
```

### **특징**
- **3단계 처리**: 블러 → 포스터라이제이션 → 엣지
- **만화 스타일**: 평면적이고 선명한 효과
- **실시간 처리**: 파이프라인 구조

## 🎨 색상 필터 비교

| 필터 | 주요 조정 | 효과 | 용도 |
|------|-----------|------|------|
| Warm | R↑, G↑, B↓ | 따뜻한 톤 | 인물 사진, 일몰 |
| Cool | R↓, G↑, B↑ | 차가운 톤 | 풍경, 야경 |
| Cartoon | 복합 처리 | 만화 스타일 | 특수 효과 |

## ⚡ 성능 특성

### **처리 복잡도**
- **Warm/Cool**: 단순 색상 변환 (1클럭)
- **Cartoon**: 복합 처리 (4클럭)

### **메모리 사용량**
- **Warm/Cool**: 라인 버퍼 불필요
- **Cartoon**: 2라인 버퍼 + 3×3 윈도우

### **하드웨어 최적화**
```systemverilog
// 색상 변환 최적화
warm_r = in_r5 + (in_r5 >> 2);  // 곱셈 대신 시프트 연산

// 클리핑 최적화
if (warm_r > 5'd31) warm_r = 5'd31;  // 조건부 할당

// 파이프라인 정렬
always_ff @(posedge clk) begin
    wAddr_out <= addr_d3;
    we_out <= we_d3;
end
```

## 🔧 공통 설계 패턴

### **1. RGB565 처리**
```systemverilog
// RGB565 분리
logic [4:0] r5 = pixel[15:11];   // 5비트 빨간색
logic [5:0] g6 = pixel[10:5];    // 6비트 녹색
logic [4:0] b5 = pixel[4:0];     // 5비트 파란색

// RGB565 재조립
pixel_out = {r5_out, g6_out, b5_out};
```

### **2. 색상 변환**
```systemverilog
// 가중치 적용
new_r = old_r + (old_r >> 2);  // 25% 증가
new_g = old_g + (old_g >> 3);  // 12.5% 증가
new_b = old_b - (old_b >> 2);  // 25% 감소
```

### **3. 클리핑 처리**
```systemverilog
// 상한 클리핑
if (r5 > 5'd31) r5 = 5'd31;
if (g6 > 6'd63) g6 = 6'd63;
if (b5 > 5'd31) b5 = 5'd31;

// 하한 클리핑
if (r5 < 5'd0) r5 = 5'd0;
if (g6 < 6'd0) g6 = 6'd0;
if (b5 < 5'd0) b5 = 5'd0;
```

## 🎯 학습 포인트

1. **색상 공간**: RGB565 포맷 이해
2. **색상 변환**: 가중치 기반 색상 조정
3. **클리핑**: 오버플로우 방지
4. **하드웨어 최적화**: 시프트 연산 활용
5. **복합 필터**: 다단계 이미지 처리

## 📝 다음 단계
- `05_Special_Filters.md`: 특수 필터들 (Ghost, Mirror, Graduation)
- `06_Sticker_Filter.md`: 스티커 필터 특별 분석
- `07_Frame_Buffer_System.md`: 메모리 시스템 분석
