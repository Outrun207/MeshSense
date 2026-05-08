<script context="module" lang="ts">
  import axios from 'axios'
  import Address from './Address.svelte'
  import { WebSocketClient } from './lib/wsc'
  import Log from './Log.svelte'
  import Nodes, { smallMode, focusNodeFilter } from './Nodes.svelte'
  import Map, { expandedMap } from './Map.svelte'
  import OpenLayersMap from './lib/OpenLayersMap.svelte'
  import Bluetooth from './Bluetooth.svelte'
  import Serial from './Serial.svelte'
  import Message from './Message.svelte'
  import { allowRemoteMessaging, connectionStatus, version } from 'api/src/vars'
  import UpdateStatus from './lib/UpdateStatus.svelte'
  import SettingsModal from './SettingsModal.svelte'
  import { hasAccess } from './lib/util'
  import News, { newsVisible } from './News.svelte'

  export const ws = new WebSocketClient(`${import.meta.env.VITE_PATH || ''}/ws`)
  axios.defaults.baseURL = import.meta.env.VITE_PATH
</script>

<script lang="ts">
  import { writable } from 'svelte/store'
  import { onMount } from 'svelte'
  import { showPage } from './SettingsModal.svelte'

  let ol: OpenLayersMap
  let isMobile = false

  type MobileTab = 'nodes' | 'map' | 'messages'
  let activeTab = writable<MobileTab>('nodes')

  function checkMobile() {
    isMobile = window.innerWidth < 1024
  }

  onMount(() => {
    checkMobile()
    window.addEventListener('resize', checkMobile)

    if (window.api?.onOpenSettings) {
      window.api.onOpenSettings(() => showPage('Settings'))
    }
    if (window.api?.onFocusNodeFilter) {
      window.api.onFocusNodeFilter(() => focusNodeFilter())
    }

    return () => window.removeEventListener('resize', checkMobile)
  })
</script>

<UpdateStatus />
<SettingsModal />

{#if !isMobile}
  <!-- Desktop layout -->
  <main class="w-full grid grid-cols-[auto_1fr] gap-2 p-2 overflow-auto h-full">
    <News />
    <div class="flex flex-col gap-2 content-start h-full overflow-auto">
      {#if $hasAccess}
        <Address class="shrink-0" />
      {/if}
      <Bluetooth class="shrink-0" />
      <Serial class="shrink-0" />
      <Nodes {ol} class="grow" />
      {#if window.location.hostname == 'localhost' || $hasAccess || $allowRemoteMessaging}
        <Message />
      {/if}
    </div>
    <div id="content" class="grid grid-rows-[5fr_3fr] content-start h-full overflow-auto gap-2 relative">
      {#if $connectionStatus == 'connected'}
        <Map class={$expandedMap ? 'row-span-full col-span-full' : ''} bind:ol />
      {:else}
        <div class="grid items-center px-5 m-auto">
          <div class="text-3xl font-bold text-white">Welcome to MeshSense!</div>
          <div class="max-w-md mt-5 flex flex-col gap-4">
            <div>Available bluetooth and serial devices will appear on the left</div>
            <div>If your device is on the network, enter it's IP address in the Device IP field and click Connect. For serial/USB, enter the port path (e.g. /dev/ttyACM0).</div>
            {#if !$hasAccess}
              <div>If you do not see the option to connect, you can get access by connecting via localhost or by setting your Access and User key.</div>
            {/if}
            <div>
              For additional information, take a look at our <a target="_blank" href="https://affirmatech.com/meshsense/faq">FAQ</a> and
              <a target="_blank" href="https://affirmatech.com/meshsense/bluetooth">Bluetooth Tips</a>
            </div>
          </div>
          <div class="font-normal absolute m-2 top-0 right-0 flex gap-2 items-center">
            <div class="text-xs text-white/50 pr-2 font-bold">MeshSense {$version}</div>
            <button class="btn btn-sm h-6 grid place-content-center" on:click={() => newsVisible.set(true)}>📰</button>
            <button class="btn btn-sm h-6 grid place-content-center" on:click={() => showPage('Settings')}>⚙</button>
          </div>
        </div>
      {/if}
      {#if !$expandedMap}
        <Log {ol} />
      {/if}
    </div>
  </main>
{:else}
  <!-- Mobile layout -->
  <div class="flex flex-col h-full w-full overflow-hidden">
    <News />
    <div class="flex-1 overflow-hidden relative">
      <!-- Nodes tab -->
      <div class="absolute inset-0 overflow-auto p-2" class:hidden={$activeTab !== 'nodes'}>
        {#if $hasAccess}
          <Address class="mb-2" />
        {/if}
        {#if $connectionStatus === 'disconnected'}
          <Bluetooth class="mb-2" />
          <Serial class="mb-2" />
        {/if}
        <Nodes {ol} class="" />
      </div>

      <!-- Map tab -->
      <div class="absolute inset-0" class:hidden={$activeTab !== 'map'}>
        {#if $connectionStatus == 'connected'}
          <Map bind:ol />
        {:else}
          <div class="text-center p-4">
            <div class="text-xl font-bold text-white">Welcome to MeshSense!</div>
            <p class="mt-2 text-sm">Connect to a device to view the map.</p>
          </div>
        {/if}
      </div>

      <!-- Messages tab -->
      <div class="absolute inset-0 overflow-auto p-2" class:hidden={$activeTab !== 'messages'}>
        {#if window.location.hostname == 'localhost' || $hasAccess || $allowRemoteMessaging}
          <Message />
        {/if}
        <Log {ol} />
      </div>
    </div>

    <!-- Bottom navigation -->
    <nav class="flex bg-gray-800 border-t border-gray-700 shrink-0">
      <button
        class="flex-1 flex flex-col items-center py-2 {$activeTab === 'nodes' ? 'text-cyan-400' : 'text-gray-400'}"
        on:click={() => ($activeTab = 'nodes')}
      >
        <span class="text-lg">📡</span>
        <span class="text-xs">Nodes</span>
      </button>
      <button
        class="flex-1 flex flex-col items-center py-2 {$activeTab === 'map' ? 'text-cyan-400' : 'text-gray-400'}"
        on:click={() => ($activeTab = 'map')}
      >
        <span class="text-lg">🗺️</span>
        <span class="text-xs">Map</span>
      </button>
      <button
        class="flex-1 flex flex-col items-center py-2 {$activeTab === 'messages' ? 'text-cyan-400' : 'text-gray-400'}"
        on:click={() => ($activeTab = 'messages')}
      >
        <span class="text-lg">💬</span>
        <span class="text-xs">Messages</span>
      </button>
      <button
        class="flex-1 flex flex-col items-center py-2 text-gray-400"
        on:click={() => showPage('Settings')}
      >
        <span class="text-lg">⚙️</span>
        <span class="text-xs">Settings</span>
      </button>
    </nav>
  </div>
{/if}

<style>
  @media (min-width: 2000px) {
    #content {
      grid-template-rows: repeat(1, minmax(0, 1fr));
      grid-template-columns: 1fr auto;
    }
  }
</style>
