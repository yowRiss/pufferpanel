<script setup>
import { ref, inject, onMounted, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import Btn from '@/components/ui/Btn.vue'
import Icon from '@/components/ui/Icon.vue'
import Toggle from '@/components/ui/Toggle.vue'
import Variables from '@/components/ui/Variables.vue'

const { t, te, locale } = useI18n()
const toast = inject('toast')

const props = defineProps({
  server: { type: Object, required: true }
})

const vars = ref({})
const flags = ref({})
const anyItems = computed(() => {
  if (Object.keys(vars.value).length > 0) return true
  if (Object.keys(flags.value).length > 0) return true
  return false
})

onMounted(async () => {
  if (props.server.hasScope('server.definition.view')) {
    vars.value = (await props.server.getDefinition())
  } else if (props.server.hasScope('server.data.view')) {
    vars.value = (await props.server.getData()) || {}
  }
  if (props.server.hasScope('server.flags.view'))
    flags.value = (await props.server.getFlags()) || {}
})

async function save() {
  const data = {}
  Object.keys(vars.value.data).map(name => {
    data[name] = vars.value.data[name].value
  })
  if (props.server.hasScope('server.data.edit.admin')) {
    await props.server.adminUpdateData(data)
  } else if (props.server.hasScope('server.data.edit')) {
    await props.server.updateData(data)
  }
  if (props.server.hasScope('server.flags.edit'))
    await props.server.setFlags(flags.value)
  toast.success(t('servers.SettingsSaved'))
}

function getFlagHint(name) {
  if (te(`servers.flags.hint.${name}`, locale))
    return t(`servers.flags.hint.${name}`)
}

const isMinecraftJava = computed(() => {
  if (!props.server) return false
  const type = (props.server.type || '').toLowerCase()
  const icon = (props.server.icon || '').toLowerCase()
  return type.includes('minecraft') || icon.includes('minecraft') || type === 'minecraft-java'
})

function goToMods() {
  location.hash = '#mods'
}
</script>

<template>
  <div>
    <div v-if="isMinecraftJava" class="minecraft-mods-banner" style="margin-bottom: 1.25rem; padding: 0.85rem 1rem; border-radius: 6px; background: rgba(27, 217, 106, 0.08); border: 1px solid rgba(27, 217, 106, 0.25); display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 0.75rem;">
      <div style="display: flex; align-items: center; gap: 0.75rem;">
        <icon name="puzzle" style="color: #1bd96a; font-size: 1.4rem;" />
        <div>
          <strong style="display: block; font-size: 0.95rem;">Minecraft Mod Marketplace</strong>
          <span style="font-size: 0.82rem; opacity: 0.75;">Browse, download, and install mods from Modrinth directly into this server.</span>
        </div>
      </div>
      <btn color="primary" @click="goToMods()"><icon name="puzzle" /> {{ t('servers.ModMarketplace') }}</btn>
    </div>

    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; flex-wrap: wrap; gap: 0.5rem;">
      <h2 style="margin: 0;" v-text="t('servers.Settings')" />
      <btn v-if="isMinecraftJava" color="primary" @click="goToMods()"><icon name="puzzle" /> {{ t('servers.ModMarketplace') }}</btn>
    </div>
    <variables v-model="vars" :disabled="!server.hasScope('server.data.edit')" />
    <div class="group-header">
      <div class="title">
        <h3 v-text="t('servers.FlagsHeader')" />
      </div>
    </div>
    <div v-for="(_, name) in flags" :key="name">
      <toggle v-model="flags[name]" :disabled="!server.hasScope('server.flags.edit')" :label="t(`servers.flags.${name}`)" :hint="getFlagHint()" />
    </div>
    <span v-if="!anyItems" v-text="t('servers.NoSettings')" />
    <div style="display: flex; gap: 0.75rem; align-items: center; margin-top: 1rem; flex-wrap: wrap;">
      <btn v-if="anyItems" color="primary" @click="save()"><icon name="save" />{{ t('servers.SaveSettings') }}</btn>
      <btn v-if="isMinecraftJava" color="primary" variant="outline" @click="goToMods()"><icon name="puzzle" /> {{ t('servers.ModMarketplace') }}</btn>
    </div>
  </div>
</template>
