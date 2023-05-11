<template>
  <v-app id="application">
    <v-app-bar app color="light-blue-accent-3">
      <v-toolbar-title>Youtube Analytics App</v-toolbar-title>
      <v-spacer></v-spacer>
    </v-app-bar>
    <v-container>
      <h1 class="text-center my-5">Youtube分析ツール</h1>
      <v-row justify="center">
        <v-col cols="12" md="8">
          <v-text-field
            v-model="channelUrl"
            label="チャンネルURL"
            outlined
            prepend-icon="mdi-youtube"
          ></v-text-field>
        </v-col>
        <v-col cols="auto">
          <v-btn height="50" color="blue-lighten-4" @click="getChannelId">
            <v-icon left>mdi-account-search</v-icon>
            チャンネルIDを取得
          </v-btn>
        </v-col>
      </v-row>

      <div v-if="channelId">
        <p class="text-center">チャンネルID: {{ channelId }}</p>
      </div>
    </v-container>
    <!-- ダイアログ -->
    <v-dialog v-model="dialog" width="80vw" scrollable>
      <v-card>
        <v-card-title class="text-h5 grey lighten-2" primary-title>
          <v-progress-circular
            v-if="!gptResponse"
            indeterminate
            color="primary"
          ></v-progress-circular>
          <span v-if="gptResponse">解析結果</span>
          <span v-else>解析中</span>
        </v-card-title>
        <v-card-text>
          <div v-if="gptResponse" class="scrolling-container">
            <pre v-html="addBreaksAfterPeriods(gptResponse)"></pre>
          </div>
        </v-card-text>
        <v-divider></v-divider>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" text @click="dialog = false"> 閉じる </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <!-- ダイアログ終わり -->

    <v-container>
      <v-row justify="center">
        <v-col cols="auto">
          <v-btn
            height="72"
            min-width="164"
            @click="callYoutbeApi"
            v-bind:disabled="activateBtnYoutube"
          >
            チャンネル情報の取得
          </v-btn>
        </v-col>
        <v-col cols="auto">
          <v-btn
            height="72"
            min-width="164"
            v-bind:disabled="activateBtn"
            @click="callGpts"
          >
            分析開始
          </v-btn>
        </v-col>
      </v-row>
    </v-container>
    <v-container>
      <div v-if="channelInfo">
        <v-container>
          <v-row>
            <v-col
              v-for="(item, index) in popularData"
              :key="index"
              cols="12"
              sm="6"
              md="4"
            >
              <v-card class="d-flex flex-column" height="100%">
                <v-img
                  class="flex-grow-1"
                  :src="item.thumbnails"
                  gradient="to bottom, rgba(0,0,0,.1), rgba(0,0,0,.5)"
                  cover
                >
                  <v-card-title class="text-bottom text-white">{{
                    item.title
                  }}</v-card-title>
                </v-img>

                <v-card-subtitle class="pt-4"> Number 10 </v-card-subtitle>

                <v-card-text class="flex-grow-1">
                  <div>再生数:{{ formatNumber(item.views) }}再生</div>
                  <div>いいね数:{{ formatNumber(item.likeCount) }}いいね</div>
                  <div>
                    コメント数:{{ formatNumber(item.commentCount) }}コメント
                  </div>
                </v-card-text>

                <v-card-actions>
                  <a
                    :href="`https://www.youtube.com/watch?v=${item.id}`"
                    target="_blank"
                  >
                    <v-btn color="orange"> 再生 </v-btn>
                  </a>
                </v-card-actions>
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </div>
    </v-container>
  </v-app>
</template>

<script setup lang="ts">
import axios from "axios";
import { Ref, ref } from "vue";
const popularData: Ref<PopularVideo[]> = ref([]);
const channelInfo: Ref<any> = ref();
const videoTitles: Ref<string[]> = ref();

const dialog: Ref<boolean> = ref(false);
const gptResponse: Ref<string | null> = ref(null);
const activateBtn: Ref<boolean> = ref(true);
const activateBtnYoutube: Ref<boolean> = ref(true);
const channelUrl: Ref<string> = ref("");
const channelId: Ref<string> = ref("");

interface PopularVideo {
  id: string;
  title: string;
  description: string;
  thumbnails: string;
  views: number;
  likeCount: number;
  commentCount: number;
}

//youtubeApiを呼ぶ
const callYoutbeApi = async (): Promise<void> => {
  try {
    videoTitles.value = [];
    const response = await axios.get(
      `${process.env.VUE_APP_BASE_URL}/api/youtube`
    );
    const popularVideos = response.data.popularVideos;
    channelInfo.value = response.data.channelInfo;
    popularData.value = popularVideos.map((video) => ({
      id: video.id,
      title: video.snippet.title,
      description: video.snippet.description,
      thumbnails: video.snippet.thumbnails.medium.url,
      views: video.statistics.viewCount,
      likeCount: video.statistics.likeCount,
      commentCount: video.statistics.commentCount,
      // その他のプロパティ
    }));
    for (let i = 0; i < popularData.value.length; i++) {
      let joinData;
      joinData = popularData.value[i].title;
      videoTitles.value.push(joinData);
    }
    activateBtn.value = false;
  } catch (error) {
    console.log(error);
  }
};
//chatGptを呼ぶ
const callGpts = async (): Promise<void> => {
  dialog.value = true;
  try {
    const response = await axios.post(
      `${process.env.VUE_APP_BASE_URL}/api/gpt`,
      {
        channelName: channelInfo.value.items[0].snippet.title,
        subscriberCount: channelInfo.value.items[0].statistics.subscriberCount,
        totalViewCount: channelInfo.value.items[0].statistics.viewCount,
        channelCreationDate: channelInfo.value.items[0].snippet.publishedAt,
        popularVideoTitles: videoTitles.value,
      }
    );
    gptResponse.value = response.data;
  } catch (error) {
    console.log(error);
  }
};
//表示単位
const formatNumber = (number: number) => {
  if (number >= 100000000) {
    return (number / 100000000).toFixed(1) + "億";
  } else if (number >= 10000) {
    return (number / 10000).toFixed(1) + "万";
  } else if (number >= 1000) {
    return (number / 1000).toFixed(1) + "千";
  } else {
    return number;
  }
};
let retryCount = 0;

//チャンネルIDの取得
const getChannelId = async (): Promise<void> => {
  if (urlCheck(channelUrl.value)) {
    try {
      const response = await axios.get(
        `${
          process.env.VUE_APP_BASE_URL
        }/api/getChannelId?channelUrl=${encodeURIComponent(channelUrl.value)}`,
        { timeout: 15000 }
      );

      if (!response.data.channelId) {
        if (retryCount >= 5) {
          alert("チャンネルが見つかりませんでした。");
          channelId.value = "";
          retryCount = 0; // リトライカウントをリセット
        } else {
          console.log("retrying");
          retryCount += 1;
          getChannelId();
        }
      } else {
        channelId.value = response.data.channelId;
        activateBtnYoutube.value = false;
        retryCount = 0; // リトライカウントをリセット
      }
    } catch (error) {
      alert(`エラーが発生しました: ${error}`);
      channelId.value = "";
    }
  }
};

const urlCheck = (url: string) => {
  if (typeof url !== "string") {
    return null;
  }
  const regex =
    // eslint-disable-next-line no-useless-escape
    /(?:https?:\/\/)?(?:www\.)?youtube\.com\//;

  if (regex.test(url)) {
    return true;
  } else {
    alert("正しいURLを入力してください");
    return null;
  }
};
//gptのレスポンスに改行を追加
const addBreaksAfterPeriods = (str) => {
  return str.replace(/。/g, "。<br>");
};
</script>
<style scoped>
li {
  list-style: none;
}
#application {
  background-color: #eeeeee;
}
#channel-url {
  background-color: white;
  border-radius: 10px;
  border: solid 2px #000;
  opacity: 0.5;
}
.scrolling-container {
  overflow-x: auto;
}
</style>
