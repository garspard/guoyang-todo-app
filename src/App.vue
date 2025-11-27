<script setup>
import { ref,computed } from 'vue'
import axios from 'axios'

getList()

const value = ref('')
const list = ref([])

//<!-- ⚠️ 2.0新增：使用 computed() 创建一个排序后的列表 -->
const sortedList = computed(() => {
  // 创建一个列表的副本，以避免直接修改原始 list
  const listCopy = [...list.value];

  // 使用 sort() 方法进行排序
  // 排序规则：
  // 1. 如果 item.isCompleted 为 false (未完成)，则排在前面 (返回 -1)
  // 2. 如果 item.isCompleted 为 true (已完成)，则排在后面 (返回 1)
  // 3. 如果两者完成状态相同，则保持原始顺序 (返回 0)
  listCopy.sort((a, b) => {
    // 未完成的项目 (false) 应该在前面
    if (a.isCompleted === false && b.isCompleted === true) {
      return -1; // a 排在 b 前面
    }
    // 已完成的项目 (true) 应该在后面
    if (a.isCompleted === true && b.isCompleted === false) {
      return 1; // a 排在 b 后面
    }
    // 状态相同，保持原有顺序（或根据 _id/创建时间进行二次排序）
    return 0;
  });

  return listCopy;
});
async function getList(){
  const res = await axios({
    url:"https://h4cpsw6xvi.hzh.sealos.run/get_list",
    method:"GET",
  })

  list.value = res.data.list
}
async function add() {
  if (!value.value.trim()) {
    alert("待办事项不能为空！");
    return;
  }

  try {
    await axios({
      url:"https://h4cpsw6xvi.hzh.sealos.run/add_todo",
      method:"POST",
      data:{
        value: value.value,
        isCompleted:false,
      },
    })

    // 成功后才清空和刷新列表
    getList()
    value.value = ''
  } catch (error) {
    // 打印错误信息，这会提示请求失败的原因
    console.error("添加待办事项失败:", error);
    alert("添加失败，请检查网络或后端服务！");
  }
}

async function update(id){
  await axios({
    url:"https://h4cpsw6xvi.hzh.sealos.run/update_todo",
    method:"POST",
    data:{
      id:id,
    },
  })

  getList()
  }

async function del(id) {
  await axios({
    url:"https://h4cpsw6xvi.hzh.sealos.run/del_todo",
    method:"POST",
    data:{
      id:id,
    },
  })

  getList()
}
</script>
<template>
  <div class="todo-app">
    <div class="title"> Guoyang 的 Todo App</div>

    <div class="todo-form">
      <input
        v-model="value"
        type="text"
        class="todo-input"
        placeholder="Add a todo"
      />
      <div @click="add" class="todo-button">Add Todo</div>
    </div>

    <div class="todo-list-container">
      <!-- ⚠️ 2.0新增：将列表容器包裹在滚动容器中 -->
      <div
      v-for="(item, index) in sortedList" :key="item._id"
      :class="[item.isCompleted ? 'completed' : 'item']"
    >
      <div>
        <input @click="update(item._id)" v-model="item.isCompleted" type="checkbox" />
        <span class="name">{{ item.value }}</span>
      </div>

      <div @click="del(item._id)" class="del">del</div>
    </div>
    </div>
  </div>
<!-- ⚠️ 2.0新增：新增公告栏 -->
  <div class="announcement">
      <div class="announcement-title">🎉 版本更新说明 (v2.0)</div>
      <ul>
        <li>新增：待办事项列表支持上下滚动。</li>
        <li>新增：公告栏模块。</li>
        <li>优化：已完成（浅色）事项将自动排到列表底部。</li>
        <li>优化：列表显示性能改进。</li>
        <li>国王语言：为爱倾心打造，郭阳专属。</li>
      </ul>
    </div>
</template>

<style scoped>
.todo-app {
  box-sizing: border-box;
  margin-top: 40px;
  margin-left: 1%;
  padding-top: 30px;
  width: 98%;
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

.todo-app {
  /* ... 保持原有样式，可以稍微增加高度以容纳更多列表 ... */
  height: 600px; /* 示例：将总高度增加到 600px */
}

/* ⚠️ 新增：列表滚动容器样式 */
.todo-list-container {
  /* 设置一个固定的最大高度，这是实现滚动的前提 */
  max-height: 400px; /* 您可以根据需要调整这个值 */
  
  /* 启用垂直滚动条 */
  overflow-y: auto;
  
  /* 保持列表项在 todo-app 内居中 */
  width: 98%;
  margin: 0 auto; 
  padding-bottom: 20px; /* 增加底部填充，防止最后一个项目被遮挡 */
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
  /* 调整宽度和边距，使其与 .todo-app (width: 98%; margin-left: 1%) 对齐 */
  width: 98%; 
  margin: 10px auto; /* 顶部留出 10px 间距，并水平居中 */
  padding: 15px;
  
  /* 保持公告栏的视觉样式 */
  border: 1px solid #d1e7dd;
  border-left: 5px solid #0f5132;
  background-color: #d1e7dd;
  color: #0f5132;
  border-radius: 5px;
  font-size: 14px;
  box-sizing: border-box; /* 确保 padding 不会使宽度超出 98% */
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
</style>
