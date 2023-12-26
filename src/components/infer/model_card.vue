<script>
import axios from "axios";
import {DownloadOutlined} from '@ant-design/icons-vue';
import {useTimer} from "vue-timer-hook";
    
export default {
  name: "model_card",

  props: ["name", "speakers", "model"],
  
  data() {
    let showPickerSpeaker = false;
    let showPickerLan = false;
    let showSpeackerName = "";
    let modelId = -1;
    const onConfirmSpeaker = ({ selectedOptions }) => {
      this.showPickerSpeaker = false;
      this.showSpeackerName = selectedOptions[0].text;
      this.modelId = selectedOptions[0].value;
      console.log("modelId:" + this.modelId)
      console.log("showSpeackerName:" + this.showSpeackerName)
      //modelId 复制给 model.id
      this.model.speaker_name = selectedOptions[0].text;
      this.model.language = "ZH"
    };

    const onConfirmLan = ({ selectedOptions }) => {
      this.showPickerLan = false;
      this.model.language = selectedOptions[0].text
    };
    return {
      speakerOptions : [
        { text: 'lilith', value: '0' },
      ],
      lanOptions : [
        { text: 'ZH', value: 'ZH' },
      ],
      onConfirmSpeaker,
      showPickerSpeaker,
      onConfirmLan,
      showPickerLan,
      time: new Date()
    }
  },
  mounted() {
    this.speaker_name = this.speakers[0]
  },
  components: {
    DownloadOutlined
  },

  methods: {
    async delete_model() {
      let url = `/models/delete`
      let params = {
        model_id: parseInt(this.model.id)
      }
      try {
        await axios.get(url, {params: params})
      } catch (error) {
        console.error("模型卸载失败", error)
      }
    },


    getName(text, name, speaker_name) {
      let name2 = name.replace(/^Data\\|^Data\//, '').replace(/models\\|models\//, '')
      return text.substring(0, 10) + "@" + name2 + "@" + speaker_name + ".wav"
    }
  }
}

</script>

<template>
  <van-cell>
    <template #extra>
      <a-switch v-model:checked="model.selected"></a-switch>
    </template>
    <van-row :style="{opacity:model.visible?1:0}">
      <van-col>
        <van-field readonly :placeholder=model.speaker_name />
      </van-col>
    </van-row>

    <van-row>
      <van-col>
        <!-- 音频相关 -->
        <van-row>
          <van-col>
            <van-cell  :spinning=model.audio.loading>
              <van-cell  :style="{opacity: model.audio.valid? 1: 0}" size="large">
                <audio :src="model.audio.data.src" controls></audio>
              </van-cell>
            </van-cell>
          </van-col>
        </van-row>
      </van-col>
    </van-row>


  </van-cell>
</template>

<style scoped>
</style>