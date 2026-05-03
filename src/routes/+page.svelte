<script lang="ts">
  import { onMount } from 'svelte';
  import { auth, db } from '$lib/firebase';
  import { APP_VERSION } from '$lib';
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

  let viewingUser = $state<any>(null);
  let userDecks = $state<any[]>([]);
  let loadingDecks = $state(false);
  let expandedDeckId = $state<string | null>(null);

  // 卡牌資訊查詢表：cardId -> { name, setCode, collectorNumber }
  interface CardInfo {
    name: string;
    setCode: string;
    collectorNumber: string;
  }
  let cardInfoMap = $state<Record<string, CardInfo>>({});
  let cardNameLoaded = false;

  const MAIN_SITE = 'https://suenz001.github.io/ptcg-tw-sim';

  async function loadCardNames() {
    if (cardNameLoaded) return;
    try {
      const indexRes = await fetch(`${MAIN_SITE}/cards/index.json`);
      const sets: any[] = await indexRes.json();
      const map: Record<string, CardInfo> = {};
      await Promise.all(sets.map(async (s: any) => {
        try {
          const res = await fetch(`${MAIN_SITE}/cards/${s.code}.json`);
          const cards: any[] = await res.json();
          for (const c of cards) {
            map[c.id] = {
              name: c.name,
              setCode: c.setCode || s.code,
              collectorNumber: c.collectorNumber || ''
            };
          }
        } catch { /* skip failed sets */ }
      }));
      cardInfoMap = map;
      cardNameLoaded = true;
    } catch (err) {
      console.error('Failed to load card names:', err);
    }
  }

  // 回傳卡牌名稱（含版本與卡號），例如：奇諾栗鼠ex M4 · 071/083
  function getCardLabel(cardId: string): string {
    const info = cardInfoMap[cardId];
    if (!info) return `#${cardId}`;
    const setInfo = info.collectorNumber
      ? `${info.setCode} · ${info.collectorNumber}`
      : info.setCode;
    return `${info.name} ${setInfo}`;
  }

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

  async function loadUserDecks(u: any) {
    viewingUser = u;
    loadingDecks = true;
    userDecks = [];
    expandedDeckId = null;
    // Load card names in parallel
    loadCardNames();
    try {
      const snap = await getDocs(collection(db, 'users', u.id, 'decks'));
      userDecks = snap.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    } catch (err: any) {
      console.error(err);
      alert('無法載入牌組：' + err.message);
    } finally {
      loadingDecks = false;
    }
  }

  function closeDecks() {
    viewingUser = null;
  }

  // Data Fetching
  async function loadData() {
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

  // Auto load data on mount
  onMount(() => {
    loadData();
  });

  function formatDate(ts: any) {
    if (!ts) return '未知';
    return new Date(ts.toDate()).toLocaleString('zh-TW');
  }

  // 解析 userAgent 成簡短的瀏覽器/平台描述
  function parseBrowser(ua: string | undefined): string {
    if (!ua) return '-';
    if (/Mobile|Android|iPhone|iPad/.test(ua)) {
      if (/iPhone|iPad/.test(ua)) return '📱 iOS';
      return '📱 Android';
    }
    if (/Edg\//.test(ua)) return '🌐 Edge';
    if (/OPR\/|Opera/.test(ua)) return '🌐 Opera';
    if (/Chrome\//.test(ua)) return '🌐 Chrome';
    if (/Firefox\//.test(ua)) return '🦊 Firefox';
    if (/Safari\//.test(ua)) return '🧭 Safari';
    return '❓ 其他';
  }
</script>

<main>
    <header class="admin-header">
      <div class="header-content">
        <h1>PTCG 後台系統 (公開測試版) <span class="version">{APP_VERSION}</span></h1>
        <div class="user-info">
          <span>訪客模式</span>
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
                    <th>裝置ID</th>
                    <th>瀏覽器</th>
                    <th>登入次數</th>
                    <th>最後登入時間</th>
                    <th>操作</th>
                  </tr>
                </thead>
                <tbody>
                  {#each users as u}
                    <tr>
                      <td class="mono">{u.id}</td>
                      <td>{u.isAnonymous ? '匿名' : '會員'}</td>
                      <td>{u.email || '-'}</td>
                      <td class="mono device-id" title={u.deviceId || '未知'}>{u.deviceId ? u.deviceId.slice(0, 8) : '-'}</td>
                      <td class="browser-cell">{parseBrowser(u.userAgent)}</td>
                      <td>{u.loginCount || 1}</td>
                      <td>{formatDate(u.lastLoginAt || u.createdAt)}</td>
                      <td>
                        <button class="action-btn" onclick={() => loadUserDecks(u)}>檢視牌組</button>
                      </td>
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

  {#if viewingUser}
    <div class="modal-overlay" onclick={closeDecks}>
      <div class="modal-content" onclick={(e)=>e.stopPropagation()}>
        <div class="modal-header">
          <h2>{viewingUser.email || viewingUser.id} 的牌組</h2>
          <button class="close-btn" onclick={closeDecks}>✕</button>
        </div>
        <div class="modal-body">
          {#if loadingDecks}
            <p>載入中...</p>
          {:else if userDecks.length === 0}
            <p>該玩家目前沒有建立任何牌組。</p>
          {:else}
            <ul class="deck-list">
              {#each userDecks as deck}
                {@const totalCards = deck.entries ? deck.entries.reduce((sum, e) => sum + e.count, 0) : 0}
                <li class="deck-item">
                  <button class="deck-header-btn" onclick={() => expandedDeckId = expandedDeckId === deck.id ? null : deck.id}>
                    <span class="deck-name">{deck.name || '未命名牌組'}</span>
                    <span class="deck-count">{totalCards} 張卡牌</span>
                    <span class="expand-icon">{expandedDeckId === deck.id ? '▲' : '▼'}</span>
                  </button>
                  {#if expandedDeckId === deck.id && deck.entries && deck.entries.length > 0}
                    <div class="deck-detail">
                      <table class="deck-table">
                        <thead>
                          <tr>
                            <th>卡牌名稱</th>
                            <th>數量</th>
                          </tr>
                        </thead>
                        <tbody>
                          {#each deck.entries as e}
                            <tr>
                              <td>{getCardLabel(e.cardId)}</td>
                              <td>x{e.count}</td>
                            </tr>
                          {/each}
                        </tbody>
                      </table>
                    </div>
                  {/if}
                </li>
              {/each}
            </ul>
          {/if}
        </div>
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
  .header-content h1 .version {
    font-size: 0.85rem;
    font-weight: normal;
    opacity: 0.65;
    margin-left: 0.4rem;
    letter-spacing: 0.03em;
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
  .device-id {
    cursor: help;
    letter-spacing: 0.05em;
  }
  .browser-cell {
    font-size: 0.9rem;
    white-space: nowrap;
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

  /* 模態視窗與操作按鈕 */
  .action-btn { padding: 0.4rem 0.8rem; background: #28a745; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 0.85rem; }
  .action-btn:hover { background: #218838; }

  .modal-overlay {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }
  .modal-content {
    background: white;
    border-radius: 12px;
    width: 90%;
    max-width: 600px;
    max-height: 80vh;
    display: flex;
    flex-direction: column;
    box-shadow: 0 10px 30px rgba(0,0,0,0.2);
  }
  .modal-header {
    padding: 1.5rem;
    border-bottom: 1px solid #eee;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .modal-header h2 { margin: 0; font-size: 1.25rem; }
  .close-btn {
    background: none; border: none; font-size: 1.5rem; cursor: pointer; color: #888;
  }
  .modal-body {
    padding: 1.5rem;
    overflow-y: auto;
  }
  .deck-list {
    list-style: none; padding: 0; margin: 0; display: flex; flex-direction: column; gap: 0.75rem;
  }
  .deck-item {
    border: 1px solid #eee; border-radius: 8px; background: #fcfcfc; overflow: hidden;
  }
  .deck-header-btn {
    width: 100%; padding: 1rem; background: none; border: none; cursor: pointer;
    display: flex; align-items: center; gap: 1rem; text-align: left; font: inherit;
  }
  .deck-header-btn:hover { background: #f0f5ff; }
  .deck-name { font-weight: bold; font-size: 1.1rem; color: #0066cc; flex: 1; }
  .deck-count { font-size: 0.85rem; color: #666; }
  .expand-icon { font-size: 0.8rem; color: #aaa; }
  .deck-detail { padding: 0 1rem 1rem; }
  .deck-table { width: 100%; border-collapse: collapse; }
  .deck-table th, .deck-table td { padding: 0.5rem 0.75rem; border-bottom: 1px solid #eee; text-align: left; font-size: 0.9rem; }
  .deck-table th { background: #f8f9fa; color: #555; font-weight: 600; }
  .deck-table td:last-child { text-align: center; width: 60px; }
</style>
