<script lang="ts">
  import { onMount } from 'svelte';
  import { auth } from '$lib/firebase';
  import { onAuthStateChanged, type User } from 'firebase/auth';

  let { children } = $props();

  let user = $state<User | null>(null);
  let initialized = $state(false);

  // 密碼保護
  const ADMIN_PWD = '750051';
  let isAuthenticated = $state(false);
  let pwdInput = $state('');
  let pwdError = $state(false);

  function checkPwd() {
    if (pwdInput === ADMIN_PWD) {
      isAuthenticated = true;
      localStorage.setItem('admin_auth', '1');
    } else {
      pwdError = true;
      pwdInput = '';
    }
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter') checkPwd();
  }

  onMount(() => {
    if (localStorage.getItem('admin_auth') === '1') {
      isAuthenticated = true;
    }
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      user = u;
      initialized = true;
    });
    return unsubscribe;
  });
</script>

{#if !initialized}
  <div class="loading">正在載入管理員系統...</div>
{:else if !isAuthenticated}
  <div class="login-container">
    <div class="login-box">
      <h2>系統登入</h2>
      <p>請輸入管理員密碼以進入後台</p>
      <input 
        type="password" 
        inputmode="numeric" 
        pattern="[0-9]*" 
        placeholder="請輸入密碼" 
        bind:value={pwdInput} 
        onkeydown={handleKeydown}
      />
      {#if pwdError}
        <div class="error">密碼錯誤，請重試</div>
      {/if}
      <button onclick={checkPwd}>登入</button>
    </div>
  </div>
{:else}
  {@render children()}
{/if}

<style>
  :global(body) {
    margin: 0;
    font-family: system-ui, -apple-system, 'Microsoft JhengHei', sans-serif;
    background: #f4f6f8;
    color: #333;
  }
  .loading {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    font-size: 1.2rem;
    color: #666;
  }
  .login-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    background: #f4f6f8;
  }
  .login-box {
    background: #fff;
    padding: 2rem 3rem;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    text-align: center;
    max-width: 400px;
    width: 90%;
    box-sizing: border-box;
  }
  .login-box h2 {
    margin-top: 0;
    color: #333;
  }
  .login-box p {
    color: #666;
    margin-bottom: 1.5rem;
  }
  .login-box input {
    width: 100%;
    padding: 0.8rem;
    font-size: 1.5rem;
    letter-spacing: 0.3em;
    text-align: center;
    border: 1px solid #ccc;
    border-radius: 4px;
    margin-bottom: 1rem;
    box-sizing: border-box;
  }
  .login-box input:focus {
    outline: none;
    border-color: #4a90e2;
  }
  .login-box button {
    width: 100%;
    padding: 0.8rem;
    font-size: 1.1rem;
    background: #4a90e2;
    color: #fff;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
  .login-box button:hover {
    background: #357abd;
  }
  .error {
    color: #d32f2f;
    margin-bottom: 1rem;
    font-size: 0.9rem;
  }
</style>
