<script setup>
import { ref, inject, onMounted, computed } from 'vue'
import { useI18n } from 'vue-i18n'
import Btn from '@/components/ui/Btn.vue'
import Icon from '@/components/ui/Icon.vue'
import Toggle from '@/components/ui/Toggle.vue'
import TextField from '@/components/ui/TextField.vue'
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

const discordConfig = ref({
  enabled: true,
  webhook_url: 'https://discord.com/api/webhooks/1550807320236269568/rfSlqbMOyuoHaJYOX8rjJVCCJGLz_HhATSWH_Q5Rd3p2d-aDY7sLQm8Plx3h52SsNumj',
  bot_name: 'algojo rama',
  avatar_url: 'https://cdn.icon-icons.com/icons2/2699/PNG/512/minecraft_logo_icon_168974.png',
  spike_threshold_mbps: 20,
  mention_critical: true,
  mention_discord_id: '586719784490631180',
  alerts: {
    disconnects: true,
    critical_errors: true,
    crashes: true,
    network_spikes: true,
    normal_leaves: false
  }
})
const discordLoading = ref(false)
const testingWebhook = ref(false)

async function loadDiscordConfig() {
  try {
    if (props.server.hasScope('server.files.view')) {
      const content = await props.server.getFile('/discord-alerts.json', true)
      if (content) {
        const parsed = JSON.parse(content)
        discordConfig.value = {
          ...discordConfig.value,
          ...parsed,
          alerts: { ...discordConfig.value.alerts, ...(parsed.alerts || {}) }
        }
      }
    }
  } catch (e) {
    // If not found yet, default config is used
  }
}

async function saveDiscordConfig() {
  discordLoading.value = true
  try {
    if (props.server.hasScope('server.files.edit')) {
      await props.server.uploadFile('/discord-alerts.json', JSON.stringify(discordConfig.value, null, 2))
      toast.success('Discord alert settings saved!')
    }
  } catch (e) {
    toast.error('Failed to save Discord alert settings')
  } finally {
    discordLoading.value = false
  }
}

async function testDiscordWebhook() {
  const url = (discordConfig.value.webhook_url || '').trim()
  if (!url || !url.startsWith('https://discord.com/api/webhooks/')) {
    toast.error('Please enter a valid Discord Webhook URL.')
    return
  }
  testingWebhook.value = true
  try {
    const cleanId = (discordConfig.value.mention_discord_id || '').replace(/\D/g, '')
    const payload = {
      username: discordConfig.value.bot_name || 'Minecraft Server Alerts',
      avatar_url: discordConfig.value.avatar_url || 'https://cdn.icon-icons.com/icons2/2699/PNG/512/minecraft_logo_icon_168974.png',
      content: discordConfig.value.mention_critical && cleanId ? `🔔 <@${cleanId}> **Test Notification from Server Settings**` : undefined,
      allowed_mentions: discordConfig.value.mention_critical && cleanId ? { users: [cleanId] } : undefined,
      embeds: [
        {
          title: '🧪 Discord Alert Test',
          description: `Test notification sent successfully from **${props.server.name || 'Minecraft Server'}** settings panel!`,
          color: 5763719,
          fields: [
            { name: 'Server ID', value: `\`${props.server.id}\``, inline: true },
            { name: 'Server Name', value: props.server.name || 'Minecraft Server', inline: true },
            {
              name: 'Active Triggers',
              value: [
                discordConfig.value.alerts.disconnects ? '• Disconnect Reasons' : null,
                discordConfig.value.alerts.critical_errors ? '• Critical Errors' : null,
                discordConfig.value.alerts.crashes ? '• Crash Reports' : null,
                discordConfig.value.alerts.network_spikes ? '• Abnormal Network Spikes & Diagnostics' : null,
                discordConfig.value.alerts.normal_leaves ? '• Normal Quits' : null,
                discordConfig.value.mention_critical && cleanId ? `• Critical Mention: <@${cleanId}>` : null
              ].filter(Boolean).join('\n') || 'None',
              inline: false
            }
          ],
          timestamp: new Date().toISOString(),
          footer: { text: 'PufferPanel Discord Logger' }
        }
      ]
    }
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })
    if (res.ok || res.status === 204) {
      toast.success('Test alert sent to Discord successfully!')
    } else {
      toast.error(`Discord returned HTTP status ${res.status}`)
    }
  } catch (e) {
    toast.error(`Error sending test webhook: ${e.message}`)
  } finally {
    testingWebhook.value = false
  }
}

onMounted(async () => {
  if (props.server.hasScope('server.definition.view')) {
    vars.value = (await props.server.getDefinition())
  } else if (props.server.hasScope('server.data.view')) {
    vars.value = (await props.server.getData()) || {}
  }
  if (props.server.hasScope('server.flags.view'))
    flags.value = (await props.server.getFlags()) || {}

  await loadDiscordConfig()
})

async function save() {
  const data = {}
  Object.keys(vars.value.data || {}).map(name => {
    data[name] = vars.value.data[name].value
  })
  if (props.server.hasScope('server.data.edit.admin')) {
    await props.server.adminUpdateData(data)
  } else if (props.server.hasScope('server.data.edit')) {
    await props.server.updateData(data)
  }
  if (props.server.hasScope('server.flags.edit'))
    await props.server.setFlags(flags.value)

  if (props.server.hasScope('server.files.edit')) {
    await props.server.uploadFile('/discord-alerts.json', JSON.stringify(discordConfig.value, null, 2))
  }
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

    <!-- Discord Webhook Alerts Card -->
    <div class="discord-alerts-section">
      <div class="group-header" style="display: flex; justify-content: space-between; align-items: center; margin-top: 2rem; margin-bottom: 0.5rem;">
        <div class="title" style="display: flex; align-items: center; gap: 10px;">
          <h3 style="margin: 0;">Discord Webhook Alerts</h3>
          <span :class="['alert-badge', discordConfig.enabled ? 'badge-enabled' : 'badge-disabled']">
            {{ discordConfig.enabled ? 'Active' : 'Disabled' }}
          </span>
        </div>
      </div>
      <p style="font-size: 0.85rem; color: var(--text-disabled); margin-top: 0; margin-bottom: 1rem;">
        Automatically monitor server logs and dispatch formatted alerts to your Discord channel for player disconnect reasons, critical errors, and crash reports.
      </p>

      <div class="discord-card">
        <toggle
          v-model="discordConfig.enabled"
          label="Enable Discord Webhook Alerts"
          hint="Toggle real-time alerts to your Discord webhook"
        />

        <div v-if="discordConfig.enabled" style="margin-top: 1.25rem; display: flex; flex-direction: column; gap: 1rem;">
          <text-field
            v-model="discordConfig.webhook_url"
            label="Discord Webhook URL"
            hint="Create a webhook in your Discord channel (Channel Settings → Integrations → Webhooks)"
            placeholder="https://discord.com/api/webhooks/..."
          />

          <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 1rem;">
            <text-field
              v-model="discordConfig.bot_name"
              label="Bot Username"
              placeholder="Minecraft Server Alerts"
            />
            <text-field
              v-model="discordConfig.avatar_url"
              label="Bot Avatar Image URL"
              placeholder="https://example.com/avatar.png"
            />
            <text-field
              v-model.number="discordConfig.spike_threshold_mbps"
              type="number"
              label="Spike Threshold (Mbps)"
              hint="Alert if bandwidth exceeds this rate"
              placeholder="20"
            />
          </div>

          <div style="margin-top: 0.5rem;">
            <strong style="display: block; font-size: 0.9rem; margin-bottom: 0.75rem;">Event Triggers</strong>
            <div style="display: flex; flex-direction: column; gap: 0.5rem;">
              <toggle
                v-model="discordConfig.alerts.disconnects"
                label="Player Disconnect Reasons"
                hint="Alert when a player drops due to timeouts, kicks, or network errors (missing mods excluded)"
              />
              <toggle
                v-model="discordConfig.alerts.critical_errors"
                label="Critical Server Errors"
                hint="Alert when severe server errors (ERROR, FATAL, or Java exceptions) occur"
              />
              <toggle
                v-model="discordConfig.alerts.crashes"
                label="Server Crash Reports"
                hint="Alert immediately when a crash report file is generated"
              />
              <toggle
                v-model="discordConfig.alerts.network_spikes"
                label="Abnormal Network Traffic Spikes"
                hint="Alert when server network bandwidth spikes abnormally and diagnose the root cause (chunk streaming, player movement, etc.)"
              />
              <toggle
                v-model="discordConfig.alerts.normal_leaves"
                label="Normal Player Quits"
                hint="Also send a notification when a player leaves normally (Disconnected)"
              />
            </div>
          </div>

          <div style="margin-top: 0.5rem; padding: 0.85rem 1rem; border-radius: 10px; background: rgba(237, 66, 69, 0.08); border: 1px solid rgba(237, 66, 69, 0.25);">
            <strong style="display: block; font-size: 0.9rem; margin-bottom: 0.5rem; color: #ed4245;">Critical Alert Auto-Tag</strong>
            <toggle
              v-model="discordConfig.mention_critical"
              label="Auto-Tag Discord User on Critical Errors & Crashes"
              hint="Mention this Discord user ID with a direct push notification (<@id>) whenever a server crash or critical fatal error occurs"
            />
            <div v-if="discordConfig.mention_critical" style="margin-top: 0.75rem;">
              <text-field
                v-model="discordConfig.mention_discord_id"
                label="Discord User ID to Tag"
                hint="Your numerical Discord user ID (e.g. 586719784490631180)"
                placeholder="586719784490631180"
              />
            </div>
          </div>

          <div style="display: flex; gap: 0.75rem; align-items: center; margin-top: 0.75rem; flex-wrap: wrap;">
            <btn
              color="neutral"
              variant="outline"
              :disabled="testingWebhook || !discordConfig.webhook_url"
              @click="testDiscordWebhook()"
            >
              <icon :name="testingWebhook ? 'loading' : 'send'" :spin="testingWebhook" />
              {{ testingWebhook ? 'Testing...' : 'Send Test Alert to Discord' }}
            </btn>
            <btn
              color="primary"
              :disabled="discordLoading"
              @click="saveDiscordConfig()"
            >
              <icon name="save" />
              Save Discord Settings
            </btn>
          </div>
        </div>
      </div>
    </div>

    <div style="display: flex; gap: 0.75rem; align-items: center; margin-top: 2rem; flex-wrap: wrap;">
      <btn v-if="anyItems" color="primary" @click="save()"><icon name="save" />{{ t('servers.SaveSettings') }}</btn>
      <btn v-if="isMinecraftJava" color="primary" variant="outline" @click="goToMods()"><icon name="puzzle" /> {{ t('servers.ModMarketplace') }}</btn>
    </div>
  </div>
</template>

<style scoped lang="scss">
.discord-alerts-section {
  margin-top: 2rem;
  padding-top: 1.5rem;
  border-top: 1px solid rgba(128, 128, 128, 0.2);
}

.alert-badge {
  font-size: 0.72rem;
  font-weight: 600;
  padding: 2px 8px;
  border-radius: 9999px;
  text-transform: uppercase;
  letter-spacing: 0.5px;

  &.badge-enabled {
    background-color: rgba(16, 185, 129, 0.18);
    color: #10b981;
    border: 1px solid rgba(16, 185, 129, 0.35);
  }

  &.badge-disabled {
    background-color: rgba(107, 114, 128, 0.18);
    color: #6b7280;
    border: 1px solid rgba(107, 114, 128, 0.3);
  }
}

.discord-card {
  background-color: var(--background);
  border: 1px solid rgba(128, 128, 128, 0.2);
  border-radius: var(--border-radius);
  padding: 1.25rem;
  margin-top: 0.75rem;
}
</style>
