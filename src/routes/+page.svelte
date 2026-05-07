<script lang="ts">
  import { onMount } from 'svelte';
  import { auth, db } from '$lib/firebase';
  import { APP_VERSION } from '$lib';
  import {
    signInWithEmailAndPassword, signOut,
    GoogleAuthProvider, signInWithPopup, onAuthStateChanged
  } from 'firebase/auth';
  import {
    collection, getDocs, query, orderBy, limit, getCountFromServer,
    doc, updateDoc, deleteDoc, serverTimestamp,
  } from 'firebase/firestore';

  // ── Auth ───────────────────────────────────────────────────────────────────
  const ADMIN_EMAIL = 'suenz001@yahoo.com.tw';
  let currentUser  = $state<any>(null);
  let authLoading  = $state(true);
  let email        = $state('');
  let password     = $state('');
  let loginError   = $state('');
  let isLoggingIn  = $state(false);

  // ── Tab & 資料 ─────────────────────────────────────────────────────────────
  let activeTab      = $state<'overview' | 'users' | 'feedback' | 'battles' | 'firebase'>('overview');
  let users          = $state<any[]>([]);
  let feedbacks      = $state<any[]>([]);
  let loadingData    = $state(false);
  let totalUserCount = $state(0);
  let lastRefreshed  = $state<Date | null>(null);

  // 玩家列表篩選
  let searchQuery   = $state('');
  let filterType    = $state<'all' | 'member' | 'anonymous'>('all');
  let filterBrowser = $state('all');

  // 玩家列表排序
  let sortCol = $state<string>('lastLoginAt');
  let sortDir = $state<'asc' | 'desc'>('desc');

  function toggleSort(col: string) {
    if (sortCol === col) { sortDir = sortDir === 'desc' ? 'asc' : 'desc'; }
    else { sortCol = col; sortDir = 'desc'; }
  }
  function sortIcon(col: string) {
    if (sortCol !== col) return '↕';
    return sortDir === 'desc' ? '↓' : '↑';
  }

  // 裝置ID合併模式
  let mergeByDevice        = $state(false);
  let expandedDeviceGroups = $state<string[]>([]);

  // 對戰紀錄
  let rooms        = $state<any[]>([]);
  let loadingRooms = $state(false);
  let battleFilter = $state<'all' | 'playing' | 'ended' | 'lobby'>('all');

  // 牌組檢視
  let viewingUser    = $state<any>(null);
  let userDecks      = $state<any[]>([]);
  let loadingDecks   = $state(false);
  let expandedDeckId = $state<string | null>(null);
  let viewingRoom    = $state<any>(null);

  // Firebase 用量估算
  let adminReadCount = $state(0);
  let adminLoadCount = $state(0);

  // ── 意見回覆（v2.53 配合主站新增）─────────────────────────────────────────
  let replyDrafts  = $state<Record<string, string>>({});
  let savingFbId   = $state<string | null>(null);

  async function submitReply(fbId: string) {
    const replyText = replyDrafts[fbId]?.trim();
    if (!replyText || savingFbId) return;
    savingFbId = fbId;
    try {
      await updateDoc(doc(db, 'feedbacks', fbId), {
        reply: replyText,
        repliedAt: serverTimestamp(),
        repliedBy: currentUser?.email ?? 'admin',
      });
      replyDrafts[fbId] = '';
      await loadData();
    } catch (err: any) {
      alert('送出回覆失敗：' + err.message);
    } finally {
      savingFbId = null;
    }
  }

  async function deleteFeedback(fbId: string) {
    if (!confirm('確定要刪除這筆回饋嗎？此動作無法復原。')) return;
    savingFbId = fbId;
    try {
      await deleteDoc(doc(db, 'feedbacks', fbId));
      await loadData();
    } catch (err: any) {
      alert('刪除失敗：' + err.message);
    } finally {
      savingFbId = null;
    }
  }

  function startEditReply(fbId: string, existingReply: string) {
    replyDrafts[fbId] = existingReply;
  }

  // 卡牌對應表
  interface CardInfo { name: string; setCode: string; collectorNumber: string; }
  let cardInfoMap    = $state<Record<string, CardInfo>>({});
  let cardNameLoaded = false;
  const MAIN_SITE    = 'https://suenz001.github.io/ptcg-tw-sim';

  // ── 總覽統計（從已載入資料計算，不額外消耗 quota）─────────────────────────
  let overviewStats = $derived((() => {
    const now = Date.now();
    const h24 = now - 86400000;
    const tsMs = (ts: any) => ts?.toMillis?.() ?? 0;

    const memberCount  = users.filter(u => !u.isAnonymous).length;
    const anonCount    = users.filter(u => u.isAnonymous).length;
    const playing      = rooms.filter(r => r.status === 'playing').length;
    const lobby        = rooms.filter(r => r.status === 'lobby').length;
    const ended        = rooms.filter(r => r.status === 'ended').length;

    // 24h 活躍（根據 lastLoginAt，僅限已載入 100 筆）
    const activeUsers24h  = users.filter(u => tsMs(u.lastLoginAt || u.createdAt) > h24).length;
    // 24h 新增對戰（根據 updatedAt）
    const newBattles24h   = rooms.filter(r => tsMs(r.updatedAt) > h24).length;
    // 24h 新增回饋
    const newFeedbacks24h = feedbacks.filter(f => tsMs(f.createdAt) > h24).length;

    // 玩家登入次數統計（從已載入資料）
    const totalLogins = users.reduce((s, u) => s + (u.loginCount || 1), 0);
    const avgLogins   = users.length > 0 ? (totalLogins / users.length).toFixed(1) : '0';

    return {
      memberCount, anonCount, playing, lobby, ended,
      activeUsers24h, newBattles24h, newFeedbacks24h,
      totalLogins, avgLogins,
    };
  })());

  // ── Auth ───────────────────────────────────────────────────────────────────
  async function handleLogin() {
    isLoggingIn = true; loginError = '';
    try { await signInWithEmailAndPassword(auth, email, password); }
    catch (err: any) { loginError = err.message || '登入失敗，請檢查帳號密碼。'; }
    finally { isLoggingIn = false; }
  }
  async function handleGoogleLogin() {
    isLoggingIn = true; loginError = '';
    try { await signInWithPopup(auth, new GoogleAuthProvider()); }
    catch (err: any) { loginError = err.message || 'Google 登入失敗。'; }
    finally { isLoggingIn = false; }
  }
  function handleLogout() { signOut(auth); }

  // ── 卡牌名稱 ────────────────────────────────────────────────────────────────
  async function loadCardNames() {
    if (cardNameLoaded) return;
    try {
      const sets: any[] = await (await fetch(`${MAIN_SITE}/cards/index.json`)).json();
      const map: Record<string, CardInfo> = {};
      await Promise.all(sets.map(async (s: any) => {
        try {
          const cards: any[] = await (await fetch(`${MAIN_SITE}/cards/${s.code}.json`)).json();
          for (const c of cards)
            map[c.id] = { name: c.name, setCode: c.setCode || s.code, collectorNumber: c.collectorNumber || '' };
        } catch { /* skip */ }
      }));
      cardInfoMap = map; cardNameLoaded = true;
    } catch (err) { console.error('Failed to load card names:', err); }
  }
  function getCardLabel(cardId: string): string {
    const info = cardInfoMap[cardId];
    if (!info) return `#${cardId}`;
    return `${info.name} ${info.collectorNumber ? `${info.setCode} · ${info.collectorNumber}` : info.setCode}`;
  }

  // ── 牌組 ────────────────────────────────────────────────────────────────────
  async function loadUserDecks(u: any) {
    viewingUser = u; loadingDecks = true; userDecks = []; expandedDeckId = null;
    loadCardNames();
    try {
      const snap = await getDocs(collection(db, 'users', u.id, 'decks'));
      userDecks = snap.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      adminReadCount += userDecks.length;
    } catch (err: any) { alert('無法載入牌組：' + err.message); }
    finally { loadingDecks = false; }
  }
  function closeDecks()       { viewingUser = null; }
  function openRoomDecks(r: any) { viewingRoom = r; loadCardNames(); }
  function closeRoomDecks()   { viewingRoom = null; }

  // ── 裝置群組 ─────────────────────────────────────────────────────────────────
  function toggleDeviceGroup(deviceId: string) {
    expandedDeviceGroups = expandedDeviceGroups.includes(deviceId)
      ? expandedDeviceGroups.filter(d => d !== deviceId)
      : [...expandedDeviceGroups, deviceId];
  }

  // ── 篩選 + 排序（玩家列表）──────────────────────────────────────────────────
  let filteredUsers = $derived(users.filter(u => {
    if (filterType === 'member'    && u.isAnonymous)  return false;
    if (filterType === 'anonymous' && !u.isAnonymous) return false;
    if (filterBrowser !== 'all' && parseBrowser(u.userAgent) !== filterBrowser) return false;
    if (searchQuery.trim()) {
      const q = searchQuery.trim().toLowerCase();
      if (!u.id?.toLowerCase().includes(q) &&
          !u.email?.toLowerCase().includes(q) &&
          !u.deviceId?.toLowerCase().includes(q)) return false;
    }
    return true;
  }));

  let sortedUsers = $derived((() => {
    const arr = [...filteredUsers];
    const dir = sortDir === 'desc' ? -1 : 1;
    arr.sort((a, b) => {
      let va: any, vb: any;
      switch (sortCol) {
        case 'loginCount':
          va = a.loginCount || 1; vb = b.loginCount || 1; break;
        case 'lastLoginAt':
          va = a.lastLoginAt?.toMillis() ?? a.createdAt?.toMillis() ?? 0;
          vb = b.lastLoginAt?.toMillis() ?? b.createdAt?.toMillis() ?? 0; break;
        case 'type':
          va = a.isAnonymous ? 1 : 0; vb = b.isAnonymous ? 1 : 0; break;
        case 'email':
          va = (a.email || '').toLowerCase(); vb = (b.email || '').toLowerCase(); break;
        case 'browser':
          va = parseBrowser(a.userAgent); vb = parseBrowser(b.userAgent); break;
        default: return 0;
      }
      return va < vb ? dir : va > vb ? -dir : 0;
    });
    return arr;
  })());

  // 裝置ID群組
  interface DeviceGroup {
    deviceId: string; accounts: any[]; totalLogins: number;
    latestLoginTs: any; browser: string; memberCount: number; anonymousCount: number;
  }
  let deviceGroups = $derived((() => {
    const groupMap: Record<string, any[]> = {};
    const noDevice: any[] = [];
    for (const u of filteredUsers) {
      if (u.deviceId) (groupMap[u.deviceId] ??= []).push(u);
      else noDevice.push(u);
    }
    const groups: DeviceGroup[] = Object.entries(groupMap)
      .map(([deviceId, accounts]) => ({
        deviceId, accounts,
        totalLogins: accounts.reduce((s, u) => s + (u.loginCount || 1), 0),
        latestLoginTs: accounts.reduce((latest, u) => {
          const ts = u.lastLoginAt || u.createdAt;
          if (!latest) return ts;
          if (!ts) return latest;
          return ts.toMillis() > latest.toMillis() ? ts : latest;
        }, null as any),
        browser: parseBrowser(accounts[0]?.userAgent),
        memberCount:   accounts.filter(u => !u.isAnonymous).length,
        anonymousCount: accounts.filter(u => u.isAnonymous).length,
      }))
      .sort((a, b) => b.accounts.length - a.accounts.length ||
        (b.latestLoginTs?.toMillis() ?? 0) - (a.latestLoginTs?.toMillis() ?? 0));
    return { groups, noDevice };
  })());

  let filteredRooms = $derived(battleFilter === 'all' ? rooms : rooms.filter(r => r.status === battleFilter));

  // ── 資料讀取 ────────────────────────────────────────────────────────────────
  async function loadData() {
    loadingData = true;
    try {
      try {
        const snap = await getCountFromServer(collection(db, 'users'));
        totalUserCount = snap.data().count;
      } catch { /* fallback */ }

      const usersSnap = await getDocs(query(collection(db, 'users'), orderBy('lastLoginAt', 'desc'), limit(100)));
      users = usersSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      if (totalUserCount === 0) totalUserCount = users.length;

      const roomsSnap = await getDocs(query(collection(db, 'rooms'), orderBy('updatedAt', 'desc'), limit(100)));
      rooms = roomsSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));

      const feedSnap = await getDocs(query(collection(db, 'feedbacks'), orderBy('createdAt', 'desc'), limit(50)));
      feedbacks = feedSnap.docs.map(doc => ({ id: doc.id, ...doc.data() }));

      adminReadCount += users.length + rooms.length + feedbacks.length;
      adminLoadCount++;
      lastRefreshed = new Date();
    } catch (err: any) {
      console.error(err);
      alert('無法載入資料：\n' + err.message);
    } finally { loadingData = false; }
  }

  onMount(() => {
    // 手機偵測（JS 雙保險）
    const checkMobile = () => { isMobile = window.innerWidth <= 768; };
    checkMobile();
    window.addEventListener('resize', checkMobile);

    const unsub = onAuthStateChanged(auth, (user) => {
      currentUser = user; authLoading = false;
      if (user && user.email === ADMIN_EMAIL) loadData();
    });
    return () => { unsub(); window.removeEventListener('resize', checkMobile); };
  });

  function formatDate(ts: any) {
    if (!ts) return '未知';
    return new Date(ts.toDate()).toLocaleString('zh-TW');
  }
  function parseBrowser(ua: string | undefined): string {
    if (!ua) return '-';
    if (/Mobile|Android|iPhone|iPad/.test(ua)) return /iPhone|iPad/.test(ua) ? '📱 iOS' : '📱 Android';
    if (/Edg\//.test(ua)) return '🌐 Edge';
    if (/OPR\/|Opera/.test(ua)) return '🌐 Opera';
    if (/Chrome\//.test(ua)) return '🌐 Chrome';
    if (/Firefox\//.test(ua)) return '🦊 Firefox';
    if (/Safari\//.test(ua)) return '🧭 Safari';
    return '❓ 其他';
  }

  // ── 手機偵測 ──────────────────────────────────────────────────────────────
  let isMobile = $state(false);

  const SPARK_DAILY_READS = 50000;
  let readPercent = $derived(Math.min(100, (adminReadCount / SPARK_DAILY_READS) * 100));
</script>

<!-- ══════════════════ AUTH GATE ══════════════════ -->
{#if authLoading}
  <div class="auth-loading">
    <div class="spinner"></div>
    <p>驗證身份中...</p>
  </div>

{:else if !currentUser}
  <div class="login-container">
    <div class="login-box">
      <h1>🛡️ PTCG 後台系統</h1>
      <p class="login-sub">僅限管理員登入</p>
      <form onsubmit={(e) => { e.preventDefault(); handleLogin(); }}>
        <input type="email" placeholder="管理員 Email" bind:value={email} required />
        <input type="password" placeholder="密碼" bind:value={password} required />
        <button type="submit" disabled={isLoggingIn}>{isLoggingIn ? '登入中...' : '登入'}</button>
      </form>
      <div class="divider">或</div>
      <button class="google-btn" onclick={handleGoogleLogin} disabled={isLoggingIn}>🔵 以 Google 帳號登入</button>
      {#if loginError}<p class="error">{loginError}</p>{/if}
    </div>
  </div>

{:else if currentUser.email !== ADMIN_EMAIL}
  <div class="login-container">
    <div class="login-box">
      <h1>🚫 無管理員權限</h1>
      <p style="color:#888; margin:1rem 0">登入帳號：<strong>{currentUser.email}</strong></p>
      <button onclick={handleLogout} style="background:#dc3545;">登出並切換帳號</button>
    </div>
  </div>

{:else}
<!-- ══════════════════ ADMIN DASHBOARD ══════════════════ -->
<main>
  <header class="admin-header">
    <div class="header-content">
      <h1>PTCG 後台系統 <span class="version">{APP_VERSION}</span></h1>
      <div class="user-info">
        <span>👤 {currentUser.email}</span>
        <button class="logout-btn" onclick={handleLogout}>登出</button>
      </div>
    </div>
  </header>

  <div class="dashboard" class:mobile={isMobile}>
    <aside class="sidebar" class:mobile={isMobile}>
      <button class:active={activeTab === 'overview'}  onclick={() => activeTab = 'overview'}>📊 總覽</button>
      <button class:active={activeTab === 'users'}     onclick={() => activeTab = 'users'}>👥 玩家列表</button>
      <button class:active={activeTab === 'feedback'}  onclick={() => activeTab = 'feedback'}>💬 意見回饋</button>
      <button class:active={activeTab === 'battles'}   onclick={() => activeTab = 'battles'}>🎮 對戰紀錄</button>
      <button class:active={activeTab === 'firebase'}  onclick={() => activeTab = 'firebase'}>🔥 Firebase 用量</button>
      <button class="refresh-btn" onclick={loadData} disabled={loadingData}>
        {loadingData ? '⏳ 載入中...' : '🔄 重新整理'}
      </button>
      {#if lastRefreshed}
        <div class="last-refresh">上次：{lastRefreshed.toLocaleTimeString('zh-TW')}</div>
      {/if}
    </aside>

    <div class="content">
      {#if loadingData}
        <div class="loading">正在載入資料...</div>

      <!-- ════════════ 總覽 ════════════ -->
      {:else if activeTab === 'overview'}
        <section>
          <div class="overview-title-row">
            <h2>系統總覽</h2>
            {#if lastRefreshed}
              <span class="refresh-time">上次更新：{lastRefreshed.toLocaleTimeString('zh-TW')}</span>
            {/if}
          </div>

          <!-- 用戶統計 -->
          <div class="ov-section-label">👥 用戶統計</div>
          <div class="ov-grid">
            <div class="ov-card ov-primary">
              <div class="ov-num">{totalUserCount}</div>
              <div class="ov-label">總人數</div>
              <div class="ov-sub">精確統計</div>
            </div>
            <div class="ov-card">
              <div class="ov-num">{overviewStats.memberCount}</div>
              <div class="ov-label">🧑 會員（有 Email）</div>
              <div class="ov-sub">最新 {users.length} 筆中</div>
            </div>
            <div class="ov-card">
              <div class="ov-num">{overviewStats.anonCount}</div>
              <div class="ov-label">👤 匿名訪客</div>
              <div class="ov-sub">最新 {users.length} 筆中</div>
            </div>
            <div class="ov-card ov-highlight">
              <div class="ov-num">+{overviewStats.activeUsers24h}</div>
              <div class="ov-label">過去 24h 活躍</div>
              <div class="ov-sub">最新 {users.length} 筆估算</div>
            </div>
          </div>

          <!-- 對戰統計 -->
          <div class="ov-section-label">🎮 對戰統計（最新 100 筆）</div>
          <div class="ov-grid">
            <div class="ov-card">
              <div class="ov-num">{rooms.length}</div>
              <div class="ov-label">已載入對戰紀錄</div>
            </div>
            <div class="ov-card ov-green">
              <div class="ov-num">{overviewStats.playing}</div>
              <div class="ov-label">⚔️ 進行中</div>
            </div>
            <div class="ov-card ov-orange">
              <div class="ov-num">{overviewStats.lobby}</div>
              <div class="ov-label">🏠 等待中</div>
            </div>
            <div class="ov-card">
              <div class="ov-num">{overviewStats.ended}</div>
              <div class="ov-label">✅ 已結束</div>
            </div>
            <div class="ov-card ov-highlight">
              <div class="ov-num">+{overviewStats.newBattles24h}</div>
              <div class="ov-label">過去 24h 新增對戰</div>
              <div class="ov-sub">最新 100 筆估算</div>
            </div>
          </div>

          <!-- 回饋統計 -->
          <div class="ov-section-label">💬 意見回饋（最新 50 筆）</div>
          <div class="ov-grid">
            <div class="ov-card">
              <div class="ov-num">{feedbacks.length}</div>
              <div class="ov-label">已載入回饋</div>
            </div>
            <div class="ov-card ov-highlight">
              <div class="ov-num">+{overviewStats.newFeedbacks24h}</div>
              <div class="ov-label">過去 24h 新增</div>
            </div>
          </div>

          <!-- 最新 3 筆回饋預覽 -->
          {#if feedbacks.length > 0}
            <div class="ov-section-label">📝 最新回饋預覽</div>
            <div class="ov-feedback-list">
              {#each feedbacks.slice(0, 3) as fb}
                <div class="ov-feedback-card">
                  <div class="ov-feedback-meta">
                    <span>{formatDate(fb.createdAt)}</span>
                    <span class="mono" style="font-size:0.8rem; color:#aaa">{fb.uid?.slice(0, 8)}…</span>
                  </div>
                  <div class="ov-feedback-content">{fb.content?.slice(0, 120)}{fb.content?.length > 120 ? '…' : ''}</div>
                </div>
              {/each}
              <button class="ov-more-btn" onclick={() => activeTab = 'feedback'}>查看全部回饋 →</button>
            </div>
          {/if}
        </section>

      <!-- ════════════ 玩家列表 ════════════ -->
      {:else if activeTab === 'users'}
        <section>
          <h2>玩家列表（最新 {users.length} 筆 / 共 {totalUserCount} 人）</h2>
          <div class="filter-bar">
            <input type="text" class="filter-input" placeholder="搜尋 UID / Email / 裝置ID..."
                   bind:value={searchQuery} />
            <select class="filter-select" bind:value={filterType}>
              <option value="all">全部帳號</option>
              <option value="member">僅有 Email 會員</option>
              <option value="anonymous">僅匿名訪客</option>
            </select>
            <select class="filter-select" bind:value={filterBrowser}>
              <option value="all">全部瀏覽器</option>
              <option value="🌐 Chrome">Chrome</option>
              <option value="🦊 Firefox">Firefox</option>
              <option value="🧭 Safari">Safari</option>
              <option value="🌐 Edge">Edge</option>
              <option value="🌐 Opera">Opera</option>
              <option value="📱 iOS">iOS</option>
              <option value="📱 Android">Android</option>
              <option value="❓ 其他">其他</option>
            </select>
            <button class="merge-toggle-btn" class:active={mergeByDevice}
                    onclick={() => { mergeByDevice = !mergeByDevice; expandedDeviceGroups = []; }}>
              🔗 合併相同裝置
            </button>
            <span class="filter-count">{filteredUsers.length} / {users.length} 筆</span>
          </div>

          {#if !mergeByDevice}
            <div class="table-container">
              <table>
                <thead>
                  <tr>
                    <th class="sortable" onclick={() => toggleSort('type')}>帳號類型 {sortIcon('type')}</th>
                    <th>UID</th>
                    <th class="sortable" onclick={() => toggleSort('email')}>Email {sortIcon('email')}</th>
                    <th>裝置ID</th>
                    <th class="sortable" onclick={() => toggleSort('browser')}>瀏覽器 {sortIcon('browser')}</th>
                    <th class="sortable" onclick={() => toggleSort('loginCount')}>登入次數 {sortIcon('loginCount')}</th>
                    <th class="sortable" onclick={() => toggleSort('lastLoginAt')}>最後登入時間 {sortIcon('lastLoginAt')}</th>
                    <th>操作</th>
                  </tr>
                </thead>
                <tbody>
                  {#each sortedUsers as u}
                    <tr>
                      <td>
                        {#if u.isAnonymous}
                          <span class="type-anon">匿名</span>
                        {:else}
                          <span class="type-member">會員</span>
                        {/if}
                      </td>
                      <td class="mono uid-cell" title={u.id}>{u.id.slice(0, 12)}…</td>
                      <td class="email-cell">{#if u.email}{u.email}{:else}<span style="color:#ccc">—</span>{/if}</td>
                      <td class="mono device-id" title={u.deviceId || '未知'}>{u.deviceId ? u.deviceId.slice(0, 8) : '—'}</td>
                      <td class="browser-cell">{parseBrowser(u.userAgent)}</td>
                      <td class="num-cell">{u.loginCount || 1}</td>
                      <td class="date-cell">{formatDate(u.lastLoginAt || u.createdAt)}</td>
                      <td><button class="action-btn" onclick={() => loadUserDecks(u)}>牌組</button></td>
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>

          {:else}
            <!-- 裝置ID合併模式 -->
            <div class="merge-legend">
              <span class="badge-multi">🔗 多帳號</span> 同一裝置有多個帳號
              <span style="margin-left:1rem; color:#888; font-size:0.85rem">
                共 {deviceGroups.groups.length} 個裝置 + {deviceGroups.noDevice.length} 個無裝置記錄
              </span>
            </div>
            <div class="table-container">
              <table>
                <thead>
                  <tr>
                    <th>裝置ID</th><th>帳號數</th><th>帳號類型</th>
                    <th>瀏覽器</th><th>總登入次數</th><th>最後登入時間</th><th>操作</th>
                  </tr>
                </thead>
                <tbody>
                  {#each deviceGroups.groups as group}
                    {@const isExpanded = expandedDeviceGroups.includes(group.deviceId)}
                    {@const isMulti = group.accounts.length > 1}
                    <tr class="group-row" class:group-multi={isMulti}>
                      <td class="mono device-id" title={group.deviceId}>{group.deviceId.slice(0, 8)}</td>
                      <td>{#if isMulti}<span class="badge-multi">🔗 {group.accounts.length}</span>{:else}<span class="badge-single">1</span>{/if}</td>
                      <td class="account-type-cell">
                        {#if group.memberCount > 0}<span class="type-member">{group.memberCount} 會員</span>{/if}
                        {#if group.anonymousCount > 0}<span class="type-anon">{group.anonymousCount} 匿名</span>{/if}
                      </td>
                      <td class="browser-cell">{group.browser}</td>
                      <td class="num-cell">{group.totalLogins}</td>
                      <td class="date-cell">{formatDate(group.latestLoginTs)}</td>
                      <td>
                        {#if isMulti}
                          <button class="expand-group-btn" onclick={() => toggleDeviceGroup(group.deviceId)}>
                            {isExpanded ? '▲' : '▼'}
                          </button>
                        {:else}
                          <button class="action-btn" onclick={() => loadUserDecks(group.accounts[0])}>牌組</button>
                        {/if}
                      </td>
                    </tr>
                    {#if isExpanded}
                      {#each group.accounts as u}
                        <tr class="sub-row">
                          <td class="mono sub-uid" title={u.id}>↳ {u.id.slice(0, 12)}…</td>
                          <td>{#if u.isAnonymous}<span class="type-anon">匿名</span>{:else}<span class="type-member">會員</span>{/if}</td>
                          <td colspan="2">{u.email || '—'}</td>
                          <td class="num-cell">{u.loginCount || 1}</td>
                          <td class="date-cell">{formatDate(u.lastLoginAt || u.createdAt)}</td>
                          <td><button class="action-btn" onclick={() => loadUserDecks(u)}>牌組</button></td>
                        </tr>
                      {/each}
                    {/if}
                  {/each}
                  {#each deviceGroups.noDevice as u}
                    <tr>
                      <td class="mono" style="color:#ccc">—</td>
                      <td>{#if u.isAnonymous}<span class="type-anon">匿名</span>{:else}<span class="type-member">會員</span>{/if}</td>
                      <td>{u.email || '—'}</td>
                      <td class="browser-cell">{parseBrowser(u.userAgent)}</td>
                      <td class="num-cell">{u.loginCount || 1}</td>
                      <td class="date-cell">{formatDate(u.lastLoginAt || u.createdAt)}</td>
                      <td><button class="action-btn" onclick={() => loadUserDecks(u)}>牌組</button></td>
                    </tr>
                  {/each}
                </tbody>
              </table>
            </div>
          {/if}
        </section>

      <!-- ════════════ 意見回饋 ════════════ -->
      {:else if activeTab === 'feedback'}
        <section>
          <h2>意見回饋（最新 50 筆）— 可加回覆給玩家</h2>
          <p class="feedback-intro">
            🛈 玩家在主站打開意見回饋視窗時，會看到自己提交過的歷史回饋與你在這裡寫的回覆。
            送出/編輯/刪除回覆會即時同步到玩家端。
          </p>
          <div class="feedback-list">
            {#each feedbacks as fb}
              <div class="feedback-card" class:has-reply={!!fb.reply}>
                <div class="fb-meta">
                  <span class="fb-time">📅 {formatDate(fb.createdAt)}</span>
                  <span class="fb-uid mono">UID: {fb.uid?.slice(0, 12)}…</span>
                  {#if fb.deviceId}
                    <span class="fb-device mono">device: {fb.deviceId.slice(0, 8)}</span>
                  {/if}
                  {#if fb.reply}<span class="fb-replied-tag">✓ 已回覆</span>{/if}
                </div>
                <div class="fb-content">{fb.content}</div>
                {#if fb.userAgent}
                  <details class="fb-ua">
                    <summary>User Agent</summary>
                    <code>{fb.userAgent}</code>
                  </details>
                {/if}
                {#if fb.reply}
                  <div class="existing-reply">
                    <div class="reply-header">
                      <strong>💬 你的回覆</strong>
                      <span class="reply-time">{formatDate(fb.repliedAt)}</span>
                      {#if fb.repliedBy}<span class="reply-by">by {fb.repliedBy}</span>{/if}
                    </div>
                    <div class="reply-text">{fb.reply}</div>
                  </div>
                {/if}
                <div class="reply-form">
                  <textarea
                    placeholder={fb.reply ? '編輯回覆內容...' : '輸入回覆內容...'}
                    bind:value={replyDrafts[fb.id]}
                    rows="3"
                    disabled={savingFbId === fb.id}
                  ></textarea>
                  <div class="reply-actions">
                    {#if fb.reply && !replyDrafts[fb.id]}
                      <button class="btn-edit" onclick={() => startEditReply(fb.id, fb.reply ?? '')}>編輯既有回覆</button>
                    {/if}
                    <button
                      class="btn-submit-reply"
                      onclick={() => submitReply(fb.id)}
                      disabled={!replyDrafts[fb.id]?.trim() || savingFbId === fb.id}
                    >
                      {savingFbId === fb.id ? '送出中...' : (fb.reply ? '更新回覆' : '送出回覆')}
                    </button>
                    <button
                      class="btn-delete-fb"
                      onclick={() => deleteFeedback(fb.id)}
                      disabled={savingFbId === fb.id}
                    >🗑 刪除</button>
                  </div>
                </div>
              </div>
            {/each}
            {#if feedbacks.length === 0}<p>目前沒有任何意見回饋。</p>{/if}
          </div>
        </section>

      <!-- ════════════ 對戰紀錄 ════════════ -->
      {:else if activeTab === 'battles'}
        <section>
          <h2>對戰紀錄（最新 100 筆）</h2>
          <div class="filter-bar">
            <button class="battle-filter-btn" class:active={battleFilter === 'all'}     onclick={() => battleFilter = 'all'}>全部 ({rooms.length})</button>
            <button class="battle-filter-btn playing" class:active={battleFilter === 'playing'} onclick={() => battleFilter = 'playing'}>⚔️ 進行中 ({rooms.filter(r=>r.status==='playing').length})</button>
            <button class="battle-filter-btn lobby"   class:active={battleFilter === 'lobby'}   onclick={() => battleFilter = 'lobby'}>🏠 等待中 ({rooms.filter(r=>r.status==='lobby').length})</button>
            <button class="battle-filter-btn ended"   class:active={battleFilter === 'ended'}   onclick={() => battleFilter = 'ended'}>✅ 已結束 ({rooms.filter(r=>r.status==='ended').length})</button>
          </div>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>房號</th><th>房間名稱</th><th>P1</th><th>P2</th>
                  <th>狀態</th><th>勝者</th><th>回合</th><th>最後更新</th><th>牌組</th>
                </tr>
              </thead>
              <tbody>
                {#each filteredRooms as room}
                  {@const p1 = room.seats?.[0]}
                  {@const p2 = room.seats?.[1]}
                  {@const gs = room.gameState}
                  {@const winnerName = gs?.winner === 0 ? p1?.name : gs?.winner === 1 ? p2?.name : null}
                  <tr>
                    <td class="mono room-code">{room.id}</td>
                    <td>{room.roomName || '-'}</td>
                    <td class:winner-cell={gs?.winner === 0}>{p1?.name || '（空）'}</td>
                    <td class:winner-cell={gs?.winner === 1}>{p2?.name || '（空）'}</td>
                    <td><span class="status-badge status-{room.status}">
                      {room.status === 'playing' ? '⚔️ 進行中' : room.status === 'lobby' ? '🏠 等待中' : '✅ 已結束'}
                    </span></td>
                    <td class="winner-col">{winnerName ? `🏆 ${winnerName}` : '-'}</td>
                    <td class="num-cell">{gs?.turn ?? '-'}</td>
                    <td class="date-cell">{formatDate(room.updatedAt)}</td>
                    <td>
                      {#if room.seats?.[0]?.deckEntries || room.seats?.[1]?.deckEntries}
                        <button class="action-btn" onclick={() => openRoomDecks(room)}>牌組</button>
                      {:else}
                        <span style="color:#ccc">—</span>
                      {/if}
                    </td>
                  </tr>
                {/each}
              </tbody>
            </table>
          </div>
          {#if filteredRooms.length === 0}
            <p style="text-align:center; color:#888; margin-top:2rem;">目前沒有符合條件的對戰紀錄</p>
          {/if}
        </section>

      <!-- ════════════ Firebase 用量 ════════════ -->
      {:else if activeTab === 'firebase'}
        <section>
          <h2>🔥 Firebase 用量監控</h2>
          <div class="firebase-card">
            <h3>本次 Admin Session 讀取估算</h3>
            <div class="firebase-stats-grid">
              <div class="firebase-stat">
                <div class="firebase-stat-value">{adminReadCount}</div>
                <div>Firestore 讀取次數</div>
                <div class="firebase-stat-percent">≈ {readPercent.toFixed(2)}% / 每日免費額度</div>
              </div>
              <div class="firebase-stat">
                <div class="firebase-stat-value">{adminLoadCount}</div>
                <div>重新整理次數</div>
              </div>
            </div>
            <p class="firebase-note">⚠️ 此數據為前端估算，僅計算本次 session 由 Admin 頁面發起的 Firestore 讀取。實際用量請至 Firebase Console 查看。</p>
          </div>
          <div class="firebase-card">
            <h3>Firebase Spark（免費方案）每日額度對照</h3>
            <table class="firebase-quota-table">
              <thead><tr><th>項目</th><th>免費額度</th><th>說明</th></tr></thead>
              <tbody>
                <tr class="quota-row-highlight">
                  <td>📖 Firestore 讀取</td><td class="quota-value">50,000 次 / 天</td><td class="quota-desc">玩家載入牌組、對戰紀錄、回饋等皆計入</td>
                </tr>
                <tr><td>✏️ Firestore 寫入</td><td class="quota-value">20,000 次 / 天</td><td class="quota-desc">建立/更新牌組、對戰狀態、回饋等</td></tr>
                <tr><td>🗑️ Firestore 刪除</td><td class="quota-value">20,000 次 / 天</td><td class="quota-desc">刪除牌組、對戰房間等</td></tr>
                <tr><td>💾 Firestore 儲存空間</td><td class="quota-value">1 GiB</td><td class="quota-desc">所有文件資料總量（含索引）</td></tr>
                <tr><td>🌐 Firestore 輸出流量</td><td class="quota-value">10 GiB / 月</td><td class="quota-desc">資料從 Firebase 傳到用戶端的總量</td></tr>
                <tr><td>🔐 Firebase Auth</td><td class="quota-value">無限制</td><td class="quota-desc">匿名登入、Email / Google 登入皆免費</td></tr>
              </tbody>
            </table>
          </div>
          <div class="firebase-card">
            <h3>Firebase Console 快捷連結</h3>
            <div class="firebase-links">
              <a class="firebase-link-btn firebase-link-primary" href="https://console.firebase.google.com/project/ptcg-tw-sim/usage"                    target="_blank" rel="noopener">📊 用量總覽</a>
              <a class="firebase-link-btn" href="https://console.firebase.google.com/project/ptcg-tw-sim/firestore"                                      target="_blank" rel="noopener">🗄️ Firestore 資料庫</a>
              <a class="firebase-link-btn" href="https://console.firebase.google.com/project/ptcg-tw-sim/authentication/users"                           target="_blank" rel="noopener">👤 Authentication 用戶</a>
              <a class="firebase-link-btn" href="https://console.firebase.google.com/project/ptcg-tw-sim/firestore/usage"                                target="_blank" rel="noopener">📈 Firestore 用量圖表</a>
            </div>
          </div>
        </section>
      {/if}
    </div>
  </div>

  <!-- ── 對戰牌組 Modal ── -->
  {#if viewingRoom}
    <div class="modal-overlay" onclick={closeRoomDecks}>
      <div class="modal-content modal-wide" onclick={(e)=>e.stopPropagation()}>
        <div class="modal-header">
          <h2>🎮 {viewingRoom.roomName || viewingRoom.id} — 使用牌組</h2>
          <button class="close-btn" onclick={closeRoomDecks}>✕</button>
        </div>
        <div class="modal-body">
          {#if !cardNameLoaded}<p>載入卡名中...</p>
          {:else}
            <div class="battle-decks-grid">
              {#each [0, 1] as playerIdx}
                {@const seat = viewingRoom.seats?.[playerIdx]}
                {@const isWinner = viewingRoom.gameState?.winner === playerIdx}
                <div class="battle-deck-col">
                  <div class="battle-deck-header" class:is-winner={isWinner}>
                    <span class="player-label">P{playerIdx+1}</span>
                    <span class="player-name">{seat?.name || '（空）'}</span>
                    {#if isWinner}<span class="winner-badge">🏆 勝者</span>{/if}
                    <span class="deck-total">{seat?.deckEntries?.reduce((s:number,e:any)=>s+e.count,0) ?? 0} 張</span>
                  </div>
                  {#if seat?.deckEntries?.length > 0}
                    <table class="deck-table">
                      <thead><tr><th>卡牌名稱</th><th>數量</th></tr></thead>
                      <tbody>
                        {#each seat.deckEntries as e}
                          <tr><td>{getCardLabel(e.cardId)}</td><td style="text-align:center">x{e.count}</td></tr>
                        {/each}
                      </tbody>
                    </table>
                  {:else}<p class="no-deck">無牌組資料</p>{/if}
                </div>
              {/each}
            </div>
          {/if}
        </div>
      </div>
    </div>
  {/if}

  <!-- ── 玩家牌組 Modal ── -->
  {#if viewingUser}
    <div class="modal-overlay" onclick={closeDecks}>
      <div class="modal-content" onclick={(e)=>e.stopPropagation()}>
        <div class="modal-header">
          <h2>{viewingUser.email || viewingUser.id} 的牌組</h2>
          <button class="close-btn" onclick={closeDecks}>✕</button>
        </div>
        <div class="modal-body">
          {#if loadingDecks}<p>載入中...</p>
          {:else if userDecks.length === 0}<p>該玩家目前沒有建立任何牌組。</p>
          {:else}
            <ul class="deck-list">
              {#each userDecks as deck}
                {@const totalCards = deck.entries?.reduce((s:number,e:any)=>s+e.count,0) ?? 0}
                <li class="deck-item">
                  <button class="deck-header-btn" onclick={() => expandedDeckId = expandedDeckId===deck.id ? null : deck.id}>
                    <span class="deck-name">{deck.name || '未命名牌組'}</span>
                    <span class="deck-count">{totalCards} 張</span>
                    <span class="expand-icon">{expandedDeckId===deck.id ? '▲' : '▼'}</span>
                  </button>
                  {#if expandedDeckId===deck.id && deck.entries?.length > 0}
                    <div class="deck-detail">
                      <table class="deck-table">
                        <thead><tr><th>卡牌名稱</th><th>數量</th></tr></thead>
                        <tbody>
                          {#each deck.entries as e}
                            <tr><td>{getCardLabel(e.cardId)}</td><td>x{e.count}</td></tr>
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
{/if}

<style>
  /* ── Auth 載入 ─────────────────────────────────────────────────────────── */
  .auth-loading { display:flex; flex-direction:column; align-items:center; justify-content:center; height:100vh; gap:1rem; color:#888; }
  .spinner { width:36px; height:36px; border:3px solid #e0e0e0; border-top-color:#0066cc; border-radius:50%; animation:spin 0.8s linear infinite; }
  @keyframes spin { to { transform:rotate(360deg); } }

  /* ── 登入畫面 ──────────────────────────────────────────────────────────── */
  .login-container { display:flex; justify-content:center; align-items:center; height:100vh; background:#f0f4f8; }
  .login-box { background:white; padding:2.5rem; border-radius:12px; box-shadow:0 10px 30px rgba(0,0,0,0.1); width:100%; max-width:400px; text-align:center; }
  .login-box h1 { margin-top:0; font-size:1.5rem; }
  .login-sub { color:#888; font-size:0.9rem; margin:0.25rem 0 0; }
  .login-box form { display:flex; flex-direction:column; gap:1rem; margin-top:1.5rem; }
  .login-box input { padding:0.75rem; border:1px solid #ccc; border-radius:6px; font-size:1rem; }
  .login-box button { padding:0.75rem; background:#0066cc; color:white; border:none; border-radius:6px; font-size:1rem; font-weight:bold; cursor:pointer; }
  .login-box button:disabled { opacity:0.7; cursor:not-allowed; }
  .error { color:#dc3545; font-size:0.9rem; margin-top:1rem; }
  .divider { margin:1.5rem 0; color:#888; position:relative; }
  .divider::before, .divider::after { content:''; position:absolute; top:50%; width:40%; height:1px; background:#ddd; }
  .divider::before { left:0; } .divider::after { right:0; }
  .google-btn { width:100%; padding:0.75rem; background:#fff; color:#444; border:1px solid #ccc; border-radius:6px; font-size:1rem; font-weight:bold; cursor:pointer; box-shadow:0 1px 3px rgba(0,0,0,0.1); }
  .google-btn:hover { background:#f9f9f9; }
  .google-btn:disabled { opacity:0.7; cursor:not-allowed; }

  /* ── 後台佈局 ──────────────────────────────────────────────────────────── */
  .admin-header { background:#1a2a3a; color:white; padding:0.85rem 2rem; }
  .header-content { display:flex; justify-content:space-between; align-items:center; max-width:1400px; margin:0 auto; }
  .header-content h1 { margin:0; font-size:1.2rem; }
  .header-content h1 .version { font-size:0.8rem; font-weight:normal; opacity:0.6; margin-left:0.4rem; }
  .user-info { display:flex; align-items:center; gap:1rem; font-size:0.9rem; }
  .logout-btn { background:rgba(255,255,255,0.1); border:1px solid rgba(255,255,255,0.2); color:white; padding:0.3rem 0.8rem; border-radius:4px; cursor:pointer; }
  .logout-btn:hover { background:rgba(255,255,255,0.2); }

  .dashboard { display:flex; max-width:1400px; margin:0 auto; min-height:calc(100vh - 58px); }
  .sidebar { width:220px; background:white; border-right:1px solid #e0e0e0; padding:1rem 0.75rem; display:flex; flex-direction:column; gap:0.3rem; }
  .sidebar button { text-align:left; padding:0.65rem 0.9rem; background:none; border:none; border-radius:6px; font-size:0.95rem; cursor:pointer; color:#444; }
  .sidebar button:hover { background:#f0f0f0; }
  .sidebar button.active { background:#e6f0fa; color:#0066cc; font-weight:bold; }
  .sidebar .refresh-btn { background:#f0f7ff; color:#0066cc; border:1px solid #c8e0ff; font-size:0.9rem; margin-top:0.5rem; }
  .sidebar .refresh-btn:hover { background:#daeeff; }
  .sidebar .refresh-btn:disabled { opacity:0.6; cursor:not-allowed; }
  .last-refresh { font-size:0.75rem; color:#aaa; text-align:center; padding:0.1rem 0.9rem; }

  .content { flex:1; padding:1.5rem 2rem; overflow-x:auto; }
  .content h2 { margin-top:0; margin-bottom:1.25rem; color:#222; font-size:1.3rem; }

  /* ── 總覽 ──────────────────────────────────────────────────────────────── */
  .overview-title-row { display:flex; align-items:baseline; gap:1rem; margin-bottom:1.25rem; }
  .overview-title-row h2 { margin:0; }
  .refresh-time { font-size:0.82rem; color:#aaa; }
  .ov-section-label { font-size:0.82rem; font-weight:700; color:#888; text-transform:uppercase; letter-spacing:0.08em; margin:1.25rem 0 0.6rem; }
  .ov-grid { display:grid; grid-template-columns:repeat(auto-fill, minmax(160px, 1fr)); gap:1rem; }
  .ov-card {
    background:white; border-radius:10px; padding:1.1rem 1.2rem;
    box-shadow:0 2px 6px rgba(0,0,0,0.06); border-top:3px solid #e0e0e0;
  }
  .ov-card.ov-primary { border-top-color:#0066cc; }
  .ov-card.ov-green   { border-top-color:#28a745; }
  .ov-card.ov-orange  { border-top-color:#fd7e14; }
  .ov-card.ov-highlight { border-top-color:#f6c90e; background:#fffdf0; }
  .ov-num   { font-size:2rem; font-weight:800; color:#222; line-height:1; margin-bottom:0.3rem; }
  .ov-card.ov-primary .ov-num  { color:#0066cc; }
  .ov-card.ov-green   .ov-num  { color:#28a745; }
  .ov-card.ov-orange  .ov-num  { color:#fd7e14; }
  .ov-card.ov-highlight .ov-num { color:#c58c00; }
  .ov-label { font-size:0.85rem; color:#555; }
  .ov-sub   { font-size:0.75rem; color:#aaa; margin-top:0.2rem; }

  .ov-feedback-list { display:flex; flex-direction:column; gap:0.6rem; }
  .ov-feedback-card { background:white; border-radius:8px; padding:0.8rem 1rem; box-shadow:0 1px 4px rgba(0,0,0,0.05); border-left:3px solid #0066cc; }
  .ov-feedback-meta { display:flex; justify-content:space-between; font-size:0.78rem; color:#aaa; margin-bottom:0.35rem; }
  .ov-feedback-content { font-size:0.88rem; color:#444; line-height:1.5; }
  .ov-more-btn { align-self:flex-start; margin-top:0.25rem; background:none; border:none; color:#0066cc; font-size:0.88rem; cursor:pointer; padding:0; }
  .ov-more-btn:hover { text-decoration:underline; }

  /* ── 表格（緊湊版）──────────────────────────────────────────────────────── */
  .table-container { background:white; border-radius:10px; box-shadow:0 2px 6px rgba(0,0,0,0.05); overflow:hidden; }
  table { width:100%; border-collapse:collapse; text-align:left; }
  th, td { padding:0.5rem 0.75rem; border-bottom:1px solid #f0f0f0; font-size:0.88rem; }
  th { background:#f8f9fa; font-weight:600; color:#555; white-space:nowrap; }
  tbody tr:last-child td { border-bottom:none; }
  tbody tr:hover { background:#fafbff; }
  .mono { font-family:ui-monospace, 'Cascadia Code', monospace; font-size:0.82rem; color:#777; }
  .uid-cell { max-width:140px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  .email-cell { max-width:200px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; font-size:0.85rem; }
  .device-id { cursor:help; letter-spacing:0.03em; }
  .browser-cell { white-space:nowrap; font-size:0.85rem; }
  .num-cell { text-align:center; font-weight:600; }
  .date-cell { white-space:nowrap; font-size:0.82rem; color:#666; }

  /* ── 可排序表頭 ────────────────────────────────────────────────────────── */
  .sortable { cursor:pointer; user-select:none; }
  .sortable:hover { background:#e9ecef; color:#0066cc; }

  /* ── 帳號類型標籤 ──────────────────────────────────────────────────────── */
  .type-member { display:inline-block; background:#d4edda; color:#155724; font-size:0.75rem; font-weight:600; padding:0.1rem 0.45rem; border-radius:8px; white-space:nowrap; }
  .type-anon   { display:inline-block; background:#e2e3e5; color:#383d41; font-size:0.75rem; font-weight:600; padding:0.1rem 0.45rem; border-radius:8px; white-space:nowrap; }

  /* ── 意見回饋 ──────────────────────────────────────────────────────────── */
  .feedback-intro { font-size:0.84rem; color:#666; background:#f7f9fc; border-left:3px solid #0066cc; padding:0.6rem 0.9rem; border-radius:4px; margin-bottom:1rem; }
  .feedback-list { display:flex; flex-direction:column; gap:0.75rem; }
  .feedback-card { background:white; padding:1.1rem 1.25rem; border-radius:10px; box-shadow:0 2px 6px rgba(0,0,0,0.05); }
  .feedback-card.has-reply { border-left:3px solid #28a745; }
  .fb-meta { display:flex; flex-wrap:wrap; gap:0.4rem 1rem; margin-bottom:0.6rem; font-size:0.78rem; align-items:center; }
  .fb-time { color:#666; }
  .fb-uid, .fb-device { color:#aaa; }
  .fb-replied-tag { background:#28a745; color:#fff; padding:0.1rem 0.5rem; border-radius:3px; font-weight:600; font-size:0.74rem; }
  .fb-content { line-height:1.6; white-space:pre-wrap; color:#333; font-size:0.9rem; margin-bottom:0.5rem; }
  .fb-ua { font-size:0.75rem; color:#888; margin:0.4rem 0; }
  .fb-ua summary { cursor:pointer; }
  .fb-ua code { display:block; margin-top:0.3rem; padding:0.4rem; word-break:break-all; background:#f8f8f8; border-radius:3px; font-size:0.72rem; }
  .existing-reply { background:#f0f9f0; border:1px solid #c8e6c8; border-radius:6px; padding:0.7rem 0.95rem; margin:0.6rem 0; }
  .reply-header { display:flex; flex-wrap:wrap; gap:0.4rem 0.8rem; align-items:baseline; font-size:0.8rem; color:#2c4a2c; margin-bottom:0.3rem; }
  .reply-time, .reply-by { color:#5a7a5a; font-weight:normal; }
  .reply-text { white-space:pre-wrap; margin:0; color:#1a3a1a; font-size:0.88rem; }
  .reply-form { margin-top:0.5rem; }
  .reply-form textarea { width:100%; box-sizing:border-box; padding:0.55rem; border:1px solid #ccc; border-radius:4px; font-family:inherit; font-size:0.88rem; resize:vertical; }
  .reply-actions { display:flex; gap:0.5rem; margin-top:0.4rem; justify-content:flex-end; }
  .btn-submit-reply, .btn-edit, .btn-delete-fb { padding:0.4rem 0.85rem; border:none; border-radius:4px; cursor:pointer; font-size:0.84rem; }
  .btn-submit-reply { background:#28a745; color:#fff; }
  .btn-submit-reply:disabled { opacity:0.4; cursor:not-allowed; }
  .btn-edit { background:#6688aa; color:#fff; }
  .btn-delete-fb { background:#c66; color:#fff; }
  .btn-delete-fb:disabled { opacity:0.5; cursor:not-allowed; }

  /* ── 按鈕 ──────────────────────────────────────────────────────────────── */
  .action-btn { padding:0.3rem 0.65rem; background:#28a745; color:white; border:none; border-radius:4px; cursor:pointer; font-size:0.82rem; }
  .action-btn:hover { background:#218838; }
  .loading { padding:3rem; color:#888; text-align:center; }

  /* ── 篩選列 ────────────────────────────────────────────────────────────── */
  .filter-bar { display:flex; align-items:center; gap:0.6rem; margin-bottom:0.85rem; flex-wrap:wrap; }
  .filter-input { flex:1; min-width:180px; padding:0.45rem 0.7rem; border:1px solid #ccc; border-radius:6px; font-size:0.88rem; }
  .filter-input:focus { outline:none; border-color:#0066cc; }
  .filter-select { padding:0.45rem 0.7rem; border:1px solid #ccc; border-radius:6px; font-size:0.88rem; background:white; cursor:pointer; }
  .filter-count { font-size:0.82rem; color:#aaa; white-space:nowrap; }

  /* ── 裝置合併 ──────────────────────────────────────────────────────────── */
  .merge-toggle-btn { padding:0.45rem 0.9rem; border:1px solid #ccc; border-radius:6px; background:white; font-size:0.88rem; cursor:pointer; color:#555; white-space:nowrap; }
  .merge-toggle-btn:hover { background:#f0f0f0; }
  .merge-toggle-btn.active { background:#fff3cd; border-color:#ffc107; color:#856404; font-weight:600; }
  .merge-legend { font-size:0.82rem; color:#666; margin-bottom:0.6rem; display:flex; align-items:center; gap:0.5rem; flex-wrap:wrap; }
  .group-row { background:#fafafa; }
  .group-multi { background:#fffbe6; }
  .group-multi:hover { background:#fff8d6; }
  .sub-row td { padding:0.4rem 0.75rem; font-size:0.84rem; border-bottom:1px dashed #eee; background:#f8f9ff; }
  .sub-uid { color:#999; font-size:0.8rem; }
  .badge-multi { display:inline-block; background:#ffc107; color:#5a3e00; font-size:0.75rem; font-weight:700; padding:0.1rem 0.45rem; border-radius:10px; white-space:nowrap; }
  .badge-single { font-size:0.8rem; color:#aaa; }
  .account-type-cell { display:flex; gap:0.35rem; align-items:center; flex-wrap:wrap; }
  .expand-group-btn { padding:0.25rem 0.6rem; border:1px solid #ccc; border-radius:4px; background:white; font-size:0.8rem; cursor:pointer; color:#555; }
  .expand-group-btn:hover { background:#f0f0f0; }

  /* ── 對戰篩選 ──────────────────────────────────────────────────────────── */
  .battle-filter-btn { padding:0.35rem 0.9rem; border:1px solid #ccc; border-radius:20px; background:white; font-size:0.88rem; cursor:pointer; color:#555; }
  .battle-filter-btn:hover { background:#f0f0f0; }
  .battle-filter-btn.active { background:#0066cc; color:white; border-color:#0066cc; }
  .battle-filter-btn.playing.active { background:#28a745; border-color:#28a745; }
  .battle-filter-btn.lobby.active   { background:#fd7e14; border-color:#fd7e14; }
  .battle-filter-btn.ended.active   { background:#6c757d; border-color:#6c757d; }
  .status-badge { display:inline-block; padding:0.15rem 0.5rem; border-radius:10px; font-size:0.78rem; font-weight:600; white-space:nowrap; }
  .status-playing { background:#d4edda; color:#155724; }
  .status-lobby   { background:#fff3cd; color:#856404; }
  .status-ended   { background:#e2e3e5; color:#383d41; }
  .room-code { font-size:0.9rem; font-weight:bold; letter-spacing:0.08em; color:#333; }
  .winner-cell { font-weight:bold; color:#0066cc; }
  .winner-col { color:#28a745; font-weight:600; }

  /* ── Modal ─────────────────────────────────────────────────────────────── */
  .modal-overlay { position:fixed; top:0;left:0;right:0;bottom:0; background:rgba(0,0,0,0.5); display:flex; justify-content:center; align-items:center; z-index:1000; }
  .modal-content { background:white; border-radius:12px; width:90%; max-width:600px; max-height:85vh; display:flex; flex-direction:column; box-shadow:0 10px 30px rgba(0,0,0,0.2); }
  .modal-header { padding:1.25rem 1.5rem; border-bottom:1px solid #eee; display:flex; justify-content:space-between; align-items:center; }
  .modal-header h2 { margin:0; font-size:1.1rem; }
  .close-btn { background:none; border:none; font-size:1.4rem; cursor:pointer; color:#888; }
  .modal-body { padding:1.25rem 1.5rem; overflow-y:auto; }
  .deck-list { list-style:none; padding:0; margin:0; display:flex; flex-direction:column; gap:0.6rem; }
  .deck-item { border:1px solid #eee; border-radius:8px; background:#fcfcfc; overflow:hidden; }
  .deck-header-btn { width:100%; padding:0.8rem 1rem; background:none; border:none; cursor:pointer; display:flex; align-items:center; gap:0.75rem; text-align:left; font:inherit; }
  .deck-header-btn:hover { background:#f0f5ff; }
  .deck-name { font-weight:bold; font-size:1rem; color:#0066cc; flex:1; }
  .deck-count { font-size:0.82rem; color:#666; }
  .expand-icon { font-size:0.75rem; color:#aaa; }
  .deck-detail { padding:0 1rem 0.75rem; }
  .deck-table { width:100%; border-collapse:collapse; }
  .deck-table th, .deck-table td { padding:0.4rem 0.65rem; border-bottom:1px solid #eee; text-align:left; font-size:0.85rem; }
  .deck-table th { background:#f8f9fa; color:#555; font-weight:600; }
  .deck-table td:last-child { text-align:center; width:55px; }

  /* ── 對戰牌組 Modal ────────────────────────────────────────────────────── */
  .modal-wide { max-width:900px; }
  .battle-decks-grid { display:grid; grid-template-columns:1fr 1fr; gap:1.25rem; }
  .battle-deck-col { border:1px solid #eee; border-radius:8px; overflow:hidden; }
  .battle-deck-header { display:flex; align-items:center; gap:0.5rem; padding:0.65rem 0.9rem; background:#f8f9fa; border-bottom:1px solid #eee; flex-wrap:wrap; }
  .battle-deck-header.is-winner { background:#d4edda; border-bottom-color:#c3e6cb; }
  .player-label { font-size:0.72rem; font-weight:bold; background:#0066cc; color:white; padding:0.1rem 0.4rem; border-radius:4px; }
  .player-name { font-weight:bold; font-size:0.95rem; flex:1; }
  .winner-badge { font-size:0.78rem; background:#28a745; color:white; padding:0.12rem 0.45rem; border-radius:10px; }
  .deck-total { font-size:0.82rem; color:#666; margin-left:auto; }
  .no-deck { padding:1rem; color:#aaa; text-align:center; }

  /* ── Firebase 用量 ─────────────────────────────────────────────────────── */
  .firebase-card { background:white; border-radius:10px; box-shadow:0 2px 6px rgba(0,0,0,0.05); padding:1.25rem 1.5rem; margin-bottom:1.25rem; }
  .firebase-card h3 { margin:0 0 1rem 0; font-size:1rem; color:#333; border-bottom:1px solid #eee; padding-bottom:0.6rem; }
  .firebase-stats-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(160px,1fr)); gap:1rem; margin-bottom:1rem; }
  .firebase-stat { background:#f8f9fa; border-radius:8px; padding:1rem; text-align:center; font-size:0.88rem; color:#555; }
  .firebase-stat-value { font-size:1.8rem; font-weight:bold; color:#e25822; margin-bottom:0.2rem; }
  .firebase-stat-percent { font-size:0.78rem; color:#888; margin-top:0.2rem; }
  .firebase-note { font-size:0.82rem; color:#888; background:#fffbe6; border-left:3px solid #f6c90e; padding:0.65rem 0.9rem; border-radius:4px; margin:0; }
  .firebase-quota-table { width:100%; border-collapse:collapse; }
  .firebase-quota-table th, .firebase-quota-table td { padding:0.6rem 0.9rem; border-bottom:1px solid #eee; text-align:left; font-size:0.88rem; }
  .firebase-quota-table th { background:#f8f9fa; font-weight:600; color:#555; }
  .quota-value { font-weight:600; color:#0066cc; white-space:nowrap; }
  .quota-desc { color:#777; font-size:0.82rem; }
  .quota-row-highlight { background:#fff8f0; }
  .quota-row-highlight .quota-value { color:#e25822; }
  .firebase-links { display:flex; flex-wrap:wrap; gap:0.65rem; }
  .firebase-link-btn { display:inline-block; padding:0.55rem 1.1rem; border-radius:6px; border:1px solid #ccc; background:#f8f9fa; color:#333; text-decoration:none; font-size:0.88rem; transition:background 0.15s; }
  .firebase-link-btn:hover { background:#e9ecef; }
  .firebase-link-primary { background:#e25822; color:white; border-color:#e25822; }
  .firebase-link-primary:hover { background:#c94d1e; }

    /* ══════════════════════════════════════════════════════════════════════════
     RWD — JS class-based（主）+ @media（備援）
     手機時 JS 會在 .dashboard / .sidebar 加上 .mobile class
  ══════════════════════════════════════════════════════════════════════════ */

  /* ── .mobile class（JS 直接寫入，優先級最高）───────────────────────────── */
  :global(.dashboard.mobile) {
    flex-direction: column !important;
    height: auto !important;
    min-height: 100vh;
    max-width: 100%;
  }
  :global(.sidebar.mobile) {
    width: 100% !important;
    height: auto !important;
    flex-direction: row !important;
    overflow-x: auto !important;
    overflow-y: hidden !important;
    border-right: none !important;
    border-bottom: 2px solid #e0e0e0 !important;
    padding: 0.5rem 0.5rem 0 !important;
    gap: 0.25rem !important;
    -webkit-overflow-scrolling: touch;
  }
  :global(.sidebar.mobile button) {
    white-space: nowrap !important;
    flex-shrink: 0 !important;
    text-align: center !important;
    padding: 0.5rem 1rem !important;
    border-radius: 8px 8px 0 0 !important;
    font-size: 0.85rem !important;
  }
  :global(.sidebar.mobile .refresh-btn) {
    white-space: nowrap !important;
    flex-shrink: 0 !important;
    margin-top: 0 !important;
    padding: 0.5rem 0.9rem !important;
  }
  :global(.sidebar.mobile .last-refresh) { display: none !important; }

  /* ── @media 備援（CSS-only 瀏覽器）────────────────────────────────────── */
  @media (max-width: 768px) {
    .dashboard {
      flex-direction: column !important;
      height: auto !important;
      min-height: 100vh;
      max-width: 100%;
    }
    .sidebar {
      width: 100% !important;
      height: auto !important;
      flex-direction: row !important;
      overflow-x: auto !important;
      overflow-y: hidden !important;
      border-right: none !important;
      border-bottom: 2px solid #e0e0e0 !important;
      padding: 0.5rem 0.5rem 0 !important;
      gap: 0.25rem !important;
      -webkit-overflow-scrolling: touch;
    }
    .sidebar button {
      white-space: nowrap !important;
      flex-shrink: 0 !important;
      text-align: center !important;
      padding: 0.5rem 1rem !important;
      border-radius: 8px 8px 0 0 !important;
      font-size: 0.85rem !important;
    }
    .sidebar .refresh-btn {
      white-space: nowrap !important;
      flex-shrink: 0 !important;
      margin-top: 0 !important;
      padding: 0.5rem 0.9rem !important;
    }
    .last-refresh { display: none !important; }
    .content { padding: 1rem !important; overflow-y: auto; }
    .admin-header { padding: 0.7rem 1rem !important; }
    .header-content h1 { font-size: 1rem !important; }
    .user-info { font-size: 0.8rem !important; gap: 0.5rem !important; }
    .ov-grid { grid-template-columns: repeat(2, 1fr) !important; gap: 0.6rem !important; }
    .ov-card { padding: 0.9rem 0.75rem !important; }
    .ov-num { font-size: 1.8rem !important; }
    .filter-bar { flex-direction: column !important; align-items: stretch !important; gap: 0.5rem !important; }
    .filter-bar > * { width: 100% !important; }
    .filter-select { width: 100% !important; }
    .table-container { overflow-x: auto !important; -webkit-overflow-scrolling: touch; }
    .table-container table { min-width: 520px !important; }
    .modal-content { width: 95vw !important; max-width: 95vw !important; padding: 1.2rem !important; }
    .battle-decks-grid { grid-template-columns: 1fr !important; gap: 0.75rem !important; }
    .firebase-stats-grid { grid-template-columns: repeat(2, 1fr) !important; }
  }

  @media (max-width: 480px) {
    .ov-grid { grid-template-columns: 1fr !important; }
    .firebase-stats-grid { grid-template-columns: 1fr !important; }
    .header-content h1 .version { display: none !important; }
    .header-content h1 { font-size: 0.95rem !important; }
    .content { padding: 0.75rem !important; }
    .sidebar button { font-size: 0.8rem !important; padding: 0.45rem 0.75rem !important; }
    .sidebar .refresh-btn { font-size: 0.8rem !important; padding: 0.45rem 0.75rem !important; }
  }
</style>
