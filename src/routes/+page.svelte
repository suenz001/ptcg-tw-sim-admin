<script lang="ts">
  import { onMount } from 'svelte';
  import { auth, db } from '$lib/firebase';
  import { signInWithEmailAndPassword, signOut, GoogleAuthProvider, signInWithPopup } from 'firebase/auth';
  import { collection, getDocs, query, orderBy, limit } from 'firebase/firestore';

  let email = $state('');
  let password = $state('');
  let loginError = $state('');
  let isLoggingIn = $state(false);

  let activeTab = $state<'overview' | 'users' | 'feedback'>('overview');

  let users = $state<any[]>([]);
  let feedbacks = $state<any[]>([]);
  let loadingData = $state(false);

  // Authentication
  async function handleLogin() {
    isLoggingIn = true;
    loginError = '';
    try {
      await signInWithEmailAndPassword(auth, email, password);
    } catch (err: any) {
      loginError = err.message || '登入失敗，請檢查帳號密碼。';
    } finally {
      isLoggingIn = false;
    }
  }

  async function handleGoogleLogin() {
    isLoggingIn = true;
    loginError = '';
    const provider = new GoogleAuthProvider();
    try {
      await signInWithPopup(auth, provider);
    } catch (err: any) {
      loginError = err.message || 'Google 登入失敗。';
    } finally {
      isLoggingIn = false;
    }
  }

  function handleLogout() {
    signOut(auth);
  }

  // Data Fetching
  async function loadData() {
    if (!auth.currentUser) return;
    loadingData = true;
    try {
      // 載入玩家
      const usersSnap = await getDocs(query(collection(db, 'users'), orderBy('lastLoginAt', 'desc'), limit(100)));
      users = usersSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));

      // 載入意見回饋
      const feedSnap = await getDocs(query(collection(db, 'feedbacks'), orderBy('createdAt', 'desc'), limit(50)));
      feedbacks = feedSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));

    } catch (err: any) {
      console.error('Failed to load data:', err);
      alert('無法載入資料，請確認是否具有管理員權限！\n' + err.message);
    } finally {
      loadingData = false;
    }
  }

  // Monitor auth state changes to load data automatically when logged in
  $effect(() => {
    if (auth.currentUser) {
      loadData();
    }
  });

  function formatDate(ts: any) {
    if (!ts) return '未知';
    return new Date(ts.toDate()).toLocaleString('zh-TW');
  }
</script>

<main>
  {#if !auth.currentUser}
    <div class="login-container">
      <div class="login-box">
        <h1>PTCG 系統管理後台</h1>
        <p>請使用管理員帳號登入</p>
        <form onsubmit={(e) => { e.preventDefault(); handleLogin(); }}>
          <input type="email" bind:value={email} placeholder="Email" required />
          <input type="password" bind:value={password} placeholder="密碼" required />
          {#if loginError}<div class="error">{loginError}</div>{/if}
          <button type="submit" disabled={isLoggingIn}>
            {isLoggingIn ? '登入中...' : '使用 Email 登入'}
          </button>
        </form>
        <div class="divider">或</div>
        <button class="google-btn" onclick={handleGoogleLogin} disabled={isLoggingIn}>
          使用 Google 登入
        </button>
      </div>
    </div>
  {:else}
    <header class="admin-header">
      <div class="header-content">
        <h1>PTCG 後台系統</h1>
        <div class="user-info">
          <span>{auth.currentUser.email}</span>
          <button class="logout-btn" onclick={handleLogout}>登出</button>
        </div>
      </div>
    </header>

    <div class="dashboard">
      <aside class="sidebar">
        <button class:active={activeTab === 'overview'} onclick={() => activeTab = 'overview'}>📊 總覽</button>
        <button class:active={activeTab === 'users'} onclick={() => activeTab = 'users'}>👥 玩家列表</button>
        <button class:active={activeTab === 'feedback'} onclick={() => activeTab = 'feedback'}>💬 意見回饋</button>
        <button class="refresh-btn" onclick={loadData} disabled={loadingData}>🔄 重新整理</button>
      </aside>

      <div class="content">
        {#if loadingData}
          <div class="loading">正在載入資料...</div>
        {:else if activeTab === 'overview'}
          <section>
            <h2>系統總覽</h2>
            <div class="stats-grid">
              <div class="stat-card">
                <h3>總註冊/訪客人數</h3>
                <div class="value">{users.length}</div>
              </div>
              <div class="stat-card">
                <h3>收到的意見回饋</h3>
                <div class="value">{feedbacks.length}</div>
              </div>
            </div>
          </section>
        {:else if activeTab === 'users'}
          <section>
            <h2>玩家列表 (最新 100 筆)</h2>
            <div class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>UID</th>
                    <th>帳號類型</th>
                    <th>Email</th>
                    <th>登入次數</th>
                    <th>最後登入時間</th>
                  </tr>
                </thead>
                <tbody>
                  {#each users as u}
                    <tr>
                      <td class="mono">{u.id}</td>
                      <td>{u.isAnonymous ? '匿名' : '會員'}</td>
                      <td>{u.email || '-'}</td>
                      <td>{u.loginCount || 1}</td>
                      <td>{formatDate(u.lastLoginAt || u.createdAt)}</td>
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>
          </section>
        {:else if activeTab === 'feedback'}
          <section>
            <h2>意見回饋 (最新 50 筆)</h2>
            <div class="feedback-list">
              {#each feedbacks as fb}
                <div class="feedback-card">
                  <div class="fb-meta">
                    <span class="fb-time">{formatDate(fb.createdAt)}</span>
                    <span class="fb-uid mono">UID: {fb.uid}</span>
                  </div>
                  <div class="fb-content">{fb.content}</div>
                </div>
              {/each}
              {#if feedbacks.length === 0}
                <p>目前沒有任何意見回饋。</p>
              {/if}
            </div>
          </section>
        {/if}
      </div>
    </div>
  {/if}
</main>

<style>
  /* 登入畫面 */
  .login-container {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }
  .login-box {
    background: white;
    padding: 2.5rem;
    border-radius: 12px;
    box-shadow: 0 10px 30px rgba(0,0,0,0.1);
    width: 100%;
    max-width: 400px;
    text-align: center;
  }
  .login-box h1 {
    margin-top: 0;
    font-size: 1.5rem;
  }
  .login-box form {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    margin-top: 1.5rem;
  }
  .login-box input {
    padding: 0.75rem;
    border: 1px solid #ccc;
    border-radius: 6px;
    font-size: 1rem;
  }
  .login-box button {
    padding: 0.75rem;
    background: #0066cc;
    color: white;
    border: none;
    border-radius: 6px;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
  }
  .login-box button:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }
  .error {
    color: red;
    font-size: 0.9rem;
  }
  .divider {
    margin: 1.5rem 0;
    color: #888;
    position: relative;
  }
  .divider::before, .divider::after {
    content: '';
    position: absolute;
    top: 50%;
    width: 40%;
    height: 1px;
    background: #ddd;
  }
  .divider::before { left: 0; }
  .divider::after { right: 0; }
  .google-btn {
    width: 100%;
    padding: 0.75rem;
    background: #fff;
    color: #444;
    border: 1px solid #ccc;
    border-radius: 6px;
    font-size: 1rem;
    font-weight: bold;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }
  .google-btn:hover {
    background: #f9f9f9;
  }
  .google-btn:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }

  /* 後台佈局 */
  .admin-header {
    background: #1a2a3a;
    color: white;
    padding: 1rem 2rem;
  }
  .header-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
    max-width: 1400px;
    margin: 0 auto;
  }
  .header-content h1 {
    margin: 0;
    font-size: 1.25rem;
  }
  .user-info {
    display: flex;
    align-items: center;
    gap: 1rem;
    font-size: 0.9rem;
  }
  .logout-btn {
    background: rgba(255,255,255,0.1);
    border: 1px solid rgba(255,255,255,0.2);
    color: white;
    padding: 0.3rem 0.8rem;
    border-radius: 4px;
    cursor: pointer;
  }
  .logout-btn:hover {
    background: rgba(255,255,255,0.2);
  }

  .dashboard {
    display: flex;
    max-width: 1400px;
    margin: 0 auto;
    min-height: calc(100vh - 64px);
  }
  .sidebar {
    width: 250px;
    background: white;
    border-right: 1px solid #e0e0e0;
    padding: 1.5rem 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
  .sidebar button {
    text-align: left;
    padding: 0.75rem 1rem;
    background: none;
    border: none;
    border-radius: 6px;
    font-size: 1rem;
    cursor: pointer;
    color: #444;
  }
  .sidebar button:hover {
    background: #f0f0f0;
  }
  .sidebar button.active {
    background: #e6f0fa;
    color: #0066cc;
    font-weight: bold;
  }
  .sidebar .refresh-btn {
    margin-top: auto;
    text-align: center;
    background: #f0f0f0;
    font-size: 0.9rem;
  }

  .content {
    flex: 1;
    padding: 2rem;
  }
  .content h2 {
    margin-top: 0;
    margin-bottom: 1.5rem;
    color: #222;
  }

  /* 總覽卡片 */
  .stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1.5rem;
  }
  .stat-card {
    background: white;
    padding: 1.5rem;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  }
  .stat-card h3 {
    margin: 0 0 0.5rem 0;
    font-size: 1rem;
    color: #666;
  }
  .stat-card .value {
    font-size: 2.5rem;
    font-weight: bold;
    color: #0066cc;
  }

  /* 表格 */
  .table-container {
    background: white;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    overflow: hidden;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    text-align: left;
  }
  th, td {
    padding: 1rem;
    border-bottom: 1px solid #eee;
  }
  th {
    background: #f8f9fa;
    font-weight: 600;
    color: #555;
  }
  .mono {
    font-family: ui-monospace, 'Cascadia Code', monospace;
    font-size: 0.85rem;
    color: #666;
  }

  /* 意見回饋卡片 */
  .feedback-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
  .feedback-card {
    background: white;
    padding: 1.5rem;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  }
  .fb-meta {
    display: flex;
    justify-content: space-between;
    margin-bottom: 0.8rem;
    font-size: 0.85rem;
  }
  .fb-time {
    color: #888;
  }
  .fb-uid {
    color: #aaa;
  }
  .fb-content {
    line-height: 1.6;
    white-space: pre-wrap;
    color: #333;
  }
</style>
