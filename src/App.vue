<template>
  <div id="app">
    <div class="main">
      <canvas id="drawBoard" width="700" height="400"></canvas>
    </div>
    <div class="tools">
      <div class="palette">
        <div
          class="palette-item"
          :class="{ active: current === index }"
          v-for="(p, index) of palette"
          :key="p"
          :style="{ background: p }"
          @click="changeColor(index)"
        ></div>
      </div>
      <div class="size">
        <input
          type="range"
          v-model="size"
          min="5"
          max="20"
          @change="changeSize"
        />
        <span>{{ size }}</span>
      </div>
      <button @click="clearCanvas">重置</button>
    </div>
  </div>
</template>
<script>
import IRealTime from "irealtime";
export default {
  data() {
    return {
      palette: ["#333", "#e91e63", "#673ab7", "#2196f3", "#4caf50"],
      current: 0,
      size: 6,
      irealtime: null,
    };
  },
  watch: {
    size(val) {
      this.ctx.lineWidth = val;
    },
    current(val) {
      this.publish(
        JSON.stringify({
          type: "changeColor",
          current: val,
        })
      );
    },
  },
  mounted() {
    this.irealtime = new IRealTime({
      host: "hk.irealtime.cn",
      appkey: "your appkey", //如何获取appkey： https://irealtime.cn/docs/get_account_and_appkey.html
      onConnected: function () {
        console.log("连接成功...");
      },
      onDisconnected: function () {
        console.log("连接断开...");
      },
      onConnectFailed: function (error) {
        console.log("连接失败...", error);
      },
    });

    this.irealtime.subscribe({
      channels: ["mychannel"],
      onMessage: (data) => {
        console.log(this.path);
        this.syncPath(data.message);
      },
      onSuccess: function () {},
      onFailed: function () {},
    });

    this.canvas = document.getElementById("drawBoard");
    this.ctx = this.canvas.getContext("2d");
    this.stage_info = this.canvas.getBoundingClientRect();
    this.path = {
      beginX: 0,
      beginY: 0,
      endX: 0,
      endY: 0,
    };
    this.ctx.lineWidth = this.size;
    this.isDraw = false;
    this.draw();
  },
  methods: {
    draw() {
      let that = this;
      this.canvas.onmousedown = (e) => {
        that.ctx.beginPath();
        that.path.beginX = e.pageX - that.stage_info.left;
        that.path.beginY = e.pageY - that.stage_info.top;
        that.ctx.moveTo(that.path.beginX, that.path.beginY);
        that.isDraw = true;
        this.publish(
          JSON.stringify({
            type: "mousedown",
            path: this.path,
          })
        );
      };
      this.canvas.onmousemove = (e) => {
        if (that.isDraw) {
          that.drawing(e);
        }
      };
      this.canvas.onmouseup = () => {
        that.isDraw = false;
        this.publish(
          JSON.stringify({
            type: "stop",
            path: this.path,
          })
        );
      };
    },
    drawing(e) {
      this.path.endX = e.pageX - this.stage_info.left;
      this.path.endY = e.pageY - this.stage_info.top;
      this.ctx.lineTo(this.path.endX, this.path.endY);
      this.ctx.stroke();
      let data = {
        type: "draw",
        size: this.size,
        current: this.current,
        path: this.path,
      };
      this.publish(JSON.stringify(data));
    },
    syncPath(data) {
      const { type, size, current, path } = JSON.parse(data);
      if (type === "changeColor") {
        this.current = current;
        this.ctx.strokeStyle = this.palette[this.current];
        return;
      }
      if (type === "changeSize") {
        this.size = size;
        this.ctx.lineWidth = this.size;
        return;
      }

      if (type == "clear") {
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        return;
      }
      if (this.path.beginX != path.beginX) {
        console.log(type === "mousedown");

        if (type === "mousedown") {
          this.ctx.beginPath();

          this.ctx.moveTo(path.beginX, path.beginY);
          this.isDraw = 1;
        } else if (type === "draw" && this.isDraw === 1) {
          this.ctx.lineTo(path.endX, path.endY);
          this.ctx.stroke();
        }
      }
      if (type === "stop") {
        this.isDraw = 0;
      }
    },
    clearCanvas() {
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.goal = "";
      this.publish(
        JSON.stringify({
          type: "clear",
        })
      );
    },
    changeColor(index) {
      this.current = index;
      this.ctx.strokeStyle = this.palette[index];
    },
    changeSize(e) {
      this.publish(
        JSON.stringify({
          type: "changeSize",
          size: e.target.value,
        })
      );
    },
    publish(msg) {
      this.irealtime.publish({
        channel: "mychannel",
        message: msg,
        onSuccess: function (data) {
          console.log("success:", data);
        },
        onFailed: function (error) {
          console.log("failed:", error);
        },
      });
    },
  },
};
</script>
<style lang="less" scoped>
#app {
  margin: 60px;
}
.main {
  display: flex;
}
canvas {
  background: #ccc;
  cursor: pointer;
}
.tools {
  display: flex;
  align-items: center;
  .palette {
    display: flex;
    align-items: center;
    &-item {
      width: 40px;
      height: 40px;
      border-radius: 50%;
      margin: 5px;
      &.active {
        border: 5px solid #ccc;
      }
    }
  }
  button {
    margin: 5px;
  }
}
</style>