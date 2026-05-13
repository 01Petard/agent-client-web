<template>
  <div v-if="visible" class="modal-overlay" @click.self="$emit('close')">
    <div class="modal-container">
      <h2>{{ isLogin ? '登录' : '注册' }}</h2>
      <form @submit.prevent="submit">
        <div class="form-group">
          <label>用户名</label>
          <input v-model="name" required type="text" placeholder="输入用户名">
        </div>
        <div class="form-group">
          <label>手机号</label>
          <input v-model="phone" required type="text" placeholder="输入手机号">
        </div>
        <p v-if="error" class="error-msg">{{ error }}</p>
        <div class="form-actions">
          <button type="submit" :disabled="loading">{{ loading ? '处理中...' : (isLogin ? '登录' : '注册') }}</button>
          <button type="button" class="link-btn" @click="isLogin = !isLogin">
            {{ isLogin ? '没有账号？去注册' : '已有账号？去登录' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup>
import {ref} from 'vue';
import {loginUser, registerUser} from '../services/api.js';

const props = defineProps({visible: Boolean});
const emit = defineEmits(['close', 'authenticated']);

const isLogin = ref(true);
const name = ref('');
const phone = ref('');
const loading = ref(false);
const error = ref('');

const submit = async () => {
  error.value = '';
  loading.value = true;
  try {
    let user;
    if (isLogin.value) {
      user = await loginUser({name: name.value, phone: phone.value});
    } else {
      user = await registerUser({name: name.value, phone: phone.value});
    }
    emit('authenticated', user);
  } catch (e) {
    error.value = e.response?.data?.message || e.message || '操作失败';
  } finally {
    loading.value = false;
  }
};
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}

.modal-container {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  width: 380px;
  max-width: 90vw;
}

h2 { margin-bottom: 1.2rem; color: #333; }

.form-group { margin-bottom: 1rem; }
.form-group label { display: block; margin-bottom: 0.3rem; color: #666; font-size: 0.9rem; }
.form-group input {
  width: 100%;
  padding: 0.6rem 0.8rem;
  border: 2px solid #e9ecef;
  border-radius: 8px;
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s;
  box-sizing: border-box;
}
.form-group input:focus { border-color: #667eea; }

.error-msg { color: #e74c3c; font-size: 0.85rem; margin: 0.5rem 0; }

.form-actions { display: flex; flex-direction: column; gap: 0.5rem; margin-top: 1rem; }
.form-actions button {
  padding: 0.6rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.95rem;
  font-weight: 500;
}
.form-actions button[type=submit] {
  background: linear-gradient(135deg, #667eea, #764ba2);
  color: white;
}
.form-actions button[type=submit]:disabled { opacity: 0.6; cursor: not-allowed; }
.link-btn { background: transparent; color: #667eea; text-decoration: underline; }
</style>
