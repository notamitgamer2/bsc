<template></template>

<script setup>
// In-app content updater for the bundled Android build only.
//
// On app open, and whenever it returns to the foreground, while online:
// fetches version.json from the "app-bundle" GitHub Release (republished
// automatically on every push to main — see .github/workflows/main.yml),
// and if it's newer than what's currently loaded, downloads the zipped
// site and swaps it in via @capgo/capacitor-updater.
//
// No-op on the regular website (Capacitor.isNativePlatform() is false
// there) — this component is mounted globally in theme/index.ts but does
// nothing outside the native shell.
import { onMounted } from 'vue'

const VERSION_URL =
  'https://github.com/notamitgamer/bsc/releases/download/app-bundle/version.json'
const CURRENT_VERSION_KEY = 'bsc_bundle_version'

async function checkForUpdate() {
  // Guard for SSR (VitePress prerenders on the server) and web builds —
  // only proceed if we're actually running inside the Capacitor native shell.
  if (typeof window === 'undefined' || !window.Capacitor?.isNativePlatform?.()) return

  const { CapacitorUpdater } = await import('@capgo/capacitor-updater')

  let remote
  try {
    const res = await fetch(VERSION_URL, { cache: 'no-store' })
    if (!res.ok) return
    remote = await res.json()
  } catch {
    // Offline or unreachable — silently skip, app keeps working on
    // whatever content is currently loaded.
    return
  }

  const currentVersion = localStorage.getItem(CURRENT_VERSION_KEY)
  if (!remote?.version || !remote?.url || remote.version === currentVersion) return

  try {
    const bundle = await CapacitorUpdater.download({
      url: remote.url,
      version: remote.version
    })
    await CapacitorUpdater.set(bundle)
    localStorage.setItem(CURRENT_VERSION_KEY, remote.version)
    // set() reloads the WebView pointed at the newly applied bundle.
  } catch (err) {
    console.error('BSC updater: failed to apply update', err)
  }
}

onMounted(() => {
  if (typeof window === 'undefined') return
  setTimeout(checkForUpdate, 1500)
  document.addEventListener('visibilitychange', () => {
    if (document.visibilityState === 'visible') checkForUpdate()
  })
})
</script>
