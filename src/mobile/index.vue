<script>
import select_model from "@/components/infer/select_model.vue";
import status_card from "@/components/infer/status_card.vue";
import model_card from "@/components/infer/model_card.vue";
import colorTable from "@/color";
import axios from "axios";
import {CheckOutlined, CloseOutlined, DownloadOutlined, UploadOutlined} from "@ant-design/icons-vue";

const remote_url = "http://43.128.55.145:5000"

export default {
  name: "infer",
  components: {DownloadOutlined, model_card, status_card, select_model, CheckOutlined, CloseOutlined, UploadOutlined},
  data() {
    return {
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
                "spk2id": {
                  "lilith": 0
                },
                "id2spk": {
                  "0": "lilith"
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
      // 仅当使用参考音频且音频有效时生效
      if (this.global_use_reference_audio && this.random_audio.valid && this.random_audio.audio_file !== null) {
        formData.append("reference_audio", this.random_audio.audio_file)
      }
      try {
        model.audio.loading = true
        let response = await axios.post(url, formData, {
          params: params,
          responseType: "blob"
        })
        if (response.status === 200) {
          let data = response.data
          console.log(data)
          model.audio.loading = false
          model.audio.data.src = URL.createObjectURL(data)
          model.audio.valid = true
          model.audio.data.texts = texts
        } else {
          model.audio.loading = false
        }
      } catch (error) {
        model.audio.loading = false
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
      this.generate_audio_loading = true
      for (let model of this.models.values()) {
        // 消除上次的音频
        model.audio.valid = false
        if (model.selected) {
          await this.infer_audio(this.texts, model, this.auto_split)
        }
      }
      this.generate_audio_loading = false
    },

    all_choices_button() {
      if (this.select_all_choices.status === false) {
        this.select_all_choices.status = true
        this.select_all_choices.type = "default"
        this.select_all_choices.text = "取消选项全选"
        this.global_sdp_ratio_selected = true
        this.global_noise_selected = true
        this.global_noisew_selected = true
        this.global_length_selected = true
        this.global_emotion_selected = true
        this.global_prompt_selected = true
        this.global_speaker_selected = true
        this.global_language_selected = true
      } else {
        this.select_all_choices.status = false
        this.select_all_choices.type = "primary"
        this.select_all_choices.text = "选项全选"
        this.global_sdp_ratio_selected = false
        this.global_noise_selected = false
        this.global_noisew_selected = false
        this.global_length_selected = false
        this.global_emotion_selected = false
        this.global_prompt_selected = false
        this.global_speaker_selected = false
        this.global_language_selected = false
      }
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
    apply_global_setting() {
      for (let model of this.models.values()) {
        if (model.selected === true) {
          if (this.global_sdp_ratio_selected === true) {
            model.sdp_ratio = this.global_sdp_ratio
          }
          if (this.global_noise_selected === true) {
            model.noise = this.global_noise
          }
          if (this.global_noisew_selected === true) {
            model.noisew = this.global_noisew
          }
          if (this.global_length_selected === true) {
            model.length = this.global_length
          }
          if (this.global_emotion_selected === true) {
            model.emotion = this.global_emotion
          }
          if (this.global_prompt_selected === true) {
            model.prompt = this.global_prompt
          }
          if (this.global_speaker_selected === true) {
            if (model.speakers.includes(this.global_speaker)) {
              model.speaker_name = this.global_speaker
            }
          }
          if (this.global_language_selected === true) {
            model.language = this.global_language
          }
        }
      }
    },

    async get_random_audio() {
      // 获取随机音频
      this.random_audio.valid = false
      let url = `/tools/random_example`
      let params = {
        "language": this.random_language,
        "root_dir": this.audio_dir
      }
      if (this.random_language === "随机") {
        params = {
          "root_dir": this.audio_dir
        }
      }
      try {
        let response = await axios.get(url, {params: params})
        if (response.status === 200) {
          let data = response.data
          console.log(data)
          if (data["status"] !== 0) {
            this.texts = data["detail"]
            return
          }
          data = data["Data"]
          this.random_audio.data.speaker = data["speaker"]
          this.random_audio.data.text = data["text"]
          this.texts = data["text"]
          this.random_audio.data.src = `tools/get_audio?path=${data["audio"]}`
          // 下载音频
          try {
            const response = await axios.get(remote_url + `/tools/get_audio?path=${data["audio"]}`, {responseType: 'blob'})
            const blob = new Blob([response.data])
            this.random_audio.audio_file = new File([blob], "random.mp3")
            this.random_audio.valid = true
            this.random_audio.data.name = ""
            console.log(this.random_audio.audio_file)
          } catch (error) {
          }
        }
      } catch (error) {
        console.error(`音频获取失败`, error)
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
  <a-row justify="start" :gutter="[16,16]" align="bottom">
    <!--
    <a-divider :style="{'background-color':colorTable[5], 'height':'2px'} "/>
    -->
    <a-col :span="12">
      <a-row justify="start" :gutter="[16,16]" align="bottom">
        <a-col :span="24">
          <select_model></select_model>
        </a-col>
        <!-- 文本栏 -->
        <van-cell-group>
          <a-card title="输入文本内容">
            <a-row justify="start" :gutter="[16,16]">
              <van-field type="textarea" v-model:value="texts" placeholder="请输入文本" :rows="7"/>
              <a-col :span="24">
                <a-space :size="24">
                  <a-button type="primary" @click="infers(false)" :loading="generate_audio_loading">生成音频</a-button>
                </a-space>
              </a-col>
            </a-row>
          </a-card>
        </van-cell-group>

        <a-col :span="24">
          <a-card>
            <a-row justify="start" :gutter="[16,16]">
              <a-col :span="24">
                <a-space :size="24">
                  <a-button @click="get_random_audio"> 随机音频示例</a-button>
                  <a-tooltip title="随机音频搜索目录">
                    <a-input v-model:value="audio_dir" placeholder="随机音频目录"/>
                  </a-tooltip>
                  <a-tooltip title="随机音频搜索语言">
                    <a-select
                        ref="select"
                        v-model:value="random_language"
                        style="width: 100%"
                        @focus="focus"
                    >
                      <a-select-option value="ZH"></a-select-option>
                      <a-select-option value="JP"></a-select-option>
                      <a-select-option value="EN"></a-select-option>
                      <a-select-option value="随机"></a-select-option>
                    </a-select>
                  </a-tooltip>
                  <a-upload
                      :show-upload-list="false"
                      :multiple="false"
                      :before-upload="get_audio"
                      accept="audio/*"
                  >
                    <a-button>
                      <template #icon>
                        <UploadOutlined/>
                      </template>
                      上传音频
                    </a-button>
                  </a-upload>
                  <a-button @click="global_use_reference_audio=!global_use_reference_audio">
                    {{ global_use_reference_audio ? "取消emo音频" : "作为emo音频" }}
                  </a-button>
                  <a-switch v-model:checked="global_use_reference_audio"></a-switch>
                </a-space>
              </a-col>
              <a-col :span="24">
                <a-space size="large" :style="{opacity:random_audio.valid?1:0}">
                  <audio :src="random_audio.data.src" controls></audio>
                  <a-button :href="random_audio.data.src"
                            :download="  random_audio.data.name===''? random_audio.data.speaker + ': ' + texts + '.wav' : random_audio.data.name">
                    <template #icon>
                      <DownloadOutlined/>
                    </template>
                    下载音频
                  </a-button>
                </a-space>
              </a-col>
            </a-row>
          </a-card>
        </a-col>
      </a-row>
    </a-col>
  </a-row>

  <van-action-bar>
      <van-action-bar-button type="danger" @click="sorry">
        生成音频
      </van-action-bar-button>
    </van-action-bar>
</template>


<style lang="less">
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
