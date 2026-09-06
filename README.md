### ALL Design Arch' ###

```text

Gemini/Grok
---+----
   |
   +---> ax630C(OpenCV + rust)
         ========+==============
                 |
                 +----UART----->  ESP32S3
                                  ====+====
                                      |
                                      |                     +---> Nema17 (X1 - PU+,DR+,EN+,AL+)
                                      |                     |
                                      +---------------------+---> Nema17 (X2 - PU+,DR+,EN+,AL+)
                                      |                     |
                                      |                     +---> Nema17 (Y - PU+,DR+, EN+,AL+)
                                      |                     |
                                      |                     +---> Nema11(Z-PEN)
                                      |
                                      +---------------------+----> ST7789(LCD Module)

    ## Power Support 
    * SMPS(24v) ---------+----------> A4988 ----> Nema11 Motor(Z)
                         |
                         +----------> Nema17 Motor Group (X1,X2,Y)
                         |
                         +----------> FlyD5(5vDC) --> ESP32S3(PROD운영)

```

## For Z-Pen Design 구상중

    신규 탑재 부품 리스트

    ```text

        - ESP32S3 DevkitC-1 (개발보드)

        - CNC Shield V3 Extension Board

        - A4988 Stepping Motor Drive 

   ```

## 1. 🛠️ 사전 하드웨어 셋팅 (점퍼 캡 & 방향 확인)①

    - CNC Shield V3 마이크로스텝 점퍼 캡 설정 (Z축 소켓 밑)Z축 드라이버 소켓 바닥면의 
      M0, M1, M2 핀 3쌍에 점퍼 캡을 모두 꽂아줍니다 ($1/16$ 마이크로스텝 적용).
    
    - 마이크로스텝을 설정해야 Nema 11 모터가 진동 없이 부드럽고 정밀하게 움직입니다.② 
    
    - A4988 드라이버 장착 방향 (★가장 중요)CNC Shield V3의 Z축 소켓(파란색 또는 노란색 소켓)에 
     A4988을 꽂습니다.방향 기준: A4988 기판에 인쇄된 DIR / EN 라벨 핀이 CNC Shield 보드 바닥에 
     인쇄된 DIR / EN 위치와 서로 맞물리도록 방향을 꼭 확인하고 꽂아주세요. 
     (보통 A4988의 작은 금속 가변저항 나사가 Shield 전원 터미널 쪽을 향하게 됩니다.)

## 2. 🔌 핀맵 및 1:1 배선 연결표
```text

구분       | ESP32-S3 DevKitC-1 핀 | CNC Shield V3 핀       |역할 및 상세 설명
----------+-----------------------+-----------------------+--------------------------
로직 전원   | 3.3V                  | 5V (or 3.3V)          | 4988 VDD 로직 칩 전원 공급
-----------+-----------------------+-----------------------+--------------------------
공통 GND    | GND                   | GND                  | 신호 기준점 맞춤 (GND 싱크)
-----------+-----------------------+-----------------------+--------------------------
Z축 STEP   | GPIO 15               | Z.STEP (또는 Z축 Step) | 스텝 펄스 신호 공급
-----------+-----------------------+-----------------------+--------------------------
Z축 DIR    | GPIO 47               |Z.DIR (또는 Z축 Dir)    | 방향 제어 신호 (CW / CCW)
-----------+-----------------------+-----------------------+--------------------------
공통 EN    | GND직결                |EN (Enable 핀)         | GND에 직결 시 Z축 중력 자중 낙하 완전 방지! 
----------+------------------------+-----------------------+---------------------------


-- ######################[ A4988 ]##########################################
[-LEFT-]    [SMPS/S3]       |       [-RIGHT-]       [SMPS/S3]
1   EN      GND/S3[TO/공통]  |       VMOT            24VDC / SMPS[From]
2   MS1                     |       GND             GND / SMPS[From]
3   MS2                     |       2B              *NEMA11[TO]
4   MS3                     |       2A              *NEMA11[TO]
5   RST                     |       1A              *NEMA11[TO]
6   SLP                     |       1B              *NEMA11[TO]
7   STEP    GPIO 15/S3[TO]  |       VDD             3.3VDC / S3/#2 VCC 3.3V[From ]
8   DIR     GPIO 47/S3[TO]  |       GND             GND / S3[TO/공통]

*참고: RST <-> SLP(쇼트연결 필수)


```

## 3. ⚡ 24V SMPS 및 모터 배선 (파워부)

    - 24V SMPS 파워 연결:
    SMPS 24V (+) $\rightarrow$ CNC Shield 파란색 스크루 터미널 (+)SMPS GND (-) $\rightarrow$ CNC Shield 파란색 스크루 터미널 (-)
    
    - Nema  11 모터 연결:Z축 A4988 드라이버 바로 옆 4핀 핀 헤더(Z축 모터 포트)에 모터 4선 커넥터를 꽂아줍니다.

### MCU - Motor/LCD MAP

```text
                +---------------------------WAGO---------------------------------------+
                |                                                                      |
                |                                                                      |
                |                                   +----- ESP32S3 -----+              |
            [PU-, DR-, EN-, AL-]                    |                   |              |
                 LCD(st7789/bl)및 기타  <--- 3.3v<--+                   +-->GND  +<-----+
                              *A988/CNC<--- 3.3v<--+                   +-->GPIO#43
                                             RST<--+                   +-->GPIO#44
    X1-X2 <--> [PU+]STEP_PIN(OUT) +<----+ GPIO#4<--+                   +-->GPIO#1  +---->+ BX6-H4/Y_BOTTOM(IN)
     X1-X2 <--> [DR+]DIR_PIN(OUT) +<----+ GPIO#5<--+                   +-->GPIO#2  +---->+ BX6-H4/X_TOP(IN)
        Y <--> [PU+]STEP_PIN(OUT) +<----+ GPIO#6<--+                   +-->GPIO#42 
        Y <--> [DR+]DIR_PIN(OUT) +<----+  GPIO#7<--+                   +-->GPIO#41
              *Z(STEP) <--> [A4988](OUT) GPIO#15<--+                   +-->GPIO#40
                            ax630c(TX)<--GPIO#16<--+                   +-->GPIO#39
                            ax630c(RX)<--GPIO#17<--+                   +-->GPIO#38 +---->+ BX6-H4/Y_TOP(IN)
            X1-X2(#1) <--> [EN+] +<----+ GPIO#18<--+                   +-->GPIO#37
                 [X1-X2-AL+(IN)] +<----+  GPIO#8<--+                   +-->GPIO#36
                             st7789/rst<--GPIO#3<--+                   +-->GPIO#35
                                         GPIO#46<--+                   +-->GPIO#0
                        st7789/dc +<----+ GPIO#9<--+                   +-->GPIO#45
                      st7789/RST +<----+ GPIO#10<--+                   +-->GPIO#48
                     st7789/mosi +<----+ GPIO#11<--+                   +-->GPIO#47 +---->+ *Z[DIR / A4988]
                     st7789/sclk +<----+ GPIO#12<--+                   +-->GPIO#21 +---->+ [Y-EN+(IN)] 
             BX6-H4/X_BOTTOM(IN) +<----+ GPIO#13<--+                   +-->GPIO#20
                    [Y-AL+(IN)] +<----+  GPIO#14<--+                   +-->GPIO#19
                                              5v<--+                   +-->GND
                                             GND<--+                   +-->GND
                                                   |                   |
                                                   +-------------------+ 


== 2.8 TFT LCD ili9341 / ST7789 핀 할당 목록 ==
VCC ➡️ 3.3V 
GND ➡️ GND 
LCD_CS ➡️ GND (SPIInterfaceNoCS 유지용 직결)
LCD_RST ➡️ GPIO 10 (RST) 
LCD_RS ➡️ GPIO 9 (DC) 
SDI(MOSI) ➡️ GPIO 11 (MOSI) 
SCK ➡️ GPIO 12 (SCLK) 
LED ➡️ 3.3V (백라이트 ON 필수 ⭐)



== 비고==


* 연결예정 Lines (공사중) 
```

## LCD - example

```rust
// GPIO 핀 초기화
let mut backlight = PinDriver::output(pins.gpio46)?; // BL
let mut reset = PinDriver::output(pins.gpio3)?;      // RST
let dc = PinDriver::output(pins.gpio9)?;             // DC

backlight.set_high()?;

// SPI2(FSPI) 설정
let spi_config = Config::new()
    .baudrate(40.MHz().into())
    .data_mode(embedded_hal::spi::MODE_0);

let spi_device = SpiDeviceDriver::new_single(
    peripherals.spi2,
    pins.gpio12,         // SCLK
    pins.gpio11,         // MOSI
    Option::<esp_idf_hal::gpio::AnyIOPin>::None,
    Some(pins.gpio10),   // CS
    &Default::default(),
    &spi_config,
)?;
```

## New Y and Z-Pen(A4988)

```rust

 use esp_idf_hal::delay::{Ets, FreeRtos};
use esp_idf_hal::gpio::*;
use esp_idf_hal::peripherals::Peripherals;

fn main() -> anyhow::Result<()> {
    esp_idf_svc::sys::link_patches();
    esp_idf_svc::log::EspLogger::initialize_default();

    println!("==================================================");
    println!("=== 🌀 NEMA11 Z-Axis Final Verified Test Mode ===");
    println!("=== Pin Map: STEP = GPIO 15, DIR = GPIO 47     ===");
    println!("==================================================");

    let peripherals = Peripherals::take()?;

    // 💡 [Grok & Gemini 최종 승인 무결점 핀 적용]
    let mut z_step = PinDriver::output(peripherals.pins.gpio15)?; // Z.STEP
    let mut z_dir = PinDriver::output(peripherals.pins.gpio47)?;  // Z.DIR (GPIO 47 무결점 핀)

    println!("⚡ Z-Axis Motor Power ALWAYS ENABLED (EN -> GND 직결)");
    FreeRtos::delay_ms(500);

    loop {
        // 1. 시계 방향 (CW) 회전 (약 3초 회전)
        println!("🌀 Z-Axis Moving CW (정방향 1000 스텝)...");
        z_dir.set_low()?;
        Ets::delay_us(100);

        for step in 0..1000 {
            z_step.set_high()?;
            Ets::delay_us(1500); // 1.5ms 정숙 저속 구동
            z_step.set_low()?;
            Ets::delay_us(1500);

            if step % 50 == 0 {
                FreeRtos::delay_ms(1);
            }
        }

        println!("⏸️ 1초 정지 대기");
        FreeRtos::delay_ms(1000);

        // 2. 반시계 방향 (CCW) 회전 (약 3초 회전)
        println!("🌀 Z-Axis Moving CCW (역방향 1000 스텝)...");
        z_dir.set_high()?;
        Ets::delay_us(100);

        for step in 0..1000 {
            z_step.set_high()?;
            Ets::delay_us(1500);
            z_step.set_low()?;
            Ets::delay_us(1500);

            if step % 50 == 0 {
                FreeRtos::delay_ms(1);
            }
        }

        println!("⏸️ 1초 정지 대기");
        FreeRtos::delay_ms(1000);
    }
} 
```

### AX630C PIN MAP

```text
      ----------------------------------------------------------------------------------------------
       2     4     6     8    10     12     14     [16]  18   20  22  24  26    28   30
       |     |     |     |    |       |      |      |     |    |   |   |   |     |    |
       G10   G8    RST   G5   G9     3.3V   G43    G17   G11  G1  G7   G0  G14   5V   BAT
       ADC PB_IN   EN   GPIO  PB_OUT        TXD0  [PC_RX]             GPIO
       -----------------------------------------------------------------------------------------------
        1    3   5       7     9   11    13   [15]    17   19   21   23       25    27    29
        |    |   |       |     |   |      |     |       |   |    |     |
       GND GND  GND     G37  G35   G36   G44   G18    G12  G2   G6   G13
                MOSI   MISO        SCX   RXD0 [PC_TX]           GPIO  I2S_DOUT
    ------------------------------------------------------------------------------------------------- 
     [ax630C 최종 픽업 단자]                    [ESP32-S3 단자]

    | AX630C (M5 단자대) | 신호 방향 | ESP32-S3 (좌측 핀열) | 비고 |
    | :--- | :---: | :--- | :--- |
    | **G17 [PC_TX]** (송신) | -> | **GPIO 16 [RX]** (수신) | 초록색 결선선 |
    | **G18 [PC_RX]** (수신) | -> | **GPIO 17 [TX]** (송신) | 파란색 결선선 | 
    | 아랫줄 1, 3, 5번 중 하나 [GND] -------->   GND

======== 최종 체크한 내용 =============
[ AX630C 단자 ]                                     [ ESP32-S3 단자 ]

  Pin 15 (TXD)  ───(주황색 라인 + 1Kohm 추가)───>   GPIO 16 (RX)
  Pin 16 (RXD)  <───(파란색 라인)───                  GPIO 17 (TX)
  Pin 1 / 3 (GND) ───────────────────              GND (공통 지선)


```

refer => <https://docs.m5stack.com/en/module/Module%20LLM%20Kit>

### M5STACK Status

```bash
root@m5stack-LLM:~# dmesg | grep -iE "uart|ttyS|ttyAMA"
[    0.000000] earlycon: uart8250 at MMIO32 0x0000000004880000 (options '')
[    0.000000] bootconsole [uart8250] enabled
[    0.000000] Kernel command line: mem=2048M console=ttyS0,115200n8 earlycon=uart8250,mmio32,0x4880000 board_id=0,boot_reason=0x0 initcall_debug=0 quiet loglevel=0 usbcore.autosuspend=-1 root=/dev/mmcblk0p16 rootfstype=ext4 rw rootwait blkdevparts=mmcblk0:768K(spl),512K(ddrinit),256K(atf),256K(atf_b),1536K(uboot),1536K(uboot_b),1024K(env),6144K(logo),6144K(logo_b),1024K(optee),1024K(optee_b),1024K(dtb),1024K(dtb_b),262144K(kernel),262144K(kernel_b),29889536K(ubuntu_rootfs)
[    0.301519] console [ttyS0] disabled
[    0.301550] 4880000.ax_uart: ttyS0 at MMIO 0x4880000 (irq = 5, base_baud = 13000000) is a 16550A
[    0.301573] console [ttyS0] enabled
[    0.301576] bootconsole [uart8250] disabled
[    0.303680] 4881000.ax_uart: ttyS1 at MMIO 0x4881000 (irq = 6, base_baud = 13000000) is a 16550A
[    0.458510] ax-apb-uart 4880000.ax_uart: forbid DMA for kernel console
root@m5stack-LLM:~# 
root@m5stack-LLM:~# chmod 666 /dev/ttyS1
root@m5stack-LLM:~# ls -al /dev/ttyS*
crw------- 1 root tty     4, 64 Aug 22 05:11 /dev/ttyS0
crw-rw-rw- 1 root dialout 4, 65 Aug 22 05:11 /dev/ttyS1
crw-rw---- 1 root dialout 4, 66 Aug 22 05:11 /dev/ttyS2
crw-rw---- 1 root dialout 4, 67 Aug 22 05:11 /dev/ttyS3
crw-rw---- 1 root dialout 4, 68 Aug 22 05:11 /dev/ttyS4
crw-rw---- 1 root dialout 4, 69 Aug 22 05:11 /dev/ttyS5

```

## 1. 제어 신호선 (ESP32-S3 ➡️ 와고 분기 ➡️ 모터 드라이버)

EN+ (이네이블 - 모터 기동/정지):
    ESP32-S3 GPIO 18 ➡️ 와고 A ➡️ X1 모터 EN+ & X2 모터 EN+
PU+ (펄스 - 모터 속도/회전):
    ESP32-S3 GPIO 4 ➡️ 와고 B ➡️ X1 모터 PU+ & X2 모터 PU+
DR+ (디렉션 - 모터 방향):
    ESP32-S3 GPIO 5 ➡️ 와고 C ➡️ X1 모터 DR+ & X2 모터 DR+
AL+ (알람 - 이중 안전장치 피드백):
    ESP32-S3 GPIO 8 ➡️ 와고 D ➡️ X1 모터 AL+ & X2 모터 AL+

## 2. 공통 음극(GND) 처리 (매우 중요 ⭐)

신호가 정상적으로 흐르려면 ESP32-S3와 모터 드라이버의 기준 전위(GND)가 같아야 합니다.N)0⭐)))버)ㅑiㅎㅎ
공통 GND 묶음:
모터의 모든 마이너스 신호선들(EN-, PU-, DR-, AL- 등)을 한데 모아 와고 E에 꽂고, 이를 ESP32-S3의 GND 핀과 연결해 줍니다.

## 3. 동력 전원선 (SMPS 24V ➡️ FlyD5 ➡️ 모터 전원)

실제 모터를 힘차게 돌려줄 전력 공급 루트입니다.
SMPS 24V 출력 ➡️ FlyD5 메인 전원 입력단
FlyD5 전원 분기 포트 ➡️ 각각 X1 모터 VCC/GND 및 X2 모터 VCC/GND

## Power Design

```text
[ 24V 5A SMPS ] 
     │
     ├─▶ [Fly-D5 메인 입력] ──▶ 우측 'POWER' 단자대 (+ / -) 결선 (보드 전원 구동)
     │                                │
     │                                └─▶ 보드 좌측 상단 '5V / G' 핀 헤더 ──▶ [ESP32-S3 5V / GND 핀] (MCU 전원 완료)
     │
     └─▶ [WAGO 분기 (센서 전원용)]
               ├─▶ (+) 24V ──▶ WAGO F단자 ──▶ TOP 센서 갈색선(VCC) & BOT 센서 갈색선(VCC)
               └─▶ (-) GND ──▶ WAGO G단자 ──▶ TOP 센서 청색선(GND) & BOT 센서 청색선(GND)



[ SMPS (-) ] ──▶ Fly-D5 메인 파워 (-) 입력단
                      │ (Fly-D5 내부 PCB 접지 패턴)
                      ▼
               Fly-D5 좌측 상단 'G' (GND) 핀
                      │
                      ▼ (전원 케이블)
               [ ESP32-S3 단 GND ] ◀◀◀ [현재 여기에 모두 묶어두신 상태!]
                      │
                      ├─▶ AX630C 단자의 GND 핀 (UART 신호 전위 일치 완료)
                      ├─▶ 모터 드라이버 묶음 (PU-, DR-, EN-, AL-)
                      └─▶ 금속 유도 센서의 청색선(GND) 
                            (※ Fly-D5 내부를 타고 SMPS (-)와 연결되므로 자동으로 루프 완성)

=========================== 최종 적용내용 =============================
[ 24V 5A SMPS ] 
     │
     ├─▶ [WAGO 모터전원 그룹] ──▶ X1 모터 VCC/GND & X2 모터 VCC/GND (대전류 구동부)
     │
     ├─▶ [WAGO 유도센서 그룹] ──▶ TOP 센서 갈색선(+) & BOT 센서 갈색선(+)
     │                          ▶ TOP 센서 청색선(-) & BOT 센서 청색선(-)
     │
     └─▶ [Fly-D5 메인 POWER 입력] (24V 공급)
               │
               ▼ (Fly-D5 내부 고품질 5A 스텝다운 강압 회로 통과 🛡️)
               │
          [Fly-D5 좌측 상단 '5V / G' 핀 헤더] 
               │
               ▼ (깨끗하게 정제된 5V 파워 및 신호 그라운드 사출)
               │
          [ESP32-S3 보드의 5V / GND 핀] ◀◀◀ (★ 여기서 모든 제어 신호 접지 통일!)
               │
               ├─▶ AX630C 메인보드의 GND 핀 (UART 시리얼 신호 기준점)
               ├─▶ 모터 드라이버 제어선 묶음 WAGO 단자 (PU-, DR-, EN-, AL-)
               │
               └─▶ [다이렉트 연결 ❌] 유도센서 흑색 신호선(S)은 S3 GPIO 2, 13번으로 직행!


##-- 안전진단 가동..
[ NPN 센서 신호선 (최대 24V 유입 대응) ]
            │
         [ 20kΩ 저항 ]
            │
            ├───> [ ESP32 GPIO 핀 ]  <-- 최대 24V 입력 시에도 약 3.41V 이하 안정권!
            │
         [ 3.3kΩ 저항 ]
            │
 [ Common GND (SMPS -V = ESP32 GND) ]


```

## GPIO 분배 출력 전압

$24\text{V} \times \frac{3.3\text{k}\Omega}{20\text{k}\Omega + 3.3\text{k}\Omega} \approx \mathbf{3.39\text{V}}$ (완벽한 안전 지대)

# [74LVT244] 기판 배선 준비

```text

74LVT244 핀 배열 명세만 깔끔하게 정리해 드릴게요.

74LVT244 (DIP-20 기준) 주요 핀 배치

    1번 (1OE) / 19번 (2OE): 둘 다 GND로 연결 (출력 상시 활성화)

    10번 (GND): ESP32-S3 Common GND 연결

    20번 (VCC): DC 3.3V 연결 (5V 절대 금지)

    채널 1 (스텝 신호 분배):

        2번 Pin (1A1) ➔ ESP32-S3 PU GPIO 핀 입력

        18번 Pin (1Y1) ➔ [X1-PU+] 및 [X2-PU+] 소켓 핀으로 동시 출력

    채널 2 (방향 신호 분배):

        4번 Pin (1A2) ➔ ESP32-S3 DR GPIO 핀 입력

        16번 Pin (1Y2) ➔ [X1-DR+] 및 [X2-DR+] 소켓 핀으로 동시 출력

    채널 3 (이네이블 신호 분배):

        6번 Pin (1A3) ➔ ESP32-S3 EN GPIO 핀 입력

        14번 Pin (1Y3) ➔ [X1-EN+] 및 [X2-EN+] 소켓 핀으로 동시 출력
```

## 🛡️ [무전원/무위험] A4988 VREF 저항 세팅 절차

```
전원 플러그와 USB를 모두 뽑은 상태에서 진행하므로 스파크나 쇼트 위험이 0%입니다.

    전원 완벽 차단

        24V SMPS 스위치를 끄고, ESP32-S3 USB 케이블도 완전히 뽑습니다.

    멀티미터 저항 모드 설정

        멀티미터 다이얼을 저항(Ω 또는 2kΩ 대역)에 놓습니다.

    측정 바늘 연결

        검은색 바늘(-): CNC Shield의 GND 단자에 댑니다.

        빨간색 바늘(+): A4988 가변저항 금속 나사 머리에 댑니다.

    저항값 맞추기 (드라이버 조절)

        멀티미터 화면의 저항값을 보면서 드라이버로 나사를 돌립니다.

        목표 저항값: 1.1kΩ ~ 1.2kΩ (화면에 1.10kΩ 또는 1100Ω 근처가 찍히면 정답)

```

## 추천하는 단계별 전원 투입 순서1단계 (SMPS 단독)

    - 1단계 (SMPS 단독): SMPS 단자대 이후 배선을 다 뺀 상태에서 $24\text{V}$ 전압 정밀 측정 및 V.ADJ 조정
    
    - 2단계 (WAGO 및 전원 분배 단자): WAGO 단자대까지만 연결 후 VCC-GND 간 쇼트(도통) 없는지 찍어보고 $24\text{V}$ 정상 출력 확인
    
    - 3단계 (MCU 단독): ESP32-S3 및 3.3V/5V 로직 전원만 인가하여 정상 부팅 및 핀 전압 확인
    
    - 4단계 (모터 드라이버 및 메인 $24\text{V}$): 마지막으로 $24\text{V}$ 동력선 연결 후 최종 인가

### 📐 만능기판 실전 안전 회로 배치도

```text
Below is a schematic layout for solder side / component placement on a perfboard:

[ 24V SMPS + ] ──( 2A 메인 퓨즈 )──┬──[ VMOT : 모터 드라이버 24V ]
                                    │
                                 [ LM2596 / DC-DC Buck Converter ]
                                    │ (24V -> 5V 변환)
                                    ▼
[ ESP32 5V IN ] ──( 500mA PPTC )──┬──[ ESP32-S3 5V VIN 핀 ]
                                   │
                           [ TVS 다이오드 (5.6V) ] (병렬: 5V - GND)
                                   │
                                 [ GND ]

------------------------------------------------------------------
[ 외부 센서/드라이버 신호선 ] ──[ 1kΩ 완충 저항 ]──► [ ESP32-S3 GPIO 핀 ]
                                        │
                                [ 3.3V 제너 다이오드 ] (병렬: GPIO - GND)

```

## 🛠 만능기판 제작 시 핵심 실전 규칙

    DC-DC 감압 모듈 필수 활용

    - $24\text{V}$ 전원에서 ESP32-S3용 전원을 뽑을 때, 만능기판 위에서 직접 강하하기보다 LM2596 같은 강압 모듈을 거쳐 $5\text{V}$로 먼저 낮춘 뒤 ESP32의 5V/VIN 핀으로 넣는 것이 가장 안전합니다.
    
    - PPTC 및 TVS 다이오드 납땜 위치:PPTC 자가복구 퓨즈: $5\text{V}$ 라인이 ESP32 핀으로 들어가는 입구 바로 앞에 직렬로 배치TVS 다이오드: PPTC 바로 뒷단에서 $5\text{V}$ 라인과 GND 라인 사이에 띠 방향(카소드, -)을 $5\text{V}$ 쪽으로 향하게 병렬 납땜
    
    - GPIO 신호선 파벽 구현:만능기판의 각 GPIO 패턴 핀 바로 직전에 $1\text{k}\Omega$ DIP 저항을 세워서(Vertical Placement) 납땜합니다.센서 신호선이 $24\text{V}$ 쪽에서 오더라도 이 $1\text{k}\Omega$ 저항을 먼저 거치게 하므로 손쉽게 신호선을 보호합니다.
    
    - 동판 동력선 패턴 강화 및 절연(Pattern Isolation):$24\text{V}$ 및 모터 대전류가 흐르는 만능기판 납땜면 패턴은 납을 두껍게 올려(Solder Bridge) 전류 허용량을 늘립니다.$24\text{V}$ 패턴 바로 옆 동판 홀(Hole)은 커터칼이나 드릴 비트로 동박 패턴을 칼같이 긁어내어(Pattern Cut) 만에 하나 발생할 신호선과의 쇼트를 물리적으로 차단합니다.
    
    - 💡 만능기판 부품 추천 규격PPTC 자가복구 퓨즈: DIP 타입 DIP 500mA 16V/30V (노란색 알약 모양)TVS 다이오드: 1SMA5.0A 또는 1N6373 ($5\text{V}$ 전원 단 보호용)완충 저항: 1/4W 1kΩ 일반 띠저항터미널 블록: 외부 $24\text{V}$ 및 모터 선 연결용 2.54mm 또는 5.08mm Screw Terminal Block (만능기판 직결 핀헤더 대신 필수 사용)

### ####################################################

### A4988 VREF 설정 및 고전압(24V) 안전 데버깅 가이드

### ##################################################

24V 메인 전원 환경에서 소형 스텝모터(NEMA11 등) 및 A4988 모터 드라이버의 파손/쇼트를 방지하기 위한 정석 VREF 세팅 매뉴얼입니다.

---

## 1. 개요 및 배경

* **24V 고전압의 장점**: 스텝모터의 토크 증가 및 고속 구동 가능
* **24V 바로 인가 시 리스크**: 드라이버 가변저항 초기 위치 과다로 인한 모터/드라이버 타버림, 조정 중 드라이버 팁 미끄러짐으로 인한 24V 스파크 및 MCU(ESP32) 동시 파괴
* **안전 절차 핵심**: **24V 모터 전원($V_{MOT}$)과 모터를 완전 분리**한 상태에서 **5V 로직 전원($V_{DD}$)**만으로 VREF를 사전 설정

---

## 2. VREF 계산 공식

$$V_{REF} = I_{MAX} \times 8 \times R_S$$

* **$I_{MAX}$**: 스텝모터 데이터시트상의 정격 전류 (안전을 위해 정격의 80% 수준 권장)
* **$R_S$**: A4988 보드에 탑재된 전류 감지 저항값 (보통 `R100` = $0.1\Omega$, `R068` = $0.068\Omega$)

### 예시 계산 (NEMA11 / $0.5\text{A}$ 정격, $R_S = 0.1\Omega$ 기준)

* 목표 전류: $0.5\text{A} \times 0.8 = 0.4\text{A}$
* 계산: $0.4\text{A} \times 8 \times 0.1\Omega = \mathbf{0.32\text{V}}$
* $\rightarrow$ 멀티미터로 VREF를 **$0.32\text{V}$**에 맞춤

---

## 3. 단계별 안전 세팅 절차 (Step-by-Step)

### Step 1. 전원 및 배선 분리 (무전원 상태)

1. 24V 메인 파워 서플라이 전원을 끕니다.
2. A4988 모듈에서 모터 케이블 및 24V 입력 전원 라인을 분리합니다.

### Step 2. 5V 로직 전원 및 제어 핀 세팅

1. MCU(ESP32)의 **5V** 라인을 A4988의 **VDD** 핀에 연결합니다.
2. MCU의 **GND** 라인을 A4988의 **GND(로직 단자)**에 연결합니다.
3. 점퍼 와이어를 이용하여 A4988의 **SLEEP 핀과 RESET 핀을 서로 쇼트(Bridge)**시킵니다. (내부 풀업을 이용해 드라이버를 활성 상태로 전환)

### Step 3. 멀티미터 측정 배선

1. 멀티미터를 **DC 전압 측정 모드**로 설정합니다.
2. 멀티미터의 **음극(-) 검침봉**을 MCU의 **GND**에 고정합니다.
3. 멀티미터의 **양극(+) 검침봉**을 악어 클립을 이용해 **정밀 금속 드라이버 끝부분**에 집어 연결합니다.

### Step 4. VREF 전압 측정 및 조절

1. MCU에 USB 전원(5V)을 공급하여 A4988 로직을 가동합니다.
2. 드라이버 팁을 A4988 중앙의 **가변저항(포텐쇼미터) 금속 홈**에 맞대고 멀티미터 전압을 확인합니다.
3. 가변저항을 천천히 돌려 목표 전압($V_{REF}$)에 맞춥니다.
   * **시계 반대 방향**: 전압 감소 (전류 한계 낮춤)
   * **시계 방향**: 전압 증가 (전류 한계 높임)

### Step 5. 결합 및 24V 최종 구동

1. 5V 로직 전원을 완전히 차단합니다.
2. SLEEP-RESET 쇼트 점퍼를 제거하고 정상 제어 핀을 연결합니다.
3. 모터 케이블 및 24V 메인 전원($V_{MOT}$)을 결합합니다.
4. 24V 전원을 켜고 축 단독 구동 및 모터/드라이버 발열 상태를 점검합니다.

---

## 4. 하드웨어 보호 체크리스트

* [ ] **Common GND**: MCU(5V/3.3V)의 GND와 모터 파워(24V)의 GND가 서로 통전(하나로 묶임)되어 있는가?
* [ ] **입력 콘덴서**: $V_{MOT}$ 단자 근처에 **100µF 이상의 전해 콘덴서**(내압 35V~50V)가 병렬로 연결되어 역전압 스파크를 방지하는가?
* [ ] **Hot-Plugging
 금지**: 24V 전원이 들어온 상태에서 모터 선이나 드라이버 모듈을 절대 탈착하지 않았는가?
* [ ] **방열판 간섭**: A4988 위에 붙인 알루미늄 방열판 밑면 금속이 주변 헤더 핀이나 SMD 부품 패턴에 직접 닿지 않았는가?

# -------------------------------------------------

# 20260826 프로젝트 아키텍처 및 하드웨어 선정 대화 요약

# --------------------------------------------------

## 1. 개요 및 배경

* **목표:** 홈 로봇(애플 필링 로봇 등)의 제어 시스템 고도화 및 하드웨어 안정성 확보. "안정성 몰빵"이라는 설계 철학 바탕.

* **기존 문제점:** 복잡한 외장 배선으로 인한 노이즈, 접촉 불량(아크) 리스크, 그리고 5V/24V 신호 전압 불일치로 인한 회로의 복잡성.

## 2. 핵심 솔루션: UMOT 초소형 마이크로 드라이버

* **특징:**
  * 모터와 드라이버의 이상적인 모듈화(컴팩트한 사이즈).
  * 32비트 DSP 디지털 칩 내장으로 진동 억제 및 저발열 제어 지원.
  * 모터 파라미터 자동 매칭 기능 탑재.
  * **3.3V 로직 신호 다이렉트 인식:** ESP32 계열(S3, C6 등)의 3.3V GPIO 핀과 레벨 시프터 없이 직결 가능하여 배선 극단적 단순화.

## 3. 라인업 및 옵션 비교

* **Open Loop 전용 (UMIP 시리즈):**
  * 예: `UMIP28` (NEMA 11용)
  * 현재 보유 중인 오픈루프 모터 자원을 그대로 활용하여 개발 및 테스트(Dev)를 진행하기에 최적의 선택.

* **Closed Loop 전용 (UMLP 시리즈):**
  * 예: `UMLP28` (NEMA 11용)
  * 엔코더 피드백을 지원하여 탈조 없는 정밀 제어가 필요한 운영(Prod) 시스템에 적합.

## 4. 최종 전략 (Dev & Prod 이관 시나리오)

1. **개발 (Dev) 단계:** 현재 가지고 있는 오픈루프 모터와 **`UMIP`** 드라이버 조합을 통해 펄스 제어 로직, ESP32 통신, 소프트웨어 안정성을 완벽히 검증.
2. **운영 (Prod) 이관:** 모터 중복 구매 없이 컨트롤러 및 배선 체계를 깔끔하게 유지하며 시스템 완성도 극대화.

** Private Region **
https://github.com/cb2024ok/RedAlert


## 5. Progress History 

    This is a collection of major demonstration videos for that date.

### 🎥 20260906-z-success 
[![active-20260904](https://i.imgur.com/qt775sk.png)](https://imgur.com/gallery/20260906-z-success-E17096q)

[![active-20260904](https://i.imgur.com/1BzjvPJ.png)](https://i.imgur.com/1BzjvPJ.png)

