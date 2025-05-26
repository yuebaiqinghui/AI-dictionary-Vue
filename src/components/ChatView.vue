<template>
  <div class="logo">
    <span @click="show = true" style="position: absolute;right: 5vw;top: 4vh;">切换语言</span>
    <nut-popup v-model:visible="show" position="right" :style="{ width: '30%', height: '100%' }">
      <nut-radio-group v-model="lan" direction="vertical"  style="margin: 20px;">
        <nut-radio label="Chinese" shape="button" style="margin-bottom: 20px;">汉语</nut-radio>
        <nut-radio label="English" shape="button" style="margin-bottom: 20px;">英语</nut-radio>
        <nut-radio label="Malay" shape="button">马来语</nut-radio>
      </nut-radio-group>
      <!-- <button @click="show = false" style="margin-top: 10px;">关闭</button> -->
    </nut-popup>
  </div>
  <div class="chat-container">
    <nut-radio-group v-model="val" direction="horizontal">
      <nut-radio label="character" shape="button">汉字</nut-radio>
      <nut-radio label="word" shape="button">词语</nut-radio>
      <nut-radio label="idiom" shape="button">成语</nut-radio>
      <nut-radio label="poetry" shape="button">古诗词</nut-radio>
    </nut-radio-group>
    <div class="input-container">
      <nut-searchbar v-model="userInput" @keydown.enter.prevent="sendMessage" placeholder="请输入查询内容...">
        <template #rightout> 
          <nut-button type="info" :loading="isLoading" @click="sendMessage">查询</nut-button>
        </template>
        <template #rightin>
          <Search2 />
        </template>
      </nut-searchbar>
      <!-- <textarea
        v-model="userInput"
        @keydown.enter.prevent="sendMessage"
        placeholder="请输入查询内容..."
        rows="1"
      ></textarea>
      <button @click="sendMessage" :disabled="isLoading">查询</button> -->
    </div>
    <div class="chat-messages" ref="messageContainer">
      <img v-if="messages.length === 0" src="../assets/bg.png" alt="" srcset="" style="width: 100%;opacity: 0.2;margin-top: 20%;">
      <div
        v-for="(message, index) in messages"
        :key="index"
        class="message"
        :class="message.role"
      >
        <!-- <div class="avatar" v-if="message.role === 'assistant'">
          <img src="https://api.dicebear.com/7.x/bottts/svg?seed=assistant" alt="Assistant" />
        </div> -->
        <div class="message-content">
          <div v-if="message.role === 'assistant'" class="typing-content">
            <span
              v-if="
                isThinking && index === messages.length - 1 && !message.content
              "
              class="thinking"
              >思考中...</span
            >
            <template v-else>
              <div v-if="index === messages.length - 1">
                <div v-if="currentReasoningContent" class="reasoning-content">
                  {{ currentReasoningContent }}
                </div>                
                <div v-html="md.render(currentTypingMessage)"></div>
              </div>
              <div v-else>
                <div v-if="message.reasoning_content" class="reasoning-content">
                  {{ message.reasoning_content }}
                </div>
                <div v-html="md.render(message.content)"></div>
              </div>
            </template>
          </div>
          <div v-else>
            {{ message.content }}
          </div>
        </div>
        <!-- <div class="avatar" v-if="message.role === 'user'">
          <img src="https://api.dicebear.com/7.x/avataaars/svg?seed=user" alt="User" />
        </div> -->
      </div>
    </div>
    
  </div>
</template>

<script>
import MarkdownIt from 'markdown-it';

export default {
  name: "ChatView",
  data() {
    return {
      md: new MarkdownIt(),
      messages: [],
      userInput: "",
      isLoading: false,
      isThinking: false,
      apiKey: process.env.VUE_APP_DEEPSEEK_API_KEY || "",
      apiEndpoint:
        "https://ark.cn-beijing.volces.com/api/v3/chat/completions",
      typingSpeed: 50, // 打字速度（毫秒）
      currentTypingMessage: null,
      currentReasoningContent: null,
      show: false,
      val: 'character', // 默认查询类型
      lan: 'Chinese', // 默认语言
    };
  },
  methods: {
    async sendMessage() {
      if (!this.userInput.trim() || this.isLoading) return;

      // 添加用户消息
      this.messages.push({
        role: "user",
        content: this.userInput.trim(),
      });

      this.userInput = "";
      this.isLoading = true;
      this.isThinking = true;

      // 添加助手消息占位
      const assistantMessage = {
        role: "assistant",
        content: "",
      };
      this.messages.push(assistantMessage);
      await this.$nextTick();
      this.scrollToBottom();
      try {
        this.currentTypingMessage = "";
        this.currentReasoningContent = "";
        await this.callDeepseekAPI(assistantMessage);
      } catch (error) {
        console.error("Error:", error);
        assistantMessage.content = "抱歉，发生了错误，请稍后重试。";
      } finally {
        this.isLoading = false;
        this.$nextTick(() => {
          this.scrollToBottom();
        });
      }
    },

    async callDeepseekAPI(assistantMessage) {
      if (!this.apiKey) {
        throw new Error("API密钥未配置");
      }
      const systemMessage = {
        role: "system",
        content: `
        Please follow the steps below to complete the task of translating and culturally adapting Chinese content into ${this.lan} based on the input type (${this.val}):

        1. Content Translation and Cultural Adaptation:
          - If translating to Chinese:
            - Use standard Mandarin Chinese for explanations(使用中文汉字输出解释).
            - Ensure the content is appropriate for students in China, avoiding overly complex or culturally specific terms.
            - Convert measurement units to the metric system, providing the conversion and reason in brackets.
          - If translating to English:
            - Choose vocabulary familiar to students in English-speaking countries.
            - Explain Chinese-specific terms with background info or analogies.
            - Adjust cultural elements to fit the target country's context, noting the reason in brackets.
          - If translating to Malay:
            - Use standard Bahasa Baku for explanations.
            - Ensure content aligns with Islamic norms, especially regarding values, beliefs, and customs.
            - Convert Chinese measurement units to international or commonly used local units, providing the conversion and reason in brackets.

        2. Handling Sensitive Content and Historical Events:
          - Stay neutral and objective regarding disputed territories, borders, or sovereignty. Mark such content with "[Needs manual review]".
          - Enhance cultural relevance by linking Chinese historical events to similar ones from the target country's history, specifying the reason for the addition in brackets.

        Ensure the output has no XML tags.
        `,
      }
      let messages = this.messages.map((msg, i) => ({
            role: msg.role,
            content: msg.content,
            prefix: this.messages.length == i + 1,
          }))
      messages.unshift(systemMessage)
      const response = await fetch(this.apiEndpoint, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${this.apiKey}`,
        },
        body: JSON.stringify({
          model: "deepseek-v3-250324",
          messages: messages, 
          stream: true,
        }),
      });

      if (!response.ok) {
        throw new Error(`API请求失败: ${response.status}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      try {
        this.isThinking = false;
        let reading = true;
        while (reading) {
          const { done, value } = await reader.read();
          if (done) {
            reading = false;
            break;
          }

          const chunk = decoder.decode(value);
          const lines = chunk.split("\n").filter((line) => line.trim() !== "");

          for (const line of lines) {
            if (line.startsWith("data: ")) {
              const data = line.slice(6);
              if (data === "[DONE]") continue;

              try {
                const parsed = JSON.parse(data);
                const content = parsed.choices[0]?.delta?.content || "";
                const reasoningContent =
                  parsed.choices[0]?.delta?.reasoning_content || "";
                if (content || reasoningContent) {
                  if (content) {
                    assistantMessage.content += content;
                    this.currentTypingMessage += content;
                    console.log("当前内容:", this.currentTypingMessage);
                  }
                  if (reasoningContent) {
                    if (!assistantMessage.reasoning_content) {
                      assistantMessage.reasoning_content = "";
                    }
                    assistantMessage.reasoning_content += reasoningContent;
                    this.currentReasoningContent += reasoningContent;
                  }
                  await this.$nextTick();
                  this.scrollToBottom();
                }
              } catch (e) {
                console.error("解析响应数据失败:", e);
              }
            }
          }
        }
      } finally {
        reader.releaseLock();
      }
    },

    scrollToBottom() {
      if (this.$refs.messageContainer) {
        this.$refs.messageContainer.scrollTop =
          this.$refs.messageContainer.scrollHeight;
      }
    },
  },
};
</script>

<style scoped>
.logo {
  width: 98vw;
  height: 8vh;
  background: url('../assets/logo1.png') no-repeat left center;
  background-size: contain;
}
.chat-container {
  display: flex;
  flex-direction: column;
  height: 92vh;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  padding-top: 0;
  box-sizing: border-box;
  position: fixed;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
  /* margin-top: 20px; */
  box-sizing: border-box;
}
.message {
  margin-bottom: 20px;
  display: flex;
  flex-direction: row;
  align-items: flex-start;
}

.message.user {
  /* flex-direction: row-reverse; */
  /* justify-content: flex-end; */
}

.avatar {
  width: 40px;
  height: 40px;
  margin: 0 10px;
  border-radius: 50%;
  overflow: hidden;
  flex-shrink: 0;
}

.avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.message-content {
  max-width: 90%;
  padding: 12px 16px;
  border-radius: 8px;
  background: white;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.reasoning-content {
  margin-top: 10px;
  padding: 8px 12px;
  background: #f8f9fa;
  border-left: 4px solid #007aff;
  font-style: italic;
  color: #495057;
  border-radius: 4px;
  font-size: 0.9em;
  margin-bottom: 10px;
}

.user .message-content {
  background: #007aff;
  color: white;
}

.input-container {
  display: flex;
  gap: 10px;
}

textarea {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  resize: none;
  font-family: inherit;
}

button {
  padding: 0 20px;
  background: #007aff;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.2s;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background: #0056b3;
}

.thinking {
  display: inline-block;
  animation: thinking 1.4s infinite;
  position: relative;
}

.thinking::after {
  content: "";
  animation: dots 1.4s linear infinite;
}

@keyframes thinking {
  0%,
  20% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 1;
  }
}

@keyframes dots {
  0% {
    content: "";
  }
  25% {
    content: ".";
  }
  50% {
    content: "..";
  }
  75% {
    content: "...";
  }
  100% {
    content: "";
  }
}
</style>