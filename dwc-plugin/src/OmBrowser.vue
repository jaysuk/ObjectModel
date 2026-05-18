<template>
  <v-container fluid class="om-browser pa-0" style="height:100%;display:flex;flex-direction:column;overflow:hidden">
    <!-- Toolbar -->
    <v-toolbar dense flat color="surface" class="flex-shrink-0">
      <v-toolbar-title class="subtitle-2 primary--text">Object Model Browser</v-toolbar-title>
      <span class="caption ml-2 grey--text">{{ modelRef }}</span>
      <v-spacer />
      <v-text-field
        v-model="searchTerm"
        dense outlined hide-details clearable
        placeholder="Search properties..."
        prepend-inner-icon="mdi-magnify"
        style="max-width:260px"
      />
      <v-btn icon small title="Expand All" @click="expandAll"><v-icon small>mdi-chevron-down-box-outline</v-icon></v-btn>
      <v-btn icon small title="Collapse All" @click="collapseAll"><v-icon small>mdi-chevron-up-box-outline</v-icon></v-btn>
    </v-toolbar>

    <!-- DSF indicator bar -->
    <div style="display:flex;align-items:center;gap:8px;height:24px;flex-shrink:0;padding:0 16px;background:#1e1e2e;border-bottom:1px solid #313244">
      <span :style="{ width:'7px', height:'7px', borderRadius:'50%', background: dsfState==='ok' ? '#a6e3a1' : '#7f849c', display:'inline-block', flexShrink:0 }" />
      <span class="caption grey--text">{{ dsfLabel }}</span>
    </div>

    <!-- Main split pane -->
    <div style="display:flex;flex:1;overflow:hidden">

      <!-- Tree panel -->
      <div ref="treePanel" style="width:360px;min-width:160px;flex-shrink:0;border-right:1px solid #313244;overflow-y:auto">
        <!-- Search results -->
        <template v-if="searchTerm && searchTerm.trim()">
          <div v-if="!searchMatches.length" class="pa-3 grey--text caption">No results</div>
          <template v-else>
            <div
              v-for="g in searchMatches" :key="g.clsName"
              class="tree-row font-weight-bold"
              @click="selectClass(g.clsName)"
            >
              <span class="tree-name">{{ g.clsName }}</span>
              <span class="tree-badge">{{ g.props.length }}</span>
            </div>
          </template>
        </template>

        <!-- Flat tree rows -->
        <template v-else>
          <div
            v-for="row in treeRows"
            :key="row.key"
            :class="['tree-row', { 'tree-row--selected': selectedClassName === row.cls.name }]"
            :style="{ paddingLeft: (8 + row.depth * 16) + 'px' }"
            @click="selectClass(row.cls.name)"
          >
            <span
              class="tree-toggle"
              @click.stop="toggleNode(row.key)"
            >{{ openNodes[row.key] ? '▼' : '▶' }}</span>
            <span class="tree-name">{{ row.propName }}</span>
            <span v-if="row.cls.name !== row.propName" class="tree-type">{{ row.cls.name }}</span>
          </div>
        </template>
      </div>

      <!-- Resizer -->
      <div class="resizer" @mousedown="startResize" />

      <!-- Detail panel -->
      <div ref="detailPanel" style="flex:1;overflow-y:auto;padding:20px 24px">
        <div v-if="!selectedClassName" class="grey--text text-center" style="margin-top:60px;font-size:14px;line-height:2">
          Select an item in the tree to view details.
        </div>

        <template v-else-if="detailType === 'class' && detailClass">
          <!-- Breadcrumb -->
          <div v-if="navStack.length" class="mb-2" style="display:flex;align-items:center;flex-wrap:wrap;gap:4px;font-size:12px">
            <span v-for="(entry, i) in navStack" :key="i">
              <a class="primary--text" style="cursor:pointer;text-decoration:underline dotted" @click="navigateBreadcrumb(i)">{{ entry.key }}</a>
              <span class="grey--text mx-1">›</span>
            </span>
            <span class="grey--text">{{ detailClass.name }}</span>
          </div>

          <!-- Header -->
          <div style="display:flex;align-items:baseline;gap:10px;flex-wrap:wrap;margin-bottom:6px">
            <h2 class="title primary--text">{{ detailClass.name }}</h2>
          </div>

          <!-- OM path chips -->
          <div v-if="currentPaths.length" style="display:flex;flex-wrap:wrap;gap:6px;margin-bottom:12px">
            <span v-for="p in currentPaths" :key="p" style="display:inline-flex;align-items:center;gap:2px">
              <code class="om-path-code">{{ p }}</code>
              <v-btn icon x-small @click="copyPath(p)" title="Copy path"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
            </span>
          </div>

          <!-- Class description -->
          <div v-if="currentClassDesc" class="class-desc mb-4">
            {{ currentClassDesc.summary }}
            <div v-if="currentClassDesc.remarks" class="grey--text mt-1" style="font-size:12px;font-style:italic">{{ currentClassDesc.remarks }}</div>
          </div>

          <!-- Properties table -->
          <div v-if="!detailClass.props || !detailClass.props.length" class="grey--text caption mt-4">No properties found.</div>
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
                  :class="{ 'row-drilldown': !!drillTarget(p) }"
                  @click="drillTarget(p) && navigateDetail('class', drillTarget(p))"
                >
                  <td>
                    <span :class="['prop-name', { 'prop-name--readonly': p.readonly }]">{{ p.name }}</span>
                    <span v-if="drillTarget(p)" class="primary--text ml-1">›</span>
                    <span v-if="propDesc(p) && propDesc(p).sbcProperty === false" class="tag tag-sbc-only ml-1">SBC only</span>
                    <span v-else-if="propDesc(p) && propDesc(p).sbcProperty === true" class="tag tag-sbc ml-1">SBC</span>
                  </td>
                  <td>
                    <span :class="['prop-type', { 'prop-type--link': typeLink(p) }]" @click.stop="typeLink(p) && navigateDetail(typeLink(p).kind, typeLink(p).name)">{{ typeDisplay(p) }}</span>
                    <span v-if="p.nullable" class="grey--text" style="font-size:11px"> or null</span>
                    <div v-if="inlineEnum(p)" style="display:flex;flex-wrap:wrap;gap:3px;margin-top:4px">
                      <span
                        v-for="m in inlineEnum(p).members" :key="m.key"
                        class="enum-pip"
                        :title="enumMemberDesc(inlineEnum(p).name, m.key)"
                      >{{ m.key }}</span>
                    </div>
                  </td>
                  <td><code class="prop-default">{{ p.default || '' }}</code></td>
                  <td v-if="hasLiveModel">
                    <span :class="['live-val', liveValClass(liveValueAt(propPath(p)))]">{{ fmtLive(liveValueAt(propPath(p))) }}</span>
                  </td>
                  <td v-if="hasAnyDesc" class="desc-cell">
                    <template v-if="propDesc(p)">
                      {{ propDesc(p).summary }}
                      <div v-if="propDesc(p).remarks" class="grey--text mt-1" style="font-size:11px;font-style:italic">{{ propDesc(p).remarks }}</div>
                    </template>
                    <span v-else class="grey--text">—</span>
                  </td>
                  <td style="text-align:center">
                    <v-btn icon x-small :title="'Copy: ' + propPath(p)" @click.stop="copyPath(propPath(p))"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>

        <template v-else-if="detailType === 'enum' && detailEnum">
          <div v-if="navStack.length" class="mb-2" style="display:flex;align-items:center;flex-wrap:wrap;gap:4px;font-size:12px">
            <span v-for="(entry, i) in navStack" :key="i">
              <a class="primary--text" style="cursor:pointer;text-decoration:underline dotted" @click="navigateBreadcrumb(i)">{{ entry.key }}</a>
              <span class="grey--text mx-1">›</span>
            </span>
            <span class="grey--text">{{ detailEnum.name }}</span>
          </div>
          <h2 class="title primary--text mb-2">{{ detailEnum.name }}</h2>
          <div v-if="currentClassDesc" class="class-desc mb-4">{{ currentClassDesc.summary }}</div>
          <v-simple-table dense class="prop-table">
            <template #default>
              <thead><tr><th>Value</th><th v-if="hasEnumMemberDescs">Description</th></tr></thead>
              <tbody>
                <tr v-for="m in detailEnum.members" :key="m.key">
                  <td>
                    <code style="color:#a6e3a1">{{ m.key }}</code>
                    <code v-if="m.value !== undefined" class="prop-default ml-2">= {{ m.value }}</code>
                  </td>
                  <td v-if="hasEnumMemberDescs" class="desc-cell">
                    <span v-if="enumMemberDesc(detailEnum.name, m.key)">{{ enumMemberDesc(detailEnum.name, m.key) }}</span>
                    <span v-else class="grey--text">—</span>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>
      </div>
    </div>

    <v-snackbar v-model="copiedSnackbar" timeout="1500" bottom right color="success" :elevation="2">
      <v-icon small class="mr-1">mdi-check</v-icon> Copied to clipboard
    </v-snackbar>
  </v-container>
</template>

<script>
import { omModel as BUNDLED_MODEL, omDescriptions as BUNDLED_DESCRIPTIONS, MODEL_REF, DSF_REF_LABEL } from './model-data.js'
import store from '@/store'

function resolveCollectionType (typeStr) {
  let m = typeStr.match(/ModelCollection<([^>]+)>/)
  if (m) return m[1].replace(/\s*\|\s*null/g, '').trim()
  m = typeStr.match(/ModelDictionary<([^>]+)>/)
  if (m) { const p = m[1].split(','); return p[p.length - 1].replace(/\s*\|\s*null/g, '').trim() }
  m = typeStr.match(/Array<([^>]+)>/)
  if (m) return m[1].trim()
  return null
}
function isCollectionType (t) { return /ModelCollection</.test(t) || /Array</.test(t) || t.endsWith('[]') }
function isDictType (t) { return /ModelDictionary</.test(t) || /Map</.test(t) }
function shortType (t) { return t.replace('ModelCollection', 'Collection').replace('ModelDictionary', 'Dict').replace('ModelSet', 'Set') }
function pascalToCamel (s) { return s ? s.charAt(0).toLowerCase() + s.slice(1) : s }

export default {
  name: 'OmBrowser',

  data () {
    return {
      omModel: BUNDLED_MODEL,
      descriptions: BUNDLED_DESCRIPTIONS,
      modelRef: MODEL_REF,
      dsfState: 'ok',
      dsfLabel: `DSF descriptions: ${DSF_REF_LABEL} (${Object.keys(BUNDLED_DESCRIPTIONS).length} types)`,

      openNodes: {},
      searchTerm: '',

      selectedClassName: null,
      detailType: null,
      navStack: [],

      copiedSnackbar: false,
      resizing: false,
      resizeStartX: 0,
      resizeStartW: 0
    }
  },

  computed: {
    liveModel () {
      try { return store.state.machine.model } catch (e) { return null }
    },
    hasLiveModel () { return !!this.liveModel },

    rootClass () { return this.omModel.classes['ObjectModel'] || null },

    // Build a flat list of tree rows from the open/closed state
    treeRows () {
      const rows = []
      if (!this.rootClass) return rows
      const walk = (cls, propName, depth) => {
        const key = cls.name + ':' + propName
        rows.push({ key, cls, propName, depth })
        if (!this.openNodes[key]) return
        for (const p of cls.props || []) {
          const inner = resolveCollectionType(p.type)
          const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
          const child = this.omModel.classes[t]
          if (child) walk(child, p.name, depth + 1)
        }
      }
      walk(this.rootClass, 'objectModel', 0)
      return rows
    },

    detailClass () {
      if (this.detailType !== 'class' || !this.selectedClassName) return null
      return this.omModel.classes[this.selectedClassName] || null
    },
    detailEnum () {
      if (this.detailType !== 'enum' || !this.selectedClassName) return null
      return this.omModel.enums[this.selectedClassName] || null
    },
    currentClassDesc () {
      if (!this.selectedClassName) return null
      return (this.descriptions[this.selectedClassName] || {}).__class__ || null
    },
    currentPaths () {
      return this.detailClass ? this.findPaths(this.detailClass.name) : []
    },
    hasAnyDesc () {
      return this.detailClass ? (this.detailClass.props || []).some(p => this.propDesc(p)) : false
    },
    hasEnumMemberDescs () {
      return this.detailEnum ? this.detailEnum.members.some(m => this.enumMemberDesc(this.detailEnum.name, m.key)) : false
    },
    searchMatches () {
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
            (desc && desc.summary && desc.summary.toLowerCase().includes(lc))
          ) {
            if (!byClass[cls.name]) byClass[cls.name] = { clsName: cls.name, props: [] }
            byClass[cls.name].props.push(prop)
          }
        }
      }
      return Object.values(byClass)
    }
  },

  mounted () {
    window.addEventListener('mousemove', this.onMouseMove)
    window.addEventListener('mouseup', this.onMouseUp)
  },

  beforeDestroy () {
    window.removeEventListener('mousemove', this.onMouseMove)
    window.removeEventListener('mouseup', this.onMouseUp)
  },

  methods: {
    toggleNode (key) {
      this.$set(this.openNodes, key, !this.openNodes[key])
    },

    expandAll () {
      const newOpen = {}
      const walk = (cls, propName, depth) => {
        if (depth > 5) return
        const key = cls.name + ':' + propName
        newOpen[key] = true
        for (const p of cls.props || []) {
          const inner = resolveCollectionType(p.type)
          const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
          const child = this.omModel.classes[t]
          if (child) walk(child, p.name, depth + 1)
        }
      }
      if (this.rootClass) walk(this.rootClass, 'objectModel', 0)
      this.openNodes = newOpen
    },

    collapseAll () { this.openNodes = {} },

    selectClass (name) {
      if (typeof name === 'object') name = name.name
      this.navStack = []
      if (this.omModel.classes[name]) {
        this.selectedClassName = name
        this.detailType = 'class'
      } else if (this.omModel.enums[name]) {
        this.selectedClassName = name
        this.detailType = 'enum'
      }
    },

    navigateDetail (type, name) {
      if (this.selectedClassName) {
        this.navStack = [...this.navStack, { type: this.detailType, key: this.selectedClassName }]
      }
      this.selectedClassName = name
      this.detailType = type
    },

    navigateBreadcrumb (idx) {
      const entry = this.navStack[idx]
      if (!entry) return
      this.navStack = this.navStack.slice(0, idx)
      this.selectedClassName = entry.key
      this.detailType = entry.type
    },

    getPropDesc (className, propName) {
      const d = this.descriptions[className]
      if (!d) return null
      return d[propName] || d[propName.charAt(0).toUpperCase() + propName.slice(1)] || null
    },
    propDesc (p) {
      if (!this.detailClass) return null
      return this.getPropDesc(this.detailClass.name, p.name)
    },
    enumMemberDesc (enumName, memberName) {
      const d = this.descriptions[enumName]
      if (!d) return null
      const e = d[memberName] || d[pascalToCamel(memberName)]
      return e ? e.summary || null : null
    },

    drillTarget (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner && this.omModel.classes[inner]) return inner
      if (isDictType(p.type) && inner && this.omModel.classes[inner]) return inner
      if (this.omModel.classes[p.type]) return p.type
      return null
    },
    typeDisplay (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner) return inner + '[]'
      if (isDictType(p.type) && inner) return inner + '{}'
      return shortType(p.type)
    },
    typeLink (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner && (this.omModel.classes[inner] || this.omModel.enums[inner])) {
        return { name: inner, kind: this.omModel.classes[inner] ? 'class' : 'enum' }
      }
      if (isDictType(p.type) && inner && this.omModel.classes[inner]) return { name: inner, kind: 'class' }
      if (this.omModel.classes[p.type]) return { name: p.type, kind: 'class' }
      if (this.omModel.enums[p.type]) return { name: p.type, kind: 'enum' }
      return null
    },
    inlineEnum (p) { return this.omModel.enums[p.type] || null },

    findPaths (targetClassName) {
      const root = this.omModel.classes['ObjectModel']
      if (!root || !this.omModel.classes[targetClassName]) return []
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
        for (const child of (direct[name] || [])) {
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
        if (cls.name === targetClassName && pathSoFar) { results.add(pathSoFar); return }
        for (const prop of cls.props || []) {
          const inner = resolveCollectionType(prop.type)
          const isCol = isCollectionType(prop.type)
          const isDct = isDictType(prop.type)
          const resolvedType = (inner || prop.type).replace(/\s*\|\s*null/g, '').trim()
          if (!this.omModel.classes[resolvedType]) continue
          const suffix = isCol ? '[]' : isDct ? '{}' : ''
          const segment = pathSoFar ? pathSoFar + '.' + prop.name + suffix : prop.name + suffix
          const toWalk = new Set([resolvedType])
          for (const sub of (subMap[resolvedType] || [])) toWalk.add(sub)
          for (const name of toWalk) { const c = this.omModel.classes[name]; if (c) walk(c, segment) }
        }
      }
      walk(root, '')
      return [...results].sort()
    },

    propPath (p) {
      if (!this.detailClass) return p.name
      const paths = this.findPaths(this.detailClass.name)
      const base = paths.length > 0 ? paths[0] : pascalToCamel(this.detailClass.name)
      const isColOrDict = isCollectionType(p.type) || isDictType(p.type)
      return (base ? base + (isColOrDict ? '[0].' : '.') : '') + p.name
    },

    liveValueAt (path) {
      if (!this.liveModel || !path) return undefined
      let cur = this.liveModel
      for (const seg of path.replace(/\[(\d+)\]/g, '.$1').split('.')) {
        if (cur == null || typeof cur !== 'object') return undefined
        cur = cur[seg]
      }
      return cur
    },
    fmtLive (val) {
      if (val === undefined) return '—'
      if (val === null) return 'null'
      if (typeof val === 'boolean') return String(val)
      if (typeof val === 'number') return String(val)
      if (typeof val === 'string') return '"' + val + '"'
      if (Array.isArray(val)) return '[' + val.length + ']'
      return '{…}'
    },
    liveValClass (val) {
      if (val === null || val === undefined) return 'live-val--null'
      if (val === true) return 'live-val--true'
      if (val === false) return 'live-val--false'
      return ''
    },

    copyPath (path) {
      if (navigator.clipboard) {
        navigator.clipboard.writeText(path).then(() => { this.copiedSnackbar = true }).catch(() => {})
      }
    },

    startResize (e) {
      this.resizing = true
      this.resizeStartX = e.clientX
      this.resizeStartW = this.$refs.treePanel ? this.$refs.treePanel.offsetWidth : 360
      document.body.style.cursor = 'col-resize'
      document.body.style.userSelect = 'none'
    },
    onMouseMove (e) {
      if (!this.resizing || !this.$refs.treePanel) return
      this.$refs.treePanel.style.width = Math.max(160, Math.min(700, this.resizeStartW + e.clientX - this.resizeStartX)) + 'px'
    },
    onMouseUp () {
      if (!this.resizing) return
      this.resizing = false
      document.body.style.cursor = ''
      document.body.style.userSelect = ''
    }
  }
}
</script>

<style scoped>
.om-browser { font-family: 'Segoe UI', system-ui, sans-serif; }

.tree-row {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 3px 8px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 13px;
  user-select: none;
}
.tree-row:hover { background: rgba(255,255,255,0.06); }
.tree-row--selected { background: rgba(137,180,250,0.15); }

.tree-toggle { width: 14px; flex-shrink: 0; font-size: 9px; color: #7f849c; text-align: center; }
.tree-name { font-weight: 500; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tree-type { font-size: 11px; color: #7f849c; margin-left: auto; padding-left: 8px; flex-shrink: 0; }
.tree-badge { font-size: 11px; background: rgba(137,180,250,0.2); color: #89b4fa; border-radius: 3px; padding: 1px 5px; margin-left: auto; }

.resizer { width: 5px; background: transparent; cursor: col-resize; flex-shrink: 0; }
.resizer:hover { background: rgba(137,180,250,0.4); }

.om-path-code {
  font-family: monospace;
  font-size: 13px;
  color: #f9e2af;
  background: rgba(249,226,175,0.08);
  border: 1px solid rgba(249,226,175,0.2);
  border-radius: 4px;
  padding: 2px 8px;
}

.class-desc {
  font-size: 13px;
  line-height: 1.6;
  background: rgba(137,180,250,0.06);
  border-left: 3px solid #89b4fa;
  padding: 8px 12px;
  border-radius: 0 4px 4px 0;
}

.prop-table { width: 100%; }
.prop-name { font-family: monospace; font-weight: 500; }
.prop-name--readonly { color: #cba6f7; }
.prop-type { font-family: monospace; color: #94e2d5; }
.prop-type--link { cursor: pointer; text-decoration: underline dotted; }
.prop-type--link:hover { color: #f5c2e7; }
.prop-default { font-family: monospace; font-size: 12px; color: #f9e2af; background: none; }
.desc-cell { font-size: 12px; line-height: 1.5; }

.row-drilldown { cursor: pointer; }
.row-drilldown:hover td { background: rgba(137,180,250,0.05); }

.tag { font-size: 10px; border-radius: 3px; padding: 1px 5px; font-weight: 500; }
.tag-sbc-only { background: rgba(250,179,135,0.2); color: #fab387; }
.tag-sbc { background: rgba(148,226,213,0.15); color: #94e2d5; }

.enum-pip {
  font-family: monospace; font-size: 11px; color: #a6e3a1;
  background: rgba(166,227,161,0.1); border: 1px solid rgba(166,227,161,0.25);
  border-radius: 3px; padding: 1px 5px; cursor: default;
}

.live-val { font-family: monospace; font-size: 12px; }
.live-val--null { color: #7f849c; }
.live-val--true { color: #a6e3a1; }
.live-val--false { color: #f38ba8; }
</style>
