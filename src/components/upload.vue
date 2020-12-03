<template>
  <div>
    <el-upload
      :show-file-list="false"
      class="upload-demo"
      drag
      action="#"
      multiple
      :http-request="handleChange"
    >
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">
        将文件拖到此处，或
        <em>点击上传</em>
      </div>
    </el-upload>
    <el-alert v-if="!isOnline" title="断网了，请检察网络，网络正常后将继续上传～" type="error"></el-alert>
    <el-progress v-if="progress" :percentage="progress" :status="progress === 100 ? 'success' : ''"></el-progress>

    <el-button type="primary" @click="handleToggleStop">暂停</el-button>

    <video v-if="['mp4', 'avi', 'mov', 'rmvb'].includes(suffix)" :src="url" controls />

    <img v-if="['png', 'gif', 'jpg', 'jpeg'].includes(suffix)" :src="url" />
  </div>
</template>

<script lang="ts">
import { ElUpload, ElProgress, ElButton, ElAlert } from "element-plus";
import {
  defineComponent,
  onMounted,
  onUnmounted,
  reactive,
  ref,
  toRefs,
} from "vue";
import SparkMD5 from "spark-md5";
import axios from "axios";
axios.defaults.baseURL = "//localhost:9999";

interface ResultData {
  code: number;
  path: string;
}

interface FileItem {
  name: string;
  chunk: File;
}

type bufferType = string | ArrayBuffer;

function getFileReader(file, type = "base64"): Promise<bufferType> {
  return new Promise((resolve) => {
    let reader: FileReader = new FileReader();
    if (type === "base64") {
      reader.readAsDataURL(file);
    } else if (type === "buffer") {
      reader.readAsArrayBuffer(file);
    }
    reader.onload = (e: ProgressEvent<FileReader>) => {
      resolve(e.target.result);
    };
  });
}

export default defineComponent({
  name: "upload",
  components: {
    [ElUpload.name]: ElUpload,
    [ElProgress.name]: ElProgress,
    [ElButton.name]: ElButton,
    [ElAlert.name]: ElAlert,
  },
  setup() {
    let hash = "";
    let abort = false;
    let curSplitArr = [];

    let state = reactive({
      progress: 0,
      url: "",
      suffix: "",
      isOnline: true,
    });

    const handleChange = async ({ file }) => {
      let buffer: bufferType = await getFileReader(file, "buffer");
      let size = file.size;
      let chunks = 100;
      let start = 0;
      let splitLen = size / chunks;
      let splitArr: FileItem[] = [];
      let spark = new SparkMD5.ArrayBuffer();
      spark.append(buffer);
      let fileMd5 = spark.end();
      hash = fileMd5;

      let suffix = /\.([0-9a-zA-Z]+)$/i.exec(file.name)[1];
      for (let i = 0; i < chunks; i++) {
        splitArr.push({
          name: `${fileMd5}_${i}.${suffix}`,
          chunk: file.slice(start, start + splitLen),
        });
        start += splitLen;
      }

      curSplitArr = splitArr;

      postAllChunks();
    };

    let postAllChunks = async () => {
      let complete = async () => {
        let resultData = await axios.get("/merge", {
          params: {
            hash: hash,
          },
        });
        let result: ResultData = resultData.data;
        if (result.code === 0) {
          let suffix = /\.([0-9a-zA-Z]+)$/.exec(result.path)[1];
          state.suffix = suffix;
          state.url = result.path;
        }
      };
      let i = 100 - curSplitArr.length;
      while (curSplitArr.length) {
        if (abort) {
          return Promise.reject();
        }
        await postFile(curSplitArr[0]);
        i++;
        state.progress = i;
      }
      complete();
    };

    const postFile = async (item: FileItem) => {
      let formData: FormData = new FormData();
      formData.append("name", item.name);
      formData.append("chunk", item.chunk);
      return axios
        .post("/upload", formData, {
          headers: { "Content-Type": "multipart/form-data" },
        })
        .then(() => {
          curSplitArr.shift();
        });
    };

    const handleToggleStop = () => {
      abort = !abort;
      if (!abort) {
        postAllChunks();
      }
    };

    const handleStop = () => {
      abort = true;
    };

    const HandleOnline = () => {
      abort = false;
      state.isOnline = true;
      postAllChunks();
    };

    const HandleOffline = () => {
      handleStop();
      state.isOnline = false;
    };

    onMounted(() => {
      window.addEventListener("online", HandleOnline, false);
      window.addEventListener("offline", HandleOffline, false);
    });

    onUnmounted(() => {
      window.removeEventListener("online", HandleOnline);
      window.removeEventListener("offline", HandleOffline);
    });

    let { progress, isOnline, url, suffix } = toRefs(state);

    return {
      isOnline,
      progress,
      url,
      suffix,
      handleChange,
      handleToggleStop,
    };
  },
});
</script>

<style scoped></style>
