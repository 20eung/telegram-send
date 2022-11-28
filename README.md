# Telegram Bot 이야기

#### 참고링크의 문서를 단순 구글번역하였음.

## 리눅스 서버 시작 시 Telegram으로 메시지 보내는 설정

## 텔레그램 봇 만들기

Telegram 그룹 또는 채널에 메시지를 보내려면 먼저 자신의 봇을 만들어야 합니다.    
Telegram을 열고 @BotFather를 찾은 다음 입력하십시오.

## 텔레그램 채널 만들기

Telegram에서 새 채널을 만들고 봇을 구성원으로 추가하십시오.    
따라서 봇은 채널에 메시지를 보낼 수 있습니다.    
채널 ID를 얻으려면 먼저 채널에 메시지를 게시하십시오.    
그런 다음 이 링크 템플릿을 사용하여 채널 ID를 가져옵니다.
```
https://api.telegram.org/bot<YourBOTToken>/getUpdates
```

다음은 응답 예시입니다.

```
{
  "ok":true,
  "result": [
    {
      "update_id":123,
      "channel_post": {
        "message_id":48,
        "chat": {
          "id":-123123123, // this is your channel id
          "title":"Notifications",
          "type":"channel"
        },
        "date":1574485277,
        "text":"test"
      }
    }
  ]
}
```

## 메시지를 보내는 스크립트

메시지를 보내려면 간단한 명령을 사용할 수 있습니다.

```
curl 'https://api.telegram.org/bot<YourBOTToken>/sendMessage?chat_id=<channel_id>&text=<text>'
```

그러나 프로그래밍에서는 저수준 구현을 숨기는 것이 좋습니다.    
따라서 우리는 Linux 터미널 명령 telegram-send를 만들고 이 간단한 명령으로 메시지를 보낼 수 있습니다.

telegram-send.sh 파일을 만듭니다.

```
touch telegram-send.sh
```

그런 다음 이 파일에 스크립트를 추가합니다. 스크립트에서 그룹 ID와 토큰을 설정합니다.
```
#!/usr/bin/bash
    
GROUP_ID=<group_id>
BOT_TOKEN=<bot_token>
DATE="$( date "+%Y-%m-%d %H:%M")"

# this 3 checks (if) are not necessary but should be convenient
if [ "$1" == "-h" ]; then
  echo "Usage: `basename $0` \"text message\""
  exit 0
fi

if [ -z "$1" ]
  then
    echo "Add message text as second arguments"
    exit 0
fi

if [ "$#" -ne 1 ]; then
    echo "You can pass only one argument. For string with spaces put it on quotes"
    exit 0
fi

#TXT="$DATE%0A$1"
TXT="$DATE"'\n'"$1"

curl -s \
    --data parse_mode=HTML \
    --data "text=$TXT" \
    --data "chat_id=$GROUP_ID" \
    'https://api.telegram.org/bot'$BOT_TOKEN'/sendMessage' > /dev/null
```

해당 장소에 토큰을 저장하는 것은 좋지 않지만 지금은 괜찮습니다.    
또한 봇이 채널에서 메시지를 보낼 때만 수행할 수 있는 작업을 제한할 수 있습니다.

이 스크립트를 실행하려면 권한을 추가해야 합니다.

```
chmod +x telegram-send.sh
```

이제 당신은 그것을 테스트 할 수 있습니다

```
./telegram-send.sh "Test message"
```

모든 곳에서 이 스크립트를 사용하고 대신 telegram-send를 입력하려면 ./telegram-send.sh를 /usr/local/bin/ 폴더에 추가하십시오.

```
sudo mv telegram-send.sh /usr/local/bin/telegram-send
```

/usr/local/bin에 있는 모든 파일의 소유자는 루트 사용자입니다.    
스크립트로 동일한 작업을 수행해 보겠습니다.

```
sudo chown root:root /usr/local/bin/telegram-send
```

이제 당신은 그것을 테스트 할 수 있습니다.

```
telegram-send "Test message"
```

## SSH 로그인 시 알림 보내기

/etc/profile.d/ 폴더에 있는 .sh 확장자를 가진 모든 파일은 bash 로그인 셸에 들어가거나 데스크톱 세션이 로드될 때마다 실행됩니다.    

알림을 보내는 새 스크립트를 추가해 보겠습니다.

```
touch login-notify.sh
```

이 코드를 스크립트에 추가

```
#!/usr/bin/bash

# prepare any message you want
login_ip="$(echo $SSH_CONNECTION | cut -d " " -f 1)"
login_date="$(date +"+%Y-%m-%d %H:%M")"
login_name="$(whoami)"

# For new line I use $'\n' here
message="$HOSTNAME"': '"New login to server"$'\nID: '"$login_name"$'\nIP: '"$login_ip"$'\nTime: '"$login_date"

#send it to telegram
/usr/local/bin/telegram-send "$message"
```

그런 다음 이 스크립트를 /etc/profile.d/ 폴더로 이동합니다.

```
sudo mv login-notify.sh /etc/profile.d/login-notify.sh
```

이제 웹 서버에 다시 로그인하여 작동하는지 확인하십시오.


## (추가) 서버 켜지고 꺼질 때 알림 보내기

서버가 켜지고 꺼질 때 알림 보내는 새 스크립트를 만들어 보겠습니다.

```
touch telegram.service
```

이 코드를 스크립트에 추가

```
[Unit]
Description=Send Telegram messagesm$
After=docker-mcje.service

[Service]
RemainAfterExit=yes
ExecStart=/usr/local/bin/telegram-send "Minecraft Server is Started"
ExecStop=/usr/local/bin/telegram-send "Minecraft Server is Stopping"

[Install]
WantedBy=multi-user.target
```

그런 다음 이 스크립트를 /etc/systemd/system 폴더로 이동합니다.

```
sudo mv telegram.service /etc/systemd/system/
```

telegram 을 서비스에 등록합니다.

```
sudo systemctl daemon-reload
sudo systemctl enable telegram
```

이제 웹 서버를 다시 시작하여 작동하는지 확인하십시오.


## 참고 링크
- Send message to Telegram on any SSH login: https://bogomolov.tech/Telegram-notification-on-SSH-login/