Secure Boot
==================
secure boot는 chip에서 code만 실행할 수 있도록 하능 기능으로 매 reset시 flash에서 load된 data는 verified되어 진다.

Background
------------------
* 대부분의 data는 flash에 저장이 된다.
* 중요한 데이터는 스프트웨어가 접근할 수 없는 칩 내부의 efuse에 저장이 되기 때문에 Flash 접근이 secure boot를 통한 물리적 접근으로 부터 보호될 필요가 없다.
* Efuse(efuse BLOCK2)는 secure bootloader key 저장을 위해 사용되어진다.
* 단일 Efuse bit(ABS_DONE_0)이 칩에 scure boot를 영구적으로 가능하게하기 위해 1로 기록된다.
* 초기 소프트웨어 bootloader를 load하거나  subsequence partition & app을 로드하는 boot process 두 단계는 secure boot process에 의해 "chain of trust" 관계로 확인된다.


Secure Boot Process Overvie
-----------------------------

1. make menuconfig 계층구조안에 제공되는 "Secure Boot Configuration" 아래의 secure boot를 활성화 한다.

2. Secure Boot는 기본적으로 build 과정에서 parition table 데이터와 image에 signing하는 것이다.
 "Secure boot pribate signing key" 구성 항목은 PEM 형식이 파일의 ECDSA 공개/개인 키 쌍에 대한 파일 경로이다.

3. software bootloader image는 esp-idf에서 보안 부팅 지원이 활성화되고 보안 부팅 서명키의 공개키 부분이 컴파일 된 상태로 만들어 진다. 이 이미지는 flash의 0x1000 offset에 저장된다.

4. 첫 번째 부팅에서 software bootloader는 secure boot을 활성화하기 위해서 다음과 같은 과정을 따른다.
<br>1) 하드웨어 보안 부팅 지원은 장치 보안 bootloader 키 (하드웨어 RNG를 통해 생성되고 efuse안에 read/write proteced되어 저장) 와 secure digest를 생성한다. digest는 ket,IV, bootloader image contents로부터 파생된다.
<br>2) secure digest는 flash의 0x0에 저장
<br>3) 보안 부팅 구성에 따라, efuse는 JTAG, ROM BASIC interpreter를 비활성화하기 위해 구워진다.
<br>4) ABS_DONE_0 efuse를 굽는 것으로 bootloader는 영구적으로 secure boot를 사용할 수 있다. software bootloader 보호되어 진다.

5. 후속 부팅시에는 ROM bootloader는 secure boot efuse가 구워지고 0x0에 저장된 digest를 읽고 하드웨어 secure boot support을 사용하여 새롭게 계산된 digest와 비교한다. -> digest가 일치하지 않는 다면 부팅을 계속 한다. (digest와 비교는 전적으로 하드웨어에서 수행되어진다, 그리고 계산된 digest는 소프트웨어로 읽을 수 없다.)

6. secure boot mode가 실행될 때 software bootloader는 secure boot signing key를 모든 subsequence partition tables, app image가 부팅되기전에 추가된 서명을 확인하기 위해 사용한다.

Key
------------------------------------
아래의 키는 secure boot process에서 사용되어진다.
* "secure bootloader key" : Efuse block 2에 저장되어지는 256-bit AES key. bootloader는 내부 하드웨어 무작위 수 생성기를 통해 자체적으로 이 키를 생성할수 있다. Efuse는 secure boot이 활성화되어지기 전에 이 키를 read&write proteced로 유지한다.
* "secure boot signing key" : PEM 형식의 표준 ECDSA public private key pair 이다.
이 key pair의 public key는 소프트 웨어 부트로더에 컴파일 되어지며 second stage booting을 확인하기 위해 사용되어 진다. publick key는 secret하게 유지될 필요가 없어서 자유롭게 분배될 수 있다. privte key는 반드시 안전하게 private상태로 유지되어야 한다. 이 키를 가진 모든 사람이 secure boot에 구성되고 public key와 매칭되는 모든 bootloader에 인증할 수 있다.



Secure boot 사용 방법
-----------------------------------------
Secure Boot option에는 "One-time Flash"와 "Reflashable" 2가지가 있다.

--One-time Flash--
<br>이 모드에서는 device가 device 외부에 절대 저장되지 않는 고유 키를 가지고 있기 때문에 production 장치에 사용하는 것이 권장되는 구성이다.
> 1) make menuconfig에서 "Secure boot Configuration" -> "One-time-Flash"를 선택한다. 
> 2) secure boot signing key 이름을 설정한다. 이 옵션은 secure boot가 활성화 되어야 나타난다. 상대경로는 프로젝트 디렉토리에서 평가되기 때문에 아직 파일이 존재하지 않아도 되며 파일은 시스템의 모든 위치에 존재할 수 있다.
> 3) 다른 메뉴를 원하는 대로 구성한다. 부트로더는 한번만 flash 할수 있음으로 "Bootloader Config"옵션은 주의해서 사용해라.
> 4) 처음 make를 실행한다. 만약 signing key를 발발견 할 수 없다면 espescure.py genreate_signing_key를 통해 signing키를 생성하라는 에러 메세지가 출력될것이다.
> <br> *** 이 signing key는 OS, Python에서 사용할 수 있는 최고의 난수 생성기를 사용하여 만들어 진다. 만약 난수가 매우 약하다면 private key 또한 매우 약할것 이다.
> <br> *** production 환경에서는 openssl 또는 다른 산업 표준암호화 프로그램을 사용하여 Key pair를 만드는 것을 추천한다.
> 5) secure boot 사용가능한 bootloader를 build하기 위해 make bootloader 실행 해라. 이 make의 출력은 esptool.py write_flash를 사용하여 flashing하는 명령어를 유도하는 것이 포함되어 있다.
> 6) flash를 위해 bootloader가 준비되면, 특정한 명령어를 실행해라 그리고 flash가 완료될 때까지 기달려라.
이 단계는 make에 의해 수행되어지지 않기 때문에 스스로 입력을 해야하며 bootloader는 "one time flash" 구성에 의해 이 단계 이후로는 바꿀수 없다.
> 7) parition table과 방금 만든 app image를 flash하고 build하기 위해 make flash를 실행해라. app image는 4)에서 생성한 signing key를 사용하여 인증된다. 만약 secure boot가 활성화되어 있다면 make flash는 bootloader를 flash하지 않는다.
> 8) ESP32가 Reset되고 flash한 소프트웨어 bootloader가 부팅된다. 소프트웨어 bootloader는 칩에서 secure boot를 사용하고 app image signature를 확인한 후 app을 부팅한다. secure boot가 사용되고 build 구성을 하는 동안 발생한 error 없는지 확인하기 위해 esp32로 부터 일련의 console 출력을 확인해야 한다.
> <br> *** Secure boot는 시스템이 완전히 구성되기 전에 사고를 예방하기 위해서 이용가능한 partition table과 app image가 flash 되어지기 전에는 활성화 되어지지 않는다.
> 9) 후속 부팅에서 secure boot 하드웨어는 소프트웨어 bootloader가 변화하지 않았다는 것을 확인할 것이다(secure bootloader key를 사용) 그리고 소프트웨어 bootloader가 parition table과 app image의 서명을 확인할 것이다.(secure boot signing key의 public key를 사용)

--Re-Flashable Software Bootloader--
<br> 이 모드에서는 secure boot loader key에 사용되는 256-bit key 파일을 제공할 수 있다. 이 키 파일을 가지고 새로운 bootloader image를 생성하고 bootloader를 보호 할 수 있다. esp-idf build process에서 이 256-bit key 파일은 위의 generate_signing_key 과정에서 생성된 app signing key로부터 파생된다. private key SHA-256 digest는 256-bit secure bootloader key로써 사용된다. 이것은 하나의 private key를 생성하고 보호하기만 하면되므로 편리하다.
<br>***하나의 secure boot key를 생성하여 모든 장치로 flash 하지 않는 것을 추천한다. 제품 생산 환경에서는 "One-Time-Flash" option을 사용하는 것을 추천한다.

> 1) make menuconfig -> Bootloader Config -> Secure Boot -> Reflashable.
> 2) secure boot signing key 이름을 설정한다. 이 옵션은 secure boot가 활성화 되어야 나타난다. 상대경로는 프로젝트 디렉토리에서 평가되기 때문에 아직 파일이 존재하지 않아도 되며 파일은 시스템의 모든 위치에 존재할 수 있다.
> 3) 다른 메뉴를 원하는 대로 구성한다. 부트로더는 한번만 flash 할수 있음으로 "Bootloader Config"옵션은 주의해서 사용해라.
> 4) 처음 make를 실행한다. 만약 signing key를 발견 할 수 없다면 espescure.py generate_signing_key를 통해 signing키를 생성하라는 에러 메세지가 출력될 것이다.
> 5) make bootloader를 실행해라. 256-bit key file이 생성될것이다, private key로부터 파생된 이 파일은 signing에 사용된다. 2 단계의 flash가 출력된다 - 1. efuse에 bootloader key를 쓰기위해 사용되는 espefuse.py burn_key 2. 이전에 계산된 digest로 bootloader를 reflash하기 위해서 사용될 수 있다.
> 6) one-time-flash 과정의 6번부터 시작해라.



Generating Secure Booot Signing Key
-------------------------------------
> oppenssl ecparam -name prime256v1 -genkey -noout -out my_secure_boot_signing_key.pem

위 openssl 명령어 라인을 사용하여 signing key를 생성할 수 있다. 그리고 signing key를 private하게 유지하는 것이 secure boot system의 장점인 것을 기억해야 한다.

Remote Signing of Images
-------------------------
production build 환경에서 build machine에 signing key를 사용하는 것 보다 원격 signing server를 사용하는 것이 더 좋다. sepsecure.py 명령 줄 프로그램은 원격 시스템에서 secure boot를 하기 위해 app image & partition table data에 sign하는 것에 사용할 수 있다.
<br>remote signing을 사용하려면 구성에 "Sign binaries during build" 옵션을 비활성화 해라. private signing key는 build system에 있을 필요가 없다 하지만 public key는 bootloader에 컴파일 되어 있기 때문에 필요하다 (OTA 업데이트에 image sign을 확인 하는데 사용).

private key로 부터 public key를 추출하는 방법
> espscure.py extract_public_key --keyfile PRIVATE_SIGNING_KEY PUBLIC_VERIFICATION_KEY

public signature verifivation key의 경로가 secure bootloader를 build하기 위해 menuconfig 아래의 "Secure boot public signature verification key"에 설정될 필요가 있다.

App image와 partition table이 build된 이후에 build system은 espsecure.py를 사용하여 signing step을 출력한다.
> espsecure.py sign_data --keyfile PRIVATE_SIGNING_KEY BINARY_FILE

위 명령어는 존재하는 binary에 image signature를 추가한다. 별개의 sign된 binary를 쓰기 위해서는 -output 인자를 사용할 수 있다.
> espsecure.py sign_data --keyfile PRIVATE_SIGNING_KEY --output SIGNED_BINARY_FILE BINARY_FILE


Secure Boot Best Practice
----------------------------------------
* 시스템에 signing key를 생성
* 매 순간 signing key를 private하게 유지.\
* espsecure.py를 사용하여 제 3자가 key 생성 또는 signing process의 모든 측면을 관찰하는 것을 금지해라.
* Secure Boot Configuration의 모든 secure boot 옵션을 활성화 해라. 여기에는 플래시 암호화, JTAG 비활성화, BASIC ROM 인터프리터 비활성화 및 UART 부트 로더 암호화 플래시 액세스 비활성화가 포함됩니다.
* flsh contents가 읽혀지는 것을 막기 위해서 flash encryption과 함께 secure boot을 사용해라.















