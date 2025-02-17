<script setup lang="ts">
import { applyNonSelectable, computed, isEeUI, isMac, useCommandPalette, useRouter, useTheme } from '#imports'
import type { CommandPaletteType } from '~/lib'

const router = useRouter()

const route = router.currentRoute

const cmdK = ref(false)

const cmdL = ref(false)

const disableBaseLayout = computed(() => route.value.path.startsWith('/nc/view') || route.value.path.startsWith('/nc/form'))

useTheme()

const { commandPalette, cmdData, cmdPlaceholder, activeScope, loadTemporaryScope, refreshCommandPalette } = useCommandPalette()

applyNonSelectable()
useEventListener(document, 'keydown', async (e: KeyboardEvent) => {
  const cmdOrCtrl = isMac() ? e.metaKey : e.ctrlKey
  if (cmdOrCtrl) {
    switch (e.key.toLowerCase()) {
      case 'a':
        // prevent Ctrl + A selection for non-editable nodes
        if (!['input', 'textarea'].includes((e.target as any).nodeName.toLowerCase())) {
          e.preventDefault()
        }
        break
      case 'k':
        e.preventDefault()
        break
      case 'l':
        e.preventDefault()
        break
      case 'j':
        e.preventDefault()
        break
    }
  }
})

// TODO: Remove when https://github.com/vuejs/core/issues/5513 fixed
const key = ref(0)

const messages = [
  `Uncaught NotFoundError: Failed to execute 'insertBefore' on 'Node': The node before which the new node is to be inserted is not a child of this node.`, // chromium based
  `NotFoundError: The object can not be found here.`, // safari
  "Cannot read properties of null (reading 'parentNode')",
]

if (typeof window !== 'undefined') {
  // @ts-expect-error using arbitrary window key
  if (!window.__ncvue) {
    window.addEventListener('error', (event) => {
      if (messages.includes(event.message)) {
        event.preventDefault()
        console.warn('Re-rendering layout because of https://github.com/vuejs/core/issues/5513')
        key.value++
      }
    })
  }

  // @ts-expect-error using arbitrary window key
  window.__ncvue = true
}

function onScope(scope: string) {
  if (scope === 'root' && isEeUI) {
    loadTemporaryScope({ scope: 'root', data: {} })
  }
}

function setActiveCmdView(cmd: CommandPaletteType) {
  if (cmd === 'cmd-k') {
    cmdK.value = true
    cmdL.value = false
  } else if (cmd === 'cmd-l') {
    cmdL.value = true
    cmdK.value = false
  } else {
    cmdL.value = false
    cmdK.value = false
    document.dispatchEvent(
      new KeyboardEvent('keydown', {
        key: 'J',
        ctrlKey: !isMac() || undefined,
        metaKey: isMac() || undefined,
      }),
    )
  }
}

onMounted(() => {
  nextTick(() => {
    refreshCommandPalette()
  })
})
</script>

<template>
  <a-config-provider>
    <NuxtLayout :name="disableBaseLayout ? false : 'base'">
      <NuxtPage :key="key" :transition="false" />
    </NuxtLayout>
  </a-config-provider>
  <!-- Command Menu -->
  <CmdK
    ref="commandPalette"
    v-model:open="cmdK"
    :scope="activeScope.scope"
    :data="cmdData"
    :placeholder="cmdPlaceholder"
    :load-temporary-scope="loadTemporaryScope"
    :set-active-cmd-view="setActiveCmdView"
    @scope="onScope"
  />
  <!-- Recent Views. Cycles through recently visited Views -->
  <CmdL v-model:open="cmdL" :set-active-cmd-view="setActiveCmdView" />
  <!-- Documentation. Integrated NocoDB Docs directlt inside the Product -->
  <CmdJ />
</template>
