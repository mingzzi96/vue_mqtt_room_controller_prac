# npm install 및 npm run serve하기 전에 env 파일 생성하셈

# VUE_APP_API_URL= 로컬 네트워크 주소

## 🎀 이 글을 읽기 전에

[겁나 쉽게 mqtt 통신 야매로 연습해보기](https://velog.io/@ooo3289/mqtt-%ED%86%B5%EC%8B%A0-%EC%97%B0%EC%8A%B5%ED%95%98%EA%B8%B0)

위 포스팅으로 가서 10분도 안걸리니까 연습하고 오기를 추천.

## 🎀 내 로컬 브로커 확인 및 websocket 설정

원래 https://test.mosquitto.org/ 라는 곳에서 제공하는
테스트용 브로커를 사용하고 있었는데,
사실 mosquitto를 설치하면 브로커가 자동으로 설치된다

```
systemctl status mosquitto.service
```

mosquitto 설치 후 위 명령어를 터미널에 입력하면
내 컴퓨터에 설치된 mosquitto의 정보를 확인할 수 있는데
현재 mosquitto가 켜져 있는지 아닌지, 그리고 브로커 정보도 확인 가능하다.

나는 이걸 몰랐단 말이지?
![](https://velog.velcdn.com/images/ooo3289/post/6b9874b2-ac4d-4eef-9e71-54180e1f387a/image.png)

암튼 그래서
`mosquitto.conf` 파일에 내 로컬 웹소켓 포트를 추가해서
굳이 다른 브로커 주소를 사용하지 않고 통신해 보겠다.

#### 1. mosquitto.conf 파일을 연다

```
sudo vim /etc/mosquitto/mosquitto.conf
```

vim 이라서 마우스 사용은 안됨 주의
그리고 접근하려면 아마 컴퓨터 비밀번호 입력하라고 함

#### 2. websocket 주소 추가

websocket을 사용해야 웹에서 mqtt와의 통신이 가능하기 때문에
반드시 websocket 설정이 필요하다.

```
listener 10000
protocol mqtt

listener 10001
protocol websockets
```

conf 파일 내용에 listener 무조건 들어가있을 건데
다른 것들은 건들지 말고 그냥 저 내용에 맞춰서 수정해 주면 됨.
대충 화살표키로 쭉쭉 움직여서 추가 ㄱㄱ

listener는 포트 번호가 되는 건데,
아무 랜덤 번호 외우기 쉬운걸로 설정해도 무관하다.

vim에서 파일 저장하고 나가는 방법은

```
:wq
```

입력하고 엔터쳐서 마무리하면 됨.

#### 3. mosquitto 재실행

```
systemctl restart mosquitto.service
```

변경사항이 있었으니, 위 명령어 사용해서 mosquitto 재실행 해준다.

## 🎀 야매 원격 시스템 구현

여기서 부터 이제 재밌다.
![](https://velog.velcdn.com/images/ooo3289/post/a3085670-17e9-4726-b75c-212cb5e97eea/image.png)

```js
<template>
  <div id="app">
    <h1 v-if="isHeatSystemOn">보일러<br />ON</h1>
    <h1 v-else>보일러<br />OFF</h1>
    <div>
      <button type="button" @click="handleClickTurnOnController()">
        원격제어 켜기
      </button>
      <button type="button" @click="handleClickTurnOffController()">
        원격제어 끄기
      </button>
    </div>
  </div>
</template>

<script>
import mqtt from "mqtt";

export default {
  name: "App",
  data() {
    return {
      isHeatSystemOn: false, // 이 상태에 따라 화면 제어
    };
  },
};
</script>
```

vue를 사용해서 간단하게 이런 화면을 만들어 보았다.
원격제어를 할 수 있는 코드를 짜볼것이다.

쁘띠 야매 컨셉 : `원격제어 켜기 버튼을 누르면 실시간으로 방 온도를 받아와서 특정 온도가 넘어서면 보일러를 켠다.`

꿀잼~~~~~~~

#### 1. mqtt 불러오기

```js
<script>import mqtt from "mqtt"; ...</script>
```

당연히 mqtt를 사용해야하니, 앞서 마크업된 코드위에 불러와 준다.

#### 2. mqtt subscribe하기

```js
  data() {
    return {
      client: mqtt.connect("ws://로컬 주소를 입력", {
        port: 10001,
      }),
      isHeatSystemOn: false,
    };
  },
```

![](https://velog.velcdn.com/images/ooo3289/post/8f215260-db52-4b8c-bbb6-8b80096ec857/image.png)

vue에서 npm run serve하면 나오는 주소 중에 Network에 해당하는 저 가려진 부분의 주소를
`ws://`다음에 넣어주면 된다.
내 로컬 주소 앞에 `ws://`만 붙여주면 그냥 websocket 사용 준비 완료인 것임!

port에는 당연히 아까 설정해 줬던 websocket 포트 번호를 넣어주면 됨.
나의 경우 10001로 설정했으니까 그걸로 ㄱㄱ 하겠음.

```js
  methods: {
    handleClickTurnOffController() {
      if (this.client) {
        this.client.end(); // mqtt 제어 종료하고 싶을 때 사용하는 메서드.
        this.isHeatSystemOn = false;
        alert("원격제어를 껐습니다. 이제 불이 꺼집니다.");
      }
    },
    handleClickTurnOnController() {
      //mqtt가 연결되어있는지 확인
      console.log(this.client.connected);
      //topic 구독
      this.client.subscribe("control");
      //구독해놓은 메시지가 들어오면 실행
      this.client.on("message", (topic, message) => {
        console.log(topic, message);
        let setTopic = topic.toString();
        let setMessage = message.toString();
        console.log(setTopic, setMessage);

        if (setTopic === "control" && setMessage === "tooCold") {
          // 온도가 지금 넘 추웡~~ 상태면 보일러 켜
          this.isHeatSystemOn = true;
        } else if (setTopic === "control" && setMessage === "tooHot") {
          // 아니면 꺼
          this.isHeatSystemOn = false;
        }
      });
    },
  },
```

`handleClickTurnOffController` : 원격제어 끄기 버튼이 눌렸을 때 실행됨.

`handleClickTurnOnController` : 원격제어 켜기 버튼이 눌렸을 때 실행됨. on 메서드를 활용해서 mqtt topic과 message를 받아와 찍어줄 수 있다. 이를 활용해서 publish 할 때 구독한 topic 정보와 메시지를 확인하고, 원하는 메시지가 도착했다면? `isHeatSystemOn`를 제어해 주면 된다.

#### 3. mqtt publish하기

그니까 여기가 이제 꿀잼 파트인것임.
내 로컬에서 내가 스스로 방에 온도 체크하는 기계가 되었다고 생각해 보고 publish 날려보면 됨 ㅋㅋㅋ

```
mosquitto_pub -h 내 네트워크 주소 -p 10000 -t 'control' -m 'tooCold'
```

원격제어 켜기 버튼을 눌러서 mqtt 실행 시키고
터미널에 위 명령어를 쳐서 넘 춥다고 publish 해보셈

```
this.client.subscribe("control");
```

토픽이 control인 이유는 위에서 control을 subscribe하고 있기 때문임.
근데 아무거나 해줘도 상관없음. ㅇㅇ

![](https://velog.velcdn.com/images/ooo3289/post/d25b194b-57e1-412e-8888-bef90cc7ae9c/image.png)

춥다 했더니 보일러 켜줌!

```
mosquitto_pub -h 내 네트워크 주소 -p 10000 -t 'control' -m 'tooHot'
```

너무 덥다고 보내면 보일러 꺼줄거고, 원격제어 끄기 버튼 누르면 mqtt 통신이 종료될 것임~~

이 터미널에서 `mosquitto_pub` publish 해주는 동작 자체를 원격 조종으로 생각해보면
mqtt 통신에 대한 이해가 쉽게 잘 되는 것 같은 느낌적인 느낌이 든다,,,,,,,,,
