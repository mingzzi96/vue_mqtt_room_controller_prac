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
      client: mqtt.connect(`ws://${process.env.VUE_APP_API_URL}`, {
        port: 10001,
      }),
      isHeatSystemOn: false,
    };
  },
  methods: {
    handleClickTurnOffController() {
      if (this.client) {
        this.client.end();
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
          this.isHeatSystemOn = true;
        } else if (setTopic === "control" && setMessage === "tooHot") {
          this.isHeatSystemOn = false;
        }
      });
    },
  },
};
</script>
