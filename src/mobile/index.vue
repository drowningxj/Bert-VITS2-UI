<script>
  // 引入css
import 'vant/es/toast/style';
import 'vant/es/notify/style' 

import select_model from "@/components/infer/select_model.vue";
import status_card from "@/components/infer/status_card.vue";
import model_card from "@/components/infer/model_card.vue";
import colorTable from "@/color";
import axios from "axios";
import { showNotify, closeNotify,showFailToast } from 'vant';
import {CheckOutlined, CloseOutlined, DownloadOutlined, UploadOutlined} from "@ant-design/icons-vue";

const remote_url = "http://43.128.55.145:5000"

export default {
  name: "infer",
  components: {DownloadOutlined, CheckOutlined, CloseOutlined, UploadOutlined},
 
  data() {
    let model_audio_loading = false;
    const selectModel = (modelId) => {
      this.audio_pic = "img/live" + modelId + ".jpg";
      
       if(modelId == 0) {
        this.audio_title = '直播带货场景';
        this.audio_desc = '为直播内容创作者、主播提供全球最好的中文声音克隆技术';
        this.checked_model_id = '0';
        this.texts = "这不仅仅是一款茶，更是西湖文化的代表。龙井绿茶可追溯到几百年前。传说，一位僧人在西湖边发现了一条龙，而龙井茶正是在这片龙经过的地方长出来的。因此，龙井绿茶被誉为“龙出井而茶香溢”的传世佳茗。"
      }
      if(modelId == 1) {
        this.audio_title = '知识付费场景';
        this.audio_desc = '为知识付费、读书、博客APP等平台的老师们提供全球最好的中文声音克隆技术';
        this.checked_model_id = '1';
        this.texts = "你好，我是少年商学院创始人，张华。欢迎来到家长营公开课。我的声音都是机器合成的。语音语调附带情感。"
      }
    }
    return {
      checked_model_id: '0',
      audio_pic: 'img/live0.jpg',
      audio_title: '直播带货场景',
      audio_desc: '为直播内容创作者、主播提供全球最好的中文声音克隆技术',
      model_audio_loading,
      model_audio_data_src: "",
      model_audio_valid: false,
      model_audio_data_texts: "",
      selectModel,
      generate_audio_loading: false,
      delete_model_loading: false,
      colorTable: colorTable,
      models: [],
      registered_model: new Set(),
      texts: "",
      global_sdp_ratio: 0.2,
      global_sdp_ratio_selected: false,
      global_noise: 0.2,
      global_noise_selected: false,
      global_noisew: 0.9,
      global_noisew_selected: false,
      global_length: 1.0,
      global_length_selected: false,
      global_emotion: 0,
      global_emotion_selected: false,
      global_prompt: "",
      global_prompt_selected: false,
      global_use_reference_audio: false,
      global_speaker: "",
      global_speaker_selected: false,
      global_language: "",
      global_language_selected: false,
      select_all_models: {
        status: false,
        type: "primary",
        text: "模型全选"
      },
      select_all_choices: {
        status: false,
        type: "primary",
        text: "选项全选"
      },
      audio_dir: "Data",
      random_audio: {
        valid: false,
        data: {
          text: "",
          src: "",
          speaker: "",
          name: ""
        },
        audio_file: null
      },
      auto_translate: false,
      auto_split: false,
      auto_blind: false,
      random_speaker: false,
      random_language: "ZH",
      txt: [],
      card_width: 8
    }
  },
  mounted() {
    this.get_models()
  },
  methods: {
    async get_models() {
      // 获取所有已加载模型
      let url = `/models/info`
      try {
        //let response = await axios.get(url)
        let response = {
            data: {
              "0": {
                "config_path": "Data/lilith/config.json",
                "model_path": "models/lilith/G_2400.pth",
                "device": "cpu",
                "language": "ZH",
                'alias': "小莉",
                "spk2id": {
                  "lilith": 0
                },
                "id2spk": {
                  "0": "lilith"
                },
                "version": "2.2"
              },
              "1": {
                "config_path": "Data/zhanghua/config.json",
                "model_path": "models/zhanghua/G_27400.pth",
                "device": "cpu",
                "language": "ZH",
                'alias': "张老师",
                "spk2id": {
                  "zhanghua": 1
                },
                "id2spk": {
                  "1": "zhanghua"
                },
                "version": "2.2"
              }
            },
            status: 200
          };

        if (response.status === 200) {
          let data = response.data
          // console.log(data)
          let loaded_model = new Set()
          for (let key in data) {
            loaded_model.add(key)
          }
          for (let model_id in data) {
            // 注册新卡片
            if (!this.registered_model.has(model_id)) {
              let spk2id = data[model_id]["spk2id"]
              let speakers = []
              for (let spk in spk2id) {
                speakers.push(spk)
              }
              this.models.push({
                id: model_id,
                name: data[model_id]["model_path"],
                device: data[model_id]["device"],
                speakers: speakers,
                speaker_name: speakers[0],
                speaker_alias: data[model_id]["alias"],
                sdp_ratio: 0.2,
                noise: 0.2,
                noisew: 0.9,
                length: 1,
                language: data[model_id]["language"],
                emotion: 0,
                prompt: "",
                selected: false,
                audio: {
                  loading: false,
                  valid: false,
                  data: {
                    texts: "",
                    src: ""
                  }
                },
                visible: true,
                version: data[model_id]["version"]
              })
              this.registered_model.add(model_id)
            }
          }
          for (let model_id of this.registered_model) {
            // 删除不存在的模型
            if (!loaded_model.has(model_id)) {
              this.registered_model.delete(model_id)
              // 从列表中弹出所有符合项
              this.models = this.models.filter(
                  model => model.id !== model_id
              )
            }
          }
        }
      } catch (exception) {
        console.log("获取已加载模型失败", exception)
      }
    },

    async infer_audio(texts, model, auto_split) {
      // 推理指定模型
      let url = remote_url + `/voice`
      let params = {
        model_id: parseInt(model.id),
        speaker_name: model.speaker_name,
        sdp_ratio: model.sdp_ratio,
        noise: model.noise,
        noisew: model.noisew,
        length: model.length,
        language: model.language,
        auto_translate: this.auto_translate,
        auto_split: auto_split,
        emotion: model.version === "2.1" ? model.emotion : model.prompt
      }
      let formData = new FormData()
      formData.append("text", texts)
      try {
        this.model_audio_loading = true
        let response = await axios.post(url, formData, {
          params: params,
          responseType: "blob"
        })
        if (response.status === 200) {
          let data = response.data
          this.model_audio_loading = false
          this.model_audio_data_src = URL.createObjectURL(data)
          this.model_audio_valid = true
          this.model_audio_data_texts = texts
        } else {
          this.model_audio_loading = false
        }
      } catch (error) {
        this.model_audio_loading = false
        console.error("推理失败", error)
      }
    },

    async infers() {
      // 推理所有选中模型
      // auto_split :是否自动切分
      // blind 是否开启盲盒
      if (this.auto_blind) {
        // 遮盖
        for (let model of this.models.values()) {
          if (model.selected) {
            model.visible = false
          }
          model.audio.valid = false
        }
        // 打乱 Fisher-Yates Shuffle
        let indexs = this.models.map((model, index, ms) => model.selected ? index : -1).filter(id => id !== -1)
        for (let i = indexs.length - 1; i >= 0; i--) {
          let j = Math.floor(Math.random() * (i + 1))
          let left = indexs[i]
          let right = indexs[j]
          let temp = this.models[right]
          this.models[right] = this.models[left]
          this.models[left] = temp
        }
      }
      
      for (let model of this.models.values()) {
        // 消除上次的音频
        
        console.log(this.checked_model_id + "model.id " + model.id)
        if (!this.texts || this.texts.trim() === '') {
          showFailToast('请输入文字');
          return;
        }
        this.generate_audio_loading = true
        model.audio.valid = false
        if (model.id === this.checked_model_id) {
          showNotify({message : model.speaker_alias + " 音频生成中" , type: 'success'});
          await this.infer_audio(this.texts, model, this.auto_split)
        }
      }
      this.generate_audio_loading = false
    },

    all_models_button() {
      if (this.select_all_models.status === false) {
        this.select_all_models.status = true
        this.select_all_models.type = "default"
        this.select_all_models.text = "取消模型全选"
        for (let model of this.models.values()) {
          model.selected = true
        }
      } else {
        this.select_all_models.status = false
        this.select_all_models.type = "primary"
        this.select_all_models.text = "模型全选"
        for (let model of this.models.values()) {
          model.selected = false
        }
      }
    },

    get_text(file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        this.texts = e.target.result
      };
      reader.readAsText(file)
      return false
    },

    get_audio(file) {
      if (file) {
        this.random_audio.audio_file = file
        console.log(this.random_audio.audio_file)
        this.random_audio.data.src = URL.createObjectURL(file)
        this.random_audio.valid = true
        this.random_audio.data.name = file.name
      }
    }
  },
  computed: {
    allSpeakers() {
      let Speakers = new Set()
      for (let model of this.models.values()) {
        for (let spk of model.speakers) {
          if (!Speakers.has(spk)) {
            Speakers.add(spk)
          }
        }
      }
      return Speakers
    }
  }
}
</script>
<template>

  <div>
    <van-nav-bar
      title="全球最好的中文声音克隆"
      @click-left="onLeftClick"
      fixed
      placeholder
      class="wechat-navbar"
    />
    <!-- 页面内容 -->
    <div class="page-content">
      <van-radio-group v-model="checked_model_id">
  <van-cell-group inset>
    <van-cell title="直播带货-小莉" clickable @click="selectModel('0')">
      <template #right-icon>
        <van-radio name="0"  />
      </template>
    </van-cell>
    <van-cell title="育儿专家-张老师" @click="selectModel('1')" clickable >
      <template #right-icon>
        <van-radio name="1" />
      </template>
    </van-cell>
  </van-cell-group>
</van-radio-group>

  <van-card
      :desc="audio_desc"
      :title="audio_title"
    >
  <template #thumb>
      <van-image ref="audioCover" id="audioCover" :src="audio_pic" fit="cover" />
  </template>
    <template #tags>
      <van-field type="textarea" v-model="texts" v-bind:maxlength="100" placeholder="请输入文本(最大100字)" :rows="4"/>
    </template>
  </van-card>

  <!-- 音频相关 -->

  <van-row>
    <van-col :spinning=model_audio_loading />
    <van-col  :style="{opacity: model_audio_valid? 1: 0}" size="large">

      <audio :src="model_audio_data_src" controls></audio>
    </van-col>
  </van-row>

<van-action-bar>
  <van-action-bar-icon icon="chat-o" text="定制声音" />
  <van-action-bar-button  type="primary" @click="infers(false)" :loading="generate_audio_loading" text="生成音频" />
</van-action-bar>

    </div>
  </div>

</template>



<style scoped>
::v-deep .van-nav-bar {
  justify-content: space-between;
  align-items: center;
  background-color: #ffffff;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
}

::v-deep .van-action-bar-icon {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 12px;
}

::v-deep .van-action-bar-icon__icon {
  font-size: 24px;
}

::v-deep .van-action-bar-icon__text {
  margin-top: 4px;
  font-size: 12px;
  color: #333333;
}



.page-content {
  padding-top: 44px; /* 保证内容在顶部栏下方 */
  /* 这里可以添加其他样式，根据你的需求进行调整 */
}
.goods {
  padding-bottom: 50px;

  &-swipe {
    img {
      width: 100%;
      display: block;
    }
  }

  &-title {
    font-size: 16px;
  }

  &-price {
    color: #f44;
  }

  &-express {
    color: #999;
    font-size: 12px;
    padding: 5px 15px;
  }

  &-cell-group {
    margin: 15px 0;

    .van-cell__value {
      color: #999;
    }
  }

  &-tag {
    margin-left: 5px;
  }

}
</style>
