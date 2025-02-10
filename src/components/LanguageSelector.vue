<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import ieltsData from '@/assets/IELTS_2.json'

// IndexedDB 工具类
const DB_NAME = 'wordsDB'
const DB_VERSION = 1

class IndexedDBHelper {
  private db: IDBDatabase | null = null

  async init() {
    return new Promise<void>((resolve, reject) => {
      const request = indexedDB.open(DB_NAME, DB_VERSION)

      request.onerror = () => reject(request.error)

      request.onsuccess = () => {
        this.db = request.result
        resolve()
      }

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result
        // 创建收藏夹存储
        if (!db.objectStoreNames.contains('favorites')) {
          db.createObjectStore('favorites', { keyPath: 'id' })
        }
        // 创建最后浏览位置存储
        if (!db.objectStoreNames.contains('lastPosition')) {
          db.createObjectStore('lastPosition', { keyPath: 'id' })
        }
      }
    })
  }

  async getFavorites(): Promise<Set<string>> {
    if (!this.db) throw new Error('Database not initialized')
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['favorites'], 'readonly')
      const store = transaction.objectStore('favorites')
      const request = store.get('favorites')

      request.onsuccess = () => {
        const favorites = request.result?.wordIds || []
        resolve(new Set(favorites))
      }
      request.onerror = () => reject(request.error)
    })
  }

  async saveFavorites(favorites: Set<string>) {
    if (!this.db) throw new Error('Database not initialized')
    return new Promise<void>((resolve, reject) => {
      const transaction = this.db!.transaction(['favorites'], 'readwrite')
      const store = transaction.objectStore('favorites')
      const request = store.put({
        id: 'favorites',
        wordIds: Array.from(favorites)
      })

      request.onsuccess = () => resolve()
      request.onerror = () => reject(request.error)
    })
  }

  async getLastWordId(): Promise<string | null> {
    if (!this.db) throw new Error('Database not initialized')
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction(['lastPosition'], 'readonly')
      const store = transaction.objectStore('lastPosition')
      const request = store.get('lastWord')

      request.onsuccess = () => {
        resolve(request.result?.wordId || null)
      }
      request.onerror = () => reject(request.error)
    })
  }

  async saveLastWordId(wordId: string) {
    if (!this.db) throw new Error('Database not initialized')
    return new Promise<void>((resolve, reject) => {
      const transaction = this.db!.transaction(['lastPosition'], 'readwrite')
      const store = transaction.objectStore('lastPosition')
      const request = store.put({
        id: 'lastWord',
        wordId
      })

      request.onsuccess = () => resolve()
      request.onerror = () => reject(request.error)
    })
  }
}

const db = new IndexedDBHelper()
const currentIndex = ref(0)
const data = ref(ieltsData)
const favorites = ref(new Set<string>())
const showFavorites = ref(false)
const favoritesList = ref<any[]>([])
const showWordList = ref(false)

// 添加搜索相关的响应式变量
const favoriteSearchKey = ref('')
const wordListSearchKey = ref('')
const filteredFavorites = ref<any[]>([])
const filteredWordList = ref<any[]>([])

const showMoreMenu = ref(false)

// 添加点击外部关闭菜单的处理函数
const closeMoreMenu = (event: MouseEvent) => {
  const target = event.target as HTMLElement
  // 如果点击的不是菜单或菜单内的元素，则关闭菜单
  if (!target.closest('.more-wrapper') && showMoreMenu.value) {
    showMoreMenu.value = false
  }
}

// 初始化数据
onMounted(async () => {
  try {
    await db.init()
    // 加载收藏数据
    favorites.value = await db.getFavorites()
    // 加载上次查看的单词位置
    const lastWordId = await db.getLastWordId()
    if (lastWordId) {
      const index = data.value.findIndex(word => word.content.word.wordId === lastWordId)
      if (index !== -1) {
        currentIndex.value = index
      }
    }
    // 初始化单词列表搜索结果
    filteredWordList.value = data.value
    document.addEventListener('click', closeMoreMenu)
  } catch (error) {
    console.error('Failed to initialize database:', error)
  }
})

// 在组件卸载时移除事件监听
onUnmounted(() => {
  document.removeEventListener('click', closeMoreMenu)
})

const toggleFavorite = async (wordId: string) => {
  if (favorites.value.has(wordId)) {
    favorites.value.delete(wordId)
  } else {
    favorites.value.add(wordId)
  }
  // 保存到 IndexedDB
  try {
    await db.saveFavorites(favorites.value)
  } catch (error) {
    console.error('Failed to save favorites:', error)
  }
}

const isFavorite = (wordId: string) => favorites.value.has(wordId)

const saveCurrentWordId = async () => {
  try {
    const currentWordId = data.value[currentIndex.value].content.word.wordId
    await db.saveLastWordId(currentWordId)
  } catch (error) {
    console.error('Failed to save last word position:', error)
  }
}

// 修改 nextWord 和 prevWord 函数
const nextWord = async () => {
  if (currentIndex.value < data.value.length - 1) {
    currentIndex.value++
    await saveCurrentWordId()
  }
}

const prevWord = async () => {
  if (currentIndex.value > 0) {
    currentIndex.value--
    await saveCurrentWordId()
  }
}

// 监听收藏夹搜索
const searchFavorites = () => {
  if (!favoriteSearchKey.value) {
    filteredFavorites.value = favoritesList.value
    return
  }
  const key = favoriteSearchKey.value.toLowerCase()
  filteredFavorites.value = favoritesList.value.filter(word =>
    word.content.word.wordHead.toLowerCase().includes(key) ||
    word.content.word.content.trans[0].tranCn.includes(key)
  )
}

// 监听单词表搜索
const searchWordList = () => {
  if (!wordListSearchKey.value) {
    filteredWordList.value = data.value
    return
  }
  const key = wordListSearchKey.value.toLowerCase()
  filteredWordList.value = data.value.filter(word =>
    word.content.word.wordHead.toLowerCase().includes(key) ||
    word.content.word.content.trans[0].tranCn.includes(key)
  )
}

// 修改 toggleFavoritesList
const toggleFavoritesList = () => {
  if (showWordList.value) {
    showWordList.value = false
  }
  showFavorites.value = !showFavorites.value
  if (showFavorites.value) {
    favoritesList.value = data.value.filter(word =>
      favorites.value.has(word.content.word.wordId)
    )
    filteredFavorites.value = favoritesList.value
    favoriteSearchKey.value = ''
  }
}

// 修改 goToWord 函数
const goToWord = async (wordId: string) => {
  const index = data.value.findIndex(word => word.content.word.wordId === wordId)
  if (index !== -1) {
    currentIndex.value = index
    showFavorites.value = false
    await saveCurrentWordId()
  }
}

const toggleWordList = () => {
  if (showFavorites.value) {
    showFavorites.value = false
  }
  showWordList.value = !showWordList.value
}

const exportWords = async () => {
  try {
    // 获取 IndexedDB 中的所有数据
    const favoritesData = await db.getFavorites()
    const lastWordId = await db.getLastWordId()

    // 准备导出的数据
    const exportData = {
      words: data.value,
      favorites: Array.from(favoritesData),
      lastWordId: lastWordId,
      exportDate: new Date().toISOString(),
      version: '1.0'
    }

    // 创建并下载文件
    const blob = new Blob([JSON.stringify(exportData, null, 2)], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    const timestamp = new Date().toISOString().split('T')[0]
    a.download = `words-backup-${timestamp}.json`
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(url)
    showMoreMenu.value = false
  } catch (error) {
    console.error('Failed to export data:', error)
  }
}

const importWords = async (event: Event) => {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (file) {
    try {
      const text = await file.text()
      const importedData = JSON.parse(text)

      // 更新收藏夹
      favorites.value = new Set(importedData.favorites)
      await db.saveFavorites(favorites.value)

      // 更新最后浏览的单词位置
      if (importedData.lastWordId) {
        await db.saveLastWordId(importedData.lastWordId)
        const index = data.value.findIndex(word => word.content.word.wordId === importedData.lastWordId)
        if (index !== -1) {
          currentIndex.value = index
        }
      }

      showMoreMenu.value = false
    } catch (error) {
      console.error('Failed to import data:', error)
    }
  }
}
</script>

<template>
  <div class="language-container">
    <!-- 收藏按钮 -->
    <div class="favorites-button" @click="toggleFavoritesList">
      收藏
      <span class="favorites-count" v-if="favorites.size">{{ favorites.size }}</span>
    </div>

    <!-- 单词表按钮 -->
    <div class="wordlist-button">
      <span @click="toggleWordList">单词表</span>
      <div class="more-wrapper" @click.stop>
        <span class="more-icon" @click="showMoreMenu = !showMoreMenu">⋮</span>
        <div class="more-menu" v-if="showMoreMenu">
          <div class="menu-item" @click="exportWords">
            导出数据
          </div>
          <label class="menu-item">
            导入数据
            <input
              type="file"
              accept=".json"
              style="display: none"
              @change="importWords"
            >
          </label>
        </div>
      </div>
    </div>

    <!-- 添加单词列表面板 -->
    <div class="wordlist-panel" :class="{ active: showWordList }">
      <div class="wordlist-header">
        <h3>单词列表</h3>
        <span class="close-button" @click="showWordList = false">×</span>
      </div>
      <div class="search-box">
        <input
          type="text"
          v-model="wordListSearchKey"
          @input="searchWordList"
          placeholder="搜索单词或释义..."
        >
      </div>
      <div class="wordlist-list">
        <div
          v-for="(word, index) in filteredWordList"
          :key="word.content.word.wordId"
          class="wordlist-item"
          @click="goToWord(word.content.word.wordId)"
          :class="{ active: index === currentIndex }"
        >
          <span class="word">{{ word.content.word.wordHead }}</span>
          <span class="translation">{{ word.content.word.content.trans[0].tranCn }}</span>
        </div>
        <div v-if="filteredWordList.length === 0" class="empty-message">
          没有找到匹配的单词
        </div>
      </div>
    </div>

    <!-- 收藏夹弹出框 -->
    <div class="favorites-panel" :class="{ active: showFavorites }">
      <div class="favorites-header">
        <h3>收藏的单词</h3>
        <span class="close-button" @click="showFavorites = false">×</span>
      </div>
      <div class="search-box">
        <input
          type="text"
          v-model="favoriteSearchKey"
          @input="searchFavorites"
          placeholder="搜索单词或释义..."
        >
      </div>
      <div class="favorites-list">
        <div
          v-for="word in filteredFavorites"
          :key="word.content.word.wordId"
          class="favorite-item"
          @click="goToWord(word.content.word.wordId)"
        >
          <span class="word">{{ word.content.word.wordHead }}</span>
          <span class="translation">{{ word.content.word.content.trans[0].tranCn }}</span>
        </div>
        <div v-if="filteredFavorites.length === 0" class="empty-message">
          {{ favoriteSearchKey ? '没有找到匹配的单词' : '还没有收藏单词' }}
        </div>
      </div>
    </div>

    <div class="main-section">
      <div class="word-header">
        <div class="word-header-content">
          <div class="word-title">
            <h2>{{ data[currentIndex].content.word.wordHead }}</h2>
            <span
              class="star"
              :class="{ active: isFavorite(data[currentIndex].content.word.wordId) }"
              @click="toggleFavorite(data[currentIndex].content.word.wordId)"
            >★</span>
          </div>
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
  background: #28465e;
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
  scrollbar-color: #28465e #f5f5f5;
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
  border-left: 3px solid #28465e;
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
  background: #28465e;
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
  background-color: #28465e;
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

.word-title {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
}

.star {
  font-size: 32px;
  cursor: pointer;
  color: rgba(255, 255, 255, 0.3);
  transition: color 0.3s ease;
  user-select: none;
}

.star:hover {
  color: rgba(255, 255, 255, 0.8);
}

.star.active {
  color: #ffd700;
}

.favorites-button {
  position: fixed;
  top: 20px;
  right: 100px;
  padding: 8px 16px;
  background: #047974;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  z-index: 1001;
  display: flex;
  align-items: center;
  gap: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  font-weight: 500;
}

.favorites-button:hover {
  background: #345a77;
}

.favorites-count {
  background: rgba(255, 255, 255, 0.2);
  padding: 2px 6px;
  border-radius: 10px;
  font-size: 12px;
}

.favorites-panel {
  position: fixed;
  top: 70px;
  right: -320px;
  width: 300px;
  height: 500px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  transition: right 0.3s ease;
  display: flex;
  flex-direction: column;
  z-index: 1000;
}

.favorites-panel.active {
  right: 20px;
}

.favorites-header {
  padding: 16px;
  background: #28465e;
  color: white;
  border-radius: 8px 8px 0 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.close-button {
  cursor: pointer;
  font-size: 24px;
  line-height: 1;
}

.favorites-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.favorite-item {
  padding: 12px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
  transition: background-color 0.2s;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.favorite-item:hover {
  background-color: #f5f5f5;
}

.favorite-item .word {
  font-weight: bold;
  color: #28465e;
}

.favorite-item .translation {
  font-size: 14px;
  color: #666;
}

.empty-message {
  text-align: center;
  color: #999;
  padding: 20px;
}

/* 添加滚动条样式 */
.favorites-list::-webkit-scrollbar {
  width: 6px;
}

.favorites-list::-webkit-scrollbar-track {
  background: #f5f5f5;
}

.favorites-list::-webkit-scrollbar-thumb {
  background-color: #28465e;
  border-radius: 3px;
}

/* 单词表按钮样式 */
.wordlist-button {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 8px 16px;
  background: #047974;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  z-index: 1001;
  display: flex;
  align-items: center;
  gap: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  font-weight: 500;
}

.wordlist-button:hover {
  background: #345a77;
}

.more-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  padding-left: 12px;
  margin-left: 4px;
  border-left: 1px solid rgba(255, 255, 255, 0.2);
}

.more-icon {
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  padding: 0 4px;
}

.more-icon:hover {
  color: rgba(255, 255, 255, 0.8);
}

.more-menu {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  min-width: 120px;
  z-index: 1002;
}

.menu-item {
  padding: 8px 16px;
  color: #333;
  cursor: pointer;
  transition: background-color 0.2s;
  display: block;
  width: 100%;
  text-align: left;
  border: none;
  background: none;
  font: inherit;
}

.menu-item:hover {
  background-color: #f5f5f5;
}

.menu-item + .menu-item {
  border-top: 1px solid #eee;
}

@media (max-width: 768px) {
  .favorites-button {
    right: 100px;
  }

  .wordlist-button {
    right: 20px;
  }
}

/* 单词列表面板样式 */
.wordlist-panel {
  position: fixed;
  top: 70px;
  right: -320px;
  width: 300px;
  height: calc(100vh - 90px);
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
  transition: right 0.3s ease;
  display: flex;
  flex-direction: column;
  z-index: 1000;
}

.wordlist-panel.active {
  right: 20px;
}

.wordlist-header {
  padding: 16px;
  background: #28465e;
  color: white;
  border-radius: 8px 8px 0 0;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.wordlist-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.wordlist-item {
  padding: 12px;
  border-bottom: 1px solid #eee;
  cursor: pointer;
  transition: background-color 0.2s;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.wordlist-item:hover {
  background-color: #f5f5f5;
}

.wordlist-item.active {
  background-color: #e3f2fd;
  border-left: 3px solid #28465e;
}

.wordlist-item .word {
  font-weight: bold;
  color: #28465e;
}

.wordlist-item .translation {
  font-size: 14px;
  color: #666;
}

/* 添加滚动条样式 */
.wordlist-list::-webkit-scrollbar {
  width: 6px;
}

.wordlist-list::-webkit-scrollbar-track {
  background: #f5f5f5;
}

.wordlist-list::-webkit-scrollbar-thumb {
  background-color: #28465e;
  border-radius: 3px;
}

@media (max-width: 768px) {
  .wordlist-panel {
    width: calc(100% - 40px);
    right: -100%;
  }

  .wordlist-panel.active {
    right: 20px;
  }
}

.search-box {
  padding: 16px;
  border-bottom: 1px solid #eee;
  background: white;
  position: sticky;
  top: 0;
  z-index: 1;
}

.search-box input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.3s;
  background: white;
}

.search-box input:focus {
  border-color: #28465e;
}

.search-box input::placeholder {
  color: #999;
}

/* 调整面板内部布局 */
.favorites-panel,
.wordlist-panel {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 90px);
}

.favorites-list,
.wordlist-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  padding-top: 0;
}

/* 确保头部在搜索框之上 */
.favorites-header,
.wordlist-header {
  z-index: 2;
}
</style>
