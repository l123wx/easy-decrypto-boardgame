<script setup lang="ts">
import { ref, onMounted } from 'vue'

interface WordsData {
  words: string
}

interface Round {
  id: number
  words: string[]
  turns: string[]
}

// 存储所有词语的数组
const allWords = ref<string[]>([])
// 当前选中的四个词语
const selectedWords = ref<string[]>([])
// 生成的数字
const generatedNumber = ref<string>('')
// 控制弹窗显示
const showModal = ref<boolean>(false)
// 游戏记录
const gameHistory = ref<Round[]>([])
// 当前轮次
const currentRound = ref<Round | null>(null)
// 显示游戏记录弹窗
const showHistoryModal = ref<boolean>(false)
// 所有可能的三位数字组合
const allPossibleNumbers = [
  '123', '124', '132', '134', '142', '143',
  '213', '214', '231', '234', '241', '243',
  '312', '314', '321', '324', '341', '342',
  '412', '413', '421', '423', '431', '432'
]
// 当前轮次可用的数字组合
const availableNumbers = ref<string[]>([])
// 显示清空确认弹窗
const showClearConfirmModal = ref<boolean>(false)

// 加载词语数据和恢复游戏状态
onMounted(async () => {
  try {
    // 修改为使用相对路径，确保在部署环境中能正确加载
    const response = await fetch('./words.json')
    
    // 检查响应状态
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }
    
    const data = await response.json() as WordsData
    allWords.value = data.words.split(';')
    
    // 尝试从本地存储加载游戏记录
    const savedHistory = localStorage.getItem('gameHistory')
    if (savedHistory) {
      gameHistory.value = JSON.parse(savedHistory)
    }
    
    // 恢复当前游戏状态
    const currentRoundId = localStorage.getItem('currentRoundId')
    if (currentRoundId && gameHistory.value.length > 0) {
      const roundId = parseInt(currentRoundId)
      const round = gameHistory.value.find(r => r.id === roundId)
      if (round) {
        currentRound.value = round
        selectedWords.value = [...round.words]
        
        // 恢复可用数字组合
        const usedNumbers = new Set(round.turns)
        availableNumbers.value = allPossibleNumbers.filter(num => !usedNumbers.has(num))
      }
    }
  } catch (error) {
    console.error('加载词语失败:', error)
  }
})

// 开始新一轮游戏
const startNewRound = () => {
  if (allWords.value.length === 0) return
  
  // 随机选择四个词语
  selectedWords.value = []
  const tempWords = [...allWords.value]
  
  for (let i = 0; i < 4; i++) {
    if (tempWords.length === 0) break
    const randomIndex = Math.floor(Math.random() * tempWords.length)
    selectedWords.value.push(tempWords[randomIndex])
    tempWords.splice(randomIndex, 1)
  }
  
  // 创建新一轮游戏记录
  const newRoundId = gameHistory.value.length > 0 
    ? Math.max(...gameHistory.value.map(r => r.id)) + 1 
    : 1
    
  currentRound.value = {
    id: newRoundId,
    words: [...selectedWords.value],
    turns: []
  }
  
  // 重置可用数字组合
  availableNumbers.value = [...allPossibleNumbers]
  
  // 添加到游戏历史
  gameHistory.value.push(currentRound.value)
  
  // 保存到本地存储
  saveGameHistory()
  saveCurrentRoundId()
}

// 生成由1234中随机三位数字组成的数字
const generateNumber = () => {
  if (!currentRound.value) {
    alert('请先开始新一轮游戏')
    return
  }
  
  // 检查是否还有可用的数字组合
  if (availableNumbers.value.length === 0) {
    alert('所有可能的数字组合已用完，请开始新一轮游戏')
    return
  }
  
  // 从可用数字中随机选择一个
  const randomIndex = Math.floor(Math.random() * availableNumbers.value.length)
  generatedNumber.value = availableNumbers.value[randomIndex]
  
  // 从可用数字中移除已使用的数字
  availableNumbers.value.splice(randomIndex, 1)
  
  showModal.value = true
  
  // 将数字添加到当前轮次
  if (currentRound.value) {
    currentRound.value.turns.push(generatedNumber.value)
    
    // 更新游戏历史中的当前轮次
    const index = gameHistory.value.findIndex(r => r.id === currentRound.value?.id)
    if (index !== -1) {
      gameHistory.value[index] = currentRound.value
    }
    
    // 保存到本地存储
    saveGameHistory()
  }
}

// 保存游戏历史到本地存储
const saveGameHistory = () => {
  localStorage.setItem('gameHistory', JSON.stringify(gameHistory.value))
}

// 保存当前轮次ID到本地存储
const saveCurrentRoundId = () => {
  if (currentRound.value) {
    localStorage.setItem('currentRoundId', currentRound.value.id.toString())
  } else {
    localStorage.removeItem('currentRoundId')
  }
}

// 显示游戏记录
const showGameHistory = () => {
  showHistoryModal.value = true
}

// 显示清空数据确认弹窗
const showClearConfirm = () => {
  showClearConfirmModal.value = true
}

// 清空所有游戏数据
const clearAllData = () => {
  gameHistory.value = []
  currentRound.value = null
  selectedWords.value = []
  availableNumbers.value = []
  
  // 清除本地存储
  localStorage.removeItem('gameHistory')
  localStorage.removeItem('currentRoundId')
  
  // 关闭确认弹窗和历史记录弹窗
  showClearConfirmModal.value = false
  showHistoryModal.value = false
}

// 关闭弹窗
const closeModal = () => {
  showModal.value = false
}

// 关闭历史记录弹窗
const closeHistoryModal = () => {
  showHistoryModal.value = false
}

// 关闭清空确认弹窗
const closeClearConfirmModal = () => {
  showClearConfirmModal.value = false
}
</script>

<template>
  <div class="container">
    <h1>谍报风云</h1>
    
    <div v-if="selectedWords.length > 0" class="words-container">
      <div v-for="(word, index) in selectedWords" :key="index" class="word-item">
        <span class="word-index">{{ index + 1 }}</span>
        <span class="word-text">{{ word }}</span>
      </div>
    </div>

    <div class="button-group">
      <button class="btn" @click="startNewRound">开始新一轮游戏</button>
      <button class="btn" @click="generateNumber">生成本轮数字</button>
      <button class="btn history-btn" @click="showGameHistory">查看游戏记录</button>
    </div>
    <a href="https://www.bilibili.com/opus/1033711144020213779/" target="_blank">游戏规则</a>

    
    <div v-if="currentRound" class="round-info">
      <div class="current-round">
        当前轮次：第 {{ currentRound.id }} 轮，回合 {{ currentRound.turns.length }}
      </div>
      <div class="available-numbers">
        剩余可用数字组合: {{ availableNumbers.length }} / {{ allPossibleNumbers.length }}
      </div>
    </div>
    
    <!-- 数字弹窗 -->
    <div v-if="showModal" class="modal-overlay" @click="closeModal">
      <div class="modal" @click.stop>
        <div class="modal-content">
          <h2>本轮数字</h2>
          <div class="number">{{ generatedNumber }}</div>
          <div class="turn-info" v-if="currentRound">
            回合 {{ currentRound.turns.length }}
          </div>
          <button class="btn close-btn" @click="closeModal">关闭</button>
        </div>
      </div>
    </div>
    
    <!-- 游戏记录弹窗 -->
    <div v-if="showHistoryModal" class="modal-overlay" @click="closeHistoryModal">
      <div class="history-modal" @click.stop>
        <div class="modal-content">
          <h2>游戏记录</h2>
          
          <div v-if="gameHistory.length === 0" class="empty-history">
            暂无游戏记录
          </div>
          
          <div v-else class="history-list">
            <div v-for="round in [...gameHistory].reverse()" :key="round.id" class="history-item">
              <h3>第 {{ round.id }} 轮</h3>
              
              <div class="history-words">
                <div>该轮词语为：</div>
                <div class="word-list">
                  <div v-for="(word, index) in round.words" :key="index">
                    {{ index + 1 }}. {{ word }}
                  </div>
                </div>
              </div>
              
              <div class="history-turns">
                <div v-if="round.turns.length === 0">暂无回合记录</div>
                <div v-for="(turn, index) in round.turns" :key="index" class="turn-record">
                  回合 {{ index + 1 }} 数字：<span class="turn-number">{{ turn }}</span>
                </div>
              </div>
            </div>
          </div>
          
          <div class="modal-footer">
            <button class="btn close-btn" @click="closeHistoryModal">关闭</button>
            <button v-if="gameHistory.length > 0" class="btn danger-btn" @click="showClearConfirm">清空数据</button>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 清空数据确认弹窗 -->
    <div v-if="showClearConfirmModal" class="modal-overlay" @click="closeClearConfirmModal">
      <div class="confirm-modal" @click.stop>
        <div class="modal-content">
          <h2>确认清空数据</h2>
          <p>此操作将清空所有游戏记录，且无法恢复。确定要继续吗？</p>
          <div class="confirm-buttons">
            <button class="btn cancel-btn" @click="closeClearConfirmModal">取消</button>
            <button class="btn danger-btn" @click="clearAllData">确认清空</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}

h1 {
  margin-bottom: 30px;
}

.button-group {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin-bottom: 30px;
  flex-wrap: wrap;
}

.btn {
  padding: 10px 20px;
  font-size: 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.btn:hover {
  background-color: #45a049;
}

.history-btn {
  background-color: #2196F3;
}

.history-btn:hover {
  background-color: #0b7dda;
}

.danger-btn {
  background-color: #f44336;
}

.danger-btn:hover {
  background-color: #d32f2f;
}

.cancel-btn {
  background-color: #9e9e9e;
}

.cancel-btn:hover {
  background-color: #757575;
}

.words-container {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
  margin: 0 auto;
  max-width: 600px;
  margin-bottom: 30px;
}

.word-item {
  display: flex;
  align-items: center;
  padding: 15px;
  background-color: #f5f5f5;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.word-index {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 30px;
  height: 30px;
  background-color: #4CAF50;
  color: white;
  border-radius: 50%;
  margin-right: 10px;
  font-weight: bold;
}

.word-text {
  font-size: 18px;
}

.round-info {
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  align-items: center;
}

.current-round {
  background-color: #e3f2fd;
  display: inline-block;
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 16px;
  color: #1565c0;
  margin-bottom: 5px;
}

.available-numbers {
  background-color: #e8f5e9;
  display: inline-block;
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 14px;
  color: #2e7d32;
}

/* 弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.modal {
  background-color: white;
  border-radius: 8px;
  padding: 20px;
  width: 300px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.history-modal {
  background-color: white;
  border-radius: 8px;
  padding: 20px;
  width: 90%;
  max-width: 600px;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.confirm-modal {
  background-color: white;
  border-radius: 8px;
  padding: 20px;
  width: 90%;
  max-width: 400px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.modal-content {
  text-align: center;
}

.number {
  font-size: 48px;
  font-weight: bold;
  margin: 20px 0 10px;
  color: #4CAF50;
}

.turn-info {
  font-size: 18px;
  margin-bottom: 20px;
  color: #666;
}

.modal-footer {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 20px;
}

.confirm-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 20px;
}

/* 历史记录样式 */
.history-list {
  max-height: 60vh;
  overflow-y: auto;
  padding: 0 10px;
}

.history-item {
  background-color: #f9f9f9;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 15px;
  text-align: left;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.history-item h3 {
  margin-top: 0;
  margin-bottom: 15px;
  color: #333;
  border-bottom: 1px solid #ddd;
  padding-bottom: 8px;
}

.history-words {
  margin-bottom: 15px;
}

.word-list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  margin-top: 8px;
}

.history-turns {
  margin-top: 15px;
}

.turn-record {
  margin: 8px 0;
}

.turn-number {
  font-weight: bold;
  color: #4CAF50;
  font-size: 18px;
}

.empty-history {
  padding: 30px;
  color: #666;
  font-style: italic;
}
</style>
