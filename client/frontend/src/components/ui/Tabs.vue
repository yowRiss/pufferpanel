<script>
import { ref, onMounted, onUpdated, onUnmounted, provide, nextTick } from 'vue'
import Icon from './Icon.vue'

export default {
  components: {
    Icon
  },
  props: {
    anchors: { type: Boolean, default: () => false }
  },
  emits: ['tabChanged'],
  setup(props, { slots, emit }) {
    const tabButtons = ref(null)
    const needsLeftScroller = ref(false)
    const needsRightScroller = ref(false)

    const tabs = ref([])
    const activeKey = ref('')
    provide('activeKey', activeKey)

    function setActive(key) {
      activeKey.value = key

      if (props.anchors) {
        history.replaceState(history.state, '', '#' + key)
      }

      // deferring emit to next tick to ensure the tab content has changed
      nextTick(() => emit('tabChanged', key))
    }

    function onResizeOrScroll() {
      if (tabButtons.value && tabButtons.value.scrollWidth > tabButtons.value.offsetWidth) {
        needsLeftScroller.value = tabButtons.value.scrollLeft > 5
        const leftMax = tabButtons.value.scrollWidth - tabButtons.value.offsetWidth
        needsRightScroller.value = (leftMax - tabButtons.value.scrollLeft) > 5
      }
    }

    function scroll(dir) {
      if (!tabButtons.value) return
      const dist = (tabButtons.value.offsetWidth / 2)
      tabButtons.value.scrollTo({
        behavior: 'smooth',
        left: tabButtons.value.scrollLeft + (dir === 'right' ? dist : dist * -1)
      })
    }

    function syncTabs() {
      if (!slots.default) return
      const raw = slots.default()
      const flat = []
      function flatten(nodes) {
        for (const node of nodes) {
          if (!node) continue
          if (Array.isArray(node)) {
            flatten(node)
          } else if (node.type && typeof node.type === 'symbol' && Array.isArray(node.children)) {
            flatten(node.children)
          } else {
            flat.push(node)
          }
        }
      }
      flatten(raw)

      const parsedTabs = flat
        .filter(e => e && e.props && e.props.title)
        .map(e => ({
          key: e.props.id || (e.props.title ? e.props.title.toLowerCase().replace(/ /g, '-') : ''),
          title: e.props.title,
          icon: e.props.icon,
          hotkey: e.props.hotkey
        }))

      const same = tabs.value.length === parsedTabs.length && tabs.value.every((t, i) => t.key === parsedTabs[i].key && t.title === parsedTabs[i].title)
      if (!same) {
        tabs.value = parsedTabs
        if (props.anchors && tabs.value.length > 0 && location.hash) {
          const tab = tabs.value.find(e => e.key === location.hash.substring(1))
          if (tab) setActive(tab.key)
        }
        if (tabs.value.length > 0 && (!activeKey.value || !tabs.value.some(t => t.key === activeKey.value))) {
          setActive(tabs.value[0].key)
        }
      }
    }

    onMounted(() => {
      window.addEventListener('resize', onResizeOrScroll)
      nextTick(() => {
        if (tabButtons.value) {
          tabButtons.value.addEventListener('scroll', onResizeOrScroll)
          onResizeOrScroll()
        }
      })

      syncTabs()
      window.addEventListener('hashchange', onHashChange)
    })

    onUpdated(() => {
      syncTabs()
      nextTick(onResizeOrScroll)
    })

    function onHashChange() {
      if (props.anchors && location.hash) {
        const key = location.hash.substring(1)
        const tab = tabs.value.find(e => e.key === key)
        if (tab) setActive(tab.key)
      }
    }

    onUnmounted(() => {
      window.removeEventListener('resize', onResizeOrScroll)
      window.removeEventListener('hashchange', onHashChange)
      if (tabButtons.value) {
        tabButtons.value.removeEventListener('scroll', onResizeOrScroll)
      }
    })

    return { tabButtons, needsLeftScroller, needsRightScroller, tabs, activeKey, setActive, scroll }
  }
}
</script>

<template>
  <div class="tabs">
    <div v-if="needsLeftScroller" class="scroll-left" @click="scroll('left')" />
    <div v-if="needsRightScroller" class="scroll-right" @click="scroll('right')" />
    <div ref="tabButtons" class="tab-buttons">
      <div
        v-for="tab in tabs"
        :key="tab.key"
        v-hotkey="tab.hotkey"
        :class="['tab-button', tab.key === activeKey ? 'active' : 'inactive', tab.icon ? 'has-icon' : '']"
        @click="setActive(tab.key)"
      >
        <icon v-if="tab.icon" :name="tab.icon" />
        <span class="title" v-text="tab.title" />
      </div>
    </div>
    <slot />
  </div>
</template>
