<template>
  <div class="app">
    <!-- 应用标题栏 -->
    <header class="app-header">
      <div class="header-left">
        <h1 class="app-title">☕ 三更咖啡Agent</h1>
        <span v-if="currentUser" class="user-badge">
          {{ currentUser.name }} (ID: {{ currentUser.id }})
        </span>
      </div>

      <nav class="tab-nav">
        <button :class="['tab-btn', { active: activeTab === 'chat' }]" @click="switchTab('chat')">
          💬 聊天
        </button>
        <button :class="['tab-btn', { active: activeTab === 'orders' }]" @click="switchTab('orders')">
          📋 订单后台
        </button>
      </nav>

      <div class="header-right">
        <button v-if="!currentUser" class="action-btn" @click="showLogin = true">🔑 登录/注册</button>
        <template v-else>
          <button class="action-btn" @click="deleteHistory">🗑️ 清除对话</button>
          <button class="action-btn logout-btn" @click="logout">🚪 退出</button>
        </template>
      </div>
    </header>

    <!-- Tab 内容 -->
    <main class="tab-content">
      <ChatInterface v-if="activeTab === 'chat'" ref="chatRef"
                     :key="'chat-' + refreshKey"
                     :current-user="currentUser"/>
      <OrderManagement v-if="activeTab === 'orders'" :key="'orders-' + refreshKey"/>
    </main>

    <!-- 登录/注册模态框 -->
    <AuthModal :visible="showLogin" @close="showLogin = false" @authenticated="onAuthenticated"/>
  </div>
</template>

<script setup>
import {provide, ref, onMounted} from 'vue';
import ChatInterface from './components/ChatInterface.vue';
import OrderManagement from './components/OrderManagement.vue';
import AuthModal from './components/AuthModal.vue';
import {deleteUserHistory} from './services/api.js';

const activeTab = ref('chat');
const refreshKey = ref(0);
const currentUser = ref(null);
const showLogin = ref(false);
const chatRef = ref(null);

const switchTab = (tab) => {
  activeTab.value = tab;
  refreshKey.value++;
};

// 从 localStorage 恢复登录状态
onMounted(() => {
  const saved = localStorage.getItem('coffee_user');
  if (saved) {
    try {
      currentUser.value = JSON.parse(saved);
    } catch { localStorage.removeItem('coffee_user'); }
  }
  // 未登录显示登录框
  if (!currentUser.value) {
    showLogin.value = true;
  }
});

const onAuthenticated = (user) => {
  currentUser.value = user;
  localStorage.setItem('coffee_user', JSON.stringify(user));
  showLogin.value = false;
  refreshKey.value++;
};

const logout = () => {
  currentUser.value = null;
  localStorage.removeItem('coffee_user');
  refreshKey.value++;
  showLogin.value = true;
};

const deleteHistory = async () => {
  if (!currentUser.value) return;
  if (!confirm('确定清除所有对话记录吗？此操作不可恢复。')) return;
  try {
    await deleteUserHistory(currentUser.value.id);
    refreshKey.value++;
    alert('对话已清除');
  } catch (e) {
    alert('清除失败');
  }
};

provide('currentUser', currentUser);
</script>

<style scoped>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  overflow: hidden;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.6rem 1.5rem;
  background: rgba(255,255,255,0.12);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255,255,255,0.15);
  flex-shrink: 0;
  gap: 1rem;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  min-width: 0;
}

.app-title {
  color: white;
  font-size: 1.2rem;
  font-weight: 600;
  margin: 0;
  white-space: nowrap;
}

.user-badge {
  color: rgba(255,255,255,0.7);
  font-size: 0.8rem;
  background: rgba(255,255,255,0.1);
  padding: 0.2rem 0.6rem;
  border-radius: 10px;
  white-space: nowrap;
}

.tab-nav {
  display: flex;
  gap: 0.4rem;
}

.tab-btn {
  color: rgba(255,255,255,0.75);
  background: transparent;
  border: 1px solid rgba(255,255,255,0.2);
  font-size: 0.9rem;
  font-weight: 500;
  padding: 0.4rem 1rem;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.tab-btn:hover { background: rgba(255,255,255,0.15); color: white; }
.tab-btn.active { background: rgba(255,255,255,0.25); color: white; border-color: rgba(255,255,255,0.5); }

.header-right {
  display: flex;
  gap: 0.5rem;
}

.action-btn {
  color: white;
  background: rgba(255,255,255,0.12);
  border: 1px solid rgba(255,255,255,0.2);
  padding: 0.4rem 0.8rem;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.action-btn:hover { background: rgba(255,255,255,0.25); }

.tab-content {
  flex: 1;
  overflow: hidden;
}
</style>
