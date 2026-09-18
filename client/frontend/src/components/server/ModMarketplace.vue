<script setup>
import { ref, computed, inject, onMounted, watch } from 'vue'
import Btn from '@/components/ui/Btn.vue'
import Icon from '@/components/ui/Icon.vue'
import Loader from '@/components/ui/Loader.vue'
import Overlay from '@/components/ui/Overlay.vue'
import markdown from '@/utils/markdown'

const props = defineProps({
  server: { type: Object, required: true }
})

const toast = inject('toast')

const activeView = ref('marketplace') // 'marketplace' | 'installed'
const loading = ref(false)
const error = ref(null)

// Search & filters
const searchQuery = ref('')
const selectedLoader = ref('')
const selectedVersion = ref('')
const selectedCategory = ref('')
const selectedEnvironment = ref('')
const sortBy = ref('downloads') // 'downloads' | 'relevance' | 'updated' | 'newest'

// Server detected info
const detectedLoader = ref('')
const detectedVersion = ref('')
const serverLoaded = ref(false)
const enablingFabric = ref(false)
const enableFabricSuccess = ref(false)

// Mod data
const mods = ref([])
const totalHits = ref(0)
const currentPage = ref(1)
const pageSize = ref(24)
const totalPages = computed(() => {
  return Math.max(1, Math.ceil(totalHits.value / pageSize.value))
})
const visiblePages = computed(() => {
  const total = totalPages.value
  const current = currentPage.value
  if (total <= 7) {
    const pages = []
    for (let i = 1; i <= total; i++) pages.push(i)
    return pages
  }
  if (current <= 4) {
    return [1, 2, 3, 4, 5, '...', total]
  }
  if (current >= total - 3) {
    return [1, '...', total - 4, total - 3, total - 2, total - 1, total]
  }
  return [1, '...', current - 1, current, current + 1, '...', total]
})

function scrollToTop() {
  const el = document.querySelector('.browse-view') || document.querySelector('.mod-marketplace')
  if (el) {
    el.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}

async function goToPage(page) {
  if (typeof page !== 'number') return
  if (page < 1 || page > totalPages.value || page === currentPage.value || loading.value) return
  scrollToTop()
  await searchMods(page)
}

async function nextPage() {
  if (currentPage.value < totalPages.value && !loading.value) {
    scrollToTop()
    await searchMods(currentPage.value + 1)
  }
}

async function prevPage() {
  if (currentPage.value > 1 && !loading.value) {
    scrollToTop()
    await searchMods(currentPage.value - 1)
  }
}

const installedMods = ref([])
const loadingInstalled = ref(false)
const installing = ref({}) // { [projectId]: { status: string } }
const uninstalling = ref({}) // { [filename]: boolean }

// Version selector modal
const versionModalOpen = ref(false)
const versionModalMod = ref(null)
const versionModalVersions = ref([])
const versionModalLoading = ref(false)

// Mod details & full description modal
const descModalOpen = ref(false)
const descModalMod = ref(null)
const descModalProject = ref(null)
const descModalLoading = ref(false)
const descModalDependencies = ref([])

const loaders = [
  { label: 'All Loaders', value: '' },
  { label: 'Fabric', value: 'fabric' },
  { label: 'Forge', value: 'forge' },
  { label: 'NeoForge', value: 'neoforge' },
  { label: 'Quilt', value: 'quilt' }
]

const sortOptions = [
  { label: 'Most Downloads', value: 'downloads' },
  { label: 'Relevance', value: 'relevance' },
  { label: 'Recently Updated', value: 'updated' },
  { label: 'Newest', value: 'newest' }
]

const categories = [
  { label: 'All Categories', value: '' },
  { label: 'Optimization', value: 'optimization' },
  { label: 'Technology', value: 'technology' },
  { label: 'Adventure', value: 'adventure' },
  { label: 'Magic', value: 'magic' },
  { label: 'Utility', value: 'utility' },
  { label: 'Storage', value: 'storage' },
  { label: 'World Gen', value: 'worldgen' }
]

const environmentOptions = [
  { label: 'All Types', value: '', dotClass: '', title: 'All mod types' },
  { label: 'Server-Side', value: 'server', dotClass: 'server-dot', title: 'Runs on dedicated server (required or optional)' },
  { label: 'Server Only', value: 'server_only', dotClass: 'server-only-dot', title: 'Dedicated server only (client does not need it)' },
  { label: 'Server & Client', value: 'both', dotClass: 'both-dot', title: 'Required on both server and client' },
  { label: 'Client-Side', value: 'client', dotClass: 'client-dot', title: 'Supports client' },
  { label: 'Client Only', value: 'client_only', dotClass: 'client-only-dot', title: 'Client-only (not supported on dedicated server)' }
]

const availableGameVersions = ref([])
const loadingGameVersions = ref(false)

const fallbackReleaseVersions = [
  '1.21.4', '1.21.3', '1.21.2', '1.21.1', '1.21',
  '1.20.6', '1.20.5', '1.20.4', '1.20.3', '1.20.2', '1.20.1', '1.20',
  '1.19.4', '1.19.3', '1.19.2', '1.19.1', '1.19',
  '1.18.2', '1.18.1', '1.18',
  '1.17.1', '1.17',
  '1.16.5', '1.16.4', '1.16.3', '1.16.2', '1.16.1', '1.16',
  '1.15.2', '1.14.4', '1.13.2', '1.12.2', '1.11.2', '1.10.2', '1.9.4', '1.8.9', '1.7.10'
]

const releaseVersions = computed(() => {
  if (!availableGameVersions.value.length) {
    return fallbackReleaseVersions.map(v => ({ version: v, version_type: 'release' }))
  }
  const list = availableGameVersions.value.filter(v => v.version_type === 'release')
  // If detected server version is not in release list (e.g. custom/snapshot), ensure it is included
  if (detectedVersion.value && !list.some(v => v.version === detectedVersion.value)) {
    list.unshift({ version: detectedVersion.value, version_type: 'release' })
  }
  return list
})

const snapshotVersions = computed(() => {
  if (!availableGameVersions.value.length) return []
  return availableGameVersions.value.filter(v => v.version_type !== 'release').slice(0, 40)
})

async function fetchGameVersions() {
  loadingGameVersions.value = true
  try {
    const res = await fetch('https://api.modrinth.com/v2/tag/game_version')
    if (res.ok) {
      availableGameVersions.value = await res.json()
    } else {
      throw new Error(`HTTP ${res.status}`)
    }
  } catch (e) {
    console.warn('Failed to fetch game versions from Modrinth, using fallback list:', e)
    availableGameVersions.value = fallbackReleaseVersions.map(v => ({ version: v, version_type: 'release' }))
  } finally {
    loadingGameVersions.value = false
  }
}

const canEditFiles = computed(() => {
  return props.server.hasScope('server.files.edit')
})

onMounted(async () => {
  await Promise.allSettled([
    detectServerConfig(),
    fetchGameVersions(),
    fetchInstalledMods()
  ])
  await searchMods()
})

async function detectServerConfig() {
  try {
    const data = await props.server.getData()
    if (data) {
      if (data.modlauncher && data.modlauncher.value) {
        const ml = data.modlauncher.value.toLowerCase()
        if (['fabric', 'forge', 'neoforge', 'quilt'].includes(ml)) {
          detectedLoader.value = ml
          selectedLoader.value = ml
        } else {
          detectedLoader.value = 'vanilla'
        }
      } else {
        detectedLoader.value = 'vanilla'
      }
      if (data.version && data.version.value && data.version.value !== 'latest') {
        detectedVersion.value = data.version.value
        selectedVersion.value = data.version.value
      }
    }
  } catch (e) {
    console.warn('Could not read server data for mod detection:', e)
  }

  // Fallback file scan if launcher not detected
  if (!selectedLoader.value) {
    try {
      const rootFiles = await props.server.getFile('')
      const names = rootFiles.map(f => f.name.toLowerCase())
      if (names.some(n => n.includes('fabric'))) {
        detectedLoader.value = 'fabric'
        selectedLoader.value = 'fabric'
      } else if (names.some(n => n.includes('neoforge'))) {
        detectedLoader.value = 'neoforge'
        selectedLoader.value = 'neoforge'
      } else if (names.some(n => n.includes('forge'))) {
        detectedLoader.value = 'forge'
        selectedLoader.value = 'forge'
      } else if (!detectedLoader.value) {
        detectedLoader.value = 'vanilla'
      }
    } catch (e) {
      console.debug('Failed to scan root files for mod launcher:', e)
      if (!detectedLoader.value) detectedLoader.value = 'vanilla'
    }
  }
  if (detectedLoader.value === 'vanilla' && !selectedLoader.value) {
    selectedLoader.value = 'fabric'
  }
  serverLoaded.value = true
}

async function enableFabric() {
  enablingFabric.value = true
  try {
    await props.server.updateData({
      modlauncher: 'fabric'
    })
    await props.server.install()
    detectedLoader.value = 'fabric'
    selectedLoader.value = 'fabric'
    enableFabricSuccess.value = true
    await searchMods()
  } catch (e) {
    console.error('Failed to enable Fabric loader:', e)
    alert('Failed to enable Fabric loader: ' + (e.message || e))
  } finally {
    enablingFabric.value = false
  }
}

async function fetchInstalledMods() {
  loadingInstalled.value = true
  try {
    const exists = await props.server.fileExists('mods')
    if (!exists) {
      if (canEditFiles.value) {
        try {
          await props.server.createFolder('mods')
        } catch (e) {
          console.debug('Failed to create mods folder:', e)
        }
      }
      installedMods.value = []
      return
    }

    const files = await props.server.getFile('mods')
    if (Array.isArray(files)) {
      installedMods.value = files.filter(f => f.isFile && f.name.toLowerCase().endsWith('.jar'))
    } else {
      installedMods.value = []
    }
  } catch (err) {
    console.error('Failed to load installed mods:', err)
    installedMods.value = []
  } finally {
    loadingInstalled.value = false
  }
}

const MOD_ALIASES = {
  'ctov': ['ct-overhaul-village', 'choicetheorems-overhauled-village', 'fgmhI8kH'],
  't_and_t': ['towns-and-towers', 'DjLobEOy'],
  'tandt': ['towns-and-towers', 'DjLobEOy'],
  'voicechat': ['simple-voice-chat', '9eGKb6K1'],
  'simplevoicechat': ['simple-voice-chat', '9eGKb6K1'],
  'c2me': ['c2me-fabric', 'VSNURh3q'],
  'c2mefabric': ['c2me-fabric', 'VSNURh3q'],
  'antixray': ['anti-xray', 'antyxray', 'sml2FMaA', '7TbtGLSU'],
  'serversleep': ['serversleep', 'server-sleep', 'Cw8IlnGM'],
  'cristellib': ['cristel-lib', 'cl223EMc'],
  'infinitetrading': ['infinite-trading', 'U3eoZT3o'],
  'morezombievillagers': ['more-zombie-villagers', 'xu15hkKT'],
  'dungeonsandtaverns': ['dungeons-and-taverns', 'tpehi7ww'],
  'fabricperplayerspawns': ['fabric-per-player-spawns', 'mHrFHOJ0'],
  'jei': ['jei', 'u6dRKJwZ'],
  'rei': ['roughly-enough-items', 'nfn13YXA'],
  'emi': ['emi', 'fRiHVvU7'],
  'xaero': ['xaeros-minimap', 'xaeros-world-map'],
  'xaerosminimap': ['xaeros-minimap'],
  'xaerosworldmap': ['xaeros-world-map'],
  'appleskin': ['appleskin', 'EsAfCjCV'],
  'ferritecore': ['ferrite-core', 'uXXizFIs'],
  'krypton': ['krypton', 'fQEb0iXm'],
  'sodium': ['sodium', 'AANobbMI'],
  'lithium': ['lithium', 'gvQqBUqZ'],
  'iris': ['iris', 'YL57xq9U'],
  'indium': ['indium', 'Orvt0mII'],
  'modmenu': ['modmenu', 'mOgUt4GM'],
  'clothconfig': ['cloth-config', '9s6osm5g'],
  'clothconfigv2': ['cloth-config', '9s6osm5g'],
  'fabricapi': ['fabric-api', 'P7dR8mSH'],
  'architectury': ['architectury-api', 'lhGA9TYQ'],
  'architecturyapi': ['architectury-api', 'lhGA9TYQ'],
  'yacl': ['yacl', 'yet-another-config-lib', '1eAoo2KR'],
  'yetanotherconfiglib': ['yacl', 'yet-another-config-lib', '1eAoo2KR'],
  'owolib': ['owo-lib', 'ccxDM1m5'],
  'owo': ['owo-lib', 'ccxDM1m5'],
  'fzzyconfig': ['fzzy-config', 'f3N2w3K6'],
  'geckolib': ['geckolib', '8BmcYKbN'],
  'curios': ['curios', 'e0M80N4U'],
  'trinkets': ['trinkets', '5aa4ISzT'],
  'cardinalcomponents': ['cardinal-components-api', 'zV5HNzda'],
  'cardinalcomponentsbase': ['cardinal-components-api', 'zV5HNzda'],
  'fabriclanguagekotlin': ['fabric-language-kotlin', 'Ha28R6CL'],
  'kotlinforforge': ['kotlin-for-forge', 'ordsPcFz'],
  'citresewn': ['cit-resewn', 'yBW8D80W'],
  'entityculling': ['entityculling', 'NNAgCjsB'],
  'dynmap': ['dynmap', 'fFcwhcuq'],
  'bluemap': ['bluemap', 'swbUVToi'],
  'chunky': ['chunky', 'fALzjamp'],
  'geyser': ['geyser', 'w0yi10ON'],
  'floodgate': ['floodgate', 'bWrNNfkb'],
  'viaversion': ['viaversion', 'P1OZGk5p'],
  'viabackwards': ['viabackwards', '140eR2qZ'],
  'spark': ['spark', 'l6YWDetect']
}

function extractModBaseName(filename) {
  let base = filename.replace(/\.jar$/i, '')
  base = base.replace(/^\[[^\]]+\]\s*/, '')
  const versionMatch = base.search(/[-_+](?:fabric|forge|neoforge|quilt|mc)?[-_+v]?\d.*$/i)
  let namePart = versionMatch !== -1 ? base.substring(0, versionMatch) : base
  namePart = namePart.replace(/[-_+](fabric|forge|neoforge|quilt|all)$/i, '')
  return namePart.replace(/^[-_.]+|[-_.]+$/g, '').trim().toLowerCase()
}

function getAcronyms(title) {
  if (!title) return []
  const words = (title.match(/[a-zA-Z0-9]+/g) || []).filter(w => !['s', 'the', 'a', 'an', 'and', 'for', 'mod', 'edition'].includes(w.toLowerCase()))
  if (words.length <= 1) return []
  const acr1 = words.map(w => w[0].toLowerCase()).join('')
  const allWords = (title.match(/[a-zA-Z0-9]+/g) || []).filter(w => w.toLowerCase() !== 's')
  const acr2 = allWords.map(w => w[0].toLowerCase()).join('')
  const acr3 = allWords.filter(w => w.length <= 3).map(w => w.toLowerCase()).join('_')
  return [acr1, acr2, acr3].filter(Boolean)
}

// Check if a mod from Modrinth is currently installed in mods/ (from marketplace or outside)
function getInstalledFile(mod) {
  if (!installedMods.value.length || !mod) return null

  const slugClean = (mod.slug || '').toLowerCase().replace(/[^a-z0-9]/g, '')
  const titleClean = (mod.title || '').toLowerCase().replace(/[^a-z0-9]/g, '')
  const projectId = (mod.project_id || '').toLowerCase()
  const titleWords = (mod.title || '').toLowerCase().match(/[a-z0-9]{3,}/g) || []
  const acronyms = getAcronyms(mod.title || '')

  return installedMods.value.find(f => {
    const rawName = f.name.toLowerCase()
    const baseName = extractModBaseName(f.name)
    const cleanBase = baseName.replace(/[^a-z0-9]/g, '')
    const cleanRaw = rawName.replace(/[^a-z0-9]/g, '')

    // 1. Direct project_id or slug match in raw filename
    if (projectId && cleanRaw.includes(projectId)) return true
    if (slugClean && cleanRaw.includes(slugClean)) return true

    // 2. Base name equals slug or title
    if (cleanBase && (cleanBase === slugClean || cleanBase === titleClean)) return true

    // 3. Substring match for substantial names (>= 4 chars)
    if (cleanBase.length >= 4) {
      if (slugClean.includes(cleanBase) || cleanBase.includes(slugClean)) return true
      if (titleClean.includes(cleanBase) || cleanBase.includes(titleClean)) return true
    }

    // 4. Aliases dictionary
    const aliases = [...(MOD_ALIASES[cleanBase] || []), ...(MOD_ALIASES[baseName] || [])]
    if (aliases.includes(mod.slug) || (projectId && aliases.includes(projectId))) return true

    // 5. Acronym check (e.g. ctov, t_and_t, c2me)
    for (const acr of acronyms) {
      if (cleanBase === acr || cleanBase === acr.replace(/_/g, '')) return true
    }

    // 6. Significant word combination check (e.g. voicechat)
    if (titleWords.length >= 2) {
      const combo = titleWords.join('')
      if (cleanBase === combo || combo.includes(cleanBase) || cleanBase.includes(combo)) return true
      if (cleanBase.length >= 4 && titleWords.includes(cleanBase)) return true
    }

    return false
  }) || null
}

const mobileFiltersOpen = ref(false)

const hasActiveFilters = computed(() => {
  return !!(
    searchQuery.value.trim() ||
    selectedCategory.value ||
    selectedEnvironment.value ||
    (selectedVersion.value && selectedVersion.value !== detectedVersion.value) ||
    (selectedLoader.value && selectedLoader.value !== detectedLoader.value) ||
    sortBy.value !== 'downloads'
  )
})

const activeFilterCount = computed(() => {
  let count = 0
  if (searchQuery.value.trim()) count++
  if (selectedLoader.value && selectedLoader.value !== detectedLoader.value) count++
  if (selectedVersion.value && selectedVersion.value !== detectedVersion.value) count++
  if (selectedCategory.value) count++
  if (selectedEnvironment.value) count++
  if (sortBy.value !== 'downloads') count++
  return count
})

function getEnvLabel(val) {
  const item = environmentOptions.find(e => e.value === val)
  return item ? item.label : val
}

function getLoaderLabel(val) {
  const item = loaders.find(l => l.value === val)
  return item ? item.label : val
}

function getCategoryLabel(val) {
  const item = categories.find(c => c.value === val)
  return item ? item.label : val
}

function resetFilters() {
  searchQuery.value = ''
  selectedLoader.value = detectedLoader.value || ''
  selectedVersion.value = detectedVersion.value || ''
  selectedCategory.value = ''
  selectedEnvironment.value = ''
  sortBy.value = 'downloads'
  currentPage.value = 1
  searchMods()
}

let searchTimeout = null
function onSearchInput() {
  if (searchTimeout) clearTimeout(searchTimeout)
  searchTimeout = setTimeout(() => {
    currentPage.value = 1
    searchMods()
  }, 400)
}

watch([selectedLoader, selectedVersion, selectedCategory, selectedEnvironment, sortBy], () => {
  currentPage.value = 1
  searchMods()
})

async function searchMods(page = null) {
  if (typeof page === 'number') {
    currentPage.value = page
  }
  loading.value = true
  error.value = null

  try {
    const facets = [['project_type:mod']]

    if (selectedLoader.value) {
      facets.push([`categories:${selectedLoader.value}`])
    }
    if (selectedVersion.value) {
      facets.push([`versions:${selectedVersion.value.trim()}`])
    }
    if (selectedCategory.value) {
      facets.push([`categories:${selectedCategory.value}`])
    }
    if (selectedEnvironment.value === 'server') {
      facets.push(['server_side:required', 'server_side:optional'])
    } else if (selectedEnvironment.value === 'server_only') {
      facets.push(['server_side:required'])
      facets.push(['client_side:unsupported'])
    } else if (selectedEnvironment.value === 'both') {
      facets.push(['server_side:required', 'server_side:optional'])
      facets.push(['client_side:required', 'client_side:optional'])
    } else if (selectedEnvironment.value === 'client') {
      facets.push(['client_side:required', 'client_side:optional'])
    } else if (selectedEnvironment.value === 'client_only') {
      facets.push(['server_side:unsupported'])
    }

    const offset = Math.max(0, (currentPage.value - 1) * pageSize.value)

    const params = new URLSearchParams({
      query: searchQuery.value.trim(),
      facets: JSON.stringify(facets),
      index: sortBy.value,
      offset: String(offset),
      limit: String(pageSize.value)
    })

    const res = await fetch(`https://api.modrinth.com/v2/search?${params.toString()}`)
    if (!res.ok) {
      throw new Error(`Modrinth returned HTTP ${res.status}`)
    }

    const data = await res.json()
    mods.value = data.hits || []
    totalHits.value = data.total_hits || 0
  } catch (err) {
    console.error('Failed to fetch mods from Modrinth:', err)
    error.value = 'Failed to load mods from Modrinth. Please check your internet connection.'
  } finally {
    loading.value = false
  }
}

// Cache for project metadata to avoid redundant lookups
const projectMetaCache = new Map()

async function getProjectMeta(projectId) {
  if (projectMetaCache.has(projectId)) return projectMetaCache.get(projectId)
  try {
    const res = await fetch(`https://api.modrinth.com/v2/project/${projectId}`)
    if (res.ok) {
      const data = await res.json()
      projectMetaCache.set(projectId, data)
      return data
    }
  } catch (e) {
    console.warn('Failed to fetch project metadata for:', projectId, e)
  }
  return null
}

async function resolveAndInstallDependencies(versionObj, parentProjectId, visitedProjects = new Set(), installedDepNames = []) {
  if (!versionObj || !versionObj.dependencies || !versionObj.dependencies.length) {
    return installedDepNames
  }

  // Filter required dependencies only
  const requiredDeps = versionObj.dependencies.filter(d => d.dependency_type === 'required')
  if (!requiredDeps.length) return installedDepNames

  for (const dep of requiredDeps) {
    let depProjectId = dep.project_id
    let depVersionId = dep.version_id
    let depVersionData = null
    let depProjectData = null

    // If version_id is provided, fetch specific version
    if (depVersionId) {
      try {
        const vRes = await fetch(`https://api.modrinth.com/v2/version/${depVersionId}`)
        if (vRes.ok) {
          depVersionData = await vRes.json()
          if (!depProjectId) depProjectId = depVersionData.project_id
        }
      } catch (e) {
        console.warn('Failed to fetch dep version:', depVersionId, e)
      }
    }

    if (!depProjectId) continue
    if (visitedProjects.has(depProjectId)) continue
    visitedProjects.add(depProjectId)

    // Fetch project details for slug & title
    depProjectData = await getProjectMeta(depProjectId)
    const depTitle = depProjectData?.title || depProjectId
    const depSlug = depProjectData?.slug || ''

    // 1. Check if dependency is ALREADY INSTALLED
    const alreadyInstalled = getInstalledFile({
      project_id: depProjectId,
      slug: depSlug,
      title: depTitle
    })

    if (alreadyInstalled) {
      console.log(`Dependency "${depTitle}" is already installed (${alreadyInstalled.name}). Skipping re-download.`)
      continue
    }

    // 2. Dependency is NOT installed -> Must auto-download it!
    console.log(`Dependency "${depTitle}" not installed. Auto-installing...`)
    installing.value[parentProjectId] = { status: `Installing dependency: ${depTitle}...` }

    // If we don't have depVersionData yet, query Modrinth for compatible version
    if (!depVersionData) {
      const loader = selectedLoader.value || (detectedLoader.value !== 'vanilla' ? detectedLoader.value : 'fabric') || 'fabric'
      const version = selectedVersion.value || detectedVersion.value || ''
      const loadersParam = loader ? JSON.stringify([loader]) : '[]'
      const versionsParam = version ? JSON.stringify([version.trim()]) : '[]'

      let verUrl = `https://api.modrinth.com/v2/project/${depProjectId}/version`
      const verParams = new URLSearchParams()
      if (loadersParam !== '[]') verParams.set('loaders', loadersParam)
      if (versionsParam !== '[]') verParams.set('game_versions', versionsParam)

      let res = await fetch(`${verUrl}?${verParams.toString()}`)
      let list = res.ok ? await res.json() : []

      // Fallback: loader only
      if (!list.length && loadersParam !== '[]') {
        res = await fetch(`${verUrl}?loaders=${loadersParam}`)
        list = res.ok ? await res.json() : []
      }

      // Fallback: all versions
      if (!list.length) {
        res = await fetch(verUrl)
        list = res.ok ? await res.json() : []
      }

      if (list.length) {
        depVersionData = list[0]
      }
    }

    if (!depVersionData || !depVersionData.files || !depVersionData.files.length) {
      console.warn(`Could not resolve compatible file for dependency "${depTitle}"`)
      continue
    }

    // Pick file
    const depFile = depVersionData.files.find(f => f.primary) || depVersionData.files[0]
    if (!depFile || !depFile.url) continue

    // Download and save dependency file
    const dlRes = await fetch(depFile.url)
    if (!dlRes.ok) throw new Error(`Failed to download dependency ${depTitle} (HTTP ${dlRes.status})`)
    const depBlob = await dlRes.blob()

    // Ensure mods/ directory exists
    const hasMods = await props.server.fileExists('mods')
    if (!hasMods) await props.server.createFolder('mods')

    // Upload dependency file
    await props.server.uploadFile(`mods/${depFile.filename}`, depBlob)
    installedDepNames.push(depTitle)

    // Add to local installedMods immediately so subsequent checks know it's installed
    installedMods.value.push({
      name: depFile.filename,
      size: depBlob.size,
      isFile: true,
      isDir: false,
      extension: 'jar'
    })

    // 3. Recursively check if THIS dependency has required dependencies of its own!
    if (depVersionData.dependencies && depVersionData.dependencies.length) {
      await resolveAndInstallDependencies(depVersionData, parentProjectId, visitedProjects, installedDepNames)
    }
  }

  return installedDepNames
}

async function installMod(mod, specificFile = null, specificVersion = null) {
  if (!canEditFiles.value) {
    toast.error('You do not have permission to upload files to this server.')
    return
  }

  installing.value[mod.project_id] = { status: 'Resolving version...' }

  try {
    let fileToDownload = specificFile
    let targetVersion = specificVersion

    if (!fileToDownload || !targetVersion) {
      // Query versions matching current loader and version
      const loadersParam = selectedLoader.value ? JSON.stringify([selectedLoader.value]) : '[]'
      const versionsParam = selectedVersion.value ? JSON.stringify([selectedVersion.value.trim()]) : '[]'

      let verUrl = `https://api.modrinth.com/v2/project/${mod.project_id}/version`
      const verParams = new URLSearchParams()
      if (selectedLoader.value) verParams.set('loaders', loadersParam)
      if (selectedVersion.value) verParams.set('game_versions', versionsParam)

      let verRes = await fetch(`${verUrl}?${verParams.toString()}`)
      let versionsData = verRes.ok ? await verRes.json() : []

      // If no version found with both filters, try loader-only
      if (!versionsData.length && selectedLoader.value) {
        verRes = await fetch(`${verUrl}?loaders=${loadersParam}`)
        versionsData = verRes.ok ? await verRes.json() : []
      }

      // If still nothing, fallback to all versions
      if (!versionsData.length) {
        verRes = await fetch(verUrl)
        versionsData = verRes.ok ? await verRes.json() : []
      }

      if (!versionsData.length) {
        throw new Error('No compatible versions found for this mod on Modrinth.')
      }

      // Pick latest version and its primary file
      targetVersion = versionsData[0]
      if (!fileToDownload) {
        fileToDownload = targetVersion.files.find(f => f.primary) || targetVersion.files[0]
      }
    }

    if (!fileToDownload || !fileToDownload.url) {
      throw new Error('No download file found in the selected version.')
    }

    // Step 1: Check and Auto-Install Required Dependencies
    const visited = new Set([mod.project_id])
    const installedDependencies = []

    if (targetVersion && targetVersion.dependencies && targetVersion.dependencies.length) {
      installing.value[mod.project_id] = { status: 'Checking dependencies...' }
      await resolveAndInstallDependencies(targetVersion, mod.project_id, visited, installedDependencies)
    }

    // Step 2: Download main mod from Modrinth CDN
    installing.value[mod.project_id] = { status: `Downloading ${mod.title}...` }
    const downloadRes = await fetch(fileToDownload.url)
    if (!downloadRes.ok) {
      throw new Error(`Failed to download mod file (HTTP ${downloadRes.status})`)
    }
    const blob = await downloadRes.blob()

    // Step 3: Ensure mods folder exists
    const hasModsFolder = await props.server.fileExists('mods')
    if (!hasModsFolder) {
      await props.server.createFolder('mods')
    }

    // Step 4: Upload main mod to server
    installing.value[mod.project_id] = { status: 'Saving to server...' }
    const targetPath = `mods/${fileToDownload.filename}`
    await props.server.uploadFile(targetPath, blob)

    // Build feedback
    if (installedDependencies.length > 0) {
      toast.success(`Installed ${mod.title} + ${installedDependencies.length} dependency: ${installedDependencies.join(', ')}`)
    } else {
      toast.success(`Installed ${mod.title} (${fileToDownload.filename})`)
    }

    await fetchInstalledMods()
    if (versionModalOpen.value) {
      versionModalOpen.value = false
    }
  } catch (err) {
    console.error('Install failed:', err)
    toast.error(err.message || 'Failed to install mod')
  } finally {
    delete installing.value[mod.project_id]
  }
}

async function uninstallMod(filename) {
  if (!canEditFiles.value) {
    toast.error('You do not have permission to delete files from this server.')
    return
  }

  uninstalling.value[filename] = true
  try {
    await props.server.deleteFile(`mods/${filename}`)
    toast.success(`Removed ${filename}`)
    await fetchInstalledMods()
  } catch (err) {
    console.error('Uninstall failed:', err)
    toast.error('Failed to uninstall mod file')
  } finally {
    delete uninstalling.value[filename]
  }
}

async function openVersionsModal(mod) {
  versionModalMod.value = mod
  versionModalVersions.value = []
  versionModalLoading.value = true
  versionModalOpen.value = true

  try {
    const res = await fetch(`https://api.modrinth.com/v2/project/${mod.project_id}/version`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const data = await res.json()
    versionModalVersions.value = data
  } catch (e) {
    console.error('Failed to load versions for mod:', e)
    toast.error('Failed to load versions from Modrinth')
  } finally {
    versionModalLoading.value = false
  }
}

const showAllVersions = ref(false)

async function openModDetails(mod) {
  descModalMod.value = mod
  descModalProject.value = null
  descModalDependencies.value = []
  descModalLoading.value = true
  showAllVersions.value = false
  descModalOpen.value = true

  try {
    const idOrSlug = mod.project_id || mod.id || mod.slug
    const res = await fetch(`https://api.modrinth.com/v2/project/${idOrSlug}`)
    if (res.ok) {
      descModalProject.value = await res.json()
    } else {
      console.warn('Could not fetch project details from Modrinth:', res.status)
    }

    // Also fetch latest compatible version to resolve dependencies
    const loader = selectedLoader.value || (detectedLoader.value !== 'vanilla' ? detectedLoader.value : 'fabric') || 'fabric'
    const version = selectedVersion.value || detectedVersion.value || ''
    const loadersParam = loader ? JSON.stringify([loader]) : '[]'
    const versionsParam = version ? JSON.stringify([version.trim()]) : '[]'

    let verUrl = `https://api.modrinth.com/v2/project/${idOrSlug}/version`
    const verParams = new URLSearchParams()
    if (loadersParam !== '[]') verParams.set('loaders', loadersParam)
    if (versionsParam !== '[]') verParams.set('game_versions', versionsParam)

    const vRes = await fetch(`${verUrl}?${verParams.toString()}`)
    const vList = vRes.ok ? await vRes.json() : []
    const latestVer = vList[0]

    if (latestVer && latestVer.dependencies && latestVer.dependencies.length) {
      const depsWithMeta = []
      for (const d of latestVer.dependencies) {
        if (d.project_id) {
          const meta = await getProjectMeta(d.project_id)
          depsWithMeta.push({
            ...d,
            title: meta?.title || d.project_id,
            slug: meta?.slug || '',
            icon_url: meta?.icon_url || null
          })
        } else if (d.version_id) {
          try {
            const singleVRes = await fetch(`https://api.modrinth.com/v2/version/${d.version_id}`)
            if (singleVRes.ok) {
              const singleV = await singleVRes.json()
              const meta = singleV.project_id ? await getProjectMeta(singleV.project_id) : null
              depsWithMeta.push({
                ...d,
                title: meta?.title || singleV.name || d.version_id,
                slug: meta?.slug || '',
                icon_url: meta?.icon_url || null
              })
            }
          } catch (e) {
            console.debug('Could not fetch single version dependency:', e)
          }
        }
      }
      descModalDependencies.value = depsWithMeta
    }
  } catch (err) {
    console.error('Failed to load project details:', err)
  } finally {
    descModalLoading.value = false
  }
}

function getSupportedVersionsList(versions) {
  if (!versions || !Array.isArray(versions)) return []
  const semverRegex = /^\d+\.\d+(\.\d+)?$/
  const releases = versions.filter(v => semverRegex.test(v))

  function semverCompare(a, b) {
    const pa = a.split('.').map(Number)
    const pb = b.split('.').map(Number)
    for (let i = 0; i < 3; i++) {
      const na = pa[i] || 0
      const nb = pb[i] || 0
      if (na !== nb) return nb - na
    }
    return 0
  }

  if (releases.length > 0) {
    return [...new Set(releases)].sort(semverCompare)
  }
  return [...new Set(versions)].reverse()
}

function getVersionRange(versions) {
  if (!versions || !versions.length) return 'All'
  const list = getSupportedVersionsList(versions)
  if (!list.length) {
    return versions.length === 1 ? versions[0] : `${versions[0]} – ${versions[versions.length - 1]}`
  }
  const oldest = list[list.length - 1]
  const newest = list[0]
  if (oldest === newest) return newest
  return `${oldest} – ${newest}`
}

function isServerVersionCompatible(targetVersion, versions) {
  if (!targetVersion || !versions || !Array.isArray(versions)) return false
  const clean = targetVersion.trim().toLowerCase()
  return versions.some(v => {
    const vc = (v || '').toLowerCase()
    return vc === clean || clean.startsWith(vc) || vc.startsWith(clean)
  })
}

function formatNumber(num) {
  if (!num && num !== 0) return '0'
  if (num >= 1000000) return (num / 1000000).toFixed(1) + 'M'
  if (num >= 1000) return (num / 1000).toFixed(1) + 'k'
  return num.toString()
}

function formatBytes(bytes) {
  if (!bytes) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + ' ' + sizes[i]
}
</script>

<template>
  <div class="mod-marketplace">
    <!-- Top Header & Navigation -->
    <div class="marketplace-header">
      <div class="title-section">
        <h2>
          <icon name="puzzle" class="title-icon" />
          Mod Marketplace
          <span class="badge modrinth-badge">Modrinth</span>
        </h2>
        <p class="subtitle">
          Browse, download, and manage Minecraft mods directly on your server.
        </p>
      </div>

      <div class="nav-controls">
        <div class="view-switch">
          <button
            :class="['view-pill', activeView === 'marketplace' ? 'active' : '']"
            type="button"
            @click="activeView = 'marketplace'"
          >
            <icon name="puzzle" />
            Browse Mods
          </button>
          <button
            :class="['view-pill', activeView === 'installed' ? 'active' : '']"
            type="button"
            @click="activeView = 'installed'"
          >
            <icon name="files" />
            Installed ({{ installedMods.length }})
          </button>
        </div>

        <btn variant="outline" color="neutral" @click="fetchInstalledMods(); searchMods()">
          <icon name="reload" />
          Refresh
        </btn>
      </div>
    </div>

    <!-- Active Environment Notice -->
    <div v-if="detectedLoader || detectedVersion" class="env-chip-bar">
      <span class="chip-label">Target Environment:</span>
      <span class="chip">
        <icon name="settings" />
        Loader: <strong>{{ detectedLoader === 'vanilla' ? 'Vanilla (No Loader)' : (detectedLoader || 'Fabric') }}</strong>
      </span>
      <span v-if="detectedVersion" class="chip">
        Minecraft: <strong>{{ detectedVersion }}</strong>
      </span>
      <span class="chip mods-count">
        Installed Mods: <strong>{{ installedMods.length }}</strong>
      </span>
    </div>

    <!-- Vanilla Minecraft One-Click Mod Loader Activation -->
    <div v-if="detectedLoader === 'vanilla'" class="vanilla-banner">
      <div class="vanilla-banner-left">
        <div class="vanilla-icon-box">
          <icon name="puzzle" />
        </div>
        <div class="vanilla-text">
          <span class="vanilla-title">Vanilla Minecraft Server</span>
          <span class="vanilla-desc">Vanilla Minecraft cannot run mods directly. Click to automatically enable <strong>Fabric Mod Loader</strong> on this server so your mods run seamlessly!</span>
        </div>
      </div>
      <btn
        color="primary"
        class="vanilla-action-btn"
        :disabled="enablingFabric || enableFabricSuccess"
        @click="enableFabric"
      >
        <loader v-if="enablingFabric" />
        <icon v-else-if="enableFabricSuccess" name="checkmark" />
        <icon v-else name="install" />
        {{ enableFabricSuccess ? 'Fabric Enabled!' : enablingFabric ? 'Configuring Fabric...' : 'Enable Fabric Loader' }}
      </btn>
    </div>

    <!-- BROWSE MARKETPLACE VIEW -->
    <div v-if="activeView === 'marketplace'" class="browse-view">
      <!-- Mobile Filter Toggle Bar -->
      <div class="mobile-filter-bar">
        <button
          type="button"
          class="mobile-filter-toggle"
          @click="mobileFiltersOpen = !mobileFiltersOpen"
        >
          <icon name="settings" />
          <span>Filters</span>
          <span v-if="activeFilterCount" class="filter-count-badge">{{ activeFilterCount }}</span>
          <icon :name="mobileFiltersOpen ? 'up' : 'down'" />
        </button>
        <button
          v-if="hasActiveFilters"
          type="button"
          class="mobile-reset-btn"
          @click="resetFilters"
        >
          Reset Filters
        </button>
      </div>

      <!-- 2-Column Marketplace Layout -->
      <div class="marketplace-columns">
        <!-- LEFT SIDEBAR: Filters (Modrinth Style) -->
        <aside :class="['filters-sidebar', mobileFiltersOpen ? 'mobile-open' : '']">
          <div class="sidebar-header">
            <div class="sidebar-title">
              <icon name="settings" />
              <span>Filters</span>
              <span v-if="activeFilterCount" class="sidebar-count-badge">{{ activeFilterCount }}</span>
            </div>
            <button
              v-if="hasActiveFilters"
              type="button"
              class="sidebar-reset-btn"
              title="Reset all filters"
              @click="resetFilters"
            >
              Reset
            </button>
          </div>

          <!-- Section 1: Side / Environment -->
          <div class="sidebar-group">
            <div class="sidebar-group-title">
              <span>Side / Environment</span>
            </div>
            <div class="sidebar-nav-list">
              <button
                v-for="env in environmentOptions"
                :key="env.value"
                type="button"
                :class="['sidebar-nav-item', selectedEnvironment === env.value ? 'active' : '']"
                :title="env.title"
                @click="selectedEnvironment = env.value"
              >
                <span v-if="env.dotClass" :class="['dot', env.dotClass]"></span>
                <span v-else class="dot dot-placeholder"></span>
                <span class="item-name">{{ env.label }}</span>
              </button>
            </div>
          </div>

          <!-- Section 2: Mod Loader -->
          <div class="sidebar-group">
            <div class="sidebar-group-title">
              <span>Mod Loader</span>
            </div>
            <div class="sidebar-nav-list">
              <button
                v-for="l in loaders"
                :key="l.value"
                type="button"
                :class="['sidebar-nav-item', selectedLoader === l.value ? 'active' : '']"
                @click="selectedLoader = l.value"
              >
                <span class="item-name">{{ l.label }}</span>
                <span v-if="l.value && l.value === detectedLoader" class="item-badge server-badge" title="Server mod loader">
                  Server
                </span>
              </button>
            </div>
          </div>

          <!-- Section 3: Minecraft Version -->
          <div class="sidebar-group">
            <div class="sidebar-group-title">
              <span>Minecraft Version</span>
            </div>
            <div class="sidebar-select-wrapper">
              <select v-model="selectedVersion">
                <option value="">All Versions</option>
                <optgroup label="Release Versions">
                  <option
                    v-for="v in releaseVersions"
                    :key="v.version"
                    :value="v.version"
                  >
                    {{ v.version }}{{ v.version === detectedVersion ? ' ★ (Server)' : '' }}
                  </option>
                </optgroup>
                <optgroup v-if="snapshotVersions.length" label="Snapshots & Pre-releases">
                  <option
                    v-for="v in snapshotVersions"
                    :key="v.version"
                    :value="v.version"
                  >
                    {{ v.version }}{{ v.version === detectedVersion ? ' ★ (Server)' : '' }}
                  </option>
                </optgroup>
              </select>
            </div>
          </div>

          <!-- Section 4: Categories -->
          <div class="sidebar-group">
            <div class="sidebar-group-title">
              <span>Categories</span>
            </div>
            <div class="sidebar-nav-list categories-list">
              <button
                v-for="c in categories"
                :key="c.value"
                type="button"
                :class="['sidebar-nav-item', selectedCategory === c.value ? 'active' : '']"
                @click="selectedCategory = c.value"
              >
                <span class="item-name">{{ c.label }}</span>
              </button>
            </div>
          </div>
        </aside>

        <!-- RIGHT MAIN AREA: Search Bar, Active Chips & Mods Grid -->
        <main class="marketplace-main">
          <!-- Top Search & Sort Bar -->
          <div class="catalog-topbar">
            <div class="search-box">
              <icon name="puzzle" class="search-icon" />
              <input
                v-model="searchQuery"
                type="text"
                placeholder="Search mods (e.g. Sodium, Lithium, Chunky, Simple Voice Chat)..."
                @input="onSearchInput"
                @keyup.enter="currentPage = 1; searchMods()"
              />
              <button
                v-if="searchQuery"
                class="clear-search-btn"
                type="button"
                title="Clear search"
                @click="searchQuery = ''; currentPage = 1; searchMods()"
              >
                <icon name="close" />
              </button>
            </div>

            <div class="sort-box">
              <label for="mod-sort">Sort:</label>
              <select id="mod-sort" v-model="sortBy">
                <option v-for="s in sortOptions" :key="s.value" :value="s.value">{{ s.label }}</option>
              </select>
            </div>
          </div>

          <!-- Active Filter Chips (if any filter is applied) -->
          <div v-if="hasActiveFilters" class="active-chips-bar">
            <span class="chips-label">Active:</span>

            <span v-if="searchQuery.trim()" class="active-chip">
              "{{ searchQuery.trim() }}"
              <button type="button" @click="searchQuery = ''; currentPage = 1; searchMods()">×</button>
            </span>

            <span v-if="selectedEnvironment" class="active-chip">
              Side: {{ getEnvLabel(selectedEnvironment) }}
              <button type="button" @click="selectedEnvironment = ''">×</button>
            </span>

            <span v-if="selectedLoader && selectedLoader !== detectedLoader" class="active-chip">
              Loader: {{ getLoaderLabel(selectedLoader) }}
              <button type="button" @click="selectedLoader = detectedLoader || ''">×</button>
            </span>

            <span v-if="selectedVersion && selectedVersion !== detectedVersion" class="active-chip">
              MC: {{ selectedVersion }}
              <button type="button" @click="selectedVersion = detectedVersion || ''">×</button>
            </span>

            <span v-if="selectedCategory" class="active-chip">
              {{ getCategoryLabel(selectedCategory) }}
              <button type="button" @click="selectedCategory = ''">×</button>
            </span>

            <button type="button" class="clear-all-chips-btn" @click="resetFilters">
              Clear All
            </button>
          </div>

      <!-- Loading State -->
      <div v-if="loading" class="state-container">
        <loader />
        <span>Searching Modrinth...</span>
      </div>

      <!-- Error State -->
      <div v-else-if="error" class="alert error">
        <icon name="close" />
        <span>{{ error }}</span>
        <btn color="neutral" @click="searchMods">Try Again</btn>
      </div>

      <!-- Empty State -->
      <div v-else-if="mods.length === 0" class="state-container empty-state">
        <icon name="puzzle" class="huge-icon" />
        <h3>No mods found</h3>
        <p>Try refining your search keyword or relaxing version/loader filters.</p>
        <btn color="primary" @click="resetFilters">
          Reset Filters
        </btn>
      </div>

      <!-- Mods Grid & Pagination -->
      <div v-else class="mods-section">
        <div class="results-header-bar">
          <span class="results-count">
            Showing <strong>{{ (currentPage - 1) * pageSize + 1 }}</strong> - <strong>{{ Math.min(currentPage * pageSize, totalHits) }}</strong> of <strong>{{ totalHits.toLocaleString() }}</strong> mods
          </span>
          <div v-if="totalPages > 1" class="top-pagination-compact">
            <span>Page <strong>{{ currentPage }}</strong> of <strong>{{ totalPages }}</strong></span>
            <button
              type="button"
              class="compact-nav-btn"
              :disabled="currentPage <= 1 || loading"
              title="Previous Page"
              @click="prevPage"
            >
              <icon name="chevron-left" />
            </button>
            <button
              type="button"
              class="compact-nav-btn"
              :disabled="currentPage >= totalPages || loading"
              title="Next Page"
              @click="nextPage"
            >
              <icon name="chevron-right" />
            </button>
          </div>
        </div>

        <div class="mods-grid">
        <div
          v-for="mod in mods"
          :key="mod.project_id"
          :class="['mod-card', getInstalledFile(mod) ? 'is-installed' : '']"
          title="Click to view full description and details"
          @click="openModDetails(mod)"
        >
          <div class="mod-card-top">
            <img
              v-if="mod.icon_url"
              :src="mod.icon_url"
              :alt="mod.title"
              class="mod-icon"
              loading="lazy"
              @error="(e) => e.target.style.display = 'none'"
            />
            <div v-else class="mod-icon-placeholder">
              <icon name="puzzle" />
            </div>

            <div class="mod-info">
              <div class="mod-header-line">
                <h4 class="mod-title" :title="mod.title">{{ mod.title }}</h4>
                <span class="mod-author">by {{ mod.author }}</span>
              </div>

              <p class="mod-desc" :title="mod.description">
                {{ mod.description }}
              </p>

              <div class="mod-tags">
                <span
                  v-for="cat in (mod.display_categories || mod.categories || []).slice(0, 2)"
                  :key="cat"
                  class="tag"
                >
                  {{ cat }}
                </span>
                <span
                  class="tag mc-version-tag"
                  :class="isServerVersionCompatible(detectedVersion, mod.versions) ? 'mc-compat-tag' : ''"
                  :title="'Supported Minecraft: ' + getVersionRange(mod.versions) + (isServerVersionCompatible(detectedVersion, mod.versions) ? ' (Compatible with server ' + detectedVersion + ')' : '')"
                >
                  <icon name="minecraft" /> {{ getVersionRange(mod.versions) }}
                </span>
                <span v-if="mod.server_side === 'required'" class="tag env-tag env-server-req" title="Required on server">
                  Server
                </span>
                <span v-else-if="mod.server_side === 'optional'" class="tag env-tag env-server-opt" title="Optional on server">
                  Server (Opt)
                </span>
                <span v-else-if="mod.server_side === 'unsupported'" class="tag env-tag env-unsupported" title="Not supported on dedicated server">
                  Client Only
                </span>
                <span v-if="mod.client_side === 'unsupported'" class="tag env-tag env-server-only" title="Dedicated server only (no client required)">
                  Server Only
                </span>
                <span v-else-if="mod.client_side === 'required' && mod.server_side !== 'unsupported'" class="tag env-tag env-client-req" title="Required on client">
                  Client
                </span>
              </div>
            </div>
          </div>

          <div class="mod-card-footer">
            <div class="mod-stats">
              <span class="stat" :title="mod.downloads + ' downloads'">
                <icon name="files" />
                {{ formatNumber(mod.downloads) }}
              </span>
              <span v-if="mod.follows" class="stat" :title="mod.follows + ' followers'">
                ★ {{ formatNumber(mod.follows) }}
              </span>
              <span class="stat mc-stat" :title="'Supported Minecraft: ' + getVersionRange(mod.versions)">
                <icon name="minecraft" /> {{ getVersionRange(mod.versions) }}
              </span>
            </div>

            <div class="mod-actions" @click.stop>
              <button
                class="versions-btn"
                type="button"
                title="Select specific version"
                @click.stop="openVersionsModal(mod)"
              >
                Versions
              </button>

              <!-- If already installed -->
              <template v-if="getInstalledFile(mod)">
                <span class="installed-badge" title="Already installed in mods/ folder">
                  ✓ Installed
                </span>
                <btn
                  variant="icon"
                  color="danger"
                  class="action-btn-small"
                  :tooltip="'Uninstall ' + getInstalledFile(mod).name"
                  :disabled="uninstalling[getInstalledFile(mod).name]"
                  @click.stop="uninstallMod(getInstalledFile(mod).name)"
                >
                  <icon :name="uninstalling[getInstalledFile(mod).name] ? 'loading' : 'close'" :spin="uninstalling[getInstalledFile(mod).name]" />
                </btn>
              </template>

              <!-- If not installed -->
              <template v-else>
                <btn
                  color="primary"
                  class="action-btn"
                  :disabled="!canEditFiles || !!installing[mod.project_id]"
                  @click.stop="installMod(mod)"
                >
                  <icon
                    :name="installing[mod.project_id] ? 'loading' : 'install'"
                    :spin="!!installing[mod.project_id]"
                  />
                  {{ installing[mod.project_id] ? installing[mod.project_id].status : 'Install' }}
                </btn>
              </template>
            </div>
          </div>
        </div>
      </div>

      <!-- Bottom Pagination Controls -->
      <div v-if="totalPages > 1" class="pagination-bar">
        <div class="pagination-info">
          Showing
          <strong>{{ (currentPage - 1) * pageSize + 1 }}</strong>
          -
          <strong>{{ Math.min(currentPage * pageSize, totalHits) }}</strong>
          of
          <strong>{{ totalHits.toLocaleString() }}</strong>
          mods
          <span class="page-indicator-text">(Page {{ currentPage }} of {{ totalPages }})</span>
        </div>

        <div class="pagination-controls">
          <button
            type="button"
            class="page-nav-btn prev-btn"
            :disabled="currentPage <= 1 || loading"
            title="Previous Page"
            @click="prevPage"
          >
            <icon name="chevron-left" />
            <span>Previous</span>
          </button>

          <div class="page-numbers-list">
            <template v-for="(p, idx) in visiblePages" :key="idx">
              <span v-if="p === '...'" class="page-ellipsis">…</span>
              <button
                v-else
                type="button"
                :class="['page-num-btn', p === currentPage ? 'active' : '']"
                :disabled="loading"
                @click="goToPage(p)"
              >
                {{ p }}
              </button>
            </template>
          </div>

          <button
            type="button"
            class="page-nav-btn next-btn"
            :disabled="currentPage >= totalPages || loading"
            title="Next Page"
            @click="nextPage"
          >
            <span>Next Page</span>
            <icon name="chevron-right" />
          </button>
        </div>
      </div>
    </div>
  </main>
</div>
</div>

    <!-- INSTALLED MODS VIEW -->
    <div v-else class="installed-view">
      <div class="installed-header-bar">
        <h3>
          Installed Mods ({{ installedMods.length }})
        </h3>
        <p class="section-desc">
          These .jar files are currently present in your server's <code>mods/</code> directory.
        </p>
      </div>

      <div v-if="loadingInstalled" class="state-container">
        <loader />
        <span>Scanning mods folder...</span>
      </div>

      <div v-else-if="installedMods.length === 0" class="state-container empty-state">
        <icon name="puzzle" class="huge-icon" />
        <h3>No mods installed yet</h3>
        <p>You haven't installed any mods into this server's <code>mods/</code> directory.</p>
        <btn color="primary" @click="activeView = 'marketplace'">
          Browse Mod Marketplace
        </btn>
      </div>

      <div v-else class="installed-list">
        <div
          v-for="file in installedMods"
          :key="file.name"
          class="installed-item"
        >
          <div class="installed-item-info">
            <icon name="file-jar" class="jar-icon" />
            <div>
              <span class="file-name">{{ file.name }}</span>
              <div class="file-meta">
                <span>{{ formatBytes(file.size) }}</span>
                <span v-if="file.modifyTime">
                  • Modified {{ new Date(file.modifyTime * 1000).toLocaleDateString() }}
                </span>
              </div>
            </div>
          </div>

          <div class="installed-item-actions">
            <btn
              variant="outline"
              color="danger"
              :disabled="uninstalling[file.name]"
              @click="uninstallMod(file.name)"
            >
              <icon :name="uninstalling[file.name] ? 'loading' : 'remove'" :spin="uninstalling[file.name]" />
              Uninstall
            </btn>
          </div>
        </div>
      </div>
    </div>

    <!-- MOD DETAILS & FULL DESCRIPTION OVERLAY -->
    <overlay
      v-model="descModalOpen"
      :title="descModalMod ? descModalMod.title : 'Mod Details'"
      closable
      class="desc-overlay"
    >
      <div v-if="descModalLoading" class="state-container">
        <loader />
        <span>Loading mod description and details...</span>
      </div>

      <div v-else-if="descModalMod" class="mod-details-content">
        <!-- Hero Header -->
        <div class="details-hero">
          <img
            v-if="descModalMod.icon_url"
            :src="descModalMod.icon_url"
            :alt="descModalMod.title"
            class="details-icon"
            @error="(e) => e.target.style.display = 'none'"
          />
          <div v-else class="details-icon-placeholder">
            <icon name="puzzle" />
          </div>

          <div class="details-main-info">
            <div class="details-title-row">
              <h2 class="details-title">{{ descModalMod.title }}</h2>
              <span class="details-author">by <strong>{{ descModalMod.author }}</strong></span>
            </div>

            <p class="details-summary">
              {{ descModalMod.description }}
            </p>

            <div class="details-chips">
              <span class="detail-chip" title="Total Downloads">
                <icon name="files" /> {{ formatNumber(descModalMod.downloads) }} downloads
              </span>
              <span v-if="descModalMod.follows" class="detail-chip" title="Followers">
                ★ {{ formatNumber(descModalMod.follows) }} followers
              </span>
              <span class="detail-chip mc-chip" :title="'Supported Minecraft Versions: ' + getVersionRange(descModalProject?.game_versions || descModalMod.versions)">
                <icon name="minecraft" /> Minecraft {{ getVersionRange(descModalProject?.game_versions || descModalMod.versions) }}
              </span>
              <span v-if="descModalProject?.license?.id || descModalProject?.license?.name" class="detail-chip license-chip">
                License: {{ descModalProject?.license?.id || descModalProject?.license?.name }}
              </span>

              <!-- Environment Chips -->
              <span v-if="descModalMod.server_side === 'required'" class="tag env-tag env-server-req" title="Required on server">
                Server: Required
              </span>
              <span v-else-if="descModalMod.server_side === 'optional'" class="tag env-tag env-server-opt" title="Optional on server">
                Server: Optional
              </span>
              <span v-else-if="descModalMod.server_side === 'unsupported'" class="tag env-tag env-unsupported" title="Not supported on dedicated server">
                Server: Unsupported
              </span>

              <span v-if="descModalMod.client_side === 'required'" class="tag env-tag env-client-req" title="Required on client">
                Client: Required
              </span>
              <span v-else-if="descModalMod.client_side === 'optional'" class="tag env-tag env-server-opt" title="Optional on client">
                Client: Optional
              </span>
              <span v-else-if="descModalMod.client_side === 'unsupported'" class="tag env-tag env-server-only" title="Dedicated server only">
                Client: Unsupported (Server Only)
              </span>
            </div>

            <!-- External Links -->
            <div class="details-links">
              <a
                v-if="descModalMod.slug"
                :href="'https://modrinth.com/mod/' + descModalMod.slug"
                target="_blank"
                rel="noopener"
                class="ext-link"
              >
                <icon name="web" /> Modrinth Page ↗
              </a>
              <a
                v-if="descModalProject?.source_url"
                :href="descModalProject.source_url"
                target="_blank"
                rel="noopener"
                class="ext-link"
              >
                <icon name="github" /> Source Code ↗
              </a>
              <a
                v-if="descModalProject?.issues_url"
                :href="descModalProject.issues_url"
                target="_blank"
                rel="noopener"
                class="ext-link"
              >
                <icon name="bug" /> Issues ↗
              </a>
              <a
                v-if="descModalProject?.wiki_url"
                :href="descModalProject.wiki_url"
                target="_blank"
                rel="noopener"
                class="ext-link"
              >
                <icon name="book" /> Wiki ↗
              </a>
            </div>
          </div>

          <!-- Hero Action Controls -->
          <div class="details-actions">
            <template v-if="getInstalledFile(descModalMod)">
              <div class="installed-status-box">
                <span class="installed-badge-large">
                  ✓ Installed
                </span>
                <span class="installed-filename" :title="getInstalledFile(descModalMod).name">
                  {{ getInstalledFile(descModalMod).name }}
                </span>
                <btn
                  color="danger"
                  variant="outline"
                  class="btn-uninstall-modal"
                  :disabled="uninstalling[getInstalledFile(descModalMod).name]"
                  @click="uninstallMod(getInstalledFile(descModalMod).name)"
                >
                  <icon :name="uninstalling[getInstalledFile(descModalMod).name] ? 'loading' : 'close'" :spin="uninstalling[getInstalledFile(descModalMod).name]" />
                  Uninstall
                </btn>
              </div>
            </template>
            <template v-else>
              <btn
                color="primary"
                class="btn-install-modal"
                :disabled="!canEditFiles || !!installing[descModalMod.project_id]"
                @click="installMod(descModalMod)"
              >
                <icon
                  :name="installing[descModalMod.project_id] ? 'loading' : 'install'"
                  :spin="!!installing[descModalMod.project_id]"
                />
                {{ installing[descModalMod.project_id] ? installing[descModalMod.project_id].status : 'Install Mod' }}
              </btn>
            </template>

            <button
              class="versions-btn modal-ver-btn"
              type="button"
              @click="openVersionsModal(descModalMod)"
            >
              Choose Version...
            </button>
          </div>
        </div>

        <hr class="details-divider" />

        <!-- Minecraft & Technical Support Section (Like Modrinth Project Page) -->
        <div class="support-meta-box">
          <div class="support-col">
            <div class="support-col-header">
              <div class="support-title-group">
                <icon name="minecraft" class="support-icon" />
                <span class="support-col-title">Minecraft Support</span>
              </div>
              <span class="support-range-badge">
                {{ getVersionRange(descModalProject?.game_versions || descModalMod.versions) }}
              </span>
            </div>

            <!-- Server Compatibility Notice -->
            <div
              v-if="detectedVersion"
              :class="['server-compat-banner', isServerVersionCompatible(detectedVersion, descModalProject?.game_versions || descModalMod.versions) ? 'compat-yes' : 'compat-warn']"
            >
              <icon :name="isServerVersionCompatible(detectedVersion, descModalProject?.game_versions || descModalMod.versions) ? 'checkbox-marked-circle' : 'alert'" />
              <span>
                <template v-if="isServerVersionCompatible(detectedVersion, descModalProject?.game_versions || descModalMod.versions)">
                  Compatible with this server's Minecraft <strong>{{ detectedVersion }}</strong>
                </template>
                <template v-else>
                  Server runs <strong>{{ detectedVersion }}</strong> — this mod might not have a tested build for this version
                </template>
              </span>
            </div>

            <!-- All Supported Game Versions -->
            <div class="versions-pill-list">
              <span
                v-for="ver in (showAllVersions ? getSupportedVersionsList(descModalProject?.game_versions || descModalMod.versions) : getSupportedVersionsList(descModalProject?.game_versions || descModalMod.versions).slice(0, 16))"
                :key="ver"
                :class="['ver-pill', ver === detectedVersion ? 'active-server-ver' : '']"
                :title="ver === detectedVersion ? 'Matches server Minecraft ' + detectedVersion : ver"
              >
                {{ ver }}
                <span v-if="ver === detectedVersion" class="check">✓</span>
              </span>
              <button
                v-if="getSupportedVersionsList(descModalProject?.game_versions || descModalMod.versions).length > 16"
                type="button"
                class="ver-pill-toggle"
                @click="showAllVersions = !showAllVersions"
              >
                {{ showAllVersions ? 'Show Less' : `+${getSupportedVersionsList(descModalProject?.game_versions || descModalMod.versions).length - 16} more` }}
              </button>
            </div>
          </div>

          <div class="support-col loaders-col">
            <div class="support-col-header">
              <div class="support-title-group">
                <icon name="puzzle" class="support-icon" />
                <span class="support-col-title">Loaders</span>
              </div>
            </div>
            <div class="loaders-pill-list">
              <span
                v-for="ldr in (descModalProject?.loaders || (descModalMod.categories || []).filter(c => ['fabric', 'forge', 'neoforge', 'quilt'].includes(c)))"
                :key="ldr"
                :class="['loader-pill', ldr.toLowerCase() === detectedLoader ? 'active-server-loader' : '']"
                :title="ldr.toLowerCase() === detectedLoader ? 'Matches server loader ' + detectedLoader : ldr"
              >
                {{ ldr }}
                <span v-if="ldr.toLowerCase() === detectedLoader" class="check">✓</span>
              </span>
            </div>
          </div>
        </div>

        <!-- Dependencies Section -->
        <div v-if="descModalDependencies && descModalDependencies.length" class="dependencies-box">
          <div class="dep-header">
            <div class="dep-header-title-group">
              <icon name="puzzle" class="dep-icon" />
              <span class="dep-header-title">Dependencies ({{ descModalDependencies.length }})</span>
            </div>
            <span class="dep-header-sub">Required dependencies are auto-detected & downloaded on install</span>
          </div>
          <div class="dep-list">
            <div
              v-for="dep in descModalDependencies"
              :key="dep.project_id || dep.version_id"
              class="dep-item"
            >
              <div class="dep-item-left">
                <img
                  v-if="dep.icon_url"
                  :src="dep.icon_url"
                  class="dep-item-icon"
                  alt="icon"
                />
                <div v-else class="dep-item-icon-ph">
                  <icon name="puzzle" />
                </div>
                <div class="dep-item-info">
                  <span class="dep-item-title">{{ dep.title }}</span>
                  <span :class="['dep-type-badge', dep.dependency_type]">
                    {{ dep.dependency_type === 'required' ? 'Required' : 'Optional' }}
                  </span>
                </div>
              </div>
              <div class="dep-item-right">
                <span v-if="getInstalledFile(dep)" class="dep-status-installed">
                  <icon name="checkbox-marked-circle" />
                  Installed ({{ getInstalledFile(dep).name }})
                </span>
                <span v-else-if="dep.dependency_type === 'required'" class="dep-status-missing">
                  <icon name="alert" />
                  Missing — Auto-installs on download
                </span>
                <span v-else class="dep-status-optional">
                  Optional
                </span>
              </div>
            </div>
          </div>
        </div>

        <hr class="details-divider" />

        <!-- Description Content -->
        <div class="details-body-wrapper">
          <div class="body-header">
            <h3 class="section-title">Full Description</h3>
            <span v-if="descModalProject?.loaders?.length" class="loaders-summary">
              Loaders: {{ descModalProject.loaders.join(', ') }}
            </span>
          </div>

          <div
            v-if="descModalProject?.body || descModalMod.description"
            class="mod-markdown-body"
            v-html="markdown(descModalProject?.body || descModalMod.description)"
          />
          <p v-else class="no-desc">
            No detailed description available for this mod.
          </p>
        </div>
      </div>
    </overlay>

    <!-- SPECIFIC VERSION SELECTOR OVERLAY -->
    <overlay
      v-model="versionModalOpen"
      :title="'Install Version: ' + (versionModalMod ? versionModalMod.title : '')"
      closable
      class="versions-overlay"
    >
      <div v-if="versionModalLoading" class="state-container">
        <loader />
        <span>Fetching versions from Modrinth...</span>
      </div>

      <div v-else class="versions-table-wrapper">
        <p class="modal-hint">
          Choose a specific release or build of <strong>{{ versionModalMod?.title }}</strong> to download into <code>mods/</code>.
        </p>

        <table class="versions-table">
          <thead>
            <tr>
              <th>Version</th>
              <th>Type</th>
              <th>Minecraft</th>
              <th>Loaders</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="v in versionModalVersions" :key="v.id">
              <td>
                <strong>{{ v.name || v.version_number }}</strong>
                <div class="subtext">{{ v.version_number }}</div>
              </td>
              <td>
                <span :class="['version-type-badge', v.version_type]">
                  {{ v.version_type }}
                </span>
              </td>
              <td>
                <span class="version-targets">
                  {{ (v.game_versions || []).slice(0, 3).join(', ') }}
                  <template v-if="(v.game_versions || []).length > 3">
                    +{{ v.game_versions.length - 3 }} more
                  </template>
                </span>
              </td>
              <td>
                <span class="version-loaders">
                  {{ (v.loaders || []).join(', ') }}
                </span>
              </td>
              <td>
                <btn
                  color="primary"
                  class="btn-sm"
                  :disabled="!!installing[versionModalMod?.project_id]"
                  @click="installMod(versionModalMod, v.files?.find(f => f.primary) || v.files?.[0], v)"
                >
                  <icon name="install" />
                  Install
                </btn>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </overlay>
  </div>
</template>

<style scoped>
.mod-marketplace {
  padding: 0.5rem 0;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.marketplace-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 1rem;
}

.title-section h2 {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0 0 0.25rem 0;
  font-size: 1.5rem;
}

.title-icon {
  color: #1bd96a;
  font-size: 1.6rem;
}

.modrinth-badge {
  background: #1bd96a;
  color: #0b1a11;
  font-size: 0.7rem;
  font-weight: 700;
  text-transform: uppercase;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  letter-spacing: 0.05em;
}

.subtitle {
  margin: 0;
  opacity: 0.75;
  font-size: 0.9rem;
}

.nav-controls {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.view-switch {
  display: flex;
  background: rgba(255, 255, 255, 0.06);
  padding: 3px;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.view-pill {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  background: transparent;
  border: none;
  color: inherit;
  opacity: 0.7;
  padding: 0.45rem 0.9rem;
  border-radius: 6px;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.view-pill:hover {
  opacity: 1;
}

.view-pill.active {
  background: var(--primary, #0070f3);
  color: #fff;
  opacity: 1;
}

.env-chip-bar {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;
  padding: 0.5rem 0.75rem;
  background: rgba(0, 150, 255, 0.06);
  border: 1px solid rgba(0, 150, 255, 0.15);
  border-radius: 6px;
  font-size: 0.85rem;
}

.chip-label {
  opacity: 0.7;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  background: rgba(255, 255, 255, 0.08);
  padding: 0.2rem 0.6rem;
  border-radius: 4px;
}

.chip.mods-count {
  margin-left: auto;
  background: rgba(27, 217, 106, 0.12);
  color: #1bd96a;
}

/* Vanilla Minecraft Banner */
.vanilla-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  background: linear-gradient(90deg, rgba(245, 158, 11, 0.12) 0%, rgba(0, 112, 243, 0.1) 100%);
  border: 1px solid rgba(245, 158, 11, 0.35);
  border-radius: 8px;
  padding: 0.85rem 1.15rem;
  flex-wrap: wrap;
}

.vanilla-banner-left {
  display: flex;
  align-items: center;
  gap: 0.85rem;
  flex: 1;
  min-width: 260px;
}

.vanilla-icon-box {
  width: 38px;
  height: 38px;
  border-radius: 8px;
  background: rgba(245, 158, 11, 0.2);
  color: #f59e0b;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.25rem;
  flex-shrink: 0;
}

.vanilla-text {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.vanilla-title {
  font-size: 0.95rem;
  font-weight: 700;
  color: #fbbf24;
}

.vanilla-desc {
  font-size: 0.82rem;
  opacity: 0.85;
  line-height: 1.4;
}

.vanilla-action-btn {
  white-space: nowrap;
  flex-shrink: 0;
}

/* Mobile Filter Bar */
.mobile-filter-bar {
  display: none;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.mobile-filter-toggle {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.55rem 0.9rem;
  background: #191b22;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 8px;
  color: #f1f5f9;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
}

.filter-count-badge {
  background: #0284c7;
  color: #ffffff;
  font-size: 0.72rem;
  padding: 0.1rem 0.45rem;
  border-radius: 10px;
  font-weight: 700;
}

.mobile-reset-btn {
  background: rgba(239, 68, 68, 0.12);
  border: 1px solid rgba(239, 68, 68, 0.25);
  color: #f87171;
  font-size: 0.82rem;
  font-weight: 600;
  padding: 0.45rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
}

/* 2-Column Marketplace Layout */
.marketplace-columns {
  display: flex;
  gap: 1.25rem;
  align-items: flex-start;
  margin-top: 0.5rem;
}

/* Left Sidebar (Modrinth Style) */
.filters-sidebar {
  width: 250px;
  flex-shrink: 0;
  background: #161820;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 10px;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1.15rem;
  position: sticky;
  top: 1rem;
}

.sidebar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-bottom: 0.75rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.sidebar-title {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  font-size: 0.95rem;
  font-weight: 700;
  color: #f1f5f9;
}

.sidebar-count-badge {
  background: #0284c7;
  color: #ffffff;
  font-size: 0.7rem;
  padding: 0.1rem 0.45rem;
  border-radius: 10px;
  font-weight: 700;
}

.sidebar-reset-btn {
  font-size: 0.76rem;
  font-weight: 600;
  color: #f87171;
  background: rgba(239, 68, 68, 0.12);
  border: 1px solid rgba(239, 68, 68, 0.25);
  padding: 0.2rem 0.55rem;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.15s ease;
}

.sidebar-reset-btn:hover {
  background: rgba(239, 68, 68, 0.22);
  border-color: rgba(239, 68, 68, 0.45);
  color: #fca5a5;
}

.sidebar-group {
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
}

.sidebar-group-title {
  font-size: 0.74rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  color: #94a3b8;
  padding: 0 0.2rem;
}

.sidebar-nav-list {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.sidebar-nav-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5rem;
  width: 100%;
  padding: 0.42rem 0.65rem;
  border-radius: 6px;
  font-size: 0.84rem;
  font-weight: 500;
  color: #cbd5e1;
  background: transparent;
  border: 1px solid transparent;
  cursor: pointer;
  text-align: left;
  transition: all 0.15s ease;
  user-select: none;
}

.sidebar-nav-item:hover:not(.active) {
  background: rgba(255, 255, 255, 0.05);
  color: #ffffff;
}

.sidebar-nav-item.active {
  background: #0284c7;
  border-color: #38bdf8;
  color: #ffffff;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(2, 132, 199, 0.35);
}

.sidebar-nav-item .dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
}

.sidebar-nav-item .dot-placeholder {
  visibility: hidden;
}

.sidebar-nav-item .server-dot {
  background-color: #22c55e;
}

.sidebar-nav-item .server-only-dot {
  background-color: #06b6d4;
}

.sidebar-nav-item .both-dot {
  background-color: #a855f7;
}

.sidebar-nav-item .client-dot {
  background-color: #38bdf8;
}

.sidebar-nav-item .client-only-dot {
  background-color: #ef4444;
}

.sidebar-nav-item.active .dot {
  box-shadow: 0 0 6px currentColor;
}

.item-name {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.server-badge {
  font-size: 0.65rem;
  font-weight: 700;
  padding: 0.1rem 0.35rem;
  border-radius: 4px;
  background: rgba(27, 217, 106, 0.18);
  color: #1bd96a;
  border: 1px solid rgba(27, 217, 106, 0.3);
}

.sidebar-nav-item.active .server-badge {
  background: rgba(255, 255, 255, 0.25);
  color: #ffffff;
  border-color: rgba(255, 255, 255, 0.4);
}

.sidebar-select-wrapper {
  position: relative;
  width: 100%;
}

.sidebar-select-wrapper select {
  width: 100%;
  color-scheme: dark;
  background-color: #111319;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 6px;
  color: #f1f5f9;
  padding: 0.45rem 1.8rem 0.45rem 0.65rem;
  font-size: 0.84rem;
  font-weight: 500;
  outline: none;
  cursor: pointer;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2.5' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m6 9 6 6 6-6'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 0.55rem center;
  background-size: 11px;
  transition: all 0.2s ease;
}

.sidebar-select-wrapper select:hover {
  background-color: #181b22;
  border-color: rgba(255, 255, 255, 0.25);
}

.sidebar-select-wrapper select:focus {
  border-color: #38bdf8;
  background-color: #181b22;
  box-shadow: 0 0 0 2px rgba(56, 189, 248, 0.2);
}

.sidebar-select-wrapper select option {
  background-color: #1e212b;
  color: #f1f5f9;
  padding: 0.45rem 0.6rem;
}

.sidebar-select-wrapper select optgroup {
  background-color: #14161d;
  color: #38bdf8;
  font-weight: 700;
  padding: 0.45rem 0.4rem;
}

.categories-list {
  max-height: 220px;
  overflow-y: auto;
  scrollbar-width: thin;
}

/* Right Main Area */
.marketplace-main {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
}

/* Top Search & Sort Bar */
.catalog-topbar {
  display: flex;
  gap: 0.75rem;
  align-items: center;
  background: #161820;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 10px;
  padding: 0.65rem 0.85rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.search-box {
  flex: 1;
  position: relative;
  display: flex;
  align-items: center;
}

.search-box input {
  width: 100%;
  padding: 0.65rem 2.2rem 0.65rem 2.5rem;
  background-color: #111319;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 8px;
  color: #f8fafc;
  font-size: 0.92rem;
  outline: none;
  transition: all 0.2s ease;
}

.search-box input::placeholder {
  color: rgba(255, 255, 255, 0.4);
}

.search-box input:focus {
  border-color: #38bdf8;
  background-color: #151720;
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.2);
}

.search-box .search-icon {
  position: absolute;
  left: 0.85rem;
  color: #94a3b8;
  pointer-events: none;
}

.clear-search-btn {
  position: absolute;
  right: 0.85rem;
  background: rgba(255, 255, 255, 0.1);
  border: none;
  color: #94a3b8;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.15s ease;
}

.clear-search-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  color: #ffffff;
}

.sort-box {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  flex-shrink: 0;
  font-size: 0.85rem;
  color: #94a3b8;
}

.sort-box label {
  font-weight: 600;
  color: #94a3b8;
  white-space: nowrap;
}

.sort-box select {
  color-scheme: dark;
  background-color: #111319;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 8px;
  color: #f1f5f9;
  padding: 0.55rem 1.85rem 0.55rem 0.65rem;
  font-size: 0.85rem;
  font-weight: 500;
  outline: none;
  cursor: pointer;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%2394a3b8' stroke-width='2.5' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='m6 9 6 6 6-6'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 0.55rem center;
  background-size: 11px;
  transition: all 0.2s ease;
}

.sort-box select:hover {
  background-color: #181b22;
  border-color: rgba(255, 255, 255, 0.25);
}

.sort-box select:focus {
  border-color: #38bdf8;
  box-shadow: 0 0 0 2px rgba(56, 189, 248, 0.2);
}

.sort-box select option {
  background-color: #1e212b;
  color: #f1f5f9;
  padding: 0.45rem 0.6rem;
}

/* Active Chips Bar */
.active-chips-bar {
  display: flex;
  align-items: center;
  gap: 0.45rem;
  flex-wrap: wrap;
  padding: 0.4rem 0.6rem;
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 7px;
}

.chips-label {
  font-size: 0.76rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #94a3b8;
  margin-right: 0.15rem;
}

.active-chip {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.78rem;
  font-weight: 600;
  padding: 0.2rem 0.55rem;
  background: rgba(56, 189, 248, 0.12);
  color: #38bdf8;
  border: 1px solid rgba(56, 189, 248, 0.25);
  border-radius: 5px;
}

.active-chip button {
  background: none;
  border: none;
  color: inherit;
  font-size: 0.95rem;
  line-height: 1;
  cursor: pointer;
  padding: 0;
  opacity: 0.7;
  transition: opacity 0.15s;
}

.active-chip button:hover {
  opacity: 1;
}

.clear-all-chips-btn {
  background: none;
  border: none;
  color: #ef4444;
  font-size: 0.78rem;
  font-weight: 600;
  cursor: pointer;
  padding: 0.2rem 0.45rem;
  text-decoration: underline;
  opacity: 0.85;
}

.clear-all-chips-btn:hover {
  opacity: 1;
}

@media (max-width: 860px) {
  .mobile-filter-bar {
    display: flex;
  }
  .marketplace-columns {
    flex-direction: column;
  }
  .filters-sidebar {
    width: 100%;
    position: static;
    display: none;
  }
  .filters-sidebar.mobile-open {
    display: flex;
  }
}

/* State Containers */
.state-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 3rem 1rem;
  gap: 1rem;
  opacity: 0.8;
}

.empty-state .huge-icon {
  font-size: 3rem;
  opacity: 0.4;
}

.empty-state h3 {
  margin: 0;
}

/* Mods Grid */
.mods-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
  gap: 1rem;
}

.mod-card {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.09);
  border-radius: 8px;
  padding: 1rem;
  transition: transform 0.15s ease, border-color 0.15s ease, box-shadow 0.15s ease;
  cursor: pointer;
}

.mod-card:hover {
  border-color: rgba(255, 255, 255, 0.3);
  box-shadow: 0 6px 18px rgba(0, 0, 0, 0.35);
  transform: translateY(-2px);
}

.mod-card:hover .mod-title {
  color: #58a6ff;
}

.mod-card.is-installed {
  border-color: rgba(27, 217, 106, 0.4);
  background: rgba(27, 217, 106, 0.02);
}

.mod-card-top {
  display: flex;
  gap: 0.85rem;
}

.mod-icon,
.mod-icon-placeholder {
  width: 52px;
  height: 52px;
  border-radius: 8px;
  object-fit: cover;
  flex-shrink: 0;
  background: rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.mod-icon-placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  opacity: 0.5;
}

.mod-info {
  flex: 1;
  min-width: 0;
}

.mod-header-line {
  display: flex;
  align-items: baseline;
  gap: 0.5rem;
  margin-bottom: 0.25rem;
}

.mod-title {
  margin: 0;
  font-size: 1.05rem;
  font-weight: 700;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.mod-author {
  font-size: 0.75rem;
  opacity: 0.6;
  white-space: nowrap;
}

.mod-desc {
  margin: 0 0 0.5rem 0;
  font-size: 0.85rem;
  opacity: 0.8;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  line-height: 1.35;
}

.mod-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.3rem;
}

.tag {
  font-size: 0.7rem;
  background: rgba(255, 255, 255, 0.08);
  padding: 0.15rem 0.45rem;
  border-radius: 4px;
  opacity: 0.85;
}

.tag.server-tag {
  background: rgba(0, 150, 255, 0.15);
  color: #58a6ff;
}

.env-tag {
  font-weight: 600;
  font-size: 0.7rem;
}

.env-server-req {
  background: rgba(27, 217, 106, 0.15);
  color: #1bd96a;
  border: 1px solid rgba(27, 217, 106, 0.3);
}

.env-server-opt {
  background: rgba(0, 210, 255, 0.15);
  color: #00d2ff;
  border: 1px solid rgba(0, 210, 255, 0.3);
}

.env-server-only {
  background: rgba(168, 85, 247, 0.15);
  color: #c084fc;
  border: 1px solid rgba(168, 85, 247, 0.3);
}

.env-client-req {
  background: rgba(245, 158, 11, 0.15);
  color: #f59e0b;
  border: 1px solid rgba(245, 158, 11, 0.3);
}

.env-unsupported {
  background: rgba(239, 68, 68, 0.15);
  color: #f87171;
  border: 1px solid rgba(239, 68, 68, 0.3);
}

.mod-card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.85rem;
  padding-top: 0.75rem;
  border-top: 1px solid rgba(255, 255, 255, 0.06);
}

.mod-stats {
  display: flex;
  gap: 0.6rem;
  font-size: 0.78rem;
  opacity: 0.7;
}

.stat {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
}

.mod-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.versions-btn {
  background: none;
  border: 1px solid rgba(255, 255, 255, 0.15);
  color: inherit;
  opacity: 0.75;
  padding: 0.3rem 0.55rem;
  border-radius: 4px;
  font-size: 0.75rem;
  cursor: pointer;
  transition: all 0.15s;
}

.versions-btn:hover {
  opacity: 1;
  border-color: rgba(255, 255, 255, 0.4);
}

.installed-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  color: #1bd96a;
  background: rgba(27, 217, 106, 0.12);
  padding: 0.3rem 0.6rem;
  border-radius: 4px;
  font-size: 0.78rem;
  font-weight: 600;
}

.action-btn {
  font-size: 0.85rem;
  padding: 0.35rem 0.8rem;
}

.action-btn-small {
  padding: 0.35rem;
}

/* Results Header Bar */
.results-header-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.85rem;
  padding: 0.2rem 0.4rem;
  font-size: 0.88rem;
  color: rgba(255, 255, 255, 0.65);
}

.results-count strong {
  color: #f1f5f9;
}

.top-pagination-compact {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.82rem;
}

.top-pagination-compact strong {
  color: #f1f5f9;
}

.compact-nav-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.14);
  color: #f1f5f9;
  cursor: pointer;
  transition: all 0.15s ease;
}

.compact-nav-btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.25);
  color: #ffffff;
}

.compact-nav-btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

/* Bottom Pagination Controls */
.pagination-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
  margin-top: 1.5rem;
  padding: 0.85rem 1.25rem;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 8px;
}

.pagination-info {
  font-size: 0.88rem;
  color: rgba(255, 255, 255, 0.7);
  display: flex;
  align-items: center;
  gap: 0.35rem;
  flex-wrap: wrap;
}

.pagination-info strong {
  color: #f1f5f9;
  font-weight: 600;
}

.page-indicator-text {
  color: rgba(255, 255, 255, 0.45);
  font-size: 0.82rem;
  margin-left: 0.25rem;
}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.page-nav-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.48rem 0.95rem;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.13);
  border-radius: 6px;
  color: #f1f5f9;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
  user-select: none;
}

.page-nav-btn:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(255, 255, 255, 0.25);
  color: #fff;
  transform: translateY(-1px);
}

.page-nav-btn:active:not(:disabled) {
  transform: translateY(0);
}

.page-nav-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}

.page-nav-btn.next-btn {
  background: rgba(14, 165, 233, 0.15);
  border-color: rgba(14, 165, 233, 0.35);
  color: #38bdf8;
}

.page-nav-btn.next-btn:hover:not(:disabled) {
  background: rgba(14, 165, 233, 0.28);
  border-color: rgba(14, 165, 233, 0.6);
  color: #fff;
}

.page-numbers-list {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.page-num-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 36px;
  height: 36px;
  padding: 0 0.35rem;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 6px;
  color: #cbd5e1;
  font-size: 0.85rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
  user-select: none;
}

.page-num-btn:hover:not(:disabled):not(.active) {
  background: rgba(255, 255, 255, 0.1);
  border-color: rgba(255, 255, 255, 0.2);
  color: #fff;
}

.page-num-btn.active {
  background: #0284c7;
  border-color: #38bdf8;
  color: #ffffff;
  font-weight: 700;
  box-shadow: 0 0 10px rgba(14, 165, 233, 0.4);
}

.page-ellipsis {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 28px;
  height: 36px;
  color: rgba(255, 255, 255, 0.4);
  font-weight: 600;
  font-size: 0.9rem;
  user-select: none;
}

@media (max-width: 640px) {
  .pagination-bar {
    flex-direction: column;
    align-items: center;
    text-align: center;
  }
  .page-numbers-list {
    display: none;
  }
}

/* Installed View */
.installed-header-bar {
  margin-bottom: 1rem;
}

.installed-header-bar h3 {
  margin: 0 0 0.25rem 0;
}

.section-desc {
  margin: 0;
  opacity: 0.7;
  font-size: 0.85rem;
}

.installed-list {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.installed-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  padding: 0.75rem 1rem;
  border-radius: 6px;
}

.installed-item-info {
  display: flex;
  align-items: center;
  gap: 0.85rem;
}

.jar-icon {
  font-size: 1.5rem;
  color: #f39c12;
}

.file-name {
  font-weight: 600;
  font-size: 0.95rem;
  display: block;
}

.file-meta {
  font-size: 0.78rem;
  opacity: 0.6;
  margin-top: 0.15rem;
}

/* Versions Overlay */
.versions-overlay {
  min-width: 600px;
  max-width: 900px;
}

.modal-hint {
  margin: 0 0 1rem 0;
  font-size: 0.9rem;
  opacity: 0.8;
}

.versions-table-wrapper {
  max-height: 480px;
  overflow-y: auto;
}

.versions-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.85rem;
}

.versions-table th,
.versions-table td {
  padding: 0.6rem 0.75rem;
  text-align: left;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.versions-table th {
  opacity: 0.6;
  font-weight: 600;
}

.version-type-badge {
  display: inline-block;
  padding: 0.15rem 0.4rem;
  border-radius: 3px;
  font-size: 0.7rem;
  text-transform: capitalize;
}

.version-type-badge.release {
  background: rgba(27, 217, 106, 0.15);
  color: #1bd96a;
}

.version-type-badge.beta {
  background: rgba(243, 156, 18, 0.15);
  color: #f39c12;
}

.version-type-badge.alpha {
  background: rgba(231, 76, 60, 0.15);
  color: #e74c3c;
}

.btn-sm {
  font-size: 0.8rem;
  padding: 0.3rem 0.6rem;
}

.subtext {
  font-size: 0.75rem;
  opacity: 0.6;
}

/* Mod Details / Description Overlay */
.desc-overlay .overlay {
  width: 94vw;
  max-width: 920px;
  max-height: 85vh;
  display: flex;
  flex-direction: column;
  padding: 1.5rem;
}

.mod-details-content {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow-y: auto;
  max-height: calc(85vh - 80px);
  padding-right: 0.5rem;
}

.details-hero {
  display: flex;
  align-items: flex-start;
  gap: 1.25rem;
  flex-wrap: wrap;
}

.details-icon {
  width: 72px;
  height: 72px;
  border-radius: 12px;
  object-fit: cover;
  background: rgba(0, 0, 0, 0.3);
  flex-shrink: 0;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.details-icon-placeholder {
  width: 72px;
  height: 72px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.06);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2rem;
  opacity: 0.5;
  flex-shrink: 0;
}

.details-main-info {
  flex: 1;
  min-width: 260px;
}

.details-title-row {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.details-title {
  margin: 0;
  font-size: 1.45rem;
  font-weight: 700;
}

.details-author {
  font-size: 0.85rem;
  opacity: 0.7;
}

.details-summary {
  margin: 0.4rem 0 0.75rem 0;
  font-size: 0.92rem;
  line-height: 1.45;
  opacity: 0.85;
}

.details-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  align-items: center;
  margin-bottom: 0.6rem;
}

.detail-chip {
  font-size: 0.75rem;
  background: rgba(255, 255, 255, 0.07);
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  opacity: 0.9;
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
}

.license-chip {
  border: 1px solid rgba(255, 255, 255, 0.12);
}

.details-links {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6rem;
  margin-top: 0.5rem;
}

.ext-link {
  font-size: 0.78rem;
  color: #58a6ff;
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  background: rgba(88, 166, 255, 0.08);
  border: 1px solid rgba(88, 166, 255, 0.2);
  transition: all 0.15s ease;
}

.ext-link:hover {
  background: rgba(88, 166, 255, 0.18);
  border-color: rgba(88, 166, 255, 0.4);
}

.details-actions {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  align-items: stretch;
  min-width: 170px;
}

.installed-status-box {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
  padding: 0.6rem 0.8rem;
  background: rgba(27, 217, 106, 0.08);
  border: 1px solid rgba(27, 217, 106, 0.25);
  border-radius: 6px;
}

.installed-badge-large {
  font-size: 0.85rem;
  font-weight: 600;
  color: #1bd96a;
}

.installed-filename {
  font-size: 0.75rem;
  opacity: 0.75;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 160px;
}

.btn-install-modal {
  width: 100%;
}

.btn-uninstall-modal {
  font-size: 0.8rem;
  padding: 0.3rem 0.6rem;
}

.modal-ver-btn {
  width: 100%;
  text-align: center;
  padding: 0.45rem 0.8rem;
  font-size: 0.85rem;
}

.details-divider {
  border: none;
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  margin: 0.5rem 0;
}

.details-body-wrapper {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.body-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.section-title {
  margin: 0;
  font-size: 1.15rem;
  font-weight: 700;
}

.loaders-summary {
  font-size: 0.8rem;
  opacity: 0.65;
}

.no-desc {
  opacity: 0.6;
  font-style: italic;
}

/* Mod Markdown Body Styling */
.mod-markdown-body {
  line-height: 1.7;
  font-size: 0.92rem;
  color: rgba(255, 255, 255, 0.88);
  overflow-wrap: break-word;
  word-break: break-word;
}

.mod-markdown-body h1,
.mod-markdown-body h2,
.mod-markdown-body h3,
.mod-markdown-body h4 {
  margin-top: 1.4rem;
  margin-bottom: 0.5rem;
  font-weight: 700;
  color: #fff;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  padding-bottom: 0.25rem;
}

.mod-markdown-body h1 { font-size: 1.4rem; }
.mod-markdown-body h2 { font-size: 1.2rem; }
.mod-markdown-body h3 { font-size: 1.05rem; }

.mod-markdown-body p {
  margin-bottom: 0.85rem;
}

.mod-markdown-body img {
  max-width: 100%;
  height: auto;
  border-radius: 6px;
  margin: 0.5rem 0;
}

.mod-markdown-body a {
  color: #58a6ff;
  text-decoration: underline;
}

.mod-markdown-body code {
  background: rgba(255, 255, 255, 0.1);
  padding: 0.15rem 0.4rem;
  border-radius: 4px;
  font-family: monospace;
  font-size: 0.88em;
}

.mod-markdown-body pre {
  background: rgba(0, 0, 0, 0.35);
  padding: 0.75rem 1rem;
  border-radius: 6px;
  overflow-x: auto;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.mod-markdown-body pre code {
  background: transparent;
  padding: 0;
}

.mod-markdown-body ul,
.mod-markdown-body ol {
  margin: 0.5rem 0 0.85rem 1.4rem;
  padding: 0;
}

.mod-markdown-body li {
  margin-bottom: 0.25rem;
}

.mod-markdown-body blockquote {
  border-left: 4px solid var(--primary, #0070f3);
  margin: 0.85rem 0;
  padding: 0.4rem 0.8rem;
  background: rgba(0, 112, 243, 0.08);
  border-radius: 0 4px 4px 0;
}

.mod-markdown-body table {
  width: 100%;
  border-collapse: collapse;
  margin: 0.85rem 0;
}

.mod-markdown-body th,
.mod-markdown-body td {
  border: 1px solid rgba(255, 255, 255, 0.12);
  padding: 0.45rem 0.75rem;
  text-align: left;
}

.mod-markdown-body th {
  background: rgba(255, 255, 255, 0.06);
  font-weight: 600;
}

/* Minecraft Support & Technical Meta Box (Modrinth Style) */
.support-meta-box {
  display: flex;
  gap: 1.25rem;
  background: rgba(0, 0, 0, 0.25);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 8px;
  padding: 1rem;
  flex-wrap: wrap;
}

.support-col {
  flex: 1;
  min-width: 280px;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.support-col.loaders-col {
  flex: 0 0 210px;
  min-width: 180px;
  border-left: 1px solid rgba(255, 255, 255, 0.08);
  padding-left: 1.25rem;
}

@media (max-width: 680px) {
  .support-col.loaders-col {
    border-left: none;
    padding-left: 0;
    border-top: 1px solid rgba(255, 255, 255, 0.08);
    padding-top: 0.75rem;
    flex: 1 1 100%;
  }
}

.support-col-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;
}

.support-title-group {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.support-col-title {
  font-size: 0.82rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  opacity: 0.9;
}

.support-icon {
  color: #1bd96a;
  font-size: 1.1rem;
}

.support-range-badge {
  font-size: 0.78rem;
  font-weight: 700;
  color: #1bd96a;
  background: rgba(27, 217, 106, 0.12);
  border: 1px solid rgba(27, 217, 106, 0.3);
  padding: 0.15rem 0.55rem;
  border-radius: 20px;
}

.server-compat-banner {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.45rem 0.75rem;
  border-radius: 6px;
  font-size: 0.82rem;
  line-height: 1.35;
}

.server-compat-banner.compat-yes {
  background: rgba(27, 217, 106, 0.12);
  border: 1px solid rgba(27, 217, 106, 0.3);
  color: #1bd96a;
}

.server-compat-banner.compat-warn {
  background: rgba(245, 158, 11, 0.12);
  border: 1px solid rgba(245, 158, 11, 0.3);
  color: #f59e0b;
}

.versions-pill-list,
.loaders-pill-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35rem;
  align-items: center;
}

.ver-pill {
  font-size: 0.75rem;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 4px;
  padding: 0.15rem 0.45rem;
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  font-family: monospace;
}

.ver-pill.active-server-ver {
  background: rgba(27, 217, 106, 0.18);
  border-color: #1bd96a;
  color: #1bd96a;
  font-weight: 700;
}

.ver-pill .check,
.loader-pill .check {
  font-size: 0.7rem;
  color: #1bd96a;
  font-weight: bold;
}

.ver-pill-toggle {
  background: none;
  border: 1px dashed rgba(255, 255, 255, 0.25);
  color: #58a6ff;
  border-radius: 4px;
  padding: 0.15rem 0.45rem;
  font-size: 0.72rem;
  cursor: pointer;
  transition: all 0.15s ease;
}

.ver-pill-toggle:hover {
  background: rgba(88, 166, 255, 0.1);
  border-color: #58a6ff;
}

.loader-pill {
  font-size: 0.78rem;
  background: rgba(255, 255, 255, 0.07);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 4px;
  padding: 0.2rem 0.5rem;
  text-transform: capitalize;
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
}

.loader-pill.active-server-loader {
  background: rgba(0, 112, 243, 0.2);
  border-color: var(--primary, #0070f3);
  color: #58a6ff;
  font-weight: 700;
}

/* Card MC version tag */
.mc-version-tag {
  background: rgba(27, 217, 106, 0.1);
  color: #1bd96a;
  border: 1px solid rgba(27, 217, 106, 0.25);
  font-weight: 600;
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
}

.mc-version-tag.mc-compat-tag {
  border-color: #1bd96a;
  background: rgba(27, 217, 106, 0.18);
}

.mc-stat {
  color: #1bd96a !important;
  opacity: 0.9 !important;
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
}

.detail-chip.mc-chip {
  background: rgba(27, 217, 106, 0.12);
  color: #1bd96a;
  border: 1px solid rgba(27, 217, 106, 0.25);
  font-weight: 600;
}

/* Dependencies Box in Details Modal */
.dependencies-box {
  background: rgba(0, 0, 0, 0.35);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 0.85rem 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.dep-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  padding-bottom: 0.5rem;
}

.dep-header-title-group {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.dep-icon {
  color: #38bdf8;
  font-size: 1.1rem;
}

.dep-header-title {
  font-size: 0.85rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  color: #f1f5f9;
}

.dep-header-sub {
  font-size: 0.75rem;
  opacity: 0.65;
}

.dep-list {
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
}

.dep-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0.45rem 0.65rem;
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 6px;
  flex-wrap: wrap;
}

.dep-item-left {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.dep-item-icon {
  width: 24px;
  height: 24px;
  border-radius: 4px;
  object-fit: cover;
}

.dep-item-icon-ph {
  width: 24px;
  height: 24px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.8rem;
  opacity: 0.6;
}

.dep-item-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.dep-item-title {
  font-size: 0.85rem;
  font-weight: 600;
  color: #f1f5f9;
}

.dep-type-badge {
  font-size: 0.68rem;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  text-transform: uppercase;
  font-weight: 700;
  letter-spacing: 0.3px;
}

.dep-type-badge.required {
  background: rgba(239, 68, 68, 0.18);
  color: #f87171;
  border: 1px solid rgba(239, 68, 68, 0.3);
}

.dep-type-badge.optional {
  background: rgba(148, 163, 184, 0.15);
  color: #94a3b8;
  border: 1px solid rgba(148, 163, 184, 0.25);
}

.dep-status-installed {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.78rem;
  color: #1bd96a;
  background: rgba(27, 217, 106, 0.12);
  padding: 0.15rem 0.5rem;
  border-radius: 4px;
  border: 1px solid rgba(27, 217, 106, 0.25);
}

.dep-status-missing {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.78rem;
  color: #f59e0b;
  background: rgba(245, 158, 11, 0.12);
  padding: 0.15rem 0.5rem;
  border-radius: 4px;
  border: 1px solid rgba(245, 158, 11, 0.25);
}

.dep-status-optional {
  font-size: 0.75rem;
  opacity: 0.6;
}
</style>

<style>
/* Ensure native dropdown popups in all desktop & mobile browsers render with dark theme */
.filter-group select {
  color-scheme: dark;
}
.filter-group select option {
  background-color: #202228 !important;
  color: #f1f5f9 !important;
}
.filter-group select optgroup {
  background-color: #15171c !important;
  color: #60a5fa !important;
}
</style>
