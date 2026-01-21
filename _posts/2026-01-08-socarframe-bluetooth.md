---
layout: post
title:  "쏘카프레임 - 블루투스 모듈"
subtitle: 쏘카는 어떻게 앱으로 차 문을 열까
date: 2026-01-08 15:00:00 +0900
category: socarframe
background : "/assets/images/onboarding-bg.jpg"
author: abel
comments: true
tags:
    - socarframe
    - mobile
    - app
---

## 쏘카는 어떻게 앱으로 차 문을 열까

<img src="/img/2026-01-08-socarframe-bluetooth/smartkey.png" width="80%"/>

쏘카는 앱에서 다음과 같은 기능들을 제공합니다.

- 쏘카 차량 제어 (문 열기 / 문 잠금 / 비상등 켜기 / 경적 울리기 등) 🚙

- 일레클 제어 (잠금 / 해제 / 반납 등) ⚡️

- 따릉이 제어 (잠금 / 해제 / 반납 등) 🚲

이러한 것들이 가능한 이유는 각 이동수단 내부에 데이터 송수신이 가능한 단말이 숨어있기 때문입니다. 이 단말들이 앱과 데이터를 송수신하는 방법에는 서버를 통한 방법, 블루투스를 통한 방법, 총 두 가지가 있으며 쏘카는 이 두 가지 방법을 적절히 혼합해서 사용하고 있습니다.

예를들어, 쏘카 차량 내부에는 데이터 송수신 단말인 STS(Socar Telematics System) 가 존재합니다. 이 단말이 서버나 블루투스로부터 “차 문을 열어라” 라는 명령 데이터를 받게 되면, 정합성 및 보안성을 검사한 뒤 차 문을 열어주게 되는 흐름입니다. 일반적으로 블루투스 통신이 서버 통신보다 신뢰성이 있기 때문에, 블루투스 통신이 가능한 상황이라면 먼저 블루투스 통신을 사용하며, 그렇지 않은 경우 서버 통신을 사용합니다.

이러한 비즈니스 로직 맥락 안에서, 쏘카의 모바일 개발자들은 여러 기능에 대한 블루투스 코드를 작성하게 됩니다. 이 글에서는 쏘카, 일레클, 따릉이 피쳐에서 모두 활용하는 쏘카프레임 - 블루투스 모듈에 대해 소개하려 합니다.



## 쏘카프레임 - 블루투스 모듈 탄생 배경

Android, iOS 플랫폼이 제공하는 블루투스 개발 환경은 본질적으로 블루투스 하드웨어 추상화 계층에 대응하는 것이기 때문에, 기대되는 동작 방식이나 운용 방식이 존재하기 마련입니다. 하지만 각 플랫폼이 제공하는 블루투스 API 는 하드웨어의 개념적 구조에 맞지 않게 파편화 되어있으며, 서비스 요구 사항에 대응하기 어렵습니다. 쏘카프레임 블루투스 모듈은 파편화된 것을 다시 하나로 추상화하고 서비스 요구 사항 대응 능력을 확보할 수 있도록 설계했습니다. 파편화는 크게 2가지 이야기로 정리할 수 있습니다.

**1\. 하드웨어 맥락(Context)을 응집하지 못하는 플랫폼 API**

어떤 블루투스 기기(DeviceX)가 있을 때, 사람은 그 DeviceX 의 고유한 식별 정보와 통신 규격, 그리고 수행 가능한 동작 등을 하나의 덩어리로 인식하게 됩니다. 예를들어 쏘카의 STS 단말기라면, CarID 를 포함한 식별 정보와 핸드셰이크 규칙, 차량 상태 리포팅 기능 등의 세계관이 하나의 클래스 안에 응집되어 있기를 기대할 것입니다. 하지만 모바일 플랫폼이 제공하는 API(iOS: CBPeripheral, Android: BluetoothDevice)는 이러한 맥락을 담지 못하는 Low-Level 인터페이스에 불과합니다. 개발자는 흩어진 스캔, 연결, 송수신 Delegate 콜백들과 의미가 부재된 Raw Data 속에서 기기의 맥락을 직접 재구성해야 하는 어려움이 있습니다.

쏘카프레임은 이를 해소하기 위해 플랫폼 API 위에 새로운 추상화 계층을 쌓아 올려, 기기의 정체성에 대한 정보를 담을 수 있도록 제공합니다. 결과적으로 앱 개발자들은 비즈니스 로직을 작성할 때 기기 클래스가 직접 제공하는 고유 인터페이스들을 참고해서 서비스 구현에만 집중한 코드를 작성할 수 있습니다. 또한, 새로운 기기가 도입되더라도 기존 구조를 해치지 않고 유연하게 확장할 수 있는 토대를 마련합니다.

**2\. iOS 와 Android 플랫폼 코드 차이**

iOS 의 CoreBluetooth 와 android.bluetooth 사이에도 동작 차이가 존재합니다. 한 가지 예로 주변 기기를 스캔하는 코드 동작성에 차이가 있는데, iOS 의 [scanForPeripherals(withServices:options:)](https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/scanforperipherals(withservices:options:)) 는 스캔 도중 새로운 스캔을 허용하지 않는 반면, Android 의 [startScan(filters, settings, callback)](https://developer.android.com/develop/connectivity/bluetooth/ble/find-ble-devices?hl=ko) 은 이를 허용합니다.

CoreBluetooth 가 가진 단일 스캔 정책 제약을 모듈 레벨에서 해결하기 위해, 쏘카프레임은 가상 스캔 매니저를 도입했습니다. 이 매니저는 앱 내 다양한 스캔 요청을 논리적으로 병합하여 OS 에는 단 하나의 최적화된 요청만 전달합니다. 이러한 동작 파편화 해소를 통해 플랫폼을 넘나든 원활한 소통 기반을 마련하며, 새로운 서비스 요구사항이 들어왔을 때의 시나리오 설계 안정성도 확보합니다.

<img src="/img/2026-01-08-socarframe-bluetooth/github_pr_comment.png" width="80%"/>


## 블루투스 모듈 구조

쏘카프레임의 블루투스 모듈은 표준 블루투스 규격 코드 위에 추상화 계층을 쌓아올립니다.

> “문 열기 명령을 보내줘”

기존 플랫폼 코드는 이 간단한 한마디를 하기 위해 기기를 스캔하고, 찾고, 연결하고, 끊어지면 다시 찾고, 에러를 처리하는 코드를 작성해야 합니다. 만약 기기마다 연결되었을 때 수행해야 할 루틴과 Read/Write 규칙이 다르다면 그에 따른 분기 코드도 작성해야합니다. 쏘카프레임에서는 이러한 코드들에 대해 화면 개발자들이 신경 쓸 필요가 없도록 지원합니다.

쏘카프레임은 블루투스 연결 관리자를 BluetoothHost 로, 블루투스 연결 대상을 BluetoothRemote 로 추상화합니다. (이하 호스트 = BluetoothHost, 리모트 = BluetoothRemote)

호스트는 기본적인 블루투스 매니저 역할뿐 아니라 리모트 목록을 관리하고 리모트의 생명주기 이벤트 콜백을 호출하는 역할을 합니다. 앱에서 다루고 싶은 리모트를 호스트에 등록하면 가상 스캔 매니저를 통한 스캔이 시작됩니다. 스캔에 성공하면 그 리모트의 생성 생명주기 콜백을 호출하며, 연결을 끊을때는 종료 생명주기 콜백을 호출합니다. STS, 일레클, 따릉이 모두 리모트로 추상화될 수 있기 때문에, 추상화된 생명주기 이벤트를 호출하기만 하면 각 기기에 맞는 루틴이 시작됩니다.

<img src="/img/2026-01-08-socarframe-bluetooth/host_remote.png" width="80%"/>

리모트 객체의 추상화는 기기 확장성을 보장하는 핵심이 됩니다. 쏘카에 새로운 블루투스 기기를 도입하더라도, 정의된 리모트 인터페이스를 따르기만 하면 기존 비즈니스 로직의 수정없이 유연하게 시스템을 확장할 수 있습니다.

또한 단순한 기기 수평 확장을 넘어, 다형성을 통한 수직적 확장까지 포용합니다. 예를 들어, 연결 상태 관리가 필수적인 기기와 그렇지 않은 기기를 구분하기 위해 BluetoothRemote 위에 연결 생명주기 제어 기능을 얹은 ConnectfulRemote 레이어를 파생시킬 수 있습니다. 필요하다면 이외의 사용자 정의 신규 레이어를 얼마든지 정의할 수도 있습니다.

리모트를 추상화했다면 호스트가 원하는 리모트를 정확히 찾아내기 위해 기기 스펙을 명시한 명세서가 있어야 합니다. 쏘카프레임은 기기의 정보를 “정적인 정체성”과 “동적인 행동 지침”으로 분리해 관리하도록 하며, 그것을 각각 BluetoothSpec 과 BluetoothHandle 로 정의합니다. 이 두 클래스는 호스트에게 무엇을(Identity) 찾고 그것을 어떻게(Behavior) 다뤄야 하는지를 알려주는 설계도 역할을 수행합니다.

정확한 기기 탐색을 위해서는 호스트에게 “CarID 가 abc123 인 STS 를 찾아” 혹은 “BikeID 가 xyz456 인 일레클을 찾아” 라고 구체적인 정보를 전달해야 합니다. CarID, BikeID, UUID 등은 기기의 정적인 정체성 정보이며, 쏘카프레임은 이러한 것들을 묶어 BluetoothSpec 으로 정의합니다.

반면, 실제 통신 과정에서 비즈니스 요구사항에 따라 가변적인 행동들이 존재합니다. STS 에서는 연결 유지를 위한 주기적 aliveMessage 전송 로직과, 보안 강화를 위한 커맨드 재발급(issueCommands) 로직 등이 그 예가 될 수 있습니다. 이러한 동적인 정보를 BluetoothHandle 로 분리하여 관리합니다.

결과적으로 이 두 관심사를 분리함으로써, 쏘카앱은 차량과의 물리적인 연결을 끊지 않고도 동적인 비즈니스 로직을 실시간으로 교체할 수 있는 유연성을 확보합니다. BluetoothSpec 이 같으면 같은 기기로 간주할 수 있기 때문입니다.

이러한 쏘카프레임의 구조 안에서, “문 열기 명령을 보내줘”라는 요구사항은 더 이상 복잡한 절차의 나열이 아닌 명확한 의도의 선언으로 바뀌게 됩니다. 다음은 쏘카프레임을 사용하지 않았을 때 앱 단에서 작성하게 되는 의사 코드(Pseudo-code) 예시입니다. 예시를 들기 위해 간략화된 의사 코드로, 실제 서비스와 디테일은 상이할 수 있습니다.

```
// 플랫폼 코드만 사용
class BluetoothManager {

    // 1. 스캔 시작
    function startScan(device) {
        scan(device.spec)
    }
    
    // 2. 연결 성립 콜백: 기기마다 다른 초기화 루틴
    function didConnect(device) {
        switch (device.type) {
            case STS:        startAliveMessage(); break;
            case ELECLE:     performHandshake();  break;
            case SEOUL_BIKE: checkFirmware();     break;
            // KICKBOARD 추가하려면 여기 코드 수정 필요
        }
    }

    // 3. 문 열기 명령 요청 시: 기기마다 다른 프로토콜
    function requestUnlock() {
        switch (device.type) {
            case STS:        writeBytes([0x01, 0xA0]); break;
            case ELECLE:     writeJSON("{cmd:unlock}"); break;
            // KICKBOARD 추가하려면 여기 코드 수정 필요
        }
    }

    // 4. 응답 수신 콜백: 기기마다 다른 파싱 규칙
    function didReceive(data) {
        switch (device.type) {
            case STS:        if (data[0] == 0x01) success(); break;
            case ELECLE:     if (json(data).ok)   success(); break;
            case SEOUL_BIKE: if (crcCheck(data))  success(); break;
            // KICKBOARD 추가하려면 여기 코드 수정 필요
        }
    }
    
    // 5. 에러 콜백: 기기마다 다른 재시도 정책
    function handleError(error) {
         switch (device.type) {
            case STS:        retry(3); break; // 3번 재시도
            case ELECLE:     disconnectImmediately(); break; // 즉시 끊기
            // KICKBOARD 추가하려면 여기 코드 수정 필요
         }
    }
}
```

앱 단에서 스캔, 연결, 명령 송수신의 모든 처리를 감당해야 합니다. 기기의 정체성이 덩어리로 인식되지 않고, 플랫폼이 분산시킨 콜백들에 파편화되어 있습니다. 새로운 기기 확장에 유리한 구조도 아닙니다. 모듈로 분리하지 않았기 때문에 쏘카 유니버스의 다른 앱에서 똑같이 STS 를 사용하고 싶어졌을 때 이 코드를 다른 프로젝트에 중복 작성하여 관리 포인트가 늘어나게 됩니다.

반대로, 다음은 쏘카프레임을 사용했을 때 앱 단에서 작성하게 되는 의사 코드 예시입니다.

```
// 1. 모듈에 정의된 BluetoothHandle 사용
// 핸들과 스펙은 포함관계
let handleSts = BluetoothHandleSts(specSts, startAliveMessage())
let handleElecle = BluetoothHandleElecle(specElecle, performHandshake())
let handleSeoulBike = BluetoothHandleSeoulBike(specSeoulBike, checkFirmware())
// KICKBOARD 추가하려면 모듈에 추가하고 이곳에 선언

// 2. 호스트에 등록 및 스캔 시작
function register() {
    // 호스트에 등록하면 곧바로 가상 스캐너 시작
    // 호스트가 리모트의 생명주기 콜백(onCreate/onConnect/onFishish)을 알아서 수행
    bluetoothHost.addHandle(handleSts)
}

// 3. 문 열기 명령 요청
function requestUnlock() {
    bluetoothHost
        .checkIsConnected(handleSts) // 연결 상태 체크
        .getBluetoothRemote() // 리모트 획득
        .request(commandUnlock) // 리모트에게 명령
        .catchError() // 리모트가 밖으로 뱉어준 에러 처리
}
```
기기 종류에 따른 모든 처리가 BluetoothRemote, BluetoothHandle 안에 캡슐화되어 있으며, 앱 단에서는 이것을 신경 쓸 필요가 없습니다. 기기의 정체성이 하나의 클래스 내에 응집되어 인식되며, 플랫폼이 분산시킨 콜백에 파편화되지 않습니다. 새로운 기기 확장성이 확보됩니다. 각 종류의 BluetoothHandle 과 BluetoothRemote 는 모듈 단에 선언되어 있기 때문에 쏘카 유니버스의 다른 앱에서 재활용 가능합니다.



## 블루투스 모듈의 가치

이렇게 쏘카는 기존의 CoreBluetooth 및 Android Bluetooth API 위에 새로운 추상화 계층인 쏘카프레임을 구축했습니다. 이 아키텍처의 도입을 통해 확보한 실효적 가치는 다음 5가지로 요약할 수 있습니다.

**1\. 수평, 수직 확장성 확보**  
BluetoothRemote 의 다형성 덕분에, 새로운 블루투스 기기가 도입되더라도 기존 BluetoothHost 코드를 수정할 필요없이 새로운 구현체만 추가하는 수평적 확장성을 제공합니다. 또한, 단순한 통신 기능을 넘어, 연결 유지나 보안 기능이 필요한 경우, 상속을 통해 새로운 계층을 쌓아 올려 고도화할 수 있습니다. 이는 서비스가 성장하며 다양한 상황을 마주하는 시점에도 아키텍처가 무너지지 않고 견고하게 버틸 수 있는 기반이 됩니다.

**2\. 인프라와 비즈니스 영역 분리**  
앱 비즈니스 계층이 하드웨어 복잡성으로부터 분리됩니다. 앱 개발자는 이제 스캔, 연결 수립, 세션 유지, 기기 고유의 규칙들에 대해 신경 쓸 필요가 없어졌습니다. 이 모든 복잡한 절차들은 모두 모듈 내부로 은닉되었습니다. 수십 줄에 달하던 블루투스 하드웨어에 대한 중복 로직이 사라지고, 기기의 종류에 따른 분기 코드를 흩어져 있는 콜백에 정의할 필요가 없어졌습니다. 개발자는 오직 어떤 기기에 어떤 명령을 내릴 것인가라는 비즈니스 의도만 작성하게 됩니다. 

**3\. 비즈니스 유연성 확보**  
iOS 와 Android 플랫폼 간의 아키텍처를 통일함으로써, 비즈니스 요구사항에 좀 더 유연하고 유리하게 대응할 수 있습니다. 두 플랫폼이 동일한 구조와 로직을 공유하므로, 의사 결정 속도를 높이며 함께 구조를 설계할 수 있습니다. 또한, 신규 기능 개발 시 한쪽 OS 의 구현 난이도 종속성으로부터 벗어나게 되고 일정 산정에 유리해집니다. 

**4\. 테스트 용이성**  
플랫폼 제공 블루투스 코드는 하드웨어와 강하게 결합되어 있기 때문에 테스트 코드를 작성하기 어렵습니다. Mocking 이 어려우며 시뮬레이터에서는 테스트를 할 수 없습니다. 쏘카프레임은 새로운 추상화 계층을 쌓았기 때문에 하드웨어 의존성을 끊어내고 Mock 객체를 구현하기 용이해집니다. 물리적으로 재현하기 힘든 예외 상황(연결 끊김, 패킷 유실 등)을 코드로 검증할 수 있게 되어 시스템 안정성을 확보할 수 있게 됩니다.

**5\. 오픈 소스로의 가능성**  
블루투스 모듈은 순수 플랫폼 로직(BluetoothCore)과 쏘카 전용 비즈니스 로직(BluetoothCommon)으로 두 개의 계층으로 분리되어있습니다. BluetoothHost, BluetoothRemote, BluetoothHandle, BluetoothSpec 등은 BluetoothCore 계층에 존재하며, 이것을 채택한 BluetoothRemoteSts, BluetoothHandleSts 등은 BluetoothCommon 계층에 존재합니다. 이러한 모듈화는 향후 BluetoothCore 를 오픈소스로 공개하여 기술 커뮤니티에 기여할 수 있는 가능성을 열어둡니다. 이는 쏘카의 기술이 사내 환경에 머물지 않고, 외부 개발자들과의 상호작용을 통해 더 견고하게 진화하며 선한 영향력을 퍼뜨릴 수 있는 생태계를 마련하는 데 의의가 있습니다.



## Reference

- [android.bluetooth](https://developer.android.com/develop/connectivity/bluetooth?hl=ko&_gl=1*1ob5auz*_up*MQ..*_ga*Njk1OTYyMjE5LjE3Njc5NDIyNTY.*_ga_6HH9YJMN9M*czE3Njc5NDIyNTYkbzEkZzAkdDE3Njc5NDIyNTYkajYwJGwwJGgxODI3MTM0NjA1)

- [CoreBluetooth](https://developer.apple.com/documentation/corebluetooth)

