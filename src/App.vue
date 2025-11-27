<script setup>
import { ref } from 'vue'
import axios from 'axios'

getList()

const value = ref('')
const list = ref([])

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

    <div
    v-for="(item, index) in list"
    :class="[item.isCompleted ? 'completed' : 'item']"
    >
      <div>
        <input @click="update(item._id)" v-model="item.isCompleted" type="checkbox" />
        <span class="name">{{ item.value }}</span>
      </div>

      <div @click="del(item._id)" class="del">del</div>
    </div>
  </div>
</template>

<style scoped>
.todo-app {
  box-sizing: border-box;
  margin-top: 40px;
  margin-left: 1%;
  padding-top: 30px;
  width: 98%;
  height: 500px;
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
</style>
