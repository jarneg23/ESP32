Application Startup Flow
============================
: ESP-IDF application의 app_main() 함수가 불려지기 전에 일어나는 다양한 단계에 대한 문서


----------------------------
High-level statup 프로세스
----------------------------
1. ROM의 First-stage bootloader는 flash offset 0x1000 으로부터 RAM에 Second-stage bootloader 이미지를 불러온다.

2. Second-stage bootloader는 flash로부터 partition table과 main app image를 불러온다.

3. Main app image 실행. 이 때 두 번째 CPU, RTOS 스케쥴러가 시작된다.


First stage bootlader
----------------------------
Soc가 reset된 이후 APP CPU가 reset 상태로 유지되는 동안 reset vector code를 실행하면 PRO CPU가 즉시 작동을 시작한다.

Startup 과정에서 PRO CPU(protocol)는 모든 초기화를 수행하고 APP CPU(application) reset은 응용프로그램 시작 코드의 call_start_cpu()에서 해제된다.

reset vector code는 esp32 칩 ROM mask의 0x40000400에 있으며 수정할 수 없다.

(soc (system on a chip) : 하나의 집적회로에 집적된 컴퓨터나 전자 시스템 부품 -> 디지털 신호, 아날로그 신호, 혼성 신호, RF기능 등이 단일 칩에 구현되어 있다. )

Startup code는 reset vector code에서 호출되며 GPIO_STRAP_REG 레지스터에서 bootstrap 핀 상태를 확인 하여 boot mode를 결정한다. 재설정된 이유에 따라 다음 단계가 수행된다.

1. deep sleep으로 부터 Rest
    <br> : RTC_CNTL_STORE6_REG 의 값이 0이 아니고 RTC_CNTL_STORE7_REG의 RTC 메모리의 CRC값이 valid할 때 RTC_CNTL_STORED6_REG를 entry point 주소로 사용하고 즉시 그곳으로 jump한다.
    그렇지 않을 경우에는 power-on은 reset하는 것 처럼 boot 절차를 진행한다.

2. power-on reset, software SOC reset, watchdog SOC reset
    <br> : UART 또는 SDIO 설치 모드가 요구되면 GPIO_STRAP_REG 레지스터를 확인한다. 이 경우 UART 또는 SDIO를 구성하고 code가 설치되기를 기다린다. 그렇지 않으면 소프트웨어 CPU reset으로 인한 것처럼 부팅을 계속한다.

3. software CPU reset, watchdog CPU reset
    <br> : EFUSE 값에 기초하여 SPI flash를 구성하고 flash로 부터 code를 불러오는 것을 시도해라.
    flash로부터 코드를 불러오는 것에 실패를 했다면 RAM에 BASIC 인터프리터를 풀고 BASIC 인터프리터를 시작한다.

    >Note: 이러한 일이 발생할 때 RTC watchdog는 여전히 이용가능하다 그래서 interpreter에 의해 어떠한 입력도 받지 않는다면 watchdog는 몇 백 ms 안에 SOC를 reset하고 전체 process를 반복한다. 만약 UART로 부터 interpreter가 input을 받으면 watchdog는 사용할수 없게 된다. 
    <br>* watchdog : 컴퓨터의 오작동을 탐지하고 복구하기 위해 쓰이는 전자 타이머.
 
Application binary image는 flash 시작 주소 0x1000에서 불려진다. 첫 flash의 4kb 섹터는 application image의 signature와 secure boot IV를 저장하기 위해서 사용되어진다. 


Second stage bootloader
-----------------------------
flash에 0x1000 offset에 위치한 Binary image가 second stage bootloader이다. Second stage bootloader source code는 components/bootloader directory에서 이용할 수 있다. 
<br> Second stage bootloader는  flash layout에 유연성을 추가하고 flash encryption, secure boot, OTA update와 관련된 다양한 flow를 허용한다.

Second stage bootloader는 offset 0x8000에 있는 partition table을 읽는다. 그리고 OTA info partition에 있는 data에 기반하여 부팅될 partition을 찾는다. 선택한 partition에 대해서 Second stage bootloader는 IRAM 및 DRAM에 매핑 된 데이터 섹션과 코드 섹션을 load address에 복사한다. Second stage bootloader는 PRO CPU와 APP CPU 모두에 flash MMU 설정하지만 PRO CPU의 flash MMU만 사용할 수 있다. 그 이유는 Second stage bootloader code가 APP CPU cache에 의해 사용되어지는 메모리 공간에 불려지기 때문이다. APP CPU의 캐시를 활성화하는 것은 application의 역할이다. code가 load되고 flash mmu가 설정이 되면 Second stage bootloader가 binary image header에 있는 application의 entry point로 이동한다.
<br>(mmu : 메모리 관리 장치, 가상 메모리 -> 실제 메모리 주소로 변환)

Application statup
---------------------
ESP-IDF application의 entry point는 call_start_cpu0 함수 이다. 이 함수의 2가지 주된 기능은 heap allocator를 사용할수 있고 APP CPU를 call_start_cpu1인 entry point로 점프시키는 것이다. PRO CPU code는 APP CPU의 entry point를 설정하고 APP CPU reset을 해제하며 APP가 실행된 것을 나타내는 APP CPU위에 실행되는 code에 의해 설정되는 global flag가 설정되기를 기다린다. 
<br>start_cpu0와 start_cpu1은 응용프로그램마다 초기화 시퀀스를 변경해야하는 경우 응용 프로그램에서 재정의 할수있다. start_cpu0의 기본 구성은 menuconfig에 선택에 따라 구성요소를 초기화하고 활성화 한다.
모든 구성요소들이 초기화되면 main task가 생성되고 FreeRTOS 스케쥴러가 시작된다.

PRO CPU가 start_cpu0함수 안에서 초기화되는 동안 start_cpu1 함수의 APP CPU spin은 PRO CPU위에서 시작되어지기 위해 스케쥴러를 기다린다. 한번 스케쥴러가 PRO CPU위에 시작되어지면 APP CPU의 code가 스케쥴러에 의해 시작된다.

Main task는 app_main 함수가 실행되는 task이다. menuconfig에서 stack 사이즈와 우선순위를 설정할 수있다.

Application memory layou
-------------------------
ESP32 chip은 flexible memory mapping 기능을 가지고 있으며 application code는 다음 메모리 공간중 한곳에 위치할 수 있다.

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1. IRAM(instruction RAM) : 내부 SRAM0 영역의 일부를 instruction RAM에 할당한다. PRO,APP CPU 캐시에 사용되는 처음 64kb 블럭을 제외한 나머지 메모리 범위는 RAM에서 실행해야하는 응용프로그램의 일부를 저장하는데 사용된다.

> 일부 application code를 IRAM에 배치해야 할 경우 IRAM_ATTR define을 사용하여 수행 할 수 있다.
<pre><code>#include "esp_attr.h"
void IRAM_ATTR gpio_isr_handler(void* arg)
{
    ....
}
</code></pre>

다음의 경우에는 application의 부분이 IRAM에 위치해야한다.
* ESP_INTR_FLAG_IRAM이 사용되면 Interrupt Handlers는 IRAM에 배치해야 한다. 이 경우 IRAM, ROM에 있는 함수만을 호출할 수 있다. 
> 현재 모든 FreeRTOS API는 IRAM에 배치되어 인터럽트 핸들러에서 안전하게 호출 할 수 있다. 만약 ISR이 IRAM에 위치해 있다면 ISR에 의해 사용되는 모든 cosntan data와 함수는 DRAM_ATTR을 사용하여 DRAM에 위치해야 한다.

* flash로부터 code를 불러오는 것과 관련한 단점을 줄이기 위해서 IRAM에 timing critical code를 저장한다.
esp32는 32kb cache를 통해 flash로부터 data와 code를 읽는다. 몇몇의 경우 IRAM에 저장된 함수는 cache miss에 의해 발생하는 delay를 줄일수 있다.

2. IROM(code executed from Flash) : 함수가 명시적으로 IRAM or RTC 메모리에 위치하지 않는다면 flash안에 위치한다. ESP-IDF 는 은 0x400f0000 - 0x40400000 영역의 시작 부분부터 flash에서 실행되어야 하는 코드를 배치한다. 위 시작시 Second stage bootloader는 플래시 MMU를 초기화하고 코드가있는 위치를이 영역의 시작 부분에 매핑합니다. 이 영역에 대한 접근은 0x40070000 - 0x40080000 범위의 두 개의 32kb 블럭을 사용하여 cache된다.

3. RTC fast memory : deep sleep mode에서 깨어난 이후 실행되는 code는 RTC memory 안에 저장된다.

4. DRAM (data RAM) : Non-constan static data와 0으로 초기화된 data는 0x3FFB0000 - 0x3FFF0000 256kb 범위 안에 linker에 의해서 저장된다. 만약 Bluetooth stack이 사용된다면 이 영역의 범위는 64kb로 줄어든다. 또한 trace memory가 사용된다면 이 영역의 길이는 16kb 또는 32kb로 줄어든다. static data가 저장된 이후 남아있는 모든 공간은 runtime시 heap에 사용된다. 만약 constant data가 isr에서 사용되어졌ㄷ마년 DRAM에 저장될 수 있다. 그렇게 할 경우 DRAM_ATTR define이 사용되어야 한다.
<pre><code>DRAM_ATTr const char[] format_string = "%p %x";
char buffer[64];
sprintf(buffer,format_string,ptr,val);
</code></pre>

ISR에서 printf 및 다른 출력함수를 사용하는 것은 권장되지 않는다, 디버깅을 위해서는 ESP_EARLY_LOGx 매크로를 사용하고 이 경우에는 TAG와 포멧 문자열이 모두 DRAM에 저장되어 있는지 확인해라.

5. DROM (data stored is Flash) : 기본적으로 constant data는 Flash MMU와 cache를 통해 외부 flash memory에 접근하기위해 사용되어지는 4MB 범위안에 linker에 의해서 저장된다. (0x3f400000 - 0x3F800000)
(Exception은 complier에의 application으로 임베디드된 literal constant 이다.)

6. RTC slow memory : RTC memory로 부터 실행되는 code에 의해 사용되는 Global , static 변수들은 반드시 RTC slow memory에 저장된다.

-------------------------



