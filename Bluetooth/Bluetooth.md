ESP-IDP의 블루투스
=======================================

Bluetooth protocol
---------------------------------------
>Controller Stack
* PHY,BaseBand,Link Controller, Link Manager, Device Manager, HCI and Other module
* 하드웨어 인터페이스 관리 및 링크 관리에 사용

>Host Stack
* L2CAP, SMP, SDP, ATT, GATT, GAP 및 다양한 프로파일
* 응용프로그램 계층에 대한 인터페이스로 작동
* 응용프로그램 계층이 Bluetooth 시스템에 엑세스 가능
* 컨트롤러와 동일한 장치 또는 다른 장치에 구현 가능


ESP-IDP의 블루투스 호스트와 컨트롤러 구조
----------------------------------------

<img src="./bluetooth structure.png">


블루투스 호스트와 컨트롤러의 사이에는 3가지 형식의 구조가 있다.

>첫 번쨰 경우
* BLUEDROID가 Bluetooth 호스트로 설정
* VHCI(소프트웨어 구현 가상 HCI 인터페이스 - Host와 연결하기 위한 인터페이스)가 Controller와    Bluedroid의 통신에 사용
* Bluedroid와 Controller가 한 장치에 구현되므로 추가 장치가 필요하지 않다.

>두 번째 경우
* ESP32는 Bluetooth 컨트롤러로만 사용
* Bluetooth 호스트를 구현 (다른 장치에)
* 휴대폰, PAD, PC오 유사.

>세 번째 경우
* 두 번째오 유사하지만 BQB 컨트롤러 테스트에서 ESP32는 UART를 IO 인터페이스로 사용하여 테스트 도구와 연결하는 경우




