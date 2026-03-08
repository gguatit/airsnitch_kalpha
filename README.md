# <div align="center">AirSnitch: Wi-Fi 클라이언트 격리(Client Isolation) 테스트</div>

<img align="right" src="airsnitch.png" width="200px" title="AirSnitch logo created by ChatGPT">

이 저장소에는 게스트 사용자가 Wi-Fi 클라이언트 격리를 우회할 수 있게 해주는 도구 및 공격 세트인 AirSnitch가 포함되어 있습니다. 이 도구를 사용하면 가정 및 기업용 Wi-Fi 네트워크에서 클라이언트 격리의 보안성을 검증하고 이를 우회할 수 있는지 테스트할 수 있습니다. 클라이언트 격리(종종 AP(Access Point) 격리라고도 함)는 Wi-Fi에서 표준화된 기능이 아닙니다. 예를 들어, 클라이언트 격리는 일반적으로 전통적인 ARP 기반 MitM(중간자) 공격을 방지합니다. 그러나 우리의 [NDSS'26 논문](https://papers.mathyvanhoef.com/ndss2026-airsnitch.pdf)에 따르면 클라이언트 격리는 종종 일관성 없고 불안전한 방식으로 구현됩니다. AirSnitch를 사용하면 네트워크의 클라이언트 격리가 예상대로 올바르게 구현되고 구성되었는지 테스트할 수 있습니다.

이 연구의 초기 공개 당시, 일각에서는 AirSnitch가 Wi-Fi 암호화를 '깨뜨릴' 수 있다고 말했습니다. 이 공격으로는 키나 암호를 알아낼 수 없기 때문에 이는 제가 보기엔 잘못된 설명입니다. '우회한다(Bypasses)'라는 용어는 좀 더 낫지만 완벽하지는 않습니다. 대부분의 공격은 현재 사용 중인 Wi-Fi 암호화 방식과는 무관하다는 것이 제 생각입니다. 즉, 단지 WPA3로 전환하거나 관리 프레임 보호(Management Frame Protection)를 활성화하는 것만으로는 이 공격을 막을 수 정없습니다. 따라서 AirSnitch 도구를 사용하여 네트워크의 클라이언트 격리 구성을 직접 테스트해 볼 것을 권장합니다.

<sub>참고: NDSS'26 논문에서 사용된 AirSnitch는 [GitHub](https://github.com/zhouxinan/airsnitch) 및 [Zenodo](https://doi.org/10.5281/zenodo.17905485)에서 찾을 수 있습니다. 이번 버전에는 다양한 업데이트가 포함되어 있습니다.</sub>

<a id="id-toc"></a>
## [목차](#id-toc)

* [1. 서론](#id-intro)
* [2. 사전 준비 (Prerequisites)](#id-prereq)
* [3. 매번 사용하기 전 (Before every usage)](#id-everyuse)
* [4. 주요 취약점 테스트 (Main Vulnerability Tests)](#id-mainflaws)
* [5. 추가 취약점 테스트 (Extra Vulnerability Tests)](#id-extraflaws)
* [6. 방어 및 완화 기법 (Defenses and mitigations)](#id-defenses)
* [7. 문제 해결 (Troubleshooting)](#id-troubleshooting)
* [8. 추가 설명 (Clarifications)](#id-clarifications)


<a id="id-intro"></a>
## [1. 서론](#id-intro)

AirSnitch는 클라이언트 격리를 우회하는 세 가지 주요 공격 범주를 테스트할 수 있습니다. 이러한 공격은 Wi-Fi 암호화를 우회하므로, 단순히 WPA1/2/3을 사용하는 것만으로는 공격을 막을 수 없습니다. [NDSS'26 AirSnitch 논문](https://papers.mathyvanhoef.com/ndss2026-airsnitch.pdf)에서 이러한 공격을 다루고 있으며, 전체적인 개요는 다음과 같습니다.

1. **GTK 그룹 키 악용 (Abusing GTK group keys)**: 공격자는 동일한 Wi-Fi 네트워크에 있는 모든 클라이언트 간에 공유되는 그룹 키를 악용할 수 있습니다. 특히 특정 피해자(들)에게 직접 패킷을 주입(inject)하는 데 GTK를 사용할 수 있습니다. 또한 우리가 테스트한 모든 운영 체제는 브로드캐스트 Wi-Fi 패킷 내부에 있는 유니캐스트 IP 패킷을 수용합니다. 즉, 공격자는 다음과 같은 형태의 Wi-Fi 프레임을 주입하여 임의의 패킷을 보낼 수 있습니다( [Scapy](https://scapy.readthedocs.io/en/latest/api/scapy.layers.dot11.html) 표기법 기준).

	```
	Dot11(dst=ff:ff:ff:ff:ff:ff, src=access point) / IP(dst=victim, src=adversary)
	```

	이 패킷은 악의적인 내부자를 포함하여 모든 클라이언트가 알고 있는 공유 그룹 키를 사용하여 암호화됩니다. 이후 지정된 대상 IP 주소를 가진 클라이언트만 주입된 패킷을 처리하므로 타겟형 공격이 계속 유지될 수 있습니다. 이는 [WPA Too: Hole 196](https://defcon.org/html/links/dc-archives/dc-18-archive.html#Ahmad) 공격과 유사하지만, 이전에 연구되지 않았던 클라이언트 격리 우회 상황에 적용된 것입니다.


2. **게이트웨이 바운싱 (Gateway Bouncing)**: 많은 네트워크가 클라이언트 격리를 MAC/이더넷 계층에서만 강제합니다. 이를 통해 공격자는 IP 계층에서 피해자에게 패킷을 전달하도록 게이트웨이를 속여 클라이언트 격리를 우회할 수 있습니다. 즉 게이트웨이에서 패킷을 '바운싱(반사)' 시킵니다. 구체적으로 공격자는 다음과 같은 유형의 패킷을 전송할 수 있습니다.

	```
	Ethernet(src=attacker, dst=gateway) / IP(src=attacker, dst=victim)
	```

	이 패킷은 지정된 대상이 다른 클라이언트가 아닌 게이트웨이이기 때문에 MAC/이더넷 계층에서 클라이언트 격리로 인해 차단되지 않습니다. 게이트웨이는 수신 후 IP 계층 기준에 따라 정상적으로 피해자에게 이 패킷을 라우팅해 줍니다.


3. **포트 스틸링 (Port Stealing - BSSID 간 도용)**: 공격자는 피해자의 업링크(송신) *및* 다운링크(수신) 트래픽을 자신에게 전달하도록 내부 스위치 및 브릿지를 조작할 수 있습니다. 피해자의 *업링크* 트래픽을 가로채는 방식은 다음 그림에 설명되어 있습니다.

	<div align="center"><img src="steal-uplink.png" width="500px"></div>

	피해자는 액세스 포인트 AP2에 연결되어 있고, 공격자는 내부 게이트웨이의 MAC 주소를 위장하면서 다른 액세스 포인트 AP1에 접속합니다(1단계). 결과적으로 피해자가 진짜 게이트웨이로 업링크 트래픽을 보내려 할 때 해당 트래픽은 라우팅되어 공격자에게 전송됩니다(2단계). 빨간색 선은 라우팅 테이블을 조작하기 위한 스푸핑된 위장 트래픽을 의미하고, 파란색 선은 가로챈 업링크 트래픽을 의미합니다.
	
	*다운링크* 트래픽을 가로채기 위해서도 유사한 공격이 가능합니다. 공격자가 피해자의 MAC 주소를 사용하여 네트워크가 인터넷에서 들어오는 다운링크 통신을 공격자에게 전송하도록 게이트웨이를 속이는 방식입니다( [이 그림 참조](steal-downlink.png) ).


<a id="id-impact"></a>
### [1.1 실질적 영향 (Practical Impact)](#id-impact)

위의 기술들을 결합하면 공격자는 **클라이언트 격리가 활성화되어 있더라도 완벽한 중간자(MitM) 기능을 복원**할 수 있습니다. [NDSS'26 논문](https://papers.mathyvanhoef.com/ndss2026-airsnitch.pdf)에서 우리는 대부분의 가정용 라우터가 취약하며 기업용 디바이스의 제한적인 보안 보장 및 기업 환경 내의 실제 취약점들을 발견했습니다. 주요 내용은 다음과 같습니다.

- 메인 네트워크 외에도 게스트 네트워크를 제공하는 가정용 라우터의 경우 주요 네트워크와 게스트 네트워크 사이의 격리가 무력화됩니다. 즉, 게스트 네트워크 사용자가 메인 네트워크의 기기를 지속적으로 공격할 수 있습니다.

- 패킷을 일방적으로 피해자에게 꽂아 넣기(inject)만 할 수 있어도 파급력이 큰 공격이 가능합니다. 예를 들어 조작된 ICMPv6 Router Advertisements 패킷을 피해자에게 넣어 피해자가 악성 DNS 서버를 사용하게 만들고 이로써 이후의 모든 IP 기반 트래픽을 중간자가 모두 가로챌 수 있습니다. 자세한 내용은 [FragAttacks](https://papers.mathyvanhoef.com/fragattacks-overview.pdf)를 참조하세요.

- 이 기법을 사용하면 내부망에 존재하는 *내부* 기기들과의 트래픽을 스니핑/인젝션 할 수도 있습니다. 무엇보다 액세스 포인트가 생성하는 RADIUS 패킷을 가로채는 위험도가 큽니다. 이는 자격 증명을 훔쳐내 로그를 취득한 뒤 내부 시스템용 악성 네트워크를 운영하는 위험으로 이어질 수 있습니다.

- 이 공격은 서로 다른 BSSID 및 액세스 포인트 간에서도 효과적입니다. 일례로 한 대학교 네트워크에서는 암호화된 WPA2/3에 있는 타겟 피해자의 트래픽을 포트 스틸링 기술로 빼앗은 뒤 패스워드가 없는 무방비의 Open 라우터에 풀어버려 근방의 모든 사람들이 패킷을 볼 수 있도록 만들 수 있었습니다.

- 정상적인 트래픽 흐름을 끊지 않고 완벽한 중간자 공격(MitM)을 매끄럽게 달성하려면 논문에 설명된 추가 기법(예: _Server-Triggered Port Restoration_ 또는 _Inter-NIC Relaying_)이 필요합니다.


<a id="id-compare-macstealer"></a>
### [1.2 MacStealer 와의 비교](#id-compare-macstealer)

우리의 이전 [USENIX Security '23 framing frames 논문](https://www.usenix.org/conference/usenixsecurity23/presentation/schepers)과 동반 도구인 [MacStealer tool](https://github.com/vanhoefm/macstealer) 에서도 클라이언트 격리 우회를 소개한 바 있습니다. AirSnitch는 이 연구를 더욱 확장한 결과입니다. 요약하자면 MacStealer는 피해자와 공격자가 *동일한 BSSID*에 접속해 있는 특정 상황에서의 `--c2c-port-steal` 테스트에 해당합니다. 즉, 원본 MacStealer는 오직 '하나의 BSSID 내에서의 다운링크 트래픽 탈취'만 다루었지만, 그 외의 **본 도구에 있는 다른 모든 우회 기법들은 완전히 새롭고 새롭게 제시되는 것들입니다**.

원래의 'MacStealer' 결함 패치 작업은 생각보다 [까다롭고 복잡합니다](https://github.com/vanhoefm/macstealer/blob/main/README.md#id-mitigations). 이상적인 해결을 위해서는 액세스 포인트와 클라이언트 *모두*가 (초안) IEEE 802.11 규격에 맞춰 ["Reassociating STA recognition" 확장 설정](https://mentor.ieee.org/802.11/dcn/23/11-23-0537-07-000m-reassociating-sta-recognition.docx)을 지원해야 하기 때문입니다. 이로 인해 이 이슈는 제조사들 사이에서 여전히 골치 아픈 문제로 남아 있습니다. 이와 반대로, 이번 AirSnitch에서 밝혀낸 새로운 취약점들은 대부분 구현 및 구성상의 결함으로 간주되므로 실제 패치 및 해결을 진행하기가 훨씬 수월합니다.


<a id="id-prereq"></a>
## [2. 사전 준비 (Prerequisites)](#id-prereq)

스크립트는 **Ubuntu 22.04.5 LTS** 에서 정상적으로 작동함을 테스트했습니다. 따라서 [Ubuntu 22.04](https://releases.ubuntu.com/jammy/)를 사용하는 것을 권장합니다. USB Wi-Fi 동글이 있다면 [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 내부에서도 사용할 수 있습니다.

리포지토리를 초기화하고 시스템에 필요한 실행 파일을 컴파일하기 위해 다음 과정을 처음 한 번만 수행하면 됩니다. 먼저 다음 의존성(Dependencies)들을 설치합니다.

	sudo apt update
	sudo apt install libnl-3-dev libnl-genl-3-dev libnl-route-3-dev \
		libssl-dev libdbus-1-dev pkg-config build-essential net-tools python3-venv \
		aircrack-ng rfkill git dnsmasq tcpreplay macchanger

그 후 이 리포지토리를 클론하고, 루트 디렉토리에서 다음 스크립트를 실행하여 수정된 hostap 릴리스 배포판을 컴파일합니다.

	./setup.sh
	cd airsnitch/research
	./build.sh
	./pysetup.sh


<a id="id-everyuse"></a>
## [3. 매번 사용하기 전 (Before every usage)](#id-everyuse)

<a id="id-everyuse-env"></a>
### [3.1. 실행 환경 (Execution environment)](#id-everyuse-env)

매 사용 전 항상 python 환경을 관리자 권한(root)으로 로드해야 합니다.

	cd airsnitch/research
	sudo su
	source venv/bin/activate

그리고 무선 연결 관리자(네트워크 매니저)가 AirSnitch의 무선 인터페이스 작업을 방해하지 않도록 [Wi-Fi를 잠시 비활성화](https://github.com/vanhoefm/libwifi/blob/master/docs/linux_tutorial.md#id-disable-wifi)해야 합니다. 추가적으로 `sudo airmon-ng check` 명령어로 네트워크 카드를 붙잡고 있어 AirSnitch와 충돌할 수 있는 다른 프로세스가 없는지 확인하는 것도 좋습니다.

<a id="id-everyuse-net"></a>
### [3.2. 네트워크 설정 (Network configuration)](#id-everyuse-net)

그 다음 단계로 테스트할 네트워크의 정보가 담긴 [`client.conf`](research/client.conf) 파일을 편집해야 합니다. 이 파일은 [`wpa_supplicant`](https://wikiarchlinux.org/title/wpa_supplicant#Connecting_with_wpa_passphrase) 통신 설정을 관리하며 주로 피해자와 공격자를 나타내는 2개의 네트워크 블록을 포함합니다. 예시로 메인 네트워크(`main-network`)와 게스트 네트워크(`guest-network`)의 WPA2/3 통신을 검사하기 위한 설정 예제는 다음과 같습니다.

	# 이 줄은 변경하지 마세요, 그렇지 않으면 AirSnitch가 작동하지 않습니다
	ctrl_interface=wpaspy_ctrl

	network={
		# 이 필드를 변경하지 마세요, 스크립트가 의존합니다
		id_str="victim"

		# 테스트할 네트워크: 시뮬레이션된 피해자가 연결할 네트워크/SSID
		ssid="main-network"
		key_mgmt=WPA-PSK
		psk="main-password"
	}

	network={
		# 이 필드를 변경하지 마세요, 스크립트가 의존합니다
		id_str="attacker"

		# 테스트할 네트워크: 시뮬레이션된 공격자가 연결할 네트워크/SSID
		ssid="guest-network"
		key_mgmt=WPA-PSK
		psk="guest-password"
	}

망을 구성할 때마다 테스트할 네트워크의 실제 이름과 암호화 구성, 그리고 자격 증명을 넣어서 파일을 작성해야 합니다. `wpa_supplicant.conf` 구성 파일을 작성/편집하는 방법과 각종 Wi-Fi 설정에 대해서는 [wpa_supplicant.conf](wpa_supplicant/wpa_supplicant.conf)의 가이드를 참고하세요. 첫 번째 블록(`id_str="victim"`)에는 피해자의 네트워크, 두 번째 블록에는 테스트할 공격자의 네트워크 정보가 들어갑니다.

위 예시에서 AirSnitch는 게스트 네트워크의 공격자가 메인 네트워크 내의 피해자를 침해할 수 있는지 검사할 것입니다.

스크립트는 기본적으로 `client.conf` 안의 값을 불러옵니다. 다른 구성 파일을 사용하려면 파라미터로 `--config network.conf` 처럼 설정하면 됩니다. 다음과 같은 주요 구성 파일 예제가 제공됩니다.
- [`eap.conf`](airsnitch/research/eap.conf): PEAP-MSCHAPv2 기반 인증, 고유한 사용자 지정 ID/비밀번호 설정을 지닌 기업(Enterprise) 네트워크용.
- [`multipsk.conf`](airsnitch/research/multipsk.conf): 신뢰하는 장치와 손님용 장치에 다른 비밀번호를 사용하는 multi-PSK 망.
- [`saepk.conf`](airsnitch/research/saepk.conf): SAE-PK를 사용하는 퍼블릭 공용 핫스팟용.

<a id="id-everyuse-bssid"></a>
### [3.3. BSSID 지정 (BSSID selection)](#id-everyuse-bssid)

AirSnitch가 정확히 올바른 망에 접속하고 있는지 검증하기 위해 다음과 같은 규칙에 따라 실행 인자를 주어야 합니다.

- `--same-bss` 또는 `--other-bss`: 시뮬레이터 피해자와 공격자가 동일한 AP/BSSID에 접속할지(`--same-bss`), 서로 다른 AP/BSSID에 접속할지(`--other-bss`) 명시해야 합니다. 둘 중 하나는 사용해야 합니다. [명시적으로 BSSID를 기재](#id-test-bss)하여 설정에 지정해줄 수도 있습니다.
- `--no-ssid-check`: 피해자와 공격자가 접속하는 SSID 대상 자체가 다를 경우 포함해야 합니다. (위의 예시와 같은 메인/게스트 분리망 등). 이 인수가 없으면 스크립트는 설정에 적힌 타겟 SSID가 동일하지 않을 경우 실행 오류를 발생시키며 종료됩니다.

<a id="id-mainflaws"></a>
## [4. 주요 취약점 테스트 (Main Vulnerability Tests)](#id-mainflaws)

<a id="id-mainflaws-gtk"></a>
### [4.1. GTK 악용 여부 테스트 (GTK Abuse)](#id-mainflaws-gtk)

다음 명령어를 실행하여 공격자와 피해자가 동일한 GTK(브로드캐스트 암호화 그룹 키)를 공유 받아 사용하고 있는지 점검합니다.

	./airsnitch.py wlan2 --check-gtk-shared wlan3 --no-ssid-check [--other-bss,--same-bss]

AirSnitch는 주어지는 두 개의 네트워크 장비 카드(NIC)를 사용해 통신망에 붙고 각자 할당받은 GTK 그룹 키를 받아옵니다. 이때 인자 `--other-bss` 및 `--same-bss`를 사용하여 키 공유가 통일된 AP 사이트에 걸쳐 일어나는지 또는 서로 다른 AP에서도 일부분 공통적으로 키가 유출/고정되어 있는지 진단할 수 있습니다.

스크립트 결과값:
	>>> The victim's GTK is (XXX)."
	>>> The attacker's GTK is (YYY)."

만약 `XXX` 키와 `YYY` 키가 동일하다면, 사용 중인 망이 위협에 취약한 것입니다. 공격자는 이를 탈취해 피해자를 직접 노릴 수 있습니다.

<a id="id-mainflaws-bounce"></a>
### [4.2. 게이트웨이 바운싱 여부 테스트 (Gateway Bouncing)](#id-mainflaws-bounce)

다음 명령을 실행하여 공격자와 피해자 간의 IP 통신 단에서의 망 분리가 정상적으로 방어되고 있는지 확인합니다.

	./airsnitch.py wlan2 --c2c-ip wlan3 --no-ssid-check [--other-bss,--same-bss]

AirSnitch는 네트워크에 접속한 뒤 앞서 [서론](#id-intro)에서 언급된 게이트웨이 유도형(반송형) L3 트래픽을 주입합니다. 다음의 결과와 같이 실패 메시지가 빨간색으로 출력된다면 침해 공격이 성립한 것으로, 네트워크가 이 결함에 뚫려 있음을 나타냅니다.

	>>> Client to client traffic at IP layer is allowed (PSK{pw_atkr} to SAE{pw_victim})

(참고: 괄호 안의 문자열은 설정한 암호 유형과 정보에 따라 다를 수 있습니다.)

<a id="id-mainflaws-port-down"></a>
### [4.3. 다운링크 포트 스틸링 (Downlink Port Stealing)](#id-mainflaws-port-down)

다음 명령을 실행하여 공격자와 피해자 간의 포트 도용 시뮬레이션(공격자가 외부에서 들어오는 피해자의 트래픽을 가로채려 함)을 진행합니다.

	./airsnitch.py wlan2 --c2c-port-steal wlan3 --no-ssid-check --other-bss --server 8.8.8.8

여기서 `--server` 매개변수는 ping 요청에 응답할 수 있는 서버의 IP 주소여야 합니다. 이 스크립트는 네트워크 장비 두 개를 통해 핑(Ping) 테스트를 수행합니다. 스크립트의 작동 방식:

1. 피해자는 서버로 연속적인 ping 요청 스트림을 보냅니다. 서버는 피해자에게 응답을 되돌려 보낼 것입니다. 공격자는 이 ping 응답을 가로채려 할 것입니다.
2. 공격자는 피해자의 MAC 주소로 스푸핑하여 네트워크에 연결하고, 위조된 피해자의 MAC 주소를 담은 (가짜) IPv4 패킷을 발송합니다. 이 공격이 성공하면, 네트워크 장비는 해당 ping 응답들을 공격자에게 전달하기 시작합니다.

다음의 결과와 같이 빨간색 텍스트가 표시되면 공격이 성공한 것입니다.

	>>> Downlink port stealing is successful.

이 공격에 있어서 `--same-bss` 매개변수는 보통 의미가 없습니다. 이 매개변수를 적용하면 공격자와 피해자가 동일한 MAC 주소로 동일한 BSSID 접속을 시도하기 때문입니다. 이렇게 되면 둘이 계속해서 서로를 튕겨내며 서로 간섭하게 됩니다. 따라서 이 테스트에는 `--other-bss` 인자만 사용하는 것이 합리적입니다.


<a id="id-mainflaws-port-up"></a>
### [4.4. 업링크 포트 스틸링 (Uplink port stealing)](#id-mainflaws-port-up)

다음 명령을 실행하여 공격자와 피해자 간의 포트 도용 시뮬레이션(피해자가 전송하는 트래픽을 가로채려 함)을 진행합니다.

	./airsnitch.py wlan2 --c2c-port-steal-uplink wlan3 --no-ssid-check [--other-bss,--same-bss] [--server 1.2.3.4]

여기에서는 어떠한 `--server` IP 값을 사용하는지는 중요하지 않습니다. 마찬가지로 두 개의 네트워크 카드를 사용해 테스트합니다.

1. 피해자는 서버를 향해 연속적인 ping 요청 스트림을 보냅니다 (서버가 꼭 응답하지 않아도 무방). 공격자는 이 전송되는 ping 요청들을 스니핑하려 시도할 것입니다.
2. 공격자는 이번에는 내부 게이트웨이의 MAC 주소로 스푸핑하여 연결한 뒤, 위조된 게이트웨이의 MAC 주소를 지닌 (가짜) IPv4 패킷을 삽입 및 전송합니다. 성공하면 네트워크는 피해자의 ping 트래픽 전송 방향을 공격자 쪽으로 돌립니다.

이 다음의 빨간색 텍스트가 표시된다면 역시 성공한 것입니다.

	>>> Uplink port stealing is successful.

다운링크 포트 스틸링과 다르게 이 경우엔 `--other-bss` 와 `--same-bss` 모두 유효합니다. 실제 조사 결과, 서로 다른 BSSID에 접속할 땐 실패했던 공격이, 동일한 BSSID에 접속했더니 성공하거나 반대의 결과를 보인 네트워크 사례들도 발견되었습니다.


<a id="id-extraflaws"></a>
## [5. 추가 취약점 테스트 (Extra Vulnerability Tests)](#id-extraflaws)

<a id="id-bcast-reflect"></a>
### [5.1. 브로드캐스트 리플렉션 (Broadcast Reflection)](#id-bcast-reflect)

이 테스트는 GTK 악용(`--check-gtk-shared`)과 밀접한 연관이 있으며, 다음과 같이 실행할 수 있습니다.

	./airsnitch.py wlan2 --c2c-broadcast --no-ssid-check [--other-bss,--same-bss]

AirSnitch는 네트워크 카드 두 개를 통해 공격을 테스트하며, 공격자는 유니캐스트 IP 목적지를 둔 채 브로드캐스트 형태로 Wi-Fi 프레임을 전송합니다. 구체적으로 공격자는 다음과 같은 (암호화될 수 있는) 형식의 프레임을 구성하여 보냅니다.

	Dot11(FCfield="to-DS", addr1=adversary, addr2=AP, addr3=ff:ff:ff:ff:ff:ff)
		/ IP(dst=victim, src=adversary)

만약 격리가 활성화되어 있음에도 AP가 이 프레임을 튕겨 피해자에게 포워딩한다면 네트워크는 취약한 것이며, 다음 항목이 출력됩니다.

	>>> Broadcast Reflection is allowed (PSK{pw_atkr} to SAE{pw_victim})

괄호 안의 문자열은 설정한 자격 증명에 따라 내용이 다를 수 있습니다.


<a id="id-test-bss"></a>
### [5.2. 특정 액세스 포인트 / BSS(공유기 특정) 테스트](#id-test-bss)

기본적으로 AirSnitch는 사용할 AP/BSS를 자동으로 선택합니다. 그러나 한 네트워크 내에 여러 AP/BSS가 있는 경우, 피해자의 네트워크 설정 블록 내에서 `bssid` 키워드를 사용해 원하는 AP/BSS를 명시적으로 지목할 수 있습니다.

	...

	network={
		# 이 필드를 변경하지 마세요, 스크립트가 의존합니다
		id_str="victim"

		# 테스트할 네트워크: 피해자가 접속할 네트워크 이름 / SSID
		ssid="main-network"
		key_mgmt=WPA-PSK
		psk="main-password"

		# 접속할 명시적 AP/BSS
		bssid=00:11:22:33:44:55
	}

	...

위 구성에 따라 AirSnitch 가상 피해자는 `00:11:22:33:44:55` 장비에만 연결됩니다. 공격자의 네트워크 블록에도 `bssid` 값이 함께 선언되거나, 인자로 `--same-bss` 가 붙지 않는 한 시뮬레이터 공격자는 어떤 무작위 AP/BSS로든 접속할 것입니다.
- 만약 위 예시가 `--other-bss` 인자와 함께 실행된다면, 피해자는 `00:11:22:33:44:55` 에 연결되겠지만 공격자는 항상 *다른* 특정 BSSID로 우회하여 붙습니다.
- 양쪽 네트워크 블록 모두에 BSS/AP를 둘 다 명시해 버리는 방법도 가능합니다.


<a id="id-manual"></a>
### [5.3. 수동 점검 요소 (Manual tests)](#id-manual)

이 스크립트는 모든 것들을 커버하지는 못하며 단지 현재 개발 중이거나 수동으로 조사되어야 하는 부분들이 존재합니다.

- 클라이언트 격리 상황 내에서 IGTK 그룹 키가 서로 다르게 분배 및 난수 생성되고 있는지 점검하는 것.


<a id="id-defenses"></a>
## [6. 방어 및 완화 기법 (Defenses and mitigations)](#id-defenses)

클라이언트 격리(AP 격리)라는 용어 자체가 공식적으로 규격화된 표준이 아니기 때문에 각 사용자에 따라 보안 기대치가 다르며, 모든 문제를 한 번에 해결할 완벽한 정답은 존재할 수 없습니다. 사용하는 제품에 맞춰 소프트웨어 패치 및 구성 개선을 조합해야만 공격을 예방할 수 있습니다. **클라이언트 격리의 안전하고 강력한 구현 정의와 실무 환경 강제 적용에 대하여는 향후 더 많은 연구와 표준화가 필요한 실정입니다.**

**한편, 그 사이 다음과 같은 완화 방법을 제시 및 권장합니다**: (1) 해당 제품의 기존 클라이언트 격리 보장 범위를 문서화합니다; (2) 그룹 키 난수화; (3) 매일 쏘고 받는 브로드캐스트 Wi-Fi 프레임 속의 유니캐스트 IP 패킷 차단; (4) 내부 VLAN망 및 방화벽 규칙 적용; (5) MAC 스푸핑 방지; (6) IP 주소 스푸핑 방지; (7) 중앙 컨트롤러의 통합 복호화 및 라우팅 도입; (8) 불완전한 설정 시 경고 기능 제공. 이 중 하나만 적용해선 모든 공격을 다 막아낼 순 없겠지만 이들을 종합 적용한다면 클라이언트 보호 능력을 극대화할 수 있을 것입니다.


<a id="id-defenses-docs"></a>
## [6.1. 적절한 문서화 작업 (Proper documentation)](#id-defenses-docs)

메인 네트워크와 게스트 네트워크 분리를 지원하는 등의 가정용 AP 같이 중앙 집중형 설정 기능을 담은 제품은, 보안 보장 범위를 아래와 같이 제품 문서로 정확히 명시해야만 합니다.

1. 게스트 네트워크 안의 기기들이 상호 간 통신이 가능한 상태인가?
2. 게스트와 메인 네트워크 간 트래픽이 허가되어 있는가? 어느 방향으로 오픈되어 있는가?
3. Wi-Fi 단말기들뿐 아니라 유선 포트 장비들끼리, 또는 무선 장비와 유선 장비 간에도 격리되어 있는가?
4. *추가 옵션:* 동일한 사용자, 동일한 EAP 자격, 나아가 동일한 [기기/장비별 개별 패스워드 장비 인증(PPSK)](https://www.supernetworks.org/pages/docs/knowledgebase/ppsk-wpa3-wifi-security)을 사용하는 자들끼리도 망 내에서 완전히 통신 차단이 되어 있는가?


<a id="id-defense-randgtk"></a>
## [6.2. 그룹 키 난수화 (Group key randomization)](#id-defense-randgtk)

서로 격리된 클라이언트에게는 서로 다른 그룹 키가 설정되어야만 합니다. 이를 확실히 하는 방법 중 하나는 **설정 옵션을 추가**하거나 기본값으로 매 기기마다 무작위 암호키를 건네는 것입니다. 이는 [Passpoint 규격 내 권장사항](https://www.wi-fi.org/file/passpoint-specification-package)과 맥락을 같이 합니다. 구체적으로 말해, 4-Way 핸드셰이크, 그룹 키 핸드셰이크, 고속 링크 초기 접속(FILS) 및 빠른 BSS 전환(FT) 핸드셰이크, WNM 절전 전환 프로세스를 포함한 미래 나올 다양한 802.11 연결 인가 단계마다 액세스 포인트가 계속 기기별로 각각 다른 무작위 키를 주어야 한다는 것입니다.

ARP 전송 및 DHCP 요청과 같이 본연의 필수 브로드캐스팅 트래픽 교환을 위해, 네트워크는 차라리 L2 브로드캐스트/멀티캐스트를 유니캐스트로 전환하는 방식을 자체 구현할 수 있습니다. 이를 위해서는 _프록시 ARP 서비스(Proxy ARP service)_ 와 _ICMPv6 라우터 알림 패킷을 일대일로 대변해서 전환(conversion of ICMPv6 Router Advertisement packets)_ 해 쏘는 기능들이 필요할 것입니다. 더 전문적인 관련 기술의 논의는 [Passpoint 규격 설명](https://www.wi-fi.org/file/passpoint-specification-package)에서 열람할 수 있습니다.

참고로 일부 장비는 VLAN별로 각각 다른 값을 나눠 갖게 해 주는 옵션을 본래 제공하기도 합니다 (예: Cambium 솔루션의 `gtk-per-vlan` 옵션). 네트워크 통신 인프라가 무작위 그룹 키 생성 자체는 할 줄 모른다 쳐도 최소한 VLAN마다 별개의 분할된 그룹 키를 쓸 수 있게 설정해 주어야 합니다.


<a id="id-defense-filter-bcast"></a>
## [6.3. 브로드캐스트 프레임 내 유니캐스트 패킷 차단 방어](#id-defense-filter-bcast)

클라이언트의 운영 체제는 설계상으로 Layer 2 멀티캐스트 프레임 안에서 특정 기기를 조준한 유니캐스트 형태의 IP 통신 패킷이 도착할 경우 기본적으로 이를 버리도록 하거나 최소한 차단 설정을 제공해야 할 것입니다. 일례로 리눅스 시스템에선 `drop_unicast_in_l2_multicast=1` sysctl 설정으로 이러한 예외 트래픽 무시(Drop) 처리를 구성할 수 있습니다.


<a id="id-defenses-vlan"></a>
## [6.4. 방화벽 룰과 VLAN 활용 (Usage of VLANs and firewall rules)](#id-defenses-vlan)

게스트 망과 일반 망 분리를 위한 1차 해결 방법은 VLAN을 도입하는 것입니다. 예를 들어 SRP의 모델과 같이 [매 단말기별로 VLAN 격리를 제공](https://www.supernetworks.org/pages/blog/guest-ssid-on-spr)하게 하면, 내부 보안 장치를 올리는 데 큰 도움을 줄 수 있습니다.

하지만 VLAN을 적용해 분리시켜 놓았다 하더라도 [위에 서술된 무작위 그룹 키 사용](#id-defense-randgtk) 지점을 달성해 내었거나 VLAN별로 각자 그룹 키(GTK/IGTK)를 따로 생성하여 뿌리고 있는지 점검하고, 방화벽 사이트 사이에 패킷 통제 룰 세팅이 확실한 지 주의를 기울여야 합니다.

방화벽 규칙을 적용할 땐 꼭 다음 두 가지 방침을 명심하십시오.
- L2(MAC) 이더넷 스위치 계층만이 아닌 L3(IP) 계층에서도 완전히 단절되어 있는지 점검할 것.
- MAC 이더넷 계층상의 브로드캐스팅 트래픽들 사이에도 완벽한 단절이 되는지 점검할 것.


<a id="id-defenses-mac-spoof"></a>
## [6.5. MAC 스푸핑 차단 (MAC spoofing preventation)](#id-defenses-mac-spoof)

네트워크 제품군은 사용자가 게이트웨이, DNS 서버, 사내 필수 서버 등의 기설정된 유선 장비들이 점유한 기존의 MAC 주소를 도용 및 허위 맵핑해 내부 통신에 참여하는 행위를 금지하도록 통제 옵션을 둬야 합니다.
이와 덧붙여 BSSID 사이에서 한 MAC이 여러 위치의 통신 신호로 엮이는 부분 역시 제한해, 도용 기반 횡단 포트 스틸링(다운링크 가로채기) 해킹이 작동하지 않게 만드십시오. 단일 BSSID를 상대로 일어나는 다운링크 포트 공격에 대비해서는 저희의 과거 연구인 [MacStealer 완화 기법 권고](https://github.com/vanhoefm/macstealer#id-mitigations)를 읽고 수행해 볼 것을 권합니다.

무선이 활성화되어 있더라도 클라이언트 내에서 프린터 서버 등을 사용할 수 있게 MAC 화이트리스트 기능(Allowlist)을 넣는 방법 또한 추천합니다(물론 그 서버들이 임의 MAC 변경을 끄고 운영된다는 보장하에).


<a id="id-defenses-ip-spoof"></a>
## [6.6. IP 스푸핑 차단 (IP spoofing preventation)](#id-defenses-ip-spoof)

나아가 저희는 MAC 차단을 넘어 IP 스푸핑 변조 방어 장치 또한 권장합니다. 이는 저희의 각종 통신 맹점 침투법을 양방향 통신 수준의 중간자 모드(Full MitM)로 심화시키는 일을 매우 고단하게 막을 것입니다. 특히나 _게이트웨이 바운싱_ 따위로 트래픽들을 가로채서 다시 외부나 망에 재포워딩하는 틈새 우회를 사단 시킬 수 있습니다.


<a id="id-defense-central"></a>
## [6.7. 중앙 집중식 통합 무선 복호화 및 라우팅 운용](#id-defense-central)

거대 사내용 엔터프라이즈 하드웨어 장비라면 중앙 Wi-Fi 컨트롤러 환경으로 몽땅 트래픽을 넘겨 라우팅과 암호 관리를 총괄해버리는 훌륭한 자체 옵션이 존재할 수도 있습니다. 저희 테스트상 이 모드를 사용하는 곳은 확실히 침범이 쉽지 않았습니다. 개별 방어 시스템 도입이 어렵다면 차라리 모든 무선 복호화 관리를 이 중앙 관리 장치에 다 터널링 위임해버어 공격 경로를 일절 허용하지 마십시오.


<a id="id-defenses-config"></a>
## [6.8. 경고 및 주의 안내 메세지 (Warnings)](#id-defenses-warnings)

끝으로 제품 제조사가 다음과 같은 기능 사용성 향상과 보안 메시지를 추가해 줄 것을 당부합니다.

- 오픈 Wi-Fi나 누구나 아는 공용 WPA 비밀번호 환경망에 클라이언트 격리 보호 기술을 설정해 본들 아무런 보안 효과를 내진 못한다고 경고해 주기.
- 게스트 전용 망을 관리자가 구성할 때, 그냥 인터넷만 따로 놀게 하는 거에 그칠 게 아니라 게스트들 본인들 서로끼리의 망에서도 접속 및 교류 여부를 끄고 단절시키는 옵션을 선택하라고 인터페이스 상 알려주기.


<a id="id-troubleshooting"></a>
# [7. 문제 해결 (Troubleshooting)](#id-troubleshooting)

- AirSnitch가 특정 망에 도무지 달라붙지를 않는다면, 자신의 `client.conf` 설정 파일 자체를 운영체제 기본 빌트인 도구인 `wpa_supplicant` 에 먹여 제대로 이어지는지 여부부터 따져 점검하십시오. 그것도 안 달린다면 client.conf의 문제거나 네트워크의 문제, 혹은 자체 동글 카드의 이슈일 확률이 큽니다. AirSnitch는 기입한 AP/BSS에 연결할 때까지 최대 30초만 대기하다, 연결하지 못하면 스크립트를 죽이고 나옵니다.
- Ubuntu 22.04를 VirtualBox 7 또는 그 상위 버전 안에서 띄울 때 일부 시스템 콘솔 창이나 터미널이 검게 뜨지 않는 증상을 확인하였습니다. 이럴 땐 [이 가이드](https://askubuntu.com/questions/1435918/terminal-not-opening-on-ubuntu-22-04-on-virtual-box-7-0-0)를 밟고 해결해 보세요. 설치할 때 차라리 "Skip Unattended Installation"(무인설치 생략) 체크를 활성화해두셨다면 펀하게 진행되었을 겁니다.
- `--c2c-gtk-inject` 공격 테스트엔 가상의 공격자가 자신이 L2 리눅스 패킷 전송 머신으로 작동하고 해당 유니캐스트를 추적 인젝션 하기 때문에 `drop_unicast_in_l2_multicast` 시스템 변수 값이 반드시 `0`으로 세팅되어 있어야 합니다.


<a id="id-clarifications"></a>
# [8. 추가 설명 (Clarifications)](#id-clarifications)

- 저희는 이 연구 논문을 진행하며 일부 유명 엔터프라이즈 AP 하드웨어들을 올려놓고, 이들이 애초에 무너질 소지가 다분히 존재하는지 취약하게 설계되었음부터 까봤습니다. 하지만 그렇다 하여 '이 기계를 쓴 네크워크는 무조건 지금 뚫린다'는 뜻은 아닙니다. 그 제품이 깔린 환경망 전체가 취약한지 어떤지는 기타 세부적인 부속 구성, 설정 및 인프라 매칭을 종합적으로 결합해 따져 보아야 하기 때문입니다. 즉 이 목록에 기업형 디저이스가 언급되었다 해서 무방비라는 게 아닙니다, 단지 이들은 가장 취약한 옵션상에서 결함의 여지가 짙으니 네트워크 관리자는 꼭 '자신의 망 단절 보안 환경'이 잘 구축되어 있는지 실 점검해 봐야 한다는 뜻을 담고 있습니다.
