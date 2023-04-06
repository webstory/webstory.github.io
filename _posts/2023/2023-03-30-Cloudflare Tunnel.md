---
category: 2023
tag:
  [
    "2023",
    "cloudflare",
    "cloudflare_tunnel",
    "argo_tunnel",
    "dev",
    "zero_trust",
    "cloudflared",
  ]
# author_profile: false
# sidebar:
#   nav: "how-to-github-pages"
# search: false # default true
---

# Cloudflare Tunnel

로컬 서버를 잠시 동안 인터넷에 열어주어야 할 때가 가끔 생긴다. 개발이 어느 정도 진척된 뒤에는 개발용으로 세팅된 서버(스테이징 서버, 알파 테스트 서버, 뭐라고 불러도 좋다)에 git push로 넘겨줘서 거기서 호스팅하지만 임시 땜빵용으로 무언가를 우회해서 서비스해줘야 하거나 정말 잠깐 서비스해주고 서비스를 삭제해야 하는 경우가 생길 수 있다. 그때마다 일일이 개발 서버에 올렸다 내렸다 하는 건 귀찮은 일이다. 하지만 로컬 환경은 보안상의 이유로 외부에서 접근할 수 없게 만들어져 있다. 구체적으로는 공유기의 NAT를 통해 인터넷에 액세스하기에 내부 포트를 직접 포워딩하기 전에는 로컬 컴퓨터에 접근할 수가 없다. 게다가 나의 로컬 개발환경은 WSL 하부에 만들어져 있는데 WSL은 별도의 설정을 하지 않는 한 NAT로 동작하기에 이쪽 포트에 localhost가 아닌 외부에서 접근하기는 상당히 까다롭다.

물론 개발 서버에 ssh tunneling 하는 방법이 있다. 예를 들어 이렇게 할 수 있다.

```sh
ssh -fN -L 8805:127.0.0.1:8805 hoya@192.168.1.11
```

이 명령으로 로컬호스트의 8805번 포트를 192.168.1.11 서버의 8805번 포트로 포워딩한 뒤 서버쪽 리버스 프록시 설정을 만져주는 방법도 있다.

하지만, 위에서 언급했듯이 영 귀찮다. 그래서 ngrok을 사용했던 건데... ngrok 는 몇 가지 단점이 있다.

1. url을 자유롭게 정하기 어렵다. 커스텀 도메인 설정이 좀 까다로운 편이다
2. 불안정하다. 한 시간 정도는 그럭저럭 버티지만 대충 오전 중에 띄워놓고 점심먹고 오면 끊겨 있는 경우가 자주 있었다

기억난 건 저정도다. 두 가지 모두 임시로 터널을 열어주는 거라 실사용 측면에서는 아무 문제가 없다. 사실 클라우드플레어 터널을 사용해 볼 생각을 한 것은 호기심에 가깝다.

## Install

cloudflared 를 다운받아 설치한다. 나는 리눅스 환경이므로 https://pkg.cloudflare.com/index.html 에서 우분투 22.04LTS 설치법을 따랐다.

```sh
# Add cloudflare gpg key
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

# Add this repo to your apt repositories
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared jammy main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# install cloudflared
sudo apt-get update && sudo apt-get install cloudflared
```

## 로그인

처음에 클라우드플레어 터널을 사용하려면 로그인해야 한다.

```sh
cloudflared tunnel login
```

이 작업은 전에 설치한 wrangler(클라우드플레어 워커를 위한 관리 도구)과는 무관한 것 같다.

## 터널 생성

```sh
cloudflared tunnel create mytunnel
Tunnel credentials written to /home/hoya/.cloudflared/189af159-082c-4378-957b-9220cf70ebd2.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.

Created tunnel mytunnel with id 189af159-082c-4378-957b-9220cf70ebd2
```

create할 때 UUID가 나올 것이다. 나중에 다시 확인하려면 `cloudflared tunnel list` 를 입력한다.

```sh
cloudflared tunnel list

You can obtain more detailed information for each tunnel with `cloudflared tunnel info <name/uuid>`
ID                                   NAME     CREATED              CONNECTIONS
189af159-082c-4378-957b-9220cf70ebd2 mytunnel 2023-04-06T01:03:03Z
```

## 라우트 설정

```sh
cloudflared tunnel route dns 189af159-082c-4378-957b-9220cf70ebd2 mytunnel.mirma.cc

2023-04-06T01:04:53Z INF Added CNAME mytunnel.mirma.cc which will route to this tunnel tunnelID=189af159-082c-4378-957b-9220cf70ebd2
```

이 정보는 클라우드플레어 웹 콘솔의 Zero Trust 에서도 볼 수 있다.
![Zero Trust](/assets/images/2023/2023-03-30-Cloudflare%20Tunnel/Zero-trust.png)

![Tunnels](/assets/images/2023/2023-03-30-Cloudflare%20Tunnel/tunnels.png)

또는 클라우드플레어 도메인 관리 페이지의 DNS항목에서도 볼 수 있다.
![CNAME](/assets/images/2023/2023-03-30-Cloudflare%20Tunnel/cname.png)

콘솔 명령을 사용하지 않고 직접 CNAME을 작성할 수도 있는데 이 경우 아래와 같은 형식이 된다.

```
CNAME mytunnel 189af159-082c-4378-957b-9220cf70ebd2.cfargotunnel.com
```

## config.yml 작성

~/.cloudflared/config.yml 파일을 작성한다

```yaml
tunnel: 189af159-082c-4378-957b-9220cf70ebd2
credentials-file: /home/hoya/.cloudflared/189af159-082c-4378-957b-9220cf70ebd2.json
ingress:
  - hostname: mytunnel.mirma.cc
    service: http://localhost:7860
  - service: http_status:404
```

## 터널 실행

```sh
cloudflared tunnel run
```

이렇게 하면 mytunnel 의 터널 상태가 HEALTHY로 변하고, mytunnel.mirma.cc url을 통해 localhost:7860 이 서비스되는 것을 확인할 수가 있다.

![Tunnel is Healthy](/assets/images/2023/2023-03-30-Cloudflare%20Tunnel/healthy.png)

### 번외: 여러 개의 터널을 작성했을 때

별도의 config.yml 파일을 작성해야 한다. 한 config.yml 파일에 여러 개의 tunnel 설정을 할 수 없게 되어 있다.

터널을 실행할 때는 해당 config.yml 파일의 경로를 명시해준다.

```sh
cloudflared tunnel --config ~/.cloudflared/mytunnel.yml run
```

하지만 hostname은 여러 개 만들 수 있다. 이 경우 터널을 새로 만들지 말고 기존 터널에 dns CNAME만을 추가한다.

```sh
cloudflared tunnel route dns mytunnel2 mytunnel2.mirma.cc
2023-04-06T02:15:36Z INF Added CNAME mytunnel2.mirma.cc which will route to this tunnel tunnelID=189af159-082c-4378-957b-9220cf70ebd2
```

tunnelID가 기존 터널인 mytunnel과 같음을 확인하고, config.yml 파일에 아래와 같이 적는다.

```yaml
tunnel: 189af159-082c-4378-957b-9220cf70ebd2
credentials-file: /home/hoya/.cloudflared/189af159-082c-4378-957b-9220cf70ebd2.json
ingress:
  - hostname: mytunnel.mirma.cc
    service: http://localhost:7860
  - hostname: mytunnel2.mirma.cc
    service: http://localhost:7861
  - service: http_status:404
```

## 터널 삭제

```sh
cloudflared tunnel delete mytunnel
```

터널을 삭제해도 도메인의 CNAME부분은 여전히 남아있으니 웹 콘솔이나 CLI를 통해 수동으로 지워 주어야 한다.
