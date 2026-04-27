<script lang="ts">
  import { onMount } from 'svelte';
  import { auth } from '$lib/firebase';
  import { onAuthStateChanged, type User } from 'firebase/auth';

  let { children } = $props();

  let user = $state<User | null>(null);
  let initialized = $state(false);

  onMount(() => {
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      user = u;
      initialized = true;
    });
    return unsubscribe;
  });
</script>

{#if !initialized}
  <div class="loading">正在載入管理員系統...</div>
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
</style>
