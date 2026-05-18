<template>
  <v-container fluid class="om-browser pa-0" style="height:100%;display:flex;flex-direction:column;overflow:hidden">
    <!-- Toolbar -->
    <v-toolbar dense flat color="surface" class="flex-shrink-0">
      <v-toolbar-title class="subtitle-2 primary--text">Object Model Browser</v-toolbar-title>
      <v-divider vertical class="mx-3" />
      <span class="caption mr-2" style="color:#7f849c">Branch/Tag:</span>
      <v-select
        v-model="selectedRef"
        :items="refItems"
        dense
        outlined
        hide-details
        style="max-width:200px"
        :loading="loadingBranches"
        :disabled="loadingBranches || loadingModel"
      />
      <v-btn small class="ml-2" color="primary" :disabled="!selectedRef || loadingModel" @click="doLoadModel">
        Load
      </v-btn>
      <v-spacer />
      <v-text-field
        v-model="searchTerm"
        dense
        outlined
        hide-details
        clearable
        placeholder="Search properties..."
        prepend-inner-icon="mdi-magnify"
        style="max-width:260px"
        :disabled="!modelLoaded"
      />
      <v-btn icon small :disabled="!modelLoaded" @click="expandAll" title="Expand All">
        <v-icon small>mdi-chevron-down-box-outline</v-icon>
      </v-btn>
      <v-btn icon small :disabled="!modelLoaded" @click="collapseAll" title="Collapse All">
        <v-icon small>mdi-chevron-up-box-outline</v-icon>
      </v-btn>
    </v-toolbar>

    <!-- DSF indicator bar -->
    <div class="dsf-bar px-4 caption" style="display:flex;align-items:center;gap:8px;height:24px;flex-shrink:0;background:#252535;border-bottom:1px solid #3d3d5c">
      <span :class="['dsf-dot', dsfState]" />
      <span style="color:#7f849c">{{ dsfLabel }}</span>
      <v-spacer />
      <span v-if="loadingModel" style="color:#7f849c">{{ loadingMsg }}</span>
    </div>

    <!-- Main split pane -->
    <div style="display:flex;flex:1;overflow:hidden">
      <!-- Tree panel -->
      <div style="width:360px;min-width:180px;flex-shrink:0;border-right:1px solid #3d3d5c;display:flex;flex-direction:column;overflow:hidden">
        <div ref="treeScroll" style="flex:1;overflow-y:auto;padding:8px 4px">
          <div v-if="!modelLoaded && !loadingModel" class="placeholder">
            Select a branch and click Load
          </div>
          <div v-else-if="loadingModel" class="placeholder">
            Loading…
          </div>
          <template v-else-if="searchTerm && searchTerm.trim()">
            <om-search-results
              :matches="searchMatches"
              :search-term="searchTerm"
              @select-class="selectClass"
            />
          </template>
          <template v-else>
            <om-tree-node
              v-if="rootClass"
              :cls="rootClass"
              :model="omModel"
              :depth="0"
              :prop-name="'objectModel'"
              :selected-class="selectedClassName"
              :open-nodes="openNodes"
              @select="selectClass"
              @toggle="toggleNode"
            />
          </template>
        </div>
      </div>

      <!-- Resizer -->
      <div
        class="resizer"
        @mousedown="startResize"
      />

      <!-- Detail panel -->
      <div ref="detailPanel" style="flex:1;overflow-y:auto;padding:20px 24px">
        <div v-if="!selectedClassName" class="placeholder" style="margin-top:60px;text-align:center">
          Select an item in the tree to view details.<br>
          <span style="font-size:12px;color:#7f849c">Use the branch selector above to switch RRF releases.</span>
        </div>
        <template v-else-if="detailType === 'class' && detailClass">
          <!-- Breadcrumb -->
          <div v-if="navStack.length" class="detail-breadcrumb mb-2">
            <span
              v-for="(entry, i) in navStack"
              :key="i"
              class="bc-item"
              @click="navigateBreadcrumb(i)"
            >{{ entry.key }}</span>
            <span class="bc-sep">›</span>
            <span class="bc-current">{{ detailClass.name }}</span>
          </div>

          <!-- Header -->
          <div style="display:flex;align-items:baseline;gap:10px;flex-wrap:wrap;margin-bottom:6px">
            <h2 class="title primary--text">{{ detailClass.name }}</h2>
          </div>

          <!-- OM path chips -->
          <div v-if="currentPaths.length" style="display:flex;flex-wrap:wrap;gap:6px;margin-bottom:12px">
            <span v-for="p in currentPaths" :key="p" style="display:inline-flex;align-items:center;gap:2px">
              <code class="om-path-code">{{ p }}</code>
              <v-btn icon x-small @click="copyPath(p)" title="Copy path">
                <v-icon x-small>mdi-content-copy</v-icon>
              </v-btn>
            </span>
          </div>

          <!-- Class description -->
          <div v-if="currentClassDesc" class="class-description mb-4">
            {{ currentClassDesc.summary }}
            <div v-if="currentClassDesc.remarks" class="remarks">{{ currentClassDesc.remarks }}</div>
          </div>

          <!-- Properties table -->
          <div v-if="!detailClass.props || !detailClass.props.length" style="color:#7f849c;font-size:13px;margin-top:16px">
            No properties found.
          </div>
          <v-simple-table v-else dense class="prop-table">
            <template #default>
              <thead>
                <tr>
                  <th>Property</th>
                  <th>Type</th>
                  <th>Default</th>
                  <th v-if="hasLiveModel">Value</th>
                  <th v-if="hasAnyDesc">Description</th>
                  <th style="width:32px" />
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="p in detailClass.props"
                  :key="p.name"
                  :class="{ 'prop-row-drilldown': drillTarget(p) }"
                  @click="drillTarget(p) && navigateDetail('class', drillTarget(p))"
                >
                  <td>
                    <span :class="['prop-name', { 'is-readonly': p.readonly }]">{{ p.name }}</span>
                    <span v-if="drillTarget(p)" class="prop-drilldown-chevron">›</span>
                    <span v-if="propDesc(p) && propDesc(p).sbcProperty === false" class="tag tag-sbc-only" title="SBC only">SBC only</span>
                    <span v-else-if="propDesc(p) && propDesc(p).sbcProperty === true" class="tag tag-sbc-also" title="SBC managed">SBC</span>
                  </td>
                  <td>
                    <span
                      :class="['prop-type', { 'prop-link': typeLink(p) }]"
                      @click.stop="typeLink(p) && navigateDetail(typeLink(p).kind, typeLink(p).name)"
                    >{{ typeDisplay(p) }}</span>
                    <span v-if="p.nullable" style="color:#7f849c;font-size:11px"> or null</span>
                    <div v-if="inlineEnum(p)" class="enum-inline">
                      <span
                        v-for="m in inlineEnum(p).members"
                        :key="m.key"
                        class="enum-pip"
                        :title="enumMemberDesc(inlineEnum(p).name, m.key)"
                      >{{ m.key }}</span>
                    </div>
                  </td>
                  <td><span class="prop-default">{{ p.default || '' }}</span></td>
                  <td v-if="hasLiveModel">
                    <span :class="['live-value', liveValueClass(liveValueAt(propPath(p)))]">
                      {{ formatLiveValue(liveValueAt(propPath(p))) }}
                    </span>
                  </td>
                  <td v-if="hasAnyDesc" class="prop-desc-cell">
                    <template v-if="propDesc(p)">
                      {{ propDesc(p).summary }}
                      <div v-if="propDesc(p).remarks" class="remarks">{{ propDesc(p).remarks }}</div>
                    </template>
                    <span v-else style="color:#7f849c">—</span>
                  </td>
                  <td style="text-align:center">
                    <v-btn icon x-small @click.stop="copyPath(propPath(p))" :title="'Copy: ' + propPath(p)">
                      <v-icon x-small>mdi-content-copy</v-icon>
                    </v-btn>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>

        <template v-else-if="detailType === 'enum' && detailEnum">
          <!-- Enum detail -->
          <div v-if="navStack.length" class="detail-breadcrumb mb-2">
            <span v-for="(entry, i) in navStack" :key="i" class="bc-item" @click="navigateBreadcrumb(i)">{{ entry.key }}</span>
            <span class="bc-sep">›</span>
            <span class="bc-current">{{ detailEnum.name }}</span>
          </div>
          <h2 class="title primary--text mb-2">{{ detailEnum.name }}</h2>
          <div v-if="currentClassDesc" class="class-description mb-4">{{ currentClassDesc.summary }}</div>
          <v-simple-table dense class="prop-table">
            <template #default>
              <thead>
                <tr>
                  <th>Value</th>
                  <th v-if="hasEnumMemberDescs">Description</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="m in detailEnum.members" :key="m.key">
                  <td>
                    <span style="font-family:monospace;color:#a6e3a1">{{ m.key }}</span>
                    <span v-if="m.value !== undefined" class="prop-default" style="margin-left:8px">= {{ m.value }}</span>
                  </td>
                  <td v-if="hasEnumMemberDescs" class="prop-desc-cell">
                    <template v-if="enumMemberDesc(detailEnum.name, m.key)">
                      {{ enumMemberDesc(detailEnum.name, m.key) }}
                    </template>
                    <span v-else style="color:#7f849c">—</span>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>
      </div>
    </div>

    <!-- Copy notification snackbar -->
    <v-snackbar v-model="copiedSnackbar" timeout="1500" bottom right color="success" :elevation="2">
      <v-icon small class="mr-1">mdi-check</v-icon>
      Copied to clipboard
    </v-snackbar>
  </v-container>
</template>

<script>
// ── GitHub constants ────────────────────────────────────────
const OM_REPO   = 'Duet3D/ObjectModel'
const DSF_REPO  = 'Duet3D/DuetSoftwareFramework'
const OM_RAW    = 'https://raw.githubusercontent.com/' + OM_REPO
const DSF_RAW   = 'https://raw.githubusercontent.com/' + DSF_REPO
const OM_API    = 'https://api.github.com/repos/' + OM_REPO
const DSF_API   = 'https://api.github.com/repos/' + DSF_REPO
const DSF_OM_PATH = 'src/DuetAPI/ObjectModel'
const CACHE_VERSION = 'om_v2'

// ── Utility functions (standalone, no Vue dependency) ──────

function pascalToCamel(s) {
  if (!s) return s
  return s.charAt(0).toLowerCase() + s.slice(1)
}

function extractXmlTag(text, tag) {
  const re = new RegExp('<' + tag + '>([\\s\\S]*?)<\\/' + tag + '>', 'i')
  const m = text.match(re)
  if (!m) return null
  return m[1].replace(/\s+/g, ' ').trim()
}

function findEnclosingType(lines, fromLine) {
  let depth = 0
  for (let i = fromLine - 1; i >= 0; i--) {
    const t = lines[i].trim()
    for (let c = t.length - 1; c >= 0; c--) {
      if (t[c] === '}') depth++
      else if (t[c] === '{') {
        if (depth > 0) { depth-- }
        else {
          for (let k = i; k >= Math.max(0, i - 4); k--) {
            const decl = lines[k].trim()
            const cm = decl.match(/(?:public|internal|private)\s+(?:(?:partial|abstract|sealed|static)\s+)*class\s+(\w+)/)
            if (cm) return cm[1]
            const em = decl.match(/(?:public|internal|private)\s+enum\s+(\w+)/)
            if (em) return em[1]
          }
          return null
        }
      }
    }
  }
  return null
}

function parseDSFFile(src) {
  const result = {}
  const lines = src.split('\n')
  let i = 0
  while (i < lines.length) {
    const trimmed = lines[i].trim()
    if (trimmed.startsWith('///')) {
      const docParts = []
      while (i < lines.length && lines[i].trim().startsWith('///')) {
        docParts.push(lines[i].trim().replace(/^\/\/\/\s?/, ''))
        i++
      }
      const docBlock = docParts.join(' ')
      const summary = extractXmlTag(docBlock, 'summary')
      const remarks = extractXmlTag(docBlock, 'remarks')
      if (!summary) continue

      let j = i
      let sbcProperty = null
      while (j < lines.length) {
        const t = lines[j].trim()
        if (t === '') { j++; continue }
        if (t.startsWith('[')) {
          const sbcM = t.match(/^\[SbcProperty\(\s*(true|false)\s*\)\]/)
          if (sbcM) sbcProperty = sbcM[1] === 'true'
          j++; continue
        }
        break
      }
      if (j >= lines.length) continue

      const declBlock = lines.slice(j, j + 3).map(l => l.trim()).join(' ')

      const classMatch = declBlock.match(/^public\s+(?:(?:partial|abstract|sealed|static)\s+)*class\s+(\w+)/)
      if (classMatch) {
        const name = classMatch[1]
        if (!result[name]) result[name] = {}
        result[name].__class__ = { summary, remarks }
        continue
      }

      const enumMatch = declBlock.match(/^public\s+enum\s+(\w+)/)
      if (enumMatch) {
        const name = enumMatch[1]
        if (!result[name]) result[name] = {}
        result[name].__class__ = { summary, remarks }
        continue
      }

      const propMatch = declBlock.match(/^public\s+(?:(?:static|virtual|override|new|readonly)\s+)*[\w?<>[\],\s]+?\s+(\w+)\s*(?:\{|=>)/)
      if (propMatch) {
        const propName = propMatch[1]
        if (propName === 'class' || propName === 'enum') { continue }
        const enclosing = findEnclosingType(lines, j)
        if (enclosing) {
          if (!result[enclosing]) result[enclosing] = {}
          const entry = { summary, remarks }
          if (sbcProperty !== null) entry.sbcProperty = sbcProperty
          result[enclosing][pascalToCamel(propName)] = entry
        }
        continue
      }

      const enumMemberMatch = declBlock.match(/^(\w+)\s*(?:=\s*-?\d+)?\s*,?(?:\s|$)/)
      if (enumMemberMatch) {
        const memberName = enumMemberMatch[1]
        if (/^(public|private|protected|internal|static|readonly|namespace|using|return|void|class|enum|if|for|new)$/.test(memberName)) {
          continue
        }
        const enclosing = findEnclosingType(lines, j)
        if (enclosing) {
          if (!result[enclosing]) result[enclosing] = {}
          result[enclosing][pascalToCamel(memberName)] = { summary }
          result[enclosing][memberName] = { summary }
        }
      }
      continue
    }
    i++
  }
  return result
}

function stripComments(src) {
  return src
    .replace(/\/\*[\s\S]*?\*\//g, m => m.replace(/[^\n]/g, ''))
    .replace(/\/\/[^\n]*/g, '')
}

function parseOMFile(src, filePath) {
  const clean = stripComments(src)
  const classes = {}
  const enums = {}

  const enumRe = /export\s+(?:const\s+)?enum\s+(\w+)\s*\{([^}]*)\}/g
  let m
  while ((m = enumRe.exec(clean)) !== null) {
    const name = m[1]
    const body = m[2]
    const members = []
    const memberRe = /(\w+)\s*(?:=\s*([^,\n}]+))?/g
    let mm
    while ((mm = memberRe.exec(body)) !== null) {
      if (!mm[1]) continue
      members.push({ key: mm[1], value: mm[2] ? mm[2].trim() : undefined })
    }
    if (members.length > 0) enums[name] = { name, members, file: filePath }
  }

  const classRe = /export\s+class\s+(\w+)(?:\s+extends\s+(\w+))?\s*\{/g
  while ((m = classRe.exec(clean)) !== null) {
    const name = m[1]
    const parent = m[2] || null
    const startIdx = m.index + m[0].length
    let depth = 1, ci = startIdx
    while (ci < clean.length && depth > 0) {
      if (clean[ci] === '{') depth++
      else if (clean[ci] === '}') depth--
      ci++
    }
    const body = clean.slice(startIdx, ci - 1)
    const props = parseClassBody(body)
    if (props.length > 0 || name !== 'ModelObject') {
      classes[name] = { name, parent, props, file: filePath }
    }
  }

  return { classes, enums }
}

function parseClassBody(body) {
  const props = []
  const ctorRe = /constructor\s*\([^)]*\)\s*\{/g
  let cleaned = body
  let m
  while ((m = ctorRe.exec(cleaned)) !== null) {
    const start = m.index
    let depth = 1, ci = start + m[0].length
    while (ci < cleaned.length && depth > 0) {
      if (cleaned[ci] === '{') depth++
      else if (cleaned[ci] === '}') depth--
      ci++
    }
    cleaned = cleaned.slice(0, start) + cleaned.slice(ci)
    ctorRe.lastIndex = start
  }
  const propRe = /^\s*(readonly\s+)?(\w+)\s*:\s*([^=;\n]+?)\s*(?:=\s*([^;\n]+?))?\s*;/gm
  while ((m = propRe.exec(cleaned)) !== null) {
    const readonly = !!m[1]
    const name = m[2]
    if (['constructor', 'super', 'return', 'this', 'static'].includes(name)) continue
    let typeStr = m[3].trim().replace(/\s+/g, ' ')
    const defaultVal = m[4] ? m[4].trim() : undefined
    const nullable = typeStr.includes('| null') || typeStr.includes('null |')
    const cleanType = typeStr.replace(/\s*\|\s*null/g, '').replace(/null\s*\|\s*/g, '').trim()
    props.push({ name, type: cleanType, nullable, readonly, default: defaultVal })
  }
  return props
}

function resolveCollectionType(typeStr) {
  let m = typeStr.match(/ModelCollection<([^>]+)>/)
  if (m) return m[1].replace(/\s*\|\s*null/g, '').trim()
  m = typeStr.match(/ModelDictionary<([^>]+)>/)
  if (m) {
    const parts = m[1].split(',')
    return parts[parts.length - 1].replace(/\s*\|\s*null/g, '').trim()
  }
  m = typeStr.match(/Array<([^>]+)>/)
  if (m) return m[1].trim()
  return null
}

function isCollectionType(t) {
  return /ModelCollection</.test(t) || /Array</.test(t) || t.endsWith('[]')
}

function isDictType(t) {
  return /ModelDictionary</.test(t) || /Map</.test(t)
}

function shortType(t) {
  return t.replace('ModelCollection', 'Collection').replace('ModelDictionary', 'Dict').replace('ModelSet', 'Set')
}

async function fetchJSON(url) {
  const r = await fetch(url, { headers: { Accept: 'application/vnd.github.v3+json' } })
  if (!r.ok) throw new Error(`HTTP ${r.status}: ${url}`)
  return r.json()
}

async function fetchText(url) {
  const r = await fetch(url)
  if (!r.ok) throw new Error(`HTTP ${r.status}: ${url}`)
  return r.text()
}

function cacheKey(omSha, dsfSha) {
  return `${CACHE_VERSION}_${omSha}_${dsfSha || 'none'}`
}

function cacheSave(omSha, dsfSha, dsfRef, modelData, descsData, dsfLabel) {
  const prefix = CACHE_VERSION + '_'
  const toRemove = []
  for (let i = 0; i < localStorage.length; i++) {
    const k = localStorage.key(i)
    if (k && k.startsWith(prefix)) toRemove.push(k)
  }
  toRemove.forEach(k => localStorage.removeItem(k))
  const key = cacheKey(omSha, dsfSha)
  try {
    localStorage.setItem(key, JSON.stringify({ omSha, dsfSha, dsfRef, model: modelData, descriptions: descsData, dsfLabel }))
  } catch (e) {
    console.warn('[om-browser] cache save failed:', e.message)
  }
}

function cacheLoad(omSha, dsfSha) {
  const key = cacheKey(omSha, dsfSha)
  try {
    const raw = localStorage.getItem(key)
    if (!raw) return null
    return JSON.parse(raw)
  } catch (e) { return null }
}

// ── Inline tree node component ─────────────────────────────
const OmTreeNode = {
  name: 'OmTreeNode',
  props: {
    cls: Object,
    model: Object,
    depth: Number,
    propName: String,
    selectedClass: String,
    openNodes: Object
  },
  data() {
    return { populated: false }
  },
  computed: {
    nodeKey() { return this.cls.name + ':' + this.propName },
    isOpen() { return !!this.openNodes[this.nodeKey] },
    childProps() {
      if (!this.populated) return []
      return (this.cls.props || []).filter(p => {
        const inner = resolveCollectionType(p.type)
        const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
        return this.model.classes[t]
      })
    },
    leafProps() {
      if (!this.populated) return []
      return (this.cls.props || []).filter(p => {
        const inner = resolveCollectionType(p.type)
        const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
        return !this.model.classes[t]
      })
    }
  },
  template: `
    <div class="tree-node">
      <div
        :class="['tree-row', { selected: selectedClass === cls.name }]"
        @click="selectThis"
      >
        <span class="tree-indent" :style="{ width: (depth * 16) + 'px' }" />
        <span class="tree-toggle" @click.stop="toggle">{{ isOpen ? '▼' : '▶' }}</span>
        <span class="tree-name">{{ propName || cls.name }}</span>
        <span v-if="cls.name !== propName" class="tree-badge badge-class">{{ cls.name }}</span>
      </div>
      <div v-if="isOpen" class="tree-children">
        <om-tree-node
          v-for="p in childProps"
          :key="p.name"
          :cls="childClass(p)"
          :model="model"
          :depth="depth + 1"
          :prop-name="p.name"
          :selected-class="selectedClass"
          :open-nodes="openNodes"
          @select="forwardSelect"
          @toggle="forwardToggle"
        />
        <div v-for="p in leafProps" :key="p.name" class="tree-row" @click="$emit('select', { type: 'class', name: cls.name })">
          <span class="tree-indent" :style="{ width: ((depth + 1) * 16) + 'px' }" />
          <span class="tree-toggle" />
          <span class="tree-name">{{ p.name }}</span>
          <span class="dim">{{ shortType(p.type) }}</span>
        </div>
        <div v-if="!childProps.length && !leafProps.length" style="padding:4px 8px;font-size:12px;color:#7f849c">(no properties)</div>
      </div>
    </div>
  `,
  methods: {
    toggle() {
      if (!this.populated) { this.populated = true }
      this.$emit('toggle', this.nodeKey)
    },
    selectThis() {
      this.$emit('select', { type: 'class', name: this.cls.name })
    },
    forwardSelect(e) { this.$emit('select', e) },
    forwardToggle(e) { this.$emit('toggle', e) },
    childClass(p) {
      const inner = resolveCollectionType(p.type)
      const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
      return this.model.classes[t]
    },
    shortType
  }
}

const OmSearchResults = {
  name: 'OmSearchResults',
  props: { matches: Array, searchTerm: String },
  methods: {
    hl(text) {
      if (!this.searchTerm) return text
      const lc = this.searchTerm.toLowerCase()
      const idx = text.toLowerCase().indexOf(lc)
      if (idx === -1) return text
      return text.slice(0, idx) + '<mark>' + text.slice(idx, idx + lc.length) + '</mark>' + text.slice(idx + lc.length)
    }
  },
  template: `
    <div>
      <div v-if="!matches.length" style="color:#7f849c;font-size:13px;padding:8px">No results</div>
      <div v-for="g in matches" :key="g.clsName">
        <div class="tree-row" style="font-weight:600" @click="$emit('select-class', g.clsName)">
          <span class="tree-name" v-html="hl(g.clsName)" />
          <span class="tree-badge badge-class">{{ g.props.length }}</span>
        </div>
        <div v-for="p in g.props" :key="p.name" class="tree-row" @click="$emit('select-class', g.clsName)">
          <span class="tree-indent" style="width:24px" />
          <span class="tree-name" v-html="hl(p.name)" />
          <span class="dim">{{ p.type }}</span>
        </div>
      </div>
    </div>
  `
}

export default {
  name: 'OmBrowser',

  components: { OmTreeNode, OmSearchResults },

  data() {
    return {
      // Ref selector
      selectedRef: 'v3.6-dev',
      refItems: [],
      loadingBranches: false,

      // Model
      omModel: { classes: {}, enums: {} },
      descriptions: {},
      loadingModel: false,
      loadingMsg: '',

      // DSF indicator
      dsfState: 'none',
      dsfLabel: 'DSF descriptions: —',

      // Tree state
      openNodes: {},
      searchTerm: '',

      // Detail panel
      selectedClassName: null,
      detailType: null,
      navStack: [],

      // UI
      copiedSnackbar: false,

      // Resize
      resizing: false,
      resizeStartX: 0,
      resizeStartW: 0
    }
  },

  computed: {
    rootClass() {
      return this.omModel.classes['ObjectModel'] || null
    },

    modelLoaded() {
      return !!this.rootClass
    },

    liveModel() {
      try {
        return this.$store.state.machine.model
      } catch (e) {
        return null
      }
    },

    hasLiveModel() {
      return !!this.liveModel
    },

    detailClass() {
      if (this.detailType !== 'class' || !this.selectedClassName) return null
      return this.omModel.classes[this.selectedClassName] || null
    },

    detailEnum() {
      if (this.detailType !== 'enum' || !this.selectedClassName) return null
      return this.omModel.enums[this.selectedClassName] || null
    },

    currentClassDesc() {
      if (!this.selectedClassName) return null
      return this.descriptions[this.selectedClassName]?.__class__ || null
    },

    currentPaths() {
      if (!this.detailClass) return []
      return this.findPaths(this.detailClass.name)
    },

    hasAnyDesc() {
      if (!this.detailClass) return false
      return (this.detailClass.props || []).some(p => this.propDesc(p))
    },

    hasEnumMemberDescs() {
      if (!this.detailEnum) return false
      return this.detailEnum.members.some(m => this.enumMemberDesc(this.detailEnum.name, m.key))
    },

    searchMatches() {
      if (!this.searchTerm || !this.searchTerm.trim()) return []
      const lc = this.searchTerm.toLowerCase()
      const byClass = {}
      for (const cls of Object.values(this.omModel.classes)) {
        for (const prop of cls.props || []) {
          const desc = this.getPropDesc(cls.name, prop.name)
          if (
            prop.name.toLowerCase().includes(lc) ||
            cls.name.toLowerCase().includes(lc) ||
            prop.type.toLowerCase().includes(lc) ||
            desc?.summary?.toLowerCase().includes(lc)
          ) {
            if (!byClass[cls.name]) byClass[cls.name] = { clsName: cls.name, props: [] }
            byClass[cls.name].props.push(prop)
          }
        }
      }
      return Object.values(byClass)
    }
  },

  async mounted() {
    window.addEventListener('mousemove', this.onMouseMove)
    window.addEventListener('mouseup', this.onMouseUp)
    await this.loadBranches()
    if (this.selectedRef) {
      await this.doLoadModel()
    }
  },

  beforeDestroy() {
    window.removeEventListener('mousemove', this.onMouseMove)
    window.removeEventListener('mouseup', this.onMouseUp)
  },

  methods: {
    // ── GitHub loading ─────────────────────────────────────

    async loadBranches() {
      this.loadingBranches = true
      try {
        const [branches, tags] = await Promise.all([
          fetchJSON(`${OM_API}/branches?per_page=100`),
          fetchJSON(`${OM_API}/tags?per_page=100`)
        ])
        const items = []
        if (branches.length) {
          items.push({ header: 'Branches' })
          branches.forEach(b => items.push({ text: b.name, value: b.name }))
        }
        if (tags.length) {
          items.push({ header: 'Tags' })
          tags.sort((a, b) => b.name.localeCompare(a.name, undefined, { numeric: true }))
          tags.forEach(t => items.push({ text: t.name, value: t.name }))
        }
        this.refItems = items
      } catch (e) {
        console.error('[om-browser] loadBranches failed:', e)
      }
      this.loadingBranches = false
    },

    async doLoadModel() {
      if (!this.selectedRef) return
      this.loadingModel = true
      this.loadingMsg = 'Loading…'
      this.omModel = { classes: {}, enums: {} }
      this.descriptions = {}
      this.selectedClassName = null
      this.navStack = []
      this.openNodes = {}
      this.setDSFIndicator('none', 'DSF descriptions: searching…')

      try {
        const ref = this.selectedRef

        // Resolve SHAs for cache
        let omSha = null, dsfRef = null, dsfSha = null
        this.loadingMsg = 'Checking for updates…'
        try {
          const data = await fetchJSON(`https://api.github.com/repos/${OM_REPO}/commits/${encodeURIComponent(ref)}?per_page=1`)
          omSha = data.sha
        } catch (e) { console.warn('[om-browser] OM SHA resolve failed:', e.message) }

        try {
          dsfRef = await this.findDSFRef(ref)
          if (dsfRef) {
            const data = await fetchJSON(`https://api.github.com/repos/${DSF_REPO}/commits/${encodeURIComponent(dsfRef)}?per_page=1`)
            dsfSha = data.sha
          }
        } catch (e) { console.warn('[om-browser] DSF SHA resolve failed:', e.message) }

        // Try cache
        if (omSha) {
          const cached = cacheLoad(omSha, dsfSha)
          if (cached) {
            this.loadingMsg = 'Loading from cache…'
            await this.$nextTick()
            this.omModel = cached.model
            this.descriptions = cached.descriptions || {}
            for (const cls of Object.values(this.omModel.classes)) {
              if (cls.parent && cls.parent !== 'ModelObject' && this.omModel.classes[cls.parent])
                cls.parentRef = this.omModel.classes[cls.parent]
            }
            if (cached.dsfLabel) this.setDSFIndicator('ok', cached.dsfLabel)
            else this.setDSFIndicator('none', 'DSF descriptions: no matching branch')
            this.loadingModel = false
            return
          }
        }

        // Fetch OM TypeScript files
        this.loadingMsg = 'Fetching OM file list…'
        const omTree = await fetchJSON(`${OM_API}/git/trees/${encodeURIComponent(ref)}?recursive=1`)
        const omFiles = omTree.tree.filter(f => f.type === 'blob' && f.path.startsWith('src/') && f.path.endsWith('.ts')).map(f => f.path)
        const total = omFiles.length
        let done = 0

        const omResults = []
        for (let i = 0; i < omFiles.length; i += 10) {
          this.loadingMsg = `Fetching OM source… (${done}/${total})`
          const batch = omFiles.slice(i, i + 10)
          const texts = await Promise.all(
            batch.map(f => fetchText(`${OM_RAW}/${encodeURIComponent(ref)}/${f}`).then(t => ({ path: f, text: t })))
          )
          omResults.push(...texts)
          done += batch.length
        }

        this.loadingMsg = 'Parsing TypeScript…'
        await this.$nextTick()
        const model = { classes: {}, enums: {} }
        for (const { path, text } of omResults) {
          try {
            const parsed = parseOMFile(text, path)
            Object.assign(model.classes, parsed.classes)
            Object.assign(model.enums, parsed.enums)
          } catch (e) { console.warn('[om-browser] OM parse error:', path, e) }
        }
        for (const cls of Object.values(model.classes)) {
          if (cls.parent && cls.parent !== 'ModelObject' && model.classes[cls.parent])
            cls.parentRef = model.classes[cls.parent]
        }

        // Fetch DSF descriptions
        let dsfLabel = null
        if (dsfRef) {
          try {
            this.loadingMsg = `Loading DSF descriptions (${dsfRef})…`
            await this.$nextTick()
            const dsfTree = await fetchJSON(`${DSF_API}/git/trees/${encodeURIComponent(dsfRef)}?recursive=1`)
            const dsfFiles = dsfTree.tree
              .filter(f => f.type === 'blob' && f.path.startsWith(DSF_OM_PATH + '/') && f.path.endsWith('.cs'))
              .map(f => f.path)

            const dsfTotal = dsfFiles.length
            let dsfDone = 0
            const dsfResults = []
            for (let i = 0; i < dsfFiles.length; i += 10) {
              this.loadingMsg = `Fetching DSF source… (${dsfDone}/${dsfTotal})`
              const batch = dsfFiles.slice(i, i + 10)
              const texts = await Promise.all(
                batch.map(f => fetchText(`${DSF_RAW}/${encodeURIComponent(dsfRef)}/${f}`).then(t => ({ path: f, text: t })))
              )
              dsfResults.push(...texts)
              dsfDone += batch.length
            }

            this.loadingMsg = 'Parsing DSF descriptions…'
            await this.$nextTick()
            const descriptions = {}
            for (const { path, text } of dsfResults) {
              try {
                const parsed = parseDSFFile(text)
                for (const [name, props] of Object.entries(parsed)) {
                  if (!descriptions[name]) descriptions[name] = {}
                  Object.assign(descriptions[name], props)
                }
              } catch (e) { console.warn('[om-browser] DSF parse error:', path, e) }
            }
            this.descriptions = descriptions

            const descCount = Object.keys(descriptions).length
            dsfLabel = `DSF descriptions: ${dsfRef} (${descCount} types)`
            this.setDSFIndicator('ok', dsfLabel)
          } catch (e) {
            console.warn('[om-browser] DSF load failed:', e)
            this.setDSFIndicator('error', 'DSF descriptions: failed')
          }
        } else {
          this.setDSFIndicator('none', 'DSF descriptions: no matching branch')
        }

        this.omModel = model

        // Cache (strip circular parentRef)
        if (omSha) {
          const modelForCache = {
            classes: Object.fromEntries(Object.entries(model.classes).map(([k, v]) => {
              const { parentRef, ...rest } = v
              return [k, rest]
            })),
            enums: model.enums
          }
          cacheSave(omSha, dsfSha, dsfRef, modelForCache, this.descriptions, dsfLabel)
        }

      } catch (e) {
        console.error('[om-browser] loadModel failed:', e)
        this.setDSFIndicator('error', 'Load failed: ' + e.message)
      }

      this.loadingModel = false
    },

    async findDSFRef(omRef) {
      const [branches, tags] = await Promise.all([
        fetchJSON(`${DSF_API}/branches?per_page=100`),
        fetchJSON(`${DSF_API}/tags?per_page=100`)
      ])
      const branchNames = branches.map(b => b.name)
      const tagNames = tags.map(t => t.name)
      const all = [...branchNames, ...tagNames]

      if (all.includes(omRef)) return omRef

      const verMatch = omRef.match(/v?(\d+\.\d+)/)
      if (!verMatch) return null
      const minorVer = verMatch[1]

      const devBranch = `v${minorVer}-dev`
      if (branchNames.includes(devBranch)) return devBranch

      const matchingTags = tagNames
        .filter(t => t.startsWith(`v${minorVer}.`) || t === `v${minorVer}`)
        .sort((a, b) => b.localeCompare(a, undefined, { numeric: true }))
      if (matchingTags.length) return matchingTags[0]

      return null
    },

    setDSFIndicator(state, label) {
      this.dsfState = state
      this.dsfLabel = label
    },

    // ── Tree ───────────────────────────────────────────────

    toggleNode(nodeKey) {
      this.$set(this.openNodes, nodeKey, !this.openNodes[nodeKey])
    },

    expandAll() {
      const newOpen = {}
      const walk = (cls, propName, depth) => {
        const key = cls.name + ':' + propName
        newOpen[key] = true
        for (const p of cls.props || []) {
          const inner = resolveCollectionType(p.type)
          const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
          const child = this.omModel.classes[t]
          if (child && depth < 6) walk(child, p.name, depth + 1)
        }
      }
      if (this.rootClass) walk(this.rootClass, 'objectModel', 0)
      this.openNodes = newOpen
    },

    collapseAll() {
      this.openNodes = {}
    },

    selectClass(e) {
      const name = typeof e === 'string' ? e : e.name
      this.navStack = []
      if (this.omModel.classes[name]) {
        this.selectedClassName = name
        this.detailType = 'class'
      } else if (this.omModel.enums[name]) {
        this.selectedClassName = name
        this.detailType = 'enum'
      }
    },

    // ── Detail navigation ─────────────────────────────────

    navigateDetail(type, name) {
      if (this.selectedClassName) {
        this.navStack = [...this.navStack, { type: this.detailType, key: this.selectedClassName }]
      }
      this.selectedClassName = name
      this.detailType = type
    },

    navigateBreadcrumb(idx) {
      const entry = this.navStack[idx]
      if (!entry) return
      this.navStack = this.navStack.slice(0, idx)
      this.selectedClassName = entry.key
      this.detailType = entry.type
    },

    // ── Helpers ────────────────────────────────────────────

    getClassDesc(className) {
      return this.descriptions[className]?.__class__ || null
    },

    getPropDesc(className, propName) {
      const d = this.descriptions[className]
      if (!d) return null
      return d[propName] || d[propName.charAt(0).toUpperCase() + propName.slice(1)] || null
    },

    propDesc(p) {
      if (!this.detailClass) return null
      return this.getPropDesc(this.detailClass.name, p.name)
    },

    enumMemberDesc(enumName, memberName) {
      const d = this.descriptions[enumName]
      if (!d) return null
      const entry = d[memberName] || d[pascalToCamel(memberName)]
      return entry?.summary || null
    },

    drillTarget(p) {
      const inner = resolveCollectionType(p.type)
      const isCol = isCollectionType(p.type)
      const isDct = isDictType(p.type)
      const isCls = !!this.omModel.classes[p.type]
      if (isCol && inner && this.omModel.classes[inner]) return inner
      if (isDct && inner && this.omModel.classes[inner]) return inner
      if (isCls) return p.type
      return null
    },

    typeDisplay(p) {
      const inner = resolveCollectionType(p.type)
      const isCol = isCollectionType(p.type)
      const isDct = isDictType(p.type)
      if (isCol && inner) return inner + '[]'
      if (isDct && inner) return inner + '{}'
      return shortType(p.type)
    },

    typeLink(p) {
      const inner = resolveCollectionType(p.type)
      const isCol = isCollectionType(p.type)
      const isDct = isDictType(p.type)
      if (isCol && inner && (this.omModel.classes[inner] || this.omModel.enums[inner])) {
        return { name: inner, kind: this.omModel.classes[inner] ? 'class' : 'enum' }
      }
      if (isDct && inner && this.omModel.classes[inner]) return { name: inner, kind: 'class' }
      if (this.omModel.classes[p.type]) return { name: p.type, kind: 'class' }
      if (this.omModel.enums[p.type]) return { name: p.type, kind: 'enum' }
      return null
    },

    inlineEnum(p) {
      return this.omModel.enums[p.type] || null
    },

    findPaths(targetClassName) {
      const root = this.omModel.classes['ObjectModel']
      if (!root || !this.omModel.classes[targetClassName]) return []

      // Build subclass map
      const direct = {}
      for (const cls of Object.values(this.omModel.classes)) {
        if (cls.parent) {
          if (!direct[cls.parent]) direct[cls.parent] = new Set()
          direct[cls.parent].add(cls.name)
        }
      }
      const subMap = {}
      const getAll = (name) => {
        if (subMap[name]) return subMap[name]
        subMap[name] = new Set()
        for (const child of direct[name] || []) {
          subMap[name].add(child)
          for (const grand of getAll(child)) subMap[name].add(grand)
        }
        return subMap[name]
      }
      for (const name of Object.keys(direct)) getAll(name)

      const results = new Set()
      const visited = new Set()
      const walk = (cls, pathSoFar) => {
        const key = cls.name + '|' + pathSoFar
        if (visited.has(key)) return
        visited.add(key)
        if (cls.name === targetClassName && pathSoFar) {
          results.add(pathSoFar)
          return
        }
        for (const prop of cls.props || []) {
          const inner = resolveCollectionType(prop.type)
          const isCol = isCollectionType(prop.type)
          const isDct = isDictType(prop.type)
          const resolvedType = (inner || prop.type).replace(/\s*\|\s*null/g, '').trim()
          if (!this.omModel.classes[resolvedType]) continue
          const suffix = isCol ? '[]' : isDct ? '{}' : ''
          const segment = pathSoFar ? pathSoFar + '.' + prop.name + suffix : prop.name + suffix
          const toWalk = new Set([resolvedType])
          for (const sub of subMap[resolvedType] || []) toWalk.add(sub)
          let ancestor = this.omModel.classes[resolvedType]
          while (ancestor) {
            for (const sub of subMap[ancestor.name] || []) toWalk.add(sub)
            ancestor = ancestor.parent ? this.omModel.classes[ancestor.parent] : null
          }
          for (const name of toWalk) {
            const c = this.omModel.classes[name]
            if (c) walk(c, segment)
          }
        }
      }
      walk(root, '')
      return [...results].sort()
    },

    // ── Live value helpers ─────────────────────────────────

    propPath(p) {
      if (!this.detailClass) return p.name
      const paths = this.findPaths(this.detailClass.name)
      const basePath = paths.length > 0 ? paths[0] : (this.detailClass.name.charAt(0).toLowerCase() + this.detailClass.name.slice(1))
      const isColOrDict = isCollectionType(p.type) || isDictType(p.type)
      return (basePath ? basePath + (isColOrDict ? '[0].' : '.') : '') + p.name
    },

    liveValueAt(path) {
      if (!this.liveModel || !path) return undefined
      let cur = this.liveModel
      for (const seg of path.replace(/\[(\d+)\]/g, '.$1').split('.')) {
        if (cur == null || typeof cur !== 'object') return undefined
        cur = cur[seg]
      }
      return cur
    },

    formatLiveValue(val) {
      if (val === undefined) return '—'
      if (val === null) return 'null'
      if (typeof val === 'boolean') return String(val)
      if (typeof val === 'number') return String(val)
      if (typeof val === 'string') return '"' + val + '"'
      if (Array.isArray(val)) return '[' + val.length + ']'
      return '{…}'
    },

    liveValueClass(val) {
      if (val === null || val === undefined) return 'is-null'
      if (val === true) return 'is-bool-true'
      if (val === false) return 'is-bool-false'
      return ''
    },

    // ── Clipboard ─────────────────────────────────────────

    copyPath(path) {
      if (navigator.clipboard) {
        navigator.clipboard.writeText(path).then(() => {
          this.copiedSnackbar = true
        }).catch(() => {})
      }
    },

    // ── Resize ────────────────────────────────────────────

    startResize(e) {
      this.resizing = true
      this.resizeStartX = e.clientX
      const treeEl = this.$el.querySelector('[style*="width:360px"]') ||
                     this.$el.querySelector('.tree-panel')
      this.resizeStartW = treeEl ? treeEl.offsetWidth : 360
      this._resizeEl = treeEl
      document.body.style.cursor = 'col-resize'
      document.body.style.userSelect = 'none'
    },

    onMouseMove(e) {
      if (!this.resizing) return
      const newW = Math.max(180, Math.min(700, this.resizeStartW + e.clientX - this.resizeStartX))
      if (this._resizeEl) this._resizeEl.style.width = newW + 'px'
    },

    onMouseUp() {
      if (!this.resizing) return
      this.resizing = false
      document.body.style.cursor = ''
      document.body.style.userSelect = ''
    }
  }
}
</script>

<style scoped>
.om-browser {
  font-family: 'Segoe UI', system-ui, sans-serif;
  background: #1e1e2e;
  color: #cdd6f4;
}

.placeholder {
  color: #7f849c;
  font-size: 14px;
  padding: 20px;
  line-height: 1.8;
}

.dsf-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: #7f849c;
  display: inline-block;
  flex-shrink: 0;
}
.dsf-dot.ok { background: #a6e3a1; }
.dsf-dot.error { background: #f38ba8; }
.dsf-dot.none { background: #7f849c; }

.tree-node { user-select: none; }

.tree-row {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 3px 4px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 13px;
  line-height: 1.4;
}
.tree-row:hover { background: #2e2e45; }
.tree-row.selected { background: rgba(137,180,250,0.15); }

.tree-indent { display: inline-block; flex-shrink: 0; }
.tree-toggle { width: 16px; flex-shrink: 0; font-size: 10px; color: #7f849c; text-align: center; cursor: pointer; }
.tree-name { font-weight: 500; color: #cdd6f4; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tree-badge {
  font-size: 10px;
  border-radius: 3px;
  padding: 1px 5px;
  margin-left: 4px;
  flex-shrink: 0;
}
.badge-class { background: rgba(137,180,250,0.2); color: #89b4fa; }
.badge-collection { background: rgba(148,226,213,0.2); color: #94e2d5; }
.badge-enum { background: rgba(166,227,161,0.2); color: #a6e3a1; }
.tree-children { padding-left: 16px; }
.dim { color: #7f849c; font-size: 12px; margin-left: auto; padding-left: 8px; flex-shrink: 0; }

.resizer {
  width: 5px;
  background: transparent;
  cursor: col-resize;
  flex-shrink: 0;
  transition: background 0.15s;
}
.resizer:hover { background: #89b4fa; }

.detail-breadcrumb {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 2px;
  font-size: 12px;
  color: #7f849c;
}
.bc-item { cursor: pointer; color: #89b4fa; text-decoration: underline dotted; }
.bc-item:hover { color: #f5c2e7; }
.bc-sep { color: #3d3d5c; margin: 0 2px; }
.bc-current { color: #7f849c; cursor: default; }

.om-path-code {
  font-family: monospace;
  font-size: 13px;
  color: #f9e2af;
  background: rgba(249,226,175,0.1);
  border: 1px solid rgba(249,226,175,0.2);
  border-radius: 4px;
  padding: 2px 8px;
  user-select: all;
}

.class-description {
  font-size: 13px;
  color: #cdd6f4;
  line-height: 1.6;
  background: #2e2e45;
  border-left: 3px solid #89b4fa;
  padding: 10px 14px;
  border-radius: 0 6px 6px 0;
}
.class-description .remarks {
  margin-top: 6px;
  font-size: 12px;
  color: #7f849c;
  font-style: italic;
}

.prop-table { width: 100%; }
.prop-name { font-weight: 500; color: #cdd6f4; font-family: monospace; }
.prop-name.is-readonly { color: #cba6f7; }
.prop-type { color: #94e2d5; font-family: monospace; }
.prop-link { cursor: pointer; text-decoration: underline dotted; }
.prop-link:hover { color: #f5c2e7; }
.prop-default { color: #f9e2af; font-family: monospace; font-size: 12px; }
.prop-desc-cell { font-size: 12px; color: #7f849c; line-height: 1.5; }
.prop-desc-cell .remarks { margin-top: 2px; font-style: italic; opacity: 0.8; }

tr.prop-row-drilldown { cursor: pointer; }
tr.prop-row-drilldown:hover td { background: rgba(137,180,250,0.07); }
.prop-drilldown-chevron { color: #89b4fa; font-size: 13px; margin-left: 6px; opacity: 0.7; }

.tag {
  font-size: 11px;
  border-radius: 4px;
  padding: 1px 6px;
  font-weight: 500;
  margin-left: 4px;
}
.tag-sbc-only { background: rgba(250,179,135,0.2); color: #fab387; }
.tag-sbc-also { background: rgba(148,226,213,0.15); color: #94e2d5; }

.enum-inline { display: flex; flex-wrap: wrap; gap: 3px; margin-top: 5px; }
.enum-pip {
  font-family: monospace;
  font-size: 11px;
  color: #a6e3a1;
  background: rgba(166,227,161,0.1);
  border: 1px solid rgba(166,227,161,0.25);
  border-radius: 3px;
  padding: 1px 6px;
  cursor: default;
}

.live-value { font-family: monospace; font-size: 12px; color: #f9e2af; }
.live-value.is-null { color: #7f849c; }
.live-value.is-bool-true { color: #a6e3a1; }
.live-value.is-bool-false { color: #f38ba8; }
</style>
