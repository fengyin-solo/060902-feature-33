<template>
  <div class="home-page">
    <section class="hero">
      <div class="hero-content">
        <h2>💌 时光里的情书</h2>
        <p class="subtitle">每一条旧短信，都是一段被遗忘的浪漫</p>
        <p class="description">上传你的短信备份，让我们帮你找出那些最动人的对话。<br>
          按关键词、发送频率自动识别"最像情书"的对话，<br>
          生成一面专属的"情书墙"。</p>
      </div>
    </section>

    <section class="upload-section" v-if="!store.processing && store.loveLetters.length === 0">
      <div class="card">
        <h3>📤 上传短信备份</h3>
        <p class="tip">支持安卓 XML 导出、iPhone 备份、CSV/文本格式</p>
        
        <div 
          class="upload-area"
          :class="{ dragover: isDragging }"
          @click="triggerFileInput"
          @dragover.prevent="isDragging = true"
          @dragleave="isDragging = false"
          @drop.prevent="handleDrop"
        >
          <div class="icon">📁</div>
          <p>点击或拖拽文件到此处上传</p>
          <p class="tip">.xml / .csv / .txt / .db</p>
          <input 
            ref="fileInput"
            type="file"
            accept=".xml,.csv,.txt,.db,.json"
            @change="handleFileSelect"
            style="display: none"
          >
        </div>

        <div class="demo-section">
          <p>或者，先看看示例效果：</p>
          <button class="btn btn-secondary" @click="loadDemo">🎭 加载演示数据</button>
        </div>

        <div class="format-info">
          <h4>💡 支持的格式</h4>
          <ul>
            <li><strong>安卓：</strong>SMS Backup & Restore 导出的 XML 文件</li>
            <li><strong>iPhone：</strong>iTunes 备份中的 sms.db 或导出的文本/CSV</li>
            <li><strong>通用：</strong>CSV（需包含号码、内容、时间、类型列）</li>
          </ul>
        </div>
      </div>
    </section>

    <section v-if="store.processing" class="processing">
      <div class="loading"></div>
      <p>正在分析您的短信，寻找最动人的对话...</p>
    </section>

    <section v-if="store.error" class="error">
      <div class="card error-card">
        <div class="icon">😢</div>
        <h3>出了点小问题</h3>
        <p>{{ store.error }}</p>
        <button class="btn btn-secondary" @click="store.error = null">重新尝试</button>
      </div>
    </section>

    <section v-if="store.loveLetters.length > 0 && !store.processing" class="results">
      <div class="results-header">
        <h3>✨ 为您找到 {{ store.loveLetters.length }} 段情书级对话</h3>
        <div class="actions">
          <button class="btn btn-secondary" @click="clearAll">🔄 重新上传</button>
          <router-link to="/wall" class="btn btn-primary">🎨 查看情书墙</router-link>
        </div>
      </div>

      <div class="stats-overview">
        <div class="stat-card">
          <div class="stat-icon">👤</div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.contactCount }}</div>
            <div class="stat-label">联系人数量</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon">💬</div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.totalMessages }}</div>
            <div class="stat-label">短信总数</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon">📅</div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.timeSpan }}</div>
            <div class="stat-label">时间跨度</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon">🎯</div>
          <div class="stat-info">
            <div class="stat-value">{{ stats.hitRate }}%</div>
            <div class="stat-label">识别命中率</div>
          </div>
        </div>
      </div>

      <div class="detector-hits" v-if="stats.detectorHits.length > 0">
        <h4>🔍 识别命中概览</h4>
        <div class="hit-items">
          <div 
            v-for="hit in stats.detectorHits" 
            :key="hit.name" 
            class="hit-item"
          >
            <span class="hit-label">{{ hit.label }}</span>
            <span class="hit-count">{{ hit.count }} 次命中</span>
          </div>
        </div>
      </div>

      <div class="conversations-list">
        <div 
          v-for="(item, idx) in store.loveLetters.slice(0, 5)" 
          :key="item.conversation.id"
          class="conversation-card card"
          @click="selectConversation(item)"
        >
          <div class="rank-badge">#{{ idx + 1 }}</div>
          <div class="conv-header">
            <h4>{{ item.conversation.name }}</h4>
            <div class="love-score">
              <span class="score-label">情书指数</span>
              <span class="score-value">{{ item.loveScore }}</span>
            </div>
          </div>
          
          <div class="tags">
            <span 
              v-for="tag in item.tags.slice(0, 6)" 
              :key="tag"
              class="tag"
              :class="getTagClass(tag)"
            >
              {{ tag }}
            </span>
          </div>

          <div class="message-preview">
            <div 
              v-for="msg in getPreviewMessages(item)" 
              :key="msg.id"
              class="preview-msg"
              :class="{ sent: msg.isSent, received: msg.isReceived }"
            >
              <span class="msg-bubble">{{ msg.body }}</span>
            </div>
          </div>

          <div class="conv-meta">
            <span>📝 {{ item.conversation.messages.length }} 条消息</span>
            <span>📊 综合得分 {{ item.totalScore }}</span>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { store } from '@/store'
import { parseSmsFile, generateDemoData } from '@/parsers'
import { findLoveLetters, getDetectors } from '@/detectors'

const router = useRouter()
const fileInput = ref(null)
const isDragging = ref(false)

const stats = computed(() => {
  const convs = store.conversations
  const letters = store.loveLetters

  const contactCount = convs.length

  let totalMessages = 0
  for (const conv of convs) {
    totalMessages += conv.messages.length
  }

  let timeSpan = '—'
  let minDate = Infinity
  let maxDate = -Infinity
  for (const conv of convs) {
    for (const msg of conv.messages) {
      if (msg.date) {
        const t = typeof msg.date === 'number' ? msg.date : new Date(msg.date).getTime()
        if (t < minDate) minDate = t
        if (t > maxDate) maxDate = t
      }
    }
  }
  if (minDate !== Infinity && maxDate !== -Infinity) {
    const diffMs = maxDate - minDate
    const diffDays = Math.floor(diffMs / 86400000)
    if (diffDays >= 365) {
      const years = Math.floor(diffDays / 365)
      const remainDays = diffDays % 365
      timeSpan = remainDays > 0 ? `${years}年${remainDays}天` : `${years}年`
    } else if (diffDays >= 30) {
      const months = Math.floor(diffDays / 30)
      const remainDays = diffDays % 30
      timeSpan = remainDays > 0 ? `${months}个月${remainDays}天` : `${months}个月`
    } else {
      timeSpan = `${diffDays}天`
    }
  }

  const hitRate = convs.length > 0
    ? Math.round((letters.length / convs.length) * 100)
    : 0

  const detectorsList = getDetectors()
  const detectorHits = detectorsList.map(d => {
    let count = 0
    for (const letter of letters) {
      if (letter.scores && letter.scores[d.name] && letter.scores[d.name] > 0) {
        count++
      }
    }
    return { name: d.name, label: d.label || d.name, count }
  }).filter(d => d.count > 0)

  return { contactCount, totalMessages, timeSpan, hitRate, detectorHits }
})

function triggerFileInput() {
  fileInput.value.click()
}

async function handleFileSelect(e) {
  const file = e.target.files[0]
  if (file) {
    await processFile(file)
  }
}

async function handleDrop(e) {
  isDragging.value = false
  const file = e.dataTransfer.files[0]
  if (file) {
    await processFile(file)
  }
}

async function processFile(file) {
  store.processing = true
  store.error = null
  
  try {
    const conversations = await parseSmsFile(file)
    store.setConversations(conversations)
    
    const loveLetters = findLoveLetters(conversations)
    store.setLoveLetters(loveLetters)
    
    if (loveLetters.length === 0) {
      store.error = '没有找到符合条件的对话，试试上传更多短信吧！'
    }
  } catch (e) {
    console.error(e)
    store.error = e.message
  } finally {
    store.processing = false
  }
}

async function loadDemo() {
  store.processing = true
  store.error = null
  
  try {
    await new Promise(resolve => setTimeout(resolve, 800))
    const conversations = generateDemoData()
    store.setConversations(conversations)
    
    const loveLetters = findLoveLetters(conversations)
    store.setLoveLetters(loveLetters)
  } catch (e) {
    store.error = e.message
  } finally {
    store.processing = false
  }
}

function clearAll() {
  store.clearAll()
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

function selectConversation(item) {
  store.setSelectedConversation(item)
  router.push('/wall')
}

function getPreviewMessages(item) {
  return item.highlightedMessages.slice(0, 3)
}

function getTagClass(tag) {
  if (tag.includes('想念') || tag.includes('思念')) return 'tag-miss'
  if (tag.includes('晚安')) return 'tag-night'
  if (tag.includes('道歉') || tag.includes('对不起')) return 'tag-sorry'
  if (tag.includes('爱意') || tag.includes('双向奔赴')) return 'tag-love'
  if (tag.includes('争吵') || tag.includes('冤家') || tag.includes('情绪')) return 'tag-quarrel'
  if (tag.includes('撒娇') || tag.includes('可爱') || tag.includes('叠字')) return 'tag-cute'
  if (tag.includes('高频') || tag.includes('秒回') || tag.includes('互动')) return 'tag-freq'
  return 'tag'
}
</script>

<style scoped>
.home-page {
  max-width: 1000px;
  margin: 0 auto;
}

.hero {
  text-align: center;
  padding: 3rem 2rem;
  background: linear-gradient(135deg, #fff5f5 0%, #fff0f6 100%);
  border-radius: 20px;
  margin-bottom: 2rem;
}

.hero h2 {
  font-size: 2.5rem;
  color: var(--love-red);
  margin-bottom: 1rem;
}

.subtitle {
  font-size: 1.2rem;
  color: var(--text-light);
  margin-bottom: 1.5rem;
}

.description {
  color: var(--text-dark);
  line-height: 1.8;
}

.upload-section h3 {
  margin-bottom: 0.5rem;
}

.tip {
  color: var(--text-light);
  font-size: 0.9rem;
  margin-bottom: 1.5rem;
}

.upload-area {
  margin-bottom: 2rem;
}

.demo-section {
  text-align: center;
  padding: 1.5rem;
  background: var(--bg-light);
  border-radius: 12px;
  margin-bottom: 2rem;
}

.demo-section p {
  margin-bottom: 1rem;
}

.format-info {
  padding-top: 1.5rem;
  border-top: 1px solid var(--border);
}

.format-info h4 {
  margin-bottom: 1rem;
  color: var(--text-dark);
}

.format-info ul {
  list-style: none;
}

.format-info li {
  padding: 0.5rem 0;
  color: var(--text-light);
}

.processing {
  text-align: center;
  padding: 4rem 2rem;
}

.processing .loading {
  margin-bottom: 1.5rem;
}

.error-card {
  text-align: center;
  max-width: 500px;
  margin: 0 auto;
}

.error-card .icon {
  font-size: 4rem;
  margin-bottom: 1rem;
}

.error-card h3 {
  color: var(--love-red);
  margin-bottom: 1rem;
}

.error-card button {
  margin-top: 1.5rem;
}

.results-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.results-header h3 {
  color: var(--love-red);
  font-size: 1.5rem;
}

.actions {
  display: flex;
  gap: 1rem;
}

.conversations-list {
  display: grid;
  gap: 1.5rem;
}

.conversation-card {
  position: relative;
  cursor: pointer;
  transition: all 0.3s;
}

.conversation-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 30px rgba(231, 76, 60, 0.15);
}

.rank-badge {
  position: absolute;
  top: -10px;
  left: -10px;
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, var(--love-red), var(--love-pink));
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  box-shadow: var(--shadow);
}

.conv-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.conv-header h4 {
  font-size: 1.2rem;
}

.love-score {
  text-align: right;
}

.score-label {
  display: block;
  font-size: 0.8rem;
  color: var(--text-light);
}

.score-value {
  font-size: 1.8rem;
  font-weight: bold;
  color: var(--love-red);
}

.tags {
  margin-bottom: 1rem;
}

.message-preview {
  padding: 1rem;
  background: var(--bg-light);
  border-radius: 8px;
  margin-bottom: 1rem;
}

.preview-msg {
  margin-bottom: 0.5rem;
  display: flex;
}

.preview-msg.sent {
  justify-content: flex-end;
}

.preview-msg .msg-bubble {
  padding: 0.5rem 1rem;
  border-radius: 16px;
  max-width: 80%;
  font-size: 0.9rem;
}

.preview-msg.sent .msg-bubble {
  background: linear-gradient(135deg, var(--love-red), var(--love-pink));
  color: white;
  border-bottom-right-radius: 4px;
}

.preview-msg.received .msg-bubble {
  background: white;
  border: 1px solid var(--border);
  border-bottom-left-radius: 4px;
}

.conv-meta {
  display: flex;
  justify-content: space-between;
  color: var(--text-light);
  font-size: 0.9rem;
}

.stats-overview {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.stat-card {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: var(--card-bg);
  border-radius: 12px;
  padding: 1.25rem;
  box-shadow: var(--shadow);
  border: 1px solid var(--border);
  transition: transform 0.2s;
}

.stat-card:hover {
  transform: translateY(-2px);
}

.stat-icon {
  font-size: 2rem;
  flex-shrink: 0;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--love-red);
  line-height: 1.2;
}

.stat-label {
  font-size: 0.85rem;
  color: var(--text-light);
  margin-top: 0.2rem;
}

.detector-hits {
  background: var(--card-bg);
  border-radius: 12px;
  padding: 1.25rem;
  box-shadow: var(--shadow);
  border: 1px solid var(--border);
  margin-bottom: 1.5rem;
}

.detector-hits h4 {
  margin-bottom: 1rem;
  color: var(--text-dark);
}

.hit-items {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.hit-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: var(--bg-light);
  border-radius: 20px;
  font-size: 0.9rem;
}

.hit-label {
  color: var(--text-dark);
  font-weight: 500;
}

.hit-count {
  color: var(--love-red);
  font-weight: 600;
}

@media (max-width: 768px) {
  .stats-overview {
    grid-template-columns: repeat(2, 1fr);
  }
}
</style>
