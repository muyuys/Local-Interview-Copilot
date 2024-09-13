<template>
  <div class="homeview_container">
    <div class="center_container">
      <div class="box">
        <div class="func_desc">
          <i class="el-icon-microphone"></i>
          Speech Recognition Results
        </div>
        <div v-if="!currentText" style="color: gray">No Content</div>
        <div class="asr_content">{{ currentText }}</div>
        <div class="single_part_bottom_bar">
          <el-button icon="el-icon-delete" :disabled="!currentText" @click="clearASRContent">
            Clear Text
          </el-button>
        </div>
      </div>
      <div class="box" style="border-left: none;">
        <div class="func_desc">
          <i class="el-icon-s-custom"></i>
          GPT Answer
        </div>
        <LoadingIcon v-show="show_ai_thinking_effect" />
        <div class="ai_result_content">{{ ai_result }}</div>
        <div class="single_part_bottom_bar">
          <el-button icon="el-icon-thumb" @click="askCurrentText" :disabled="!isGetGPTAnswerAvailable">
            Ask GPT
          </el-button>
          <el-button @click="clearAnswer">
            Clear Answer
          </el-button>
        </div>
      </div>
    </div>
    <div class="title_function_bar">
      <el-button type="success" @click="startCopilot" v-show="state === 'end'" :loading="copilot_starting"
        :disabled="copilot_starting">Start Copilot
      </el-button>
      <el-button :loading="copilot_stopping" @click="userStopCopilot" v-show="state === 'ing'">Stop Copilot
      </el-button>
      <MyTimer ref="MyTimer" />
    </div>

  </div>
</template>

<script>
import Assert from "assert-js"
import LoadingIcon from "@/components/LoadingIcon.vue";
import MyTimer from "@/components/MyTimer.vue";
import * as SpeechSDK from "microsoft-cognitiveservices-speech-sdk";
import OpenAI from "openai";
import config_util from "../utils/config_util"

export default {
  name: 'HomeView',
  props: {},
  computed: {
    isDevMode() {
      return (process.env.NODE_ENV === 'development')
    },
    isGetGPTAnswerAvailable() {
      // return this.state === "ing" && !!this.currentText
      return !!this.currentText

    }
  },
  components: { LoadingIcon, MyTimer },
  data() {
    return {
      currentText: "",
      state: "end", //end\ing
      ai_result: null,
      copilot_starting: false, //显示loading
      copilot_stopping: false,
      show_ai_thinking_effect: false,
      popStyle: {},
      record: null,
      socket: null,
      timeInte: null,
    }
  },
  async mounted() {
    console.log("mounted")
    if (this.isDevMode) {
      // this.currentText = demo_text
    }
    var that = this
    navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia;
    if (!navigator.getUserMedia) {
      alert('Your browser does not support audio input');
    } else {
      navigator.getUserMedia(
        { audio: true },
        function (mediaStream) {
          that.updateRecord(new Recorder(mediaStream));
          // this.record = new Recorder(mediaStream);
          // init(new Recorder(mediaStream));
        },
        function (error) {
          console.log(error);
        }
      );
    }
  },
  beforeDestroy() {
  },
  methods: {
    clearAnswer() {
      this.ai_result = ""
    },
    updateRecord(rec) {
      this.record = rec;
    },
    async askCurrentText() {
      const apiKey = localStorage.getItem("openai_key")
      // this.show_ai_thinking_effect = true
      const model = config_util.gpt_model()
      let content = this.currentText
      this.ai_result = ""
      this.show_ai_thinking_effect = true
      // const model = "qwen2:7b"
      const model_prompt = config_util.gpt_system_prompt()
      content = model_prompt + "\n" + content

      try {
        if (!apiKey) {
          throw new Error("You should setup an Open AI Key!")
        }
        const config = {
          apiKey: apiKey,
          dangerouslyAllowBrowser: true
        }
        if (apiKey === "ollama") {
          config['baseURL'] = 'http://localhost:11434/v1/'
        }
        if(model==="glm-4-flash"){
          config['baseURL'] = 'https://open.bigmodel.cn/api/paas/v4/'
        }
        console.log("config", config)
        const openai = new OpenAI(config)
        const stream = await openai.chat.completions.create({
          model: model,
          messages: [{ role: "user", content: content }],
          stream: true,
        });
        this.show_ai_thinking_effect = false

        for await (const chunk of stream) {
          const text = chunk.choices[0]?.delta?.content || ""
          this.ai_result += text
        }
      } catch (e) {
        this.show_ai_thinking_effect = false
        this.ai_result = "" + e
      }
    },
    async initWebSocket() {
      var queryParams = [];
      queryParams.push('lang=zh-CN');
      // if (sv) {
      //   queryParams.push('sv=1');
      // }
      var queryString = queryParams.length > 0 ? `?${queryParams.join('&')}` : '';
      this.socket = new WebSocket(`http://127.0.0.1:443/ws/transcribe${queryString}`);
      this.socket.binaryType = 'arraybuffer';

      var that = this
      this.socket.onopen = function (event) {
        console.log('WebSocket connection established');
        that.record.start();
        that.timeInte = setInterval(function () {
          if (that.socket.readyState === 1) {
            var audioBlob = that.record.getBlob();
            console.log('Blob size: ', audioBlob.size);

            // Read the Blob content for debugging
            var reader = new FileReader();
            reader.onloadend = function () {
              // console.log('Blob content: ', new Uint8Array(reader.result));
              that.socket.send(audioBlob);
              console.log('Sending audio data');
              that.record.clear();
            };
            reader.readAsArrayBuffer(audioBlob);
          }
        }, 500);
      };

      this.socket.onmessage = function (evt) {
        console.log('Received message: ' + evt.data);
        try {
          var resJson = JSON.parse(evt.data)
          var jsonResponse = JSON.stringify(resJson, null, 4);
          // debugger;
          that.currentText = that.currentText + "\n" + (resJson.data || 'No speech recognized');
        } catch (e) {
          console.error('Failed to parse response data', e);
          that.currentText = that.currentText + "\n" + evt.data;
          // transcriptionResult.textContent += "\n" + evt.data;
        }
      };

      this.socket.onclose = function () {
        console.log('WebSocket connection closed');
      };

      this.socket.onerror = function (error) {
        console.error('WebSocket error: ' + error);
      };
    },
    clearASRContent() {
      this.currentText = ""
    },
    async startCopilot() {
      await this.initWebSocket()
      this.state = "ing"
    },
    userStopCopilot() {
      // console.log('try to close WebSocket connection');
      if (this.socket) {
        // console.log('Closing WebSocket connection');
        this.socket.close();
        this.record.stop();
        clearInterval(this.timeInte);
      }
      this.state = "end"
    }
  }
}


const demo_text = `
Hello, thank you for coming for the interview. Please introduce yourself.

I'm Jhon, currently an undergraduate student majoring in Data Science at HK University. I am in the top 10% of my class, specializing in deep learning and proficient in web development. Additionally, I have contributed to several well-known open-source projects as mentioned in my resume.

Alright, let me ask you a machine learning question.

Sure, go ahead.

Can you explain the Hidden Markov Model?
`

async function sleep(ms) {
  return new Promise((resolve => setTimeout(resolve, ms)))
}

var Recorder = function (stream) {
  var sampleBits = 16; // Sample bits
  var inputSampleRate = 48000; // Input sample rate
  var outputSampleRate = 16000; // Output sample rate
  var channelCount = 1; // Single channel
  var context = new AudioContext();
  var audioInput = context.createMediaStreamSource(stream);
  var recorder = context.createScriptProcessor(4096, channelCount, channelCount);
  var audioData = {
    size: 0,
    buffer: [],
    inputSampleRate: inputSampleRate,
    inputSampleBits: sampleBits,
    clear: function () {
      this.buffer = [];
      this.size = 0;
    },
    input: function (data) {
      this.buffer.push(new Float32Array(data));
      this.size += data.length;
    },
    encodePCM: function () {
      var bytes = new Float32Array(this.size);
      var offset = 0;
      for (var i = 0; i < this.buffer.length; i++) {
        bytes.set(this.buffer[i], offset);
        offset += this.buffer[i].length;
      }
      var dataLength = bytes.length * (sampleBits / 8);
      var buffer = new ArrayBuffer(dataLength);
      var data = new DataView(buffer);
      offset = 0;
      for (var i = 0; i < bytes.length; i++, offset += 2) {
        var s = Math.max(-1, Math.min(1, bytes[i]));
        data.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
      }
      return new Blob([data], { type: 'audio/pcm' });
    }
  };

  this.start = function () {
    audioInput.connect(recorder);
    recorder.connect(context.destination);
  };

  this.stop = function () {
    recorder.disconnect();
  };

  this.getBlob = function () {
    return audioData.encodePCM();
  };

  this.clear = function () {
    audioData.clear();
  };

  function downsampleBuffer(buffer, inputSampleRate, outputSampleRate) {
    if (outputSampleRate === inputSampleRate) {
      return buffer;
    }
    var sampleRateRatio = inputSampleRate / outputSampleRate;
    var newLength = Math.round(buffer.length / sampleRateRatio);
    var result = new Float32Array(newLength);
    var offsetResult = 0;
    var offsetBuffer = 0;
    while (offsetResult < result.length) {
      var nextOffsetBuffer = Math.round((offsetResult + 1) * sampleRateRatio);
      var accum = 0, count = 0;
      for (var i = offsetBuffer; i < nextOffsetBuffer && i < buffer.length; i++) {
        accum += buffer[i];
        count++;
      }
      result[offsetResult] = accum / count;
      offsetResult++;
      offsetBuffer = nextOffsetBuffer;
    }
    return result;
  }

  recorder.onaudioprocess = function (e) {
    // console.log('onaudioprocess called');
    var resampledData = downsampleBuffer(e.inputBuffer.getChannelData(0), inputSampleRate, outputSampleRate);
    audioData.input(resampledData);
  };
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.homeview_container {
  display: flex;
  flex-direction: column;
}

.title_function_bar {
  margin-top: 10px;
  text-align: center;
  margin-bottom: 10px;
}

.center_container {
  flex-grow: 1;
  display: flex;
  height: calc(100vh - 150px);
}

.box {
  flex: 1;
  /* 设置flex属性为1，使两个div平分父容器的宽度 */
  border: 1px lightgray solid;
  /* 为了演示，添加边框样式 */
  padding: 10px;
  /* 为了演示，添加内边距 */
  white-space: pre-wrap;
  display: flex;
  flex-direction: column;
}

.asr_content {
  overflow-y: auto;
  flex-grow: 1;
}


.func_desc {
  text-align: center;
}

.single_part_bottom_bar {
  display: flex;
}

.single_part_bottom_bar>.el-button {
  flex-grow: 1;
}


.ai_result_content {
  overflow-y: auto;
  flex-grow: 1;
}

.popup-tag {
  position: absolute;
  display: none;
  background-color: #785448d4;
  color: white;
  padding: 5px;
  font-size: 15px;
  font-weight: bold;
  text-decoration: underline;
  cursor: pointer;
  -webkit-filter: drop-shadow(0 1px 10px rgba(113, 158, 206, 0.8));
}

.error_msg {
  color: red;
  text-align: center;
}
</style>
