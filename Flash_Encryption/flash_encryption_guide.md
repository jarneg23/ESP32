Flash_Encryption
====================

Background
------------------------------

* 플래의 내용은 AES(고급 암호화 표준)와 256 비트 키를 사용하여 암호화 된다.
* 암호화 키는 칩 내부의 efuse에 저장되며 소프트웨어의 접근으로부터 보호된다.
<br>(efuse : 제작이 완료된 칩의 프로그램을 재 설정하기 위한 장치 )
* 플래시 접근은 플래시 캐시 매핑 기능에 의해 가능해진다.
* 주소 공간에 매핑된 모든 플래시 영역은 읽을 때 알기 쉽게 해독된다.
* 암호화는 ESP32를 PlainText로 flashing 하며 bootloader는 처음 부팅을 할 때 데이터를 암호화 한다
<br>(암호화가 설정되어 있을 경우)
* 모든 종류의 flash가 암호화 되는 것이 아니며 아래와 같은 flash 데이터가 암호화 된다.

1. Bootloader
2. Secure boot Bootloader (secure boot가 활성화 된 경우)
3. Partition talbe
4. 모든 "App" 형식의 partitions
5. 파티션 테이블에 "encrypt" flag가 표시된 parition

> 일부 데이터 파티션은 쉽게 접근 할 수 있도록 암호화 되지 않은 상태로 유지되거나 데이터가 암호화 된 경우
효과가 없는 flash-friendly update 알고리즘을 사용하는 것이 좋다. "NVS" 파티션은 암호화를 할 수 없다.
<br>Partition: 작업 별로 메모리를 분할한 것 , NVS : non volatile storage

* 플래시 암호화 키는 ESP32 칩 내부에 있는 efuse 키 블럭 1에 저장된다. 이 키는 읽기 쓰기가 방지되어 있어서 소프트웨어가 접근할 수 없습니다.
* 칩에서 실행되는 소프트웨어는 플래시 내용을 해독할 수 있지만 플래시 암호화가 진행되면 UART 부트로더가 데이터를 암호화(복호화) 할 수 없다.

Flash Encryption process
------------------------------
1. make menuconfig -> Security Feautre -> Enamble flash encryption on boot 활성화

2. make flash and build -> 초기에는 비암호화 상태로 flash에 쓰여진다.

3. 초기 부팅시 부트 로더는 FLASH_CRYPT_CNT efuse가 0 (공장 기본값)으로 설정되어 있는지 확인하고 하드웨어 Random Number 생성기를 사용하여 플래시 암호화 키를 생성합니다.

4. 암호화 될 모든 파티션이 부트 로더에 의해 그 자리에서 암호화

* (주의) 암호화 과정에서 전원을 차단하지마라 ->  flash의 내용이 손상 되고 암호화 되지 않은 데이터가 flash 된다.

5. 한번 flash가 완료되면 efuse는 uart boot loader가 실행되는 동안 암호화된 flash의 접근을 비활성화 하기 위해 끊어진다.

6. FLASH_CRYPT_CONFIG efuse가 키 비트수를 최대화 하기 위해 최대값으로 기록된다.

7. 마지막으로 efuse 초기값이 1로 기록이 된다. 이 값은 알기 쉽게 flash 암호화 레이어를 활성화 하고 리플래시 횟수를 제한한다.

8. bootloader 가 새로 암호화 된 플래시에서 재부팅 하도록 재설정 된다.

> 처음 부팅시 플래시 암호화가 활성화되면 하드웨어는 Serial reflashing을 통해 최대 3번의 후속 flash update를 한다. 
<br>* 보안 부팅이 활성화 된 경우 물리적으로 reflash 할 수 없다. 
<br>* OTA 업데이트는 이 제한을 고려하지 않고 flash 업데이트를 할 수 있다.
<br>* 개발시 플래시 암호화를 활성화하려면 미리 생성된 키를 사용하여 무제한 적으로 reflashing 할 수 있다.


FLASH_CRYPT_CNT efuse
------------------------
Flash 암호화를 제어하기 위한 8-bit efuse field.
<br>이 efuse값에 "1"로 설정된 비트의 수에 따라 활성 비활성화 할 수 있다.
<br>
* 짝수 비트 (0,2,4,6,8)로 설정될 경우 플래시 암호화가 비활성화되며 암호화된 데이터를 해독할 수가 없다.
* 홀수 비트 (1,3,5,7,9)로 설정될 경우 암호화된 플래시의 Transparent Reading이 활성화 된다.
* efuse 값이 0xFF로 8비트가 모두 설정되면 암호화된 플래시의 Transparent Reading이 비활성화 되고 영구적으로 암호화된 데이터를 읽을 수가 없게 된다.
-> 권한이 없는 코드를 불러오기 위해 이 상태를 사용하는 것을 피하기 위해서 secure boot를 사용하거나
FLASH_CRYPT_CNT efuse가 write_protected되어야 한다.

-- FLASH_CRYPT_CNT efuse process
1.  boot loader가 Enable flash encryption on boot가 활성화 된 경우 암호화되지 않은 데이터가 있는 곳이면 즉시 플래시를 재 암호화 한다. 완료 후 에는 efuse의 다른 비트를 1로 설정하여 홀수 개의 비트로 설정한다.
2.  첫 번째 PlainText 부팅시 비트 수는 완전히 새로운 값 0이고 부트 로더는 암호화 후 비트 수 1 (값 0x01)으로 변경합니다.
3.  다음 PlainText flash update 후 비트 수는 수동으로 4 (값 0x0F)로 업데이트됩니다. 부트 로더를 다시 암호화 한 후 efuse 비트 수를 5 (값 0x1F)로 변경합니다.
4.  다음 PlainText flash update 후 비트 수는 수동으로 4 (값 0x0F)로 업데이트됩니다. 부트 로더를 다시 암호화 한 후 efuse 비트 수를 5 (값 0x1F)로 변경합니다.
5.  최종 PlainText flash update 후 비트 수는 수동으로 6 (값 0x3F)으로 업데이트됩니다. 부트 로더를 다시 암호화 한 후 efuse 비트 수를 7 (값 0x7F)으로 변경합니다.

Enable UART Bootloader Encryption/Decryption
------------------------------------------------
default : flash encryption process의 첫 부팅에서 efuse를 DISABLE_DL_ENCRYPT, DISABLE_DL_DECRYPT and DISABLE_DL_CACHE 로 burn한다.
 * DISABLE_DL_ENCRYPT : UART 부트로더가 동작할 때 플래시 암호화 작업을 사용할수 없게 한다.
 * DISABLE_DL_DECRYPT : UART 부트로더가 동작할 때 FLASH_CRYPT_CNT efuse가 정상적으로 작동할 수 있게 설정되어 있더라도 transparent flash 해독을 불가능하게 한다.
 * DISABLE_DL_CACHE : UART 부트로더가 동작할 때 MMU flash cache 전체를 불가능 하게 한다.

이 efuse 중 일부만 기록하고 처음 부팅 하기 전에 나머지 값을 쓰기 방지할 수 있다.
이 3개의 efuse는 write_protect 비트 1개에 의해 모두 disable되어 지기 때문에 쓰기 보호를 하기
전에 모든 bit를 설정해야 한다.

>esptool.py는 암호화된 Flash를 읽고 쓰기가 지원되지 않기 때문에 write_protecting efuse를 유지하기 위해서 unset하는 것은 현재 유용하지 않다. 
<br>DISABLE_DL_DECRYPT가 설정되지 않은 경우 (0) 물리적 액세스 권한이있는 공격자는 UART 부트 로더 모드 (사용자 정의 스텁 코드 사용)를 사용하여 플래시 내용을 읽을 수 있으므로 플래시 암호화를 비활성화를 하는 것이 좋다.

Using Encrypted Flash
-------------------------
 esp_flash_encryption_enalbed()함수를 호출하는 것으로 esp32 app은 플래시 암호화가 사용가능 한지 코드로 확인할 수 있다.
 <br> - 한번 flash가 암호화 되면 코드에서 flash contents에 접근할 떄는 많은 주의가 필요하다.

 Scope of Flash Encryption
-------------------------
FLASH_CRYPT_CNT efuse의 값이 홀수 비트로 설정이 될 때마다 모든 flash content는 MMU의 flash cache를 통해 알기 쉽게 복호화된다. 여기에는 다음이 포함된다.
<br>
1. Flash 안에 있는 실행 가능한 app code
2. Flash에 저장된 모든 읽기 전용 data
3. esp_spi_flash_mmap()을 통해 접근되는 모든 data
4. ROM bootloader에 의해 읽혀지는 software bootlader image

> MMU flash cache는 모든 데이터를 강제적으로 복호화한다. 암호화 되지 않은 상태로 저장된 데이터는 flash cache를 통해 읽기 쉽게 decrypted 되고 임의의 garbage로 소프트웨어에 표시가 된다.

* 	• SPI(Serial Periphral interface) : 동기화 직렬 데이터 연결 표준
<br>근거리 전용 통신 인터페이스로 동기 Synchronous 통신이라고 한다.
<br>SPI는 Full Duplex 방식이며 마스터 슬레이브 모드가 있다 ( 마스터: 클럭을 담당하는 곳)
<br>SPI는 송수신 라인이 나누어져 있어서 송신과 수신을 동시에 할 수 있다.

* 직렬 통신: 통신 채널이나 컴퓨터 버스를 거쳐 한 번에 하나의 비트 단위로 데이터를 전송하는 과정

Updating Encrypted Flash
--------------------------
-OTA Update
<br> esp_partition_write 기능이 사용되는 한 암호화된 파티션에 대한 OTA update 는 자동으로 암호화된다.

-Serial Flashing
<br> secure booting이 사용되지 않는 다는 가정하에 FLASH_CRYPT_CNT efuse는 serial flashing을 통해 새로운 plaintext로 flash가 업데이트 되도록 허락한다.
<br> 이 process는 plaintext data를 flashing 한 후 이 데이터를 bootloader가 re-encrypt 하도록 FLASH_CRYPT_CNT efuse의 값을 지정한다.

Limited Update
---------------
초기 flash 암호화를 포함하여 이 방식의 serial update cycle은 4회까지만 가능하다.
<br>4번 암호화한 이 후에는 사용할 수 없으며 FLASH_CRYPT_CNT efuse가 최대값 0xff가 되면 암호화는 영원히 사용할 수 없다.
<br>OTA update와 미리 생성한 flash 암호화 키를 사용한 reflash의 경우에는 이 제한을 초과할 수 있다.

Serial Flashing 주의 사항
-------------------------------
	1) Serial을 통해 reflashing 할 때 초기에 plaintext data로 쓰여진 모든 partition을 reflash한다. -> 현재 선택된 OTA 파티션이 아닌 app 파티션을 넘어갈 수 있다. (plaintext app 이미지가 없으면 다시 암호화 될 수 없다)
	
	2) "encrypt" 표시가 된 모든 파티션은 무조건 적으로 재 암호화 된다 -> 이미 암호화된 데이터도 두번 암호화 되어 손상될 수 있다. (make flash를 사용하여 암호화해야 하는 모든 파티션을 flash 해야 한다.)
	
    3) Secure boot가 사용가능하다면 Secure booting은 "Reflashable" 옵션을 사용하고 미리 생선된 키를 제외하고 ESP32에 구워지지 않으면 직렬을 통해 reflash 할 수 없다. -> 이 경우 plaintext secure boot digest와 bootloader image를 offset 0x0에서 다시 flashing 할 수 있다. -> 다른 plain text data를 flashing 하기 전에 이 digest를 re-flash 할 필요가 있다.


Serial Re-Flashing Procedure
-----------------------------
1) 평소처럼 application을 build 한다.

2) (make flash or esptool.py) 처럼 plaintext와 함께 device를 Flash한다. - bootloader를 포함한 이전에 암호화된 partition을 모두 flash 한다.

3) 이 시점에서 bootloader가 암호화 되어 있을거라 생각하지만 plaintext이기 때문에 flash read err,1000 이 발생한다.

4) espefuse.py burn_efuse FLASH_CRYPT_CNT 명령어를 실행하여 FLASH_CRYPT_CNT efuse를 굽는다.
-> 자동으로 bit count를 1 증가 시킨다.

5) device를 Reset한다 그리고 plaintext partition을 재 암호화 한 후 FLASH_CRYPT_CNT efuse를 다시 구워서 다시 암호화를 활성화 한다.


Disabling Serial Updates
--------------------------
Serial을 통해 plaintext가 업데이트 되는 것을 방지하기 위해 처음 booting이 완료되고 flash 암호화가 활성화된 이후 FLASH_CRYPT_CNT efuse가 write protect 되도록 espefuse.py를 사용한다 

>espefuse.py --port PORT write_protect_efuse FLASH_CRYPT_CNT

이로 인해 flash 암호화를 다시 활성화 비활성화 할 수 없습니다.

Reflashing via Pregenerated Flash Encryption Key
--------------------------
호스트 컴퓨터에서 flash 암호화 키를 미리 생성하여 ESP32의 efuse 키 블럭에 구워서 일반 Plaintext flash update없이 esp32로 flash를 할 수 있다.

bootloader가 매번 reflash될 필요가 없기 때문에 secure boot가 활성화된 경우 4 번의 reflash 제한을 제거할 수 있어서 유용하다.

<h4>다만 이 경우 제품을 위한 암호화를 사전 생성할 때 고품질의 난수 생성프로그램으로 키를 생성하고 여러 장치에서 동일한 flash 암호화 키를 공유하지 않도록 해야 한다.

Pregenerating a Flash Encryption Key
-------------------------------------
Flash 암호화 키는 32 바이트 random data 이며
>espsecure.py generate_flash_encryption_key my_flash_encryption_key.bin

위 명령어를 통해 생성할 수 있다.

만약 아래 명령어를 사용하여 secure boot를 사용하며 secure boot signing key를 가지고 있는 환경에서 flash 암호화 키로 사용할 수 있게 secure boot signing key에서 중요한 SHA-256 digest를 생성할 수 있다.
>espsecure.py digest_private-key --keyfile secure_boot_signing_key.pem my_flash_encryption_key.bin

secure boot의 reflashable mode가 활성화되어 있다면 같은 32바이트를 secure boot digest key로 사용할 수 있다.

Burning Flash Encryption Key
-------------------------------------
한번 flash 암호화 키를 생성했다면 ESP32의 efuse key block에 암호화 키를 구울 필요가 있다. 이 작업은 반드시 첫 암호화 booting 전에 해야한다. 그렇지 않으면 ESP32가 software가 접근 수정할 수 없는 임의의 키값을 생성한다.

> espefuse.py --port PORT burn_key flash_encryption my_flash_encryption_key.bin

Reflashing with pregenerated key
-------------------------------------
첫 부팅에서 암호화가 활성화된 이후 암호화된 image reflashing은 추가적인 메뉴얼 단계를 요구한다.
plaintext data를 flash 하기 위해 사용하는 일반적인 command는 아래와 같다.

 > esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x10000 build/my-app.bin

위 명령어를 사용하면 Binary app image build/my-app.bin는 offset 0x10000에 쓰여진다. 

 > espsecure.py encrypt_flash_data --keyfile my_flash_encryption_key.bin --address 0x10000 -o build/my-app-encrypted.bin build/my-app.bin

이 명령어는 supplied key를 사용하여 my-app.bin 파일을 암호화 하여 암호화된 파일 my-app-encrypted.bin 파일을 생성한다. address 인수가 바이너리를 플래싱 할 주소와 일치하는지 확인해라.

확인 후 
 > esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash 0x10000 build/my-app-encrypted.bin

 위 명령어를 통해 flash에 data를 쓸 수 있다. 이 경우 우리가 flash할 데이터가 이미 암호화 되어 있기 때문에 efuse 조작이 필요한 단계를 생략할 수 있다.                                                                                                                                                                                        

Disabling Flash Encryption
-----------------------------------
만약 실수로 flash 암호화를 활성화 했다면 다음 boot에서 지속적으로 flash read err, 1000이라는 에러가 출력이 될 것이다. 그럴경우 다음의 순서대로 암호화를 비활성화 할수 있다. 

1. make menuconfig -> Enable flash encryption boot를 체크해제 한다.
2. 메뉴를 종료하고 구성사항을 저장한다.
3. make menuconfig 다시 한번 실행 후 -> disabled 옵션이 체크되어 있지 않은지 다시 확인한다.
4. make flash를 실행한다 -> bootloader, app 등이 flash 암호화 활성화 없이 build되고 flash된다.
5. espefuse.py를 실행하여 FLASH_CRYPT_CNT efuse를 비활성화 해라 :: burn_efuse FLASH_CRYPT_CNT       

그러면 eps32가 reset되고 flash 암호화가 비활성화 되며 bootloader는 보통의 경우처럼 boot된다.

Limitations of Flash Encryption
------------------------------------
Flash 암호화는 암호화 된 Flash의 Plaintext 판독을 방지하여 권한이없는 판독 및 수정으로부터 펌웨어를 보호한다.

* flash 암호화는 키만큼 강력하다. 따라서 첫 번째 부팅시 장치에서 키를 생성하는 것이 좋다. 장치가 꺼진 상태에서 키를 생성하는 경우 (미리 생성 된 플래시 암호화 키로 새로 고침 참조) 올바른 방식으로 만들어라.
* 모든 data가 암호화 되어 저장되는 것이 아니라 flash에 데이터를 저장하는 경우 사용중인 방법, data가 flash 암호화를 지원하는지 확인해라
* 플래시 암호화는 침입자가 플래시의 상위 레벨 레이아웃을 이해하는 것을 방해하지 않습니다. 인접한 16 바이트 AES 블록의 모든 쌍에 대해 동일한 AES 키가 사용되기 때문입니다. 이러한 인접한 16 바이트 블록에 동일한 내용 (예 : 빈 영역 또는 패딩 영역)이 있으면이 블록이 암호화되어 암호화 된 블록 쌍을 일치시켜 생성합니다. 이것은 공격자가 암호화 된 장치를 상위 수준으로 비교할 수있게합니다 (예 : 두 장치가 동일한 펌웨어 버전을 실행하고 있는지 확인).
* 같은 이유로 공격자는 인접한 16 바이트 블록 쌍 (32 바이트 정렬)에 동일한 내용이 포함되어 있는지 항상 알 수 있습니다. 민감한 데이터를 플래시에 저장하는 경우 플래시 메모리를 설계하여 카운터 바이트 또는 다른 16 비트가 아닌 다른 값을 사용하면 충분합니다.
* 플래시 암호화만으로 공격자가 장치의 펌웨어를 수정할 수있는 것은 아닙니다. 장치에서 인증되지 않은 펌웨어가 실행되는 것을 방지하려면 Secure Boot와 함께 플래시 암호화를 사용하십시오.


Burning a key
------------------------------------
This command can brick your ESP32

>espefuse.py --port /dev/DONOTDOTHIS burn_key secure_boot keyfile.bin

일반적으로 efuse 블록이 이전에 기록되지 않은 경우에만 키가 레코딩 됩니다.
--force-write-always 옵션을 사용하면 이것을 무시하고 키를 구울 수 있다.