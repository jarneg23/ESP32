Secure Boot
==================

Background
------------------

* 대부분의 data는 flash에 저장이 된다.
* 중요한 데이터는 스프트웨어가 접근할 수 없는 칩 내부의 efuse에 저장이 되기 때문에 Flash 접근이 secure boot를 위해 물리적 접근으로 부터 보호될 필요가 없다.
* Efuse(efuse BLOCK2)는 secure bootloader key 저장을 위해 사용되어진다.
* 단일 Efuse bit(ABS_DONE_0)이 칩에 scure boot를 영구적으로 가능하게하기 위해 1로 기록된다.
* 초기 소프트웨어 bootloader를 load하거나  subsequence partition & app을 로드하는 boot process 두 단계는 secure boot process에 의해 "chain of trust" 관계로 확인된다.


Secure Boot Process Overvie
-----------------------------

1. make menuconfig 계층구조안에 제공되는 "Secure Boot Configuration" 아래의 secure boot를 활성화 한다.

2. Secure Boot는 기본적으로 build 과정에서 parition table 데이터와 image에 signing하는 것이다.
 "Secure boot pribate signing key" 구성 항목은 PEM 형식이 파일의 ECDSA 공개/개인 키 쌍에 대한 파일 경로이다.

3. software bootloader image는  

4. 