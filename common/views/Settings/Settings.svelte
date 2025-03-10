<script context='module'>
  import { toast } from 'svelte-sonner'
  import { click } from '@/modules/click.js'
  import { settings } from '@/modules/settings.js'
  import IPC from '@/modules/ipc.js'

  if (settings.value.enableDoH) IPC.emit('doh', settings.value.doHURL)
  export const platformMap = {
    aix: 'Aix',
    darwin: 'MacOS',
    android: 'Android',
    ios: 'iOS',
    freebsd: 'Linux',
    linux: 'Linux',
    openbsd: 'Linux',
    sunos: 'SunOS',
    win32: 'Windows'
  }
  let version = '1.0.0'
  IPC.on('version', data => (version = data))
  IPC.emit('version')

  let wasUpdated = false
  IPC.on('update-available', () => {
    if (!wasUpdated) {
      wasUpdated = true
      toast('Auto Updater', {
        description: 'A new version of Miru is available. Downloading!'
      })
    }
  })
  IPC.on('update-downloaded', () => {
    toast.success('Auto Updater', {
      description: 'A new version of Miru has downloaded. You can restart to update!'
    })
  })

  const changeLog = (async () => {
    const res = await fetch('https://api.github.com/repos/ThaUnknown/miru/releases')
    const json = await res.json()
    return json.map(({ body, tag_name: version, published_at: date, assets }) => ({ body, version, date, assets }))
  })()
  IPC.emit('show-discord-status', settings.value.showDetailsInRPC)
</script>

<script>
  import { Tabs, TabLabel, Tab } from '@/components/Tabination.js'
  import { onDestroy } from 'svelte'
  import PlayerSettings from './PlayerSettings.svelte'
  import TorrentSettings from './TorrentSettings.svelte'
  import InterfaceSettings from './InterfaceSettings.svelte'
  import AppSettings from './AppSettings.svelte'
  import { anilistClient } from '@/modules/anilist.js'
  import { logout } from '@/components/Logout.svelte'
  import smoothScroll from '@/modules/scroll.js'

  const groups = {
    player: {
      name: 'Player',
      icon: 'play_arrow'
    },
    torrent: {
      name: 'Torrent',
      icon: 'rss_feed'
    },
    interface: {
      name: 'Interface',
      icon: 'settings'
    },
    app: {
      name: 'App',
      icon: 'info'
    },
    changelog: {
      name: 'Changelog',
      icon: 'description'
    }
  }
  function pathListener (data) {
    $settings.torrentPath = data
  }

  function loginButton () {
    if (anilistClient.userID) {
      $logout = true
    } else {
      IPC.emit('open', 'https://anilist.co/api/v2/oauth/authorize?client_id=4254&response_type=token') // Change redirect_url to miru://auth
      if (platformMap[window.version.platform] === 'Linux') {
        toast('Support Notification', {
          description: "If your linux distribution doesn't support custom protocol handlers, you can simply paste the full URL into the app.",
          duration: 300000
        })
      }
    }
  }
  onDestroy(() => {
    IPC.off('path', pathListener)
  })
  $: IPC.emit('show-discord-status', $settings.showDetailsInRPC)
  IPC.on('path', pathListener)
</script>

<Tabs>
  <div class='d-flex w-full h-full position-relative settings root flex-md-row flex-column overflow-y-auto overflow-y-md-hidden' use:smoothScroll>
    <div class='d-flex flex-column flex-row h-full w-md-300 bg-dark position-relative px-20 px-md-0 flex-basis-0-md-custom'>
      <div class='px-20 py-15 font-size-24 font-weight-semi-bold'>Settings</div>
      {#each Object.values(groups) as group}
        <TabLabel>
          <div class='px-20 py-10 d-flex'>
            <span class='material-symbols-outlined font-size-24 pr-10 d-inline-flex justify-content-center align-items-center'>{group.icon}</span>
            <div class='font-size-16'>{group.name}</div>
          </div>
        </TabLabel>
      {/each}
      <div class='pointer my-5 rounded' tabindex='0' role='button' use:click={() => IPC.emit('open', 'https://github.com/sponsors/ThaUnknown/')}>
        <div class='px-20 py-10 d-flex'>
          <span class='material-symbols-outlined font-size-24 pr-10 d-inline-flex justify-content-center align-items-center'>favorite</span>
          <div class='font-size-16'>Donate</div>
        </div>
      </div>
      <div class='pointer my-5 rounded' use:click={loginButton}>
        <div class='px-20 py-10 d-flex'>
          {#if anilistClient.userID}
            {#await anilistClient.userID}
              <span class='material-symbols-outlined font-size-24 pr-10 d-inline-flex justify-content-center align-items-center'>login</span>
              <div class='font-size-16'>Login With AniList</div>
            {:then result}
              <span class='material-symbols-outlined rounded mr-10'>
                <img src={result.data.Viewer.avatar.medium} class='h-30 rounded' alt='logo' />
              </span>
              <div class='font-size-16 login-image-text'>Logout</div>
            {/await}
          {:else}
            <span class='material-symbols-outlined font-size-24 pr-10 d-inline-flex justify-content-center align-items-center'>login</span>
            <div class='font-size-16'>Login With AniList</div>
          {/if}
        </div>
      </div>
      <p class='text-muted px-20 py-10 m-0 mt-md-auto'>Restart may be required for some settings to take effect.</p>
      <p class='text-muted px-20 pb-10 m-0'>If you don't know what settings do what, use defaults.</p>
      <p class='text-muted px-20 m-0 mb-md-20'>v{version} {platformMap[window.version.platform] || 'dev'} {window.version.arch || 'dev'}</p>
    </div>
    <Tab>
      <div class='root h-full w-full overflow-y-md-auto p-20' use:smoothScroll>
        <PlayerSettings bind:settings={$settings} />
        <!-- spacing element to make space for miniplayer on mobile -->
        <div class='h-250 d-md-none' />
      </div>
    </Tab>
    <Tab>
      <div class='root h-full w-full overflow-y-md-auto p-20' use:smoothScroll>
        <TorrentSettings bind:settings={$settings} />
        <div class='h-250 d-md-none' />
      </div>
    </Tab>
    <Tab>
      <div class='root h-full w-full overflow-y-md-auto p-20' use:smoothScroll>
        <InterfaceSettings bind:settings={$settings} />
        <div class='h-250 d-md-none' />
      </div>
    </Tab>
    <Tab>
      <div class='root h-full w-full overflow-y-md-auto p-20' use:smoothScroll>
        <AppSettings />
        <div class='h-250 d-md-none' />
      </div>
    </Tab>
    <Tab>
      <div class='root my-20 px-20 overflow-y-md-auto w-full' use:smoothScroll>
        <div class='h-300 row px-20 px-sm-0'>
          <div class='col-sm-3 d-none d-sm-flex' />
          <div class='col-sm-6 d-flex justify-content-center flex-column'>
            <h1 class='font-weight-bold text-white title'>Changelog</h1>
            <div class='font-size-18 text-muted'>New updates and improvements to Miru.</div>
          </div>
        </div>
        {#await changeLog}
          {#each Array(5) as _}
            <hr class='my-20' />
            <div class='row py-20 px-20 px-sm-0'>
              <div class='col-sm-3 my-20 order-last order-sm-first '>
                <div class='skeloader rounded w-100 h-10 bg-very-dark'>
                  <div class='skeleloader-swipe' />
                </div>
              </div>
              <div class='col-sm-9'>
                <div class='skeloader rounded w-150 h-25 bg-very-dark mb-10'>
                  <div class='skeleloader-swipe' />
                </div>
                <div class='skeloader rounded w-250 h-10 bg-very-dark mt-20'>
                  <div class='skeleloader-swipe' />
                </div>
                <div class='skeloader rounded w-200 h-10 bg-very-dark mt-15'>
                  <div class='skeleloader-swipe' />
                </div>
              </div>
            </div>
          {/each}
        {:then changelog}
          {#each changelog as { version, date, body }}
            <hr class='my-20' />
            <div class='row py-20 px-20 px-sm-0 position-relative'>
              <div class='col-sm-3 order-last order-sm-first text-white'>
                <div class='position-sticky top-0 pt-20'>
                  {new Date(date).toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}
                </div>
              </div>
              <div class='col-sm-9 pre-wrap text-muted'>
                <h2 class='mt-0 font-weight-bold text-white'>{version}</h2>{body.replaceAll('- ', '')}</div>
            </div>
          {/each}
        {:catch}
          {#each Array(5) as _}
            <hr class='my-20' />
            <div class='row py-20 px-20 px-sm-0'>
              <div class='col-sm-3 my-20 order-last order-sm-first '>
                <div class='skeloader rounded w-100 h-10 bg-very-dark'>
                  <div class='skeleloader-swipe' />
                </div>
              </div>
              <div class='col-sm-9'>
                <div class='skeloader rounded w-150 h-25 bg-very-dark mb-10'>
                  <div class='skeleloader-swipe' />
                </div>
                <div class='skeloader rounded w-250 h-10 bg-very-dark mt-20'>
                  <div class='skeleloader-swipe' />
                </div>
                <div class='skeloader rounded w-200 h-10 bg-very-dark mt-15'>
                  <div class='skeleloader-swipe' />
                </div>
              </div>
            </div>
          {/each}
        {/await}
        <div class='h-250 d-md-none' />
      </div>
    </Tab>
  </div>
</Tabs>

<style>
  .settings :global(select.form-control:invalid) {
    color: var(--dm-input-placeholder-text-color);
  }
  .login-image-text {
    display: inline-flex;
    justify-content: center;
    align-items: center;
  }
  .h-30 {
    width: 3rem;
    height: 3rem;
  }
  .settings :global(input:not(:focus):invalid) {
    box-shadow: 0 0 0 0.2rem var(--danger-color) !important;
  }

  .flex-basis-0-md-custom {
    flex-basis: 0%;
  }

  @media (min-width: 769px) {
    .flex-basis-0-md-custom  {
      flex-basis: auto;
    }
  }
  .title {
    font-size: 5rem
  }
</style>
