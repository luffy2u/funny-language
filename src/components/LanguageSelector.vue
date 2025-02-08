<script setup lang="ts">
import { ref } from 'vue'
import ieltsData from '@/assets/IELTS_2.json'

const currentIndex = ref(0)
const data = ref(ieltsData)

const nextWord = () => {
  if (currentIndex.value < data.value.length - 1) {
    currentIndex.value++
  }
}

const prevWord = () => {
  if (currentIndex.value > 0) {
    currentIndex.value--
  }
}
</script>

<template>
  <div class="language-container">
    <div class="main-section">
      <div class="word-header">
        <div class="word-header-content">
          <h2>{{ data[currentIndex].content.word.wordHead }}</h2>
          <div class="phonetic">
            <span>美音: {{ data[currentIndex].content.word.content.usphone }}</span>
            <span class="divider"> | </span>
            <span>英音: {{ data[currentIndex].content.word.content.ukphone }}</span>
          </div>
          <div class="translations-header">
            <div v-for="(trans, index) in data[currentIndex].content.word.content.trans" :key="index" class="trans-item">
              <span class="pos">{{ trans.pos }}.</span>
              <span class="translation-text">{{ trans.tranCn }}</span>
            </div>
          </div>
        </div>
      </div>

      <div class="word-content">
        <div class="left-panel">
          <div class="word-block examples" v-if="data[currentIndex].content.word.content.sentence">
            <h3>例句：</h3>
            <div v-for="(sentence, index) in data[currentIndex].content.word.content.sentence.sentences" :key="index">
              <p class="example">{{ sentence.sContent }}</p>
              <p class="example-translation">{{ sentence.sCn }}</p>
            </div>
          </div>

          <div class="word-block phrases" v-if="data[currentIndex].content.word.content.phrase">
            <h3>短语：</h3>
            <div v-for="(phrase, index) in data[currentIndex].content.word.content.phrase.phrases" :key="index">
              <p class="phrase-text">{{ phrase.pContent }}</p>
              <p class="phrase-translation">{{ phrase.pCn }}</p>
            </div>
          </div>
        </div>

        <div class="right-panel">
          <div class="word-block syno" v-if="data[currentIndex].content.word.content.syno">
            <h3>同近词：</h3>
            <div v-for="(syno, index) in data[currentIndex].content.word.content.syno.synos" :key="index">
              <p>
                <span class="pos">{{ syno.pos }}.</span>
                <span class="tran">{{ syno.tran }}</span>
              </p>
              <p class="related-words">
                <span v-for="(hwd, idx) in syno.hwds" :key="idx">
                  {{ hwd.w }}{{ idx < syno.hwds.length - 1 ? '、' : '' }}
                </span>
              </p>
            </div>
          </div>

          <div class="word-block relWord" v-if="data[currentIndex].content.word.content.relWord">
            <h3>同根词：</h3>
            <div v-for="(rel, index) in data[currentIndex].content.word.content.relWord.rels" :key="index">
              <p>
                <span class="pos">{{ rel.pos }}.</span>
                <span v-for="(word, idx) in rel.words" :key="idx">
                  {{ word.hwd }} - {{ word.tran }}{{ idx < rel.words.length - 1 ? '；' : '' }}
                </span>
              </p>
            </div>
          </div>

          <div class="word-block anto" v-if="data[currentIndex].content.word.content.anto">
            <h3>反义词：</h3>
            <div v-for="(anto, index) in data[currentIndex].content.word.content.anto.antos" :key="index">
              <p>
                <span class="pos">{{ anto.pos }}.</span>
                <span class="tran">{{ anto.tran }}</span>
              </p>
              <p class="related-words">
                <span v-for="(hwd, idx) in anto.hwds" :key="idx">
                  {{ hwd.w }}{{ idx < anto.hwds.length - 1 ? '、' : '' }}
                </span>
              </p>
            </div>
          </div>

          <div class="word-block memory" v-if="data[currentIndex].content.word.content.remMethod">
            <h3>记忆方法：</h3>
            <div class="memory-content">
              {{ data[currentIndex].content.word.content.remMethod.val }}
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="controls">
      <button @click="prevWord" :disabled="currentIndex === 0">上一个</button>
      <span class="counter">{{ currentIndex + 1 }} / {{ data.length }}</span>
      <button @click="nextWord" :disabled="currentIndex === data.length - 1">下一个</button>
    </div>
  </div>
</template>

<style scoped>
.language-container {
  width: 100%;
  height: 100%;
  padding: 0;
  margin: 0;
  background: #f5f5f5;
  display: flex;
  flex-direction: column;
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
}

.main-section {
  flex: 1;
  width: 100%;
  display: flex;
  flex-direction: column;
  background: white;
  overflow: hidden;
}

.word-header {
  padding: 30px 20px;
  background: #4CAF50;
  color: white;
  flex-shrink: 0;
  display: flex;
  justify-content: center;
}

.word-header-content {
  text-align: center;
  max-width: 800px;
  width: 100%;
}

.word-header h2 {
  font-size: 42px;
  color: white;
  margin-bottom: 12px;
}

.phonetic {
  color: rgba(255, 255, 255, 0.9);
  font-size: 16px;
  margin-bottom: 20px;
}

.translations-header {
  display: flex;
  flex-direction: column;
  gap: 12px;
  align-items: center;
  margin-top: 20px;
}

.trans-item {
  display: flex;
  align-items: baseline;
  gap: 12px;
  justify-content: center;
  width: 100%;
}

.translations-header .pos {
  color: rgba(255, 255, 255, 0.8);
  font-size: 18px;
  font-weight: 500;
  min-width: 30px;
  text-align: right;
}

.translations-header .translation-text {
  color: white;
  font-size: 20px;
  font-weight: 500;
  text-align: left;
}

.word-content {
  flex: 1;
  display: flex;
  gap: 40px;
  overflow: hidden;
  position: relative;
  height: calc(100% - 60px);
}

.left-panel, .right-panel {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  height: 100%;
  scrollbar-width: thin;
  scrollbar-color: #4CAF50 #f5f5f5;
}

.left-panel {
  border-right: 1px solid #eee;
  padding-right: 40px;
}

.word-block {
  background: #f9f9f9;
  padding: 16px;
  border-radius: 6px;
  margin-bottom: 20px;
  border-left: 3px solid #4CAF50;
}

.word-block h3 {
  color: #2c3e50;
  font-size: 16px;
  margin-bottom: 12px;
  font-weight: bold;
}

.translations {
  margin-bottom: 30px;
}

.translations h3 {
  color: #2c3e50;
  font-size: 16px;
  margin-bottom: 12px;
  font-weight: bold;
}

.pos {
  color: #2c3e50;
  font-weight: bold;
  margin-bottom: 8px;
}

.translation {
  font-size: 18px;
  color: #333;
  margin: 8px 0;
}

.eng-translation {
  color: #666;
  font-style: italic;
  margin-bottom: 16px;
}

.example {
  color: #333;
  margin: 12px 0 8px;
}

.example-translation {
  color: #666;
  margin-bottom: 16px;
}

.phrase-text {
  color: #2c3e50;
  margin: 8px 0 4px 0;
}

.phrase-translation {
  color: #666;
  margin-left: 16px;
  margin-bottom: 12px;
}

.related-words {
  color: #666;
  margin-left: 20px;
  margin-bottom: 12px;
}

.memory-content {
  color: #666;
  line-height: 1.6;
}

.controls {
  padding: 20px;
  background: white;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: center;
  gap: 20px;
  flex-shrink: 0;
}

button {
  padding: 10px 24px;
  border: none;
  background: #4CAF50;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.counter {
  font-size: 16px;
  color: #666;
  line-height: 40px;
}

.left-panel::-webkit-scrollbar,
.right-panel::-webkit-scrollbar {
  width: 6px;
}

.left-panel::-webkit-scrollbar-track,
.right-panel::-webkit-scrollbar-track {
  background: #f5f5f5;
}

.left-panel::-webkit-scrollbar-thumb,
.right-panel::-webkit-scrollbar-thumb {
  background-color: #4CAF50;
  border-radius: 3px;
}

@media (max-width: 768px) {
  .word-header {
    padding: 20px;
  }

  .word-header h2 {
    font-size: 32px;
  }

  .translations-header {
    gap: 8px;
    margin-top: 16px;
  }

  .trans-item {
    gap: 8px;
  }

  .translations-header .pos {
    font-size: 16px;
    min-width: 25px;
  }

  .translations-header .translation-text {
    font-size: 18px;
  }

  .word-content {
    flex-direction: column;
    height: auto;
    overflow-y: auto;
  }

  .left-panel, .right-panel {
    height: auto;
    overflow: visible;
  }

  .main-section {
    overflow-y: auto;
  }

  .left-panel {
    border-right: none;
    padding-right: 0;
    margin-bottom: 20px;
  }
}
</style>
