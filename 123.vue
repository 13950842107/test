<template>
  <div class="camera-container">
    <div class="error-message" @click="redeploy()" v-if="err"><span
        style="color: red;font-size: larger;font-weight: bolder;line-height: 100%;">{{ msg }}</span></div>
    <div class="video-container">
      <div :ref="camera.deviceSerial" :id="camera.deviceSerial" class="video-element"></div>
    </div>
    <div class="camera-info">
      <span class="camera-name">{{ camera.deviceName }}
        <el-dropdown @command="handleCommand" v-if="permission">
          <span class="dropdown-link">
            <i class="el-icon-arrow-down el-icon--right" ></i>
          </span>
          <el-dropdown-menu slot="dropdown">
            <el-dropdown-item command="modify">编辑</el-dropdown-item>
            <el-dropdown-item command="delete">删除</el-dropdown-item>
          </el-dropdown-menu>
        </el-dropdown>
      </span>
    </div>
  </div>
</template>
<script>
import EZUIKit from 'ezuikit-js';
import { userRequest } from "@/network/request";


export default {
  inject: ['reload'],
  // components: {Cameralook},
  name: "CameraComponents",
  props: {
    id: {
      type: String
    },
    permission: {
      type: Boolean
    },
    camera: {
      type: Object
    },
    accessToken: {
      type: String
    }
  },
  data() {
    // console.log("data", this.id, this.camera.liveUrl, /^https:\/\//.test(this.camera.liveUrl));
    return {
      player: null,
      isUrl: /^https:\/\//.test(this.camera.liveUrl),
      dialogVisible: false,
      player: null,
      err: false,
      msg: ''

    };
  },
  watch: {
    camera: {
      handler(newVal, oldVal) {
        // console.log(this.dialogVisible, 'dialogVisible')
        this.redeploy()
      },

      deep: true
    },

    data() {
      this.isUrl = /^https:\/\//.test(this.camera.liveUrl);
    },

  },
  methods: {
    redeploy() {
      this.player.stop();
      this.err = false
      let that = this
      let liveId = this.camera.deviceSerial
      this.player.play({
        url: 'ezopen://open.ys7.com/' + liveId + '/1.live',

      })
    },
    closeSound() {
      var closeSoundPromise = this.player.closeSound();
      closeSoundPromise.then((data) => {
        console.log("声音关闭", data)
      })
    },
    handleCommand(command) {
      switch (command) {
        // case "rec":
        //   this.rec();
        //   break;
        case "modify":
          this.modify();
          break;
        case "delete":
          this.delete();
          break;
      }
    },
    delete() {
      this.$confirm('确定要删除“' + this.camera.deviceName + '”摄像头吗？', '警告', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.deleteRequest();
      });
    },
    modify() {
      this.$prompt('摄像头名称（选填）', '编辑摄像头', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        inputValue: this.camera.deviceName,
        closeOnClickModal: false
      }).then(({ value }) => {
        value = value.trim();
        if (value === this.camera.deviceName) {
          this.$message({
            type: 'info',
            message: '监控名称未发生修改'
          });
        } else {
          this.modifyRequest(value);
        }
      });
    },
    modifyRequest(value) {
      userRequest({
        url: '/monitor/modifyMonitor',
        method: 'POST',
        data: {
          deviceSerial: this.camera.deviceSerial,
          newDeviceName: value
        }
      }).then(data => {
        if (!data || data.code !== 200) {
          this.$message({
            type: "error",
            message: data ? data.message : "未知错误"
          });
          return;
        }
        // console.log(value);
        this.$message({
          type: 'success',
          message: data.data
        });
        this.$emit('onMonitorChange', "reload");
      });
    },
    deleteRequest() {
      userRequest({
        url: '/monitor/deleteMonitor',
        method: 'DELETE',
        data: {
          deviceSerial: this.camera.deviceSerial
        }
      }).then(data => {
        if (!data || data.code !== 200) {
          this.$message({
            type: "error",
            message: data ? data.message : "未知错误"
          });
          return;
        }
        this.$message({
          type: 'success',
          message: data.data
        });
        this.reload();
      });
    },
    info(data) {//保证获取到新的accessTonken再重新执行播放
      this.err = true
      if (data.code == "10002") {
        // this.msg='摄像头授权过期，重新连接中。。。'
        Promise.all([
          this.$emit('getAccessToken')
        ]).then(res => {
          this.player.play({
            accessToken: window.localStorage.getItem("accessToken")
          });
        })
      } else if (data.code == "5404") {
        // this.err = true
        this.msg = '检测到摄像头离线，请联系医院管理员检查设备电源或网络状况，点击重试。'
        // console.log("摄像头离线，请联系医院管理员。")
        this.player.stop()
        return
      } else {
        // this.err = true
        this.msg = '摄像头连接超时，点击重试'
        // console.log("")
        this.player.stop()
        return
      }
    }
  },

  mounted() {
    let that = this
    Promise.resolve().then(() => {
      let liveId = this.camera.deviceSerial
      let height = window.getComputedStyle(this.$refs[`${liveId}`]).height;
      let width = window.getComputedStyle(this.$refs[`${liveId}`]).width;
      this.player = new EZUIKit.EZUIKitPlayer({
        id: this.camera.deviceSerial, // 视频容器ID
        accessToken: window.localStorage.getItem("accessToken"),//window.localStorage.getItem("accessToken")
        url: 'ezopen://open.ys7.com/' + liveId + '/1.live',
        template: 'security',
        // header: ["capturePicture", "save", "zoom"],
        // footer: [ "hd", "fullScreen"],
        audio: 0,//默认不播放声音
        autoplay: true,
        height: height,
        width: width,
        handleSuccess: (data) => {
         
            that.err = false
            that.closeSound()
          

        },
        handleError: data => this.info(data),
        // closeSoundCallBack: data => console.log("已经关闭声音", data),
        // // startSaveCallBack: data => console.log("开始录像回调", data),
        // // stopSaveCallBack: data => console.log("录像回调", data),
        // // capturePictureCallBack: data => console.log("截图成功回调", data),
        // fullScreenCallBack:(data)=>{
        //   that.player.closeSound().then(()=>{
        //     console.log(data,3333)
        // })
        // },
      })
    })


  }
};
// handleError: data => this.info(data),
//         // closeSoundCallBack: data => console.log("已经关闭声音", data),
//         // // startSaveCallBack: data => console.log("开始录像回调", data),
//         // // stopSaveCallBack: data => console.log("录像回调", data),
//         // // capturePictureCallBack: data => console.log("截图成功回调", data),
//         // fullScreenCallBack:(data)=>{
//         //   that.player.closeSound().then(()=>{
//         //     console.log(data,3333)
//         // })
//         // },
</script>

<style scoped>
.camera-container {
  position: relative;
}

.error-message {
  position: absolute;
  background-color: #000000;
  padding: 0 20px;
  width: calc(100% - 40px);
  height: calc(100% - 40px);
  z-index: 4;
  /* margin:0 auto; */
  text-overflow: ellipsis;
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
}

.video-container {
  position: relative;
  width: 100%;
  height: 0;
  padding-bottom: 56.25%;
  overflow: hidden;
}

.video-element {
  z-index: 1;
  object-fit: cover;
  position: absolute;
  top: -49px;
  right: 0;
  bottom: 0;
  left: 0;
}

.camera-info {
  position: relative;
  top: -35px;
  padding: 0;
  margin: 0 0 0 10px;
  width: 50%;
  z-index: 3;
}

.camera-name {
  color: white;
  position: absolute;
  font-size: 18px;
  font-weight: bold;
  width: calc(100% - 40px);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.dropdown-link {
  /* font-weight: 900; */
  cursor: pointer;
  color: #409EFF;
  font-size: 20px;

}
</style>

