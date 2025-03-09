---
layout: single
title: cocursor 서버 배포
categories: cocursor
excerpt: AWS 사용해서 서버 배포하기
---

npm 모듈도 배포했으니, 모듈과 소통할 서버도 배포해야 한다.
그 전에 Vercel을 이용해 Node.js 서버를 배포해본 적은 있지만, Vercel은 지속적인 통신(웹소켓, SSL) 배포는 지원하지 않기 때문에 직접 AWS를 이용해 서버를 배포하기로 결정했다.

cocursor 프로젝트를 진행하면서 가장 재밌었고, 배운 게 많은 지점이다.
아무래도 새롭게 도전해보는 부분이라 그런 것 같다.

## 1. AWS EC2

EC2는 AWS로부터 컴퓨터 한 대를 빌릴 수 있는 서비스이다.  
홈 서버를 통해 서버를 배포하면, 나의 IP 주소가 노출되기에 보안 위험이 있다.  
따라서 보통 서비스는 클라우드 호스팅 업체인 AWS, Naver Cloud, Azure 등을 사용한다.

EC2 생성 및 설정은 다음 영상을 주로 참고하였다.  
https://www.youtube.com/watch?v=LU8x1UEcPFA&list=PLtUgHNmvcs6qr33RT-UiguSsCr_2Gq0S3

### EC2 인스턴스 설정

- OS: Ubuntu
- 인스턴스 유형: t2.micro
- 인바운드 보안 그룹: 22, 80, 443, 8080

### 탄력적 IP 설정

IP 주소는 한정되어 있기 때문에 DHCP에 의해 자동으로 동적으로 변경된다.
만약 서비스를 배포하는 경우, 동적으로 변경되는 IP를 고정할 필요가 있다.

AWS에서는 탄력적 IP 설정을 통해 EC2 인스턴스의 IP 주소를 고정할 수 있다.

네트워크 및 보안 탭 > 탄력적 IP 주소 할당에서 내 인스턴스에 대해 고정 IP 주소를 연결할 수 있다.
이후 EC2가 재부팅되더라도 고정 IP 주소를 유지할 수 있다.

### 서버 실행

인스턴스에 연결하면 AWS에 있는 내 컴퓨터에 원격으로 접속할 수 있다.
`git clone`으로 내 프로젝트를 다운받고, `npm i`로 모듈을 설치하자.

git ignore에 의해 숨겨진 파일들은 직접 만들어줘야 한다.

이후 Node.js 서버를 자동으로 재시작할 수 있는 PM2를 사용하였다.
이를 통해 서버가 다운되더라도 자동으로 재시작되게 설정하였다.

```
# PM2 설치
npm install -g pm2

# 서버 실행
pm2 start main.js --name cocursor-server

# 서버 재부팅 시 자동 실행 설정
pm2 startup
pm2 save
```

이전에 설정한 탄력적 IP로 클라이언트에서 연결했을 때, 정상적으로 기능이 작동하는 것을 확인하였다! :)

```tsx
new WebSocket("ws://15.165.148.251/");
```

## 2. SSL/TLS 인증서

업데이트한 npm 모듈과 함께 서버까지 로컬에서 잘 작동하는 것을 보고,  
기분 좋게 배포했을 때도 잘 작동하는지 확인하였다.

하지만 다음과 같은 에러와 함께 앱 자체에서 런타임 에러가 발생하였다! :(

```
index-a679ab04d4de696e.js:1 Mixed Content: The page at 'https://panda-market-alexgoni.vercel.app/' was loaded over HTTPS, but attempted to connect to the insecure WebSocket endpoint 'ws://15.165.148.251:8080/'. This request has been blocked; this endpoint must be available over WSS.

2
installHook.js:1 SecurityError: Failed to construct 'WebSocket': An insecure WebSocket connection may not be initiated from a page loaded over HTTPS.
    at index-a679ab04d4de696e.js:1:7273
    at Rj (framework-0c7baedefba6b077.js:9:84178)
    at Ik (framework-0c7baedefba6b077.js:9:113177)
    at w (framework-0c7baedefba6b077.js:9:107725)
    at J (framework-0c7baedefba6b077.js:33:1364)
    at MessagePort.R (framework-0c7baedefba6b077.js:33:1894)
installHook.js:1 A client-side exception has occurred, see here for more info: https://nextjs.org/docs/messages/client-side-exception-occurred
```

세 가지 에러가 발생하였는데, 공통적으로 하나를 가리키고 있다.

1. Mixed Content  
   페이지가 HTTPS로 로드되었지만, 웹소켓 연결은 ws를 사용해서 에러가 발생했다.  
   기존 로컬 환경에서는 `http://localhost:3000`이여서 에러가 발생하지 않았던 것이였다.  
   모던 브라우저에서는 HTTPS 페이지에서는 HTTP를 사용하는 웹소켓 연결을 차단한다.

2. SecurityError  
   마찬가지로 HTTPS 페이지에서 ws를 사용해 보안 에러가 발생하였다.

3. Client-side Exeption  
   위 에러로 인해 클라이언트에서 예외가 발생한 것으로 추정된다.

<br />

해당 부분은 생각도 못했는데 배포를 하면서 알게되었다.  
ws에서 wss로 업그레이드하기 위해 SSL/TLS을 적용해보자.

### Nginx

SSL/TLS 인증서를 적용하는 방법에는 여러가지 방법이 있지만,  
가장 대중적으로 쓰이고 학습적인 측면에서 Nginx를 채택하였다.

클라이언트 요청을 Node.js 서버로 전달하는 리버스 프록시로 Nginx를 사용할 예정이다.
이 Nginx에 SSL/TLS 인증서를 적용하여 HTTPS 요청을 처리하도록 설정할 것이다.

이때 인증서를 적용하려면 도메인 이름이 필요하다.  
도메인 구매는 최대한 피하려고 했지만, 서비스 배포를 위해서는 어쩔 수 없다는 것을 이번에 깨달았다.  
마침 가비아에서 할인 중인 가장 저렴한 도메인을 발견해, `cocursor-server.store` 라는 도메인 네임을 1년에 500원에 구매했다.  
막상 사고나니 나만의 URL을 가지게 된 것 같아 꽤 기분이 꽤 괜찮았다.

이후 가비아 DNS에서 해당 주소를 입력했을 때 실질적으로 어디 IP 주소로 이동해야 하는지 알려줘야 한다.  
이를 레코드 설정이라고 하고, 가비아 DNS 관리툴에서 다음과 같이 설정하였다.

![alt text](/images/2025-02-16-cocursor-server-deploy/image.png)

이제 EC2에서 Cerabot으로 Let's Encrypt SSL 인증서를 적용해보자.

```
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d cocursor-server.store
```

이렇게 하면 인스턴스 안에 인증서 파일이(cert.pem, chain.pem, fullchain.pem, private.pem) 생성된다.
참고로 Let's Encrypt의 인증서는 3개월마다 갱신 해야되는데, 아래 명령어를 통해 자동으로 재생성할 수 있다.

```
sudo certbot renew --dry-run
```

이제 Nginx 설정을 해보자.

```nginx
server {
    listen 80;
    server_name cocursor-server.store;
    return 301 https://$host$request_uri; # 모든 HTTP 요청을 HTTPS로 리디렉션
}

server {
    listen 443 ssl; # HTTPS 요청을 수락
    server_name cocursor-server.store;

    # 인증서 적용
    ssl_certificate /etc/letsencrypt/live/cocursor-server.store/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cocursor-server.store/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/cocursor-server.store/chain.pem;

    location /ws/ {
        proxy_pass http://localhost:8080; # WebSocket 요청을 로컬 서버(8080 포트)로 전달
        proxy_http_version 1.1; # WebSocket이 HTTP/1.1을 사용하도록 설정
        proxy_set_header Upgrade $http_upgrade; # WebSocket 업그레이드 요청 처리
        proxy_set_header Connection "upgrade"; # 연결을 업그레이드하도록 설정
        proxy_set_header Host $host; # 원래 요청의 호스트 정보를 유지
    }
}
```

이제 `/ws/` 주소로 요청이 오면, Nginx가 이를 내 Node.js 서버로 전달하게 된다.
이를 통해 Node.js 코드 변경 없이 ws를 wss로 업그레이드할 수 있었다.

또한, 기존에는 EC2에서 8080 포트를 열어 직접 Node.js 서버와 연결했지만,
이제는 Nginx가 요청을 프록시하므로 8080 포트를 닫아도 된다.

마지막으로, 배포된 사이트에서도 에러 없이 정상적으로 작동하는 것을 확인했다! 🚀

## 3. 전체 아키텍처

전체 아키텍처를 그려보면 다음과 같다.

1. cocursor를 사용하려는 개발자는 서비스에서 API Key를 발급받는다.
   이때 DB에 해당 API Key를 저장한다.
2. 개발자는 프로젝트에 `npm i cocursor`로 모듈을 설치하고, 페이지 컴포넌트 가장 상위에 선언한다.
3. API Key를 통해 cocursor 서버와 웹소켓 통신을 하고, 이를 통해 사용자들의 커서 정보를 페이지에 표기한다.

![alt text](/images/2025-02-16-cocursor-server-deploy/image2.png)
