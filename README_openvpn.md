## 가상머신에 OpenVpn 서버 설치하기

[이전](https://github.com/kangsunku/Iac-k8sv1.26--forVM/blob/main/README_k8s.md)에 가상 머신에 개발 환경을 구성하고 나서 한가지 불편한 점이 생겼다. 바로 외부 환경에서 가상머신으로 접속이 까다롭다는 점이다.

가상머신이 설치되어 있는 서버에서는 쉽게 접근할 수 있지만 외부에서 접근하려면 포트포워딩이 가능하도록 외부 포트를 설정하고 개방해줘야한다.



### 외부에서 가상머신으로 요청하기

ssh 포트가 위와 같이 설정되어 있다고 가정해보자.  
192.168.50.1 서버를 외부 클라이언트에서 접속하기 위해서는 외부 ip와 12000 포트로 ssh 요청을 하면 된다. 그렇게 되면 내부 ip의 9000 포트로 전달되고 최종적으로 가상머신 192.168.29.1 서버의 22번 포트로 요청이 전달이 된다.

-   외부 ip:12000 -> 내부 ip:9000 -> 192.168.50.1:22

192.168.50.2 서버는 외부 포트가 열려 있지 않으므로 외부에서 접속을 할 수 없다. 호스트 서버에 직접 접속하여 가상머신에 요청을 전달해야 한다.

-   외부 요청 불가능
-   내부 ip:9001 -> 192.168.29.2:22

192.168.29.3 서버의 경우는 가상 머신 설정에 따라서 호스트에서 접근이 가능할 수도 있고 접근이 불가능할 수 있다. VirtualBox의 호스트 전용 네트워크의 경우 가상 머신과 네트워크를 공유하므로 직접 192.168.29.3:22로 요청이 가능하다.

-   외부 요청 불가능
-   가상 머신의 설정의 따라 요청이 가능 or 불가능

따라서 외부 요청이 가능하려면 내부 포트와 외부 포트를 전부 설정해주고 개방을 해야하는데 이는 번거로운 작업이다. 또, 외부에 서버를 직접 노출하게 되는 만큼 보안도 취약하다.

이러한 불편함을 줄이는 방법은 없을까? 가상 머신 네트워크 안에 VPN 서버를 설치하면 이러한 문제를 해결할 수 있다.

### OpenVPN 서버 구성


OpenVPN 서버를 192.168.30.1 서버에 설치했다고 해보자. OpenVPN은 UDP로 통신하는 경우 1194 포트를 통해 전달되므로 편의상 외부와 내부포트도 동일하게 1194로 설정하고 개방한다면, OpenVPN 클라이언트를 통해 외부에서 1194 포트로 연결이 가능하다.

vpn이 연결되면 모든 요청을 OpenVPN 서버를 통해 우회할 수 있으므로 가상 머신에 직접 요청이 가능하다.  
예를 들어, 192.168.50.1 서버의 ssh에 접근 하기 위해서 외부 ip와 port로 요청할 필요가 없다. 외부에서 192.168.50.1의 22번 포트로 직접 ssh로 요청해도 정상적으로 응답을 받을 수 있다.

이제 설치 과정을 보자

### 가상머신에 OpenVPN 서버 설치하기

OpenVpn 서버를 설치하기 위한 방법을 찾던 도중 아주 쉬운 방법을 찾을 수 있었다.  
[How To Set up OpenVPN Server In 5 Minutes on Ubuntu Linux](https://www.cyberciti.biz/faq/howto-setup-openvpn-server-on-ubuntu-linux-14-04-or-16-04-lts/)

우분투에서 5분만에 OpenVPN 서버를 설치하는 방법인데 설치 스크립트를 미리 정의해 놓아서 입력만으로 빠르게 설치가 가능하다.

```bash
$ wget https://git.io/vpn -O openvpn-install.sh
```

다음과 같은 설치 스크립트를 다운받는다.

```bash
$ ./openvpn-install.sh
```

설치 스크립트를 실행한다.

```null
Welcome to this OpenVPN road warrior installer! Which IPv4 address should be used? 1) 10.0.2.15 2) 192.168.30.1 - 가상머신 ip IPv4 address [1]: (엔터 입력) This server is behind NAT. What is the public IPv4 address or hostname? Public Ipv4 address / hostname [xx.xx.xx.xx] (엔터 입력) Which protocol should OpenVPN use? 1) UDP (recommended) 2) TCP Protocol [1]: (엔터 입력) What port should OpenVpn listen to? Port [1194]: (엔터 입력) Select a DNS server for the clients: 1) Current system resolvers 2) Google 3) 1.1.1.1 4) OpenDNS 5) QUad9 6) AdGuard DNS server [1]: 2 Enter a name for the first client: Name [client]: yellowsunn
```

스크립트가 실행되면 다음과 같은 옵션을 설정할 수 있다.  
첫번 째 `Which IPv4 address should be used?` 질문의 경우 가상머신 설정에 따라 IP 주소가 여러개 표시된다. 위의 표시된 IP 주소는 Virtual Box 네트워크의 NAT(10.0.2.15), 호스트 전용 네트워크 (192.168.30.1) 다. 여기서는 기본 1번인 NAT 주소를 입력해야 하는데 호스트 전용 네트워크의 경우 요청을 인터넷으로 전달할 수 없기 때문에 동작하지 않는다.

> NAT: 가상 머신 내부의 구성된 네트워크
> 
> -   기본 IP는 10.0.2.15 주소로 고정되어 있으며 호스트나 인터넷으로 요청이 가능하다.
> -   반대로 호스트나 인터넷에서 가상머신으로 요청이 불가능하고 가상머신끼리 통신도 불가능.
> 
> 호스트 전용 네트워크: 호스트와 연결하기 위해 구성된 네트워크
> 
> -   호스트에서 가상 머신으로 요청이 가능하다
> -   반대로 가상 머신에서 호스트, 외부로 요청이 불가능
> -   가상머신끼리 통신 가능

그 외 다른 설정을 환경에 맞게 설정하고 마지막 OpenVPN 클라이언트 이름을 지정하면 된다.  
필자의 경우 `yellowsunn` 을 설정했는데

```bash
$ ls yellowsunn.ovpn
```

설정한 이름으로 ovpn 확장자의 파일이 생성된 것을 확인할 수 있다. 이 파일을 접근할 클라이언트에 전달해주도록하자.

### OpenVPN 클라이언트 연결하기

이제 OS환경에 맞게 클라이언트에서 OpenVPN Client 를 다운로드 받아야한다.  
[맥 OS에서 OpenVPN Client 설치하기](https://openvpn.net/client-connect-vpn-for-mac-os/)  
[윈도우에서 OpenVPN Client 설치하기](https://openvpn.net/client-connect-vpn-for-windows/)

그 다음 서버에서 생성된 ovpn 파일을 불러오면 vpn 서버와 연결이 가능하다.  
![](https://velog.velcdn.com/images/yellowsunn/post/6e131300-58be-43ea-8336-717150620a97/image.png)

![](https://velog.velcdn.com/images/yellowsunn/post/b521a7e0-7218-44b1-a0d4-2c11e2b97ba0/image.png)

#### 연결 테스트

-   ping으로 가상머신 서버(192.168.29.10)에 직접요청 했을 때 정상적으로 응답이 오는 것을 확인할 수 있다.  
    ![](https://velog.velcdn.com/images/yellowsunn/post/000c36ae-8947-4dd8-ba78-d6e315d73d97/image.png)

-   가상 머신 서버(192.168.29.20)에 argocd를 올렸더니 웹에서도 정상 확인 가능하다.  
    ![](https://velog.velcdn.com/images/yellowsunn/post/2d20a3e7-2397-434f-b31c-e808b9552425/image.png)

참고자료  
[https://whitekeyboard.tistory.com/730](https://whitekeyboard.tistory.com/730)  
[https://www.cyberciti.biz/faq/howto-setup-openvpn-server-on-ubuntu-linux-14-04-or-16-04-lts](https://www.cyberciti.biz/faq/howto-setup-openvpn-server-on-ubuntu-linux-14-04-or-16-04-lts)
