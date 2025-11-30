<script setup>
import { ref, computed } from 'vue'
import axios from 'axios'
import Gomoku from './Gomoku.vue' // 引入五子棋组件

// --- 核心状态 ---
const activeModule = ref('todo') // 当前大模块: 'todo' 或 'entertainment'
const gameView = ref('list')     // 娱乐模块子视图: 'list' (列表) 或 'gomoku' (游戏)

// --- Todo List 逻辑 (保持不变) ---
getList()
const value = ref('')
const list = ref([])

const sortedList = computed(() => {
  const listCopy = [...list.value];
  listCopy.sort((a, b) => {
    if (a.isCompleted === false && b.isCompleted === true) return -1;
    if (a.isCompleted === true && b.isCompleted === false) return 1;
    return 0;
  });
  return listCopy;
});

async function getList(){
  try {
    const res = await axios({
      url:"https://h4cpsw6xvi.hzh.sealos.run/get_list",
      method:"GET",
    })
    list.value = res.data.list
  } catch (error) { console.error(error); }
}

async function add() {
  if (!value.value.trim()) { alert("待办事项不能为空！"); return; }
  try {
    await axios({
      url:"https://h4cpsw6xvi.hzh.sealos.run/add_todo",
      method:"POST",
      data:{ value: value.value, isCompleted:false },
    })
    getList(); value.value = '';
  } catch (error) { alert("添加失败，请检查网络！"); }
}

async function update(id){
  await axios({ url:"https://h4cpsw6xvi.hzh.sealos.run/update_todo", method:"POST", data:{ id:id } })
  getList()
}

async function del(id) {
  await axios({ url:"https://h4cpsw6xvi.hzh.sealos.run/del_todo", method:"POST", data:{ id:id } })
  getList()
}

// --- 娱乐模块辅助函数 ---
function enterEntertainment() {
  activeModule.value = 'entertainment';
  gameView.value = 'list';
}
</script>

<template>
  <div class="app-container">
    
    <div class="app-controls">
      <button 
        :class="{'active': activeModule === 'todo'}" 
        @click="activeModule = 'todo'">
        📝 待办清单
      </button>
      <button 
        :class="{'active': activeModule === 'entertainment'}" 
        @click="enterEntertainment"> 
        🎮 阳阳小游戏
      </button>
    </div>

    <div v-if="activeModule === 'todo'">
      <div class="todo-app">
        <div class="title"> Guoyang 的 Todo App</div>

        <div class="todo-form">
          <input
            v-model="value"
            type="text"
            class="todo-input"
            placeholder="Add a todo"
            @keyup.enter="add"
          />
          <div @click="add" class="todo-button">Add Todo</div>
        </div>

        <div class="todo-list-container">
          <div
            v-for="(item, index) in sortedList"
            :key="item._id"
            :class="[item.isCompleted ? 'completed' : 'item']"
          >
            <div>
              <input @click="update(item._id)" :checked="item.isCompleted" type="checkbox" />
              <span class="name">{{ item.value }}</span>
            </div>

            <div @click="del(item._id)" class="del">del</div>
          </div>
        </div>
      </div>
      
      <div class="announcement">
        <div class="announcement-title">🎉 版本更新说明 (v2.1)</div>
        <ul>
          <li>新增：阳阳游戏模块。</li>
          <li>新增：技能五子棋~。</li>
          <li>国王语录：你是一个德智体美劳全面发展的小宝宝，受你一靠子郭小阳💗</li>
        </ul>
      </div>
    </div>

    <div v-else-if="activeModule === 'entertainment'" class="entertainment-box">
      
      <div v-if="gameView === 'list'" class="game-hall">
        <h2>🕹️ 阳阳小游戏</h2>
        
        <div class="game-card" @click="gameView = 'gomoku'">
          <div class="card-icon">⚔️</div>
          <div class="card-info">
            <h3>技能五子棋</h3>
            <p>阳了个阳五子棋</p>
          </div>
          <div class="card-tag">推荐</div>
        </div>

        <div class="game-card disabled">
          <div class="card-icon">🚧</div>
          <div class="card-info">
            <h3>2048 (待开发)</h3>
            <p>经典的数字消除游戏。</p>
          </div>
        </div>
      </div>

      <div v-else-if="gameView === 'gomoku'" class="game-view">
        <button class="back-btn" @click="gameView = 'list'">⬅ 返回大厅</button>
        <Gomoku />
      </div>

    </div>

  </div>
</template>

<style scoped>
/* --- 全局容器 (修复：移除 max-width 限制) --- */
.app-container {
  /* max-width: 600px;  <-- 删除这行，恢复全宽 */
  margin: 0 auto;
  font-family: Arial, sans-serif;
}

/* --- 顶部导航 --- */
.app-controls {
  text-align: center;
  margin: 20px 0;
}
.app-controls button {
  padding: 10px 20px;
  margin: 0 5px;
  border: 1px solid #ccc;
  background-color: #f0f0f0;
  cursor: pointer;
  border-radius: 5px;
  transition: all 0.2s;
  font-weight: bold;
}
.app-controls button.active {
  background: linear-gradient(to right, rgb(215, 132, 211), rgb(136, 83, 189));
  color: white;
  border-color: transparent;
  box-shadow: 0 4px 8px rgba(136, 83, 189, 0.4);
}

/* --- Todo List 样式 (完全恢复原样) --- */
.todo-app {
  box-sizing: border-box;
  margin-top: 40px;
  margin-left: 1%;
  padding-top: 30px;
  width: 98%; /* 恢复 98% 宽度 */
  height: 600px;
  background: #ffffff;
  border-radius: 5px;
}

.title {
  text-align: center;
  font-size: 30px;
  font-weight: 700;
}

.todo-form {
  display: flex;
  margin: 20px 0 30px 20px;
}

.todo-button {
  width: 100px;
  height: 52px;
  border-radius: 0 20px 20px 0;
  text-align: center;
  background: linear-gradient(to right, rgb(215, 132, 211), rgb(136, 83, 189));
  color: #fff;
  line-height: 52px;
  cursor: pointer;
  font-size: 14px;
  user-select: none;
}

.todo-input {
  padding: 0px 15px 0px 15px;
  border-radius: 20px 0 0 20px;
  border: 1px solid #dfe1e5;
  outline: none;
  width: 60%;
  height: 50px;
}

.todo-list-container {
  max-height: 400px; 
  overflow-y: auto;
  width: 98%;
  margin: 0 auto; 
  padding-bottom: 20px;
}

.item {
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 80%;
  height: 50px;
  margin: 8px auto;
  padding: 16px;
  border-radius: 20px;
  box-shadow: rgba(149, 157, 165, 0.2) 0px 8px 20px;
}

.del {
  color: red;
  cursor: pointer;
}

.completed {
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 80%;
  height: 50px;
  margin: 8px auto;
  padding: 16px;
  border-radius: 20px;
  box-shadow: rgba(149, 157, 165, 0.2) 0px 8px 20px;
  text-decoration: line-through;
  opacity: 0.4;
}

.announcement {
  width: 98%; 
  margin: 10px auto; 
  padding: 15px;
  border: 1px solid #d1e7dd;
  border-left: 5px solid #0f5132;
  background-color: #d1e7dd;
  color: #0f5132;
  border-radius: 5px;
  font-size: 14px;
  box-sizing: border-box;
}

.announcement-title {
  font-weight: bold;
  margin-bottom: 5px;
  border-bottom: 1px dashed #0f513250;
  padding-bottom: 5px;
}

.announcement ul {
  list-style-type: none;
  padding-left: 0;
  margin-top: 5px;
}

.announcement li {
  margin-bottom: 3px;
}

/* --- 娱乐空间样式 (新增：单独设置宽度限制，使其居中美观) --- */
.entertainment-box {
  max-width: 600px;
  margin: 0 auto;
}

.game-hall {
  padding: 20px;
  background: #fdfdfd;
  border-radius: 10px;
  border: 1px solid #eee;
}
.game-hall h2 {
  text-align: center;
  color: #333;
  margin-bottom: 20px;
  font-size: 22px;
}
.game-card {
  display: flex;
  align-items: center;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  padding: 15px;
  margin-bottom: 15px;
  cursor: pointer;
  transition: all 0.2s;
  position: relative;
  overflow: hidden;
}
.game-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 5px 15px rgba(0,0,0,0.1);
  border-color: #bfaee3;
}
.game-card.disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background: #f9f9f9;
}
.card-icon {
  font-size: 30px;
  margin-right: 15px;
}
.card-info h3 {
  margin: 0 0 5px 0;
  font-size: 18px;
  color: #333;
}
.card-info p {
  margin: 0;
  font-size: 13px;
  color: #666;
}
.card-tag {
  position: absolute;
  top: 10px;
  right: 10px;
  background: #ff9800;
  color: white;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
}
.back-btn {
  background: #607d8b;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 5px;
  cursor: pointer;
  margin-bottom: 15px;
  font-size: 14px;
}
.back-btn:hover {
  background: #546e7a;
}
</style>