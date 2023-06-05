<script setup>
import { ref } from "vue";
import { Cloud } from "laf-client-sdk";
import { ElMessage } from "element-plus";
import axios from "axios";
import MarkdownIt from "markdown-it";
import hljs from "highlight.js";

const cloud = new Cloud({
  baseUrl: "https://d0cmwb.laf.dev",
  getAccessToken: () => '',
  timeout: 60000,
});

//消息列表
const list = ref([]);
//输入框绑定消息
const question = ref("");
//判断消息是否为空
const parentMessageId = ref("");
//获取消息loading
const loading = ref(false);
//判断设备
const isMobile = ref(false);
// 密令
const validText = ref("");
//登录dialog
const validIdentityVisible = ref(false);
//输入框提示
const placeholder = ref("请输入问题");
// 判断是否为移动设备
if (
  /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
    navigator.userAgent
  )
) {
  isMobile.value = true;
} else {
  placeholder.value = "请输入问题";
}

//发送消息
async function send() {
  //发送时验证登录
  if (localStorage.getItem("isAuth") == null) {
    validIdentityVisible.value = true
    return;
  }
  
  //判断是否回复
  if (loading.value) return;
  list.value.push({
    text: question.value,
    avatar: "/avatar.jpeg",
  });
  //定位页面位置
  setScreen();
  const message = question.value;
  question.value = "";
  loading.value = true;
  let res;
  if (message == "") {
    loading.value = false;
    list.value.push({
      text: "问题不能为空！",
      avatar: "/logo.jpg",
    });
    setScreen();
    return;
  }
  try {
    const md = new MarkdownIt({
      highlight: function (str, lang) {
        if (lang && hljs.getLanguage(lang)) {
          try {
            return (
              '<pre class="hljs"><code>' +
              hljs.highlight(lang, str, true).value +
              "</code></pre>"
            );
          } catch (__) {}
        }
        return (
          '<pre class="hljs"><code>' +
          md.utils.escapeHtml(str.replace(/[\r\n]+/g, "\n")) +
          "</code></pre>"
        );
      },
    });

    list.value.push({
      text: "",
      avatar: "/logo.jpg",
    });

    const obj = { message };
    if (parentMessageId.value) obj.parentMessageId = parentMessageId.value;

    axios({
      url: `https://d0cmwb.laf.dev/chat-api`,
      method: "post",
      data: obj,
      responseType: "text",
      headers: {},
      onDownloadProgress: function (progressEvent) {
        const xhr = progressEvent.event.target;

        let { responseText = '' } = xhr;
        

        let result = {}
        if (typeof responseText === 'string') {
          try {
            result = JSON.parse(responseText)
            parentMessageId.value = result.parentMessageId;
            list.value[list.value.length - 1].text = md.render(result.text);
            loading.value = false;
            setScreen();
          } catch (error) {}
        }
      },
    }).then(() => {});
    // 返回 id 并保存
  } catch (error) {
    console.log(error);
    loading.value = false;
    list.value.push({
      text: "出错了，请重试！",
      avatar: "/logo.jpg",
    });
    setScreen();
    return;
  }
}

// 验证密令
function valid() {
  if (validText.value === 'yang') {
    localStorage.setItem("isAuth", '1')
    validIdentityVisible.value = false
    return ElMessage({ message: "密令输入成功！", type: "success" });
  } else {
    return ElMessage({ message: "密令输入错误！", type: "error" });
  }
}

//定位页面位置
function setScreen() {
  setTimeout(() => {
    window.scrollTo(0, document.body.scrollHeight);
  }, 0);
}

//发送消息适配PC或phone
function handleEnter(e) {
  if (e.key === "Enter" && !isMobile.value && !e.shiftKey) {
    send();
  }
}
</script>

<template>
  <div class="page">
    <div v-show="!list.length" class="begintitle">
      <!-- 这里介绍 -->
    </div>

    <!-- 页面消息列表 -->
    <div id="myList">
      <div
        v-show="item.text"
        :class="item.type === 0 ? 'problemList' : 'answerList'"
        v-for="item in list"
      >
        <img class="listImg" :src="item.avatar" alt="" />
        <div v-html="item.text" class="listText"></div>
      </div>

      <div v-show="loading" class="answerList">
        <img class="listImg" src="/logo.jpg" alt="" />
        <img class="addin" src="/loading.gif" alt="" />
      </div>
    </div>

    <el-dialog v-model="validIdentityVisible" title="密令验证" center style="width:75%;">
      <el-input
        class="elinput"
        v-model="validText"
        placeholder="请输入密令"
      />
      <div class="loginbutbox">
        <el-button type="primary" @click="valid" class="loginbut">验证密令</el-button>
      </div>
    </el-dialog>


    <!-- 输入框 -->
    <div class="inputbox">
      <el-input
        v-model="question"
        v-bind:readonly="loading"
        maxlength="2000"
        tabindex="0"
        :autosize="{ minRows: 1, maxRows: 5 }"
        type="textarea"
        :placeholder="placeholder"
        @keypress="handleEnter"
      />

      <!-- 发送按钮小飞机 -->
      <div class="btn-send" @click="send">
        <div class="send-view" style="display: flex">
          <svg
            stroke="currentColor"
            fill="none"
            stroke-width="2"
            viewBox="0 0 24 24"
            class="h-4 w-4 mr-1"
            height="1.5em"
            width="1.5em"
          >
            <line x1="22" y1="2" x2="11" y2="13"></line>
            <polygon points="22 2 15 22 11 13 2 9 22 2"></polygon>
          </svg>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.page {
  position: relative;
  height: 100vh;
}

.defbut {
  position: fixed;
  right: 2px;
  bottom: 152px;
}
.btn-send {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: 10px;
  width: 48px;
  height: 32px;
  border-radius: 6px;
  color: rgba(0, 0, 0, 0.6);
  background: rgba(0, 0, 0, 0.1);
}
.btn-send:hover {
  cursor: pointer;
  opacity: 0.85;
}
.text {
  position: absolute;
  top: 50px;
  border: 1px solid #e5e5e5;
  height: 60px;
  padding: 10px;
  width: 90%;
}

#myList {
  max-width: 1000px;
  margin: 0 auto;
  padding: 0px 0 60px 0;
  overflow-x: hidden;
  overflow-y: auto;
}

.problemList {
  display: flex;
  padding: 0px 200px;
}

.answerList {
  position: relative;
  padding: 20px 18px;
  font-size: 15px;
  display: flex;
  overflow-x: auto;
  white-space: pre-wrap;
  border-top: 1px solid #e5e5e5;
  border-bottom: 1px solid #e5e5e5;
}

.listImg {
  margin-top: 5px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
}

.listText {
  margin-left: 20px;
  padding-top: 10px;
  width: 100%;
  white-space: pre-wrap;
}

.inputbox {
  position: fixed;
  bottom: 30px;
  left: 0;
  right: 0;
  margin: auto;
  width: 90%;
  max-width: 1000px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  background: #fff;
  border-radius: 8px;
}
.pre {
  white-space: pre-wrap;
  text-indent: 2em;
  word-wrap: break-word;
  padding: 0 0 10% 0;
}

.inputbox button {
  margin-left: 15px;
  width: 56px;
  height: 82%;
  border-radius: 6px;
  border: 0;
  background: silver;
  color: #333;
  font-size: 14px;
  outline: none;
}

.inputbox button:hover {
  cursor: pointer;
  opacity: 0.8;
}

.addin {
  margin: 10px 20px;
  width: 30px;
  height: 30px;
}

.steppingstone {
  width: 100%;
  height: 160px;
}

.amount {
  margin: 5px;
  width: 100px;
  height: 40px;
  line-height: 44px;
  text-align: center;
  font-size: 16px;
  color: #606266;
}

.begintitle {
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  /* padding: 50px 50px 30px 50px; */
}

.begintitle img {
  width: 200px;
}

.begintitle h1 {
  padding: 50px;
  font-size: 28px;
  font-weight: bold;
  text-align: center;
}

.exhibition {
  width: 80%;
  margin: auto;
  display: flex;
  justify-content: space-around;
}

.witem p {
  margin: auto;
  padding: 10px;
  margin-top: 15px;
  font-size: 16px;
  border-radius: 5px;
  text-align: center;
}

.witem h3 {
  padding: 15px;
  font-size: 20px;
  color: #606266;
  text-align: center;
}

textarea {
  border: none;
  resize: none;
  cursor: pointer;
  outline: none;
  overflow-y: hidden;
}
.head {
  position: fixed;
  width: 100%;
  padding: 0px 30px;
  line-height: 50px;
  justify-content: end;
  border-bottom: 1px solid #e5e5e5;
  background-color: #fff;
  z-index: 10;
}

.cardbox {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
}
.box-card {
  cursor: pointer;
  width: auto;
  margin: 20px;
  width: 160px;
  height: 140px;
  border: 1px solid #e6a23c;
  box-shadow: 0 16rpx 16rpx rgba(10, 16, 20, 0.24), 0 0 16rpx rgba(10, 16, 20, 0.12);
}

.boxCard {
  width: auto;
  margin: 20px;
  width: 160px;
  height: 140px;
  border: 1px solid #fff;
  box-shadow: 0 16rpx 16rpx rgba(10, 16, 20, 0.24), 0 0 16rpx rgba(10, 16, 20, 0.12);
}

.useNumber {
  width: 50%;
  height: 30px;
  margin: auto;
  text-align: center;
  font-size: 18px;
}
.money {
  width: 50%;
  height: 80px;
  line-height: 80px;
  margin: auto;
}
.sign {
  font-size: 16px;
  color: #e6a23c;
}
.number {
  width: 100%;
  text-align: center;
  font-size: 28px;
  font-weight: 700;
  color: #e6a23c;
}
.accountbox {
  margin: auto;
  margin-top: 10px;
  display: flex;
}
.cheerbox {
  margin-top: 20px;
  width: 100%;
  display: flex;
  justify-content: right;
}
.qrcode {
  margin: auto;
  width: 300px;
}
.qrcodeText {
  margin: 20px 0 0 0;
  font-size: 20px;
  width: 100%;
  text-align: center;
}
.cheer {
  width: 100px;
  height: 50px;
}

.loginbutbox {
  margin: auto;
  margin-top: 20px;
  width: 120px;
  height: 40px;
  line-height: 40px;
}
.loginbut {
  width: 120px;
  height: 40px;
}

.inputname {
  width: 80px;
  line-height: 40px;
}
.contentText {
}
.lafText {
  margin-top: 50px;
  color: #56585d;
}

:deep(.el-dialog__body) {
  padding: 0;
}

@media screen and (max-width: 600px) {
  .text {
    position: absolute;
    top: 30px;
    border: 1px solid #e5e5e5;
    height: 45px;
    padding: 10px;
    width: 80%;
  }
  .dialog {
    width: 100%;
  }
  :deep(.el-dialog) {
    width: 100%;
  }

  .useNumber {
    width: 100%;
  }
  .money {
    width: 100%;
    text-align: center;
  }
  .head {
    padding: 0px 5px;
  }
  .amount {
    margin: 5px;
    width: 70px;
    height: 40px;
    line-height: 44px;
    text-align: center;
    font-size: 12px;
    color: #606266;
  }
}
</style>
