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
      <v-btn icon small title="Refresh" @click="refresh"><v-icon small>mdi-refresh</v-icon></v-btn>
    </v-toolbar>

    <!-- Status bar -->
    <div style="display:flex;align-items:center;gap:8px;height:24px;flex-shrink:0;padding:0 16px;background:#1e1e2e;border-bottom:1px solid #313244">
      <span :style="{ width:'7px', height:'7px', borderRadius:'50%', background: hasLiveModel ? '#a6e3a1' : '#7f849c', display:'inline-block', flexShrink:0 }" />
      <span class="caption grey--text">{{ hasLiveModel ? 'Live — ' + liveKeyCount + ' keys' : 'Reference only (no printer connected)' }}</span>
      <span class="caption grey--text ml-4">{{ dsfLabel }}</span>
    </div>

    <!-- Main split pane -->
    <div style="display:flex;flex:1;overflow:hidden">

      <!-- Tree panel -->
      <div ref="treePanel" style="width:340px;min-width:160px;flex-shrink:0;border-right:1px solid #313244;overflow-y:auto">
        <!-- Search results -->
        <template v-if="searchTerm && searchTerm.trim()">
          <div v-if="!searchMatches.length" class="pa-3 grey--text caption">No results</div>
          <div
            v-for="g in searchMatches" :key="g.clsName"
            :class="['tree-row', { 'tree-row--selected': selectedNode === g.clsName }]"
            style="padding-left:8px"
            @click="selectSearchResult(g.clsName)"
          >
            <span class="tree-toggle" />
            <span class="tree-name">{{ g.clsName }}</span>
            <span class="tree-type">{{ g.props.length }} props</span>
          </div>
        </template>

        <!-- Flat tree rows -->
        <template v-else>
          <div
            v-for="row in treeRows"
            :key="row.id"
            :class="['tree-row', { 'tree-row--selected': selectedNode === row.id }]"
            :style="{ paddingLeft: (8 + row.depth * 16) + 'px' }"
            @click="selectRow(row)"
          >
            <span class="tree-toggle" @click.stop="toggleNode(row.id)">{{ row.hasChildren ? (openNodes[row.id] ? '▼' : '▶') : '' }}</span>
            <span class="tree-name">{{ row.label }}</span>
            <span class="tree-type">{{ row.typeName }}</span>
          </div>
        </template>
      </div>

      <!-- Resizer -->
      <div class="resizer" @mousedown="startResize" />

      <!-- Detail panel -->
      <div ref="detailPanel" style="flex:1;overflow-y:auto;padding:20px 24px">
        <div v-if="!selectedNode" class="grey--text text-center" style="margin-top:60px;font-size:14px;line-height:2">
          Select an item in the tree to view details.
        </div>

        <!-- Live object/array detail -->
        <template v-else-if="detailMode === 'live'">
          <div v-if="navStack.length" class="mb-2" style="display:flex;align-items:center;flex-wrap:wrap;gap:4px;font-size:12px">
            <span v-for="(entry, i) in navStack" :key="i">
              <a class="primary--text" style="cursor:pointer;text-decoration:underline dotted" @click="navBack(i)">{{ entry.label }}</a>
              <span class="grey--text mx-1">›</span>
            </span>
            <span class="grey--text">{{ detailLabel }}</span>
          </div>

          <div style="display:flex;align-items:baseline;gap:10px;margin-bottom:6px;flex-wrap:wrap">
            <h2 class="title primary--text">{{ detailLabel }}</h2>
            <code class="om-path-code">{{ selectedNode }}</code>
            <v-btn icon x-small @click="copyPath(selectedNode)" title="Copy path"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
          </div>

          <!-- Class description from bundled docs -->
          <div v-if="detailClassDesc" class="class-desc mb-4">
            {{ detailClassDesc.summary }}
            <div v-if="detailClassDesc.remarks" class="grey--text mt-1" style="font-size:12px;font-style:italic">{{ detailClassDesc.remarks }}</div>
          </div>

          <v-simple-table v-if="detailRows.length" dense class="prop-table">
            <template #default>
              <thead>
                <tr>
                  <th>Property</th>
                  <th>Value</th>
                  <th>Type</th>
                  <th v-if="hasAnyLiveDesc">Description</th>
                  <th style="width:32px" />
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="row in detailRows"
                  :key="row.key"
                  :class="{ 'row-drilldown': row.drillable }"
                  @click="row.drillable && drillInto(row)"
                >
                  <td>
                    <span class="prop-name">{{ row.key }}</span>
                    <span v-if="row.drillable" class="primary--text ml-1">›</span>
                    <span v-if="row.desc && row.desc.sbcProperty === false" class="tag tag-sbc-only ml-1">SBC only</span>
                    <span v-else-if="row.desc && row.desc.sbcProperty === true" class="tag tag-sbc ml-1">SBC</span>
                  </td>
                  <td>
                    <span :class="['live-val', liveValClass(row.value)]">{{ fmtLive(row.value) }}</span>
                  </td>
                  <td>
                    <span class="prop-type">{{ row.typeName }}</span>
                    <span v-if="row.nullable" class="grey--text" style="font-size:11px"> or null</span>
                    <div v-if="row.enumMembers" style="display:flex;flex-wrap:wrap;gap:3px;margin-top:4px">
                      <span v-for="m in row.enumMembers" :key="m" class="enum-pip">{{ m }}</span>
                    </div>
                  </td>
                  <td v-if="hasAnyLiveDesc" class="desc-cell">
                    <template v-if="row.desc">
                      {{ row.desc.summary }}
                      <div v-if="row.desc.remarks" class="grey--text mt-1" style="font-size:11px;font-style:italic">{{ row.desc.remarks }}</div>
                    </template>
                    <span v-else class="grey--text">—</span>
                  </td>
                  <td style="text-align:center">
                    <v-btn icon x-small :title="'Copy: ' + row.path" @click.stop="copyPath(row.path)"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
          <div v-else class="grey--text caption mt-4">No properties.</div>
        </template>

        <!-- Reference class detail (no live model) -->
        <template v-else-if="detailMode === 'ref' && refClass">
          <div v-if="navStack.length" class="mb-2" style="display:flex;align-items:center;flex-wrap:wrap;gap:4px;font-size:12px">
            <span v-for="(entry, i) in navStack" :key="i">
              <a class="primary--text" style="cursor:pointer;text-decoration:underline dotted" @click="navBack(i)">{{ entry.label }}</a>
              <span class="grey--text mx-1">›</span>
            </span>
            <span class="grey--text">{{ refClass.name }}</span>
          </div>
          <div style="display:flex;align-items:baseline;gap:10px;margin-bottom:6px;flex-wrap:wrap">
            <h2 class="title primary--text">{{ refClass.name }}</h2>
          </div>
          <div v-if="currentPaths.length" style="display:flex;flex-wrap:wrap;gap:6px;margin-bottom:12px">
            <span v-for="p in currentPaths" :key="p" style="display:inline-flex;align-items:center;gap:2px">
              <code class="om-path-code">{{ p }}</code>
              <v-btn icon x-small @click="copyPath(p)" title="Copy path"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
            </span>
          </div>
          <div v-if="refClassDesc" class="class-desc mb-4">
            {{ refClassDesc.summary }}
            <div v-if="refClassDesc.remarks" class="grey--text mt-1" style="font-size:12px;font-style:italic">{{ refClassDesc.remarks }}</div>
          </div>
          <div v-if="!refClass.props || !refClass.props.length" class="grey--text caption mt-4">No properties.</div>
          <v-simple-table v-else dense class="prop-table">
            <template #default>
              <thead>
                <tr>
                  <th>Property</th>
                  <th>Type</th>
                  <th>Default</th>
                  <th v-if="hasAnyRefDesc">Description</th>
                  <th style="width:32px" />
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="p in refClass.props"
                  :key="p.name"
                  :class="{ 'row-drilldown': !!refDrillTarget(p) }"
                  @click="refDrillTarget(p) && refNavigate('class', refDrillTarget(p))"
                >
                  <td>
                    <span :class="['prop-name', { 'prop-name--readonly': p.readonly }]">{{ p.name }}</span>
                    <span v-if="refDrillTarget(p)" class="primary--text ml-1">›</span>
                    <span v-if="refPropDesc(p) && refPropDesc(p).sbcProperty === false" class="tag tag-sbc-only ml-1">SBC only</span>
                    <span v-else-if="refPropDesc(p) && refPropDesc(p).sbcProperty === true" class="tag tag-sbc ml-1">SBC</span>
                  </td>
                  <td>
                    <span :class="['prop-type', { 'prop-type--link': refTypeLink(p) }]" @click.stop="refTypeLink(p) && refNavigate(refTypeLink(p).kind, refTypeLink(p).name)">{{ refTypeDisplay(p) }}</span>
                    <span v-if="p.nullable" class="grey--text" style="font-size:11px"> or null</span>
                    <div v-if="omModel.enums[p.type]" style="display:flex;flex-wrap:wrap;gap:3px;margin-top:4px">
                      <span v-for="m in omModel.enums[p.type].members" :key="m.key" class="enum-pip">{{ m.key }}</span>
                    </div>
                  </td>
                  <td><code class="prop-default">{{ p.default || '' }}</code></td>
                  <td v-if="hasAnyRefDesc" class="desc-cell">
                    <template v-if="refPropDesc(p)">
                      {{ refPropDesc(p).summary }}
                      <div v-if="refPropDesc(p).remarks" class="grey--text mt-1" style="font-size:11px;font-style:italic">{{ refPropDesc(p).remarks }}</div>
                    </template>
                    <span v-else class="grey--text">—</span>
                  </td>
                  <td style="text-align:center">
                    <v-btn icon x-small :title="'Copy: ' + refPropPath(p)" @click.stop="copyPath(refPropPath(p))"><v-icon x-small>mdi-content-copy</v-icon></v-btn>
                  </td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>

        <!-- Reference enum detail -->
        <template v-else-if="detailMode === 'ref-enum' && refEnum">
          <h2 class="title primary--text mb-2">{{ refEnum.name }}</h2>
          <v-simple-table dense class="prop-table">
            <template #default>
              <thead><tr><th>Value</th><th>Description</th></tr></thead>
              <tbody>
                <tr v-for="m in refEnum.members" :key="m.key">
                  <td><code style="color:#a6e3a1">{{ m.key }}</code><code v-if="m.value !== undefined" class="prop-default ml-2">= {{ m.value }}</code></td>
                  <td class="desc-cell">{{ enumMemberDesc(refEnum.name, m.key) || '—' }}</td>
                </tr>
              </tbody>
            </template>
          </v-simple-table>
        </template>
      </div>
    </div>

    <v-snackbar v-model="copiedSnackbar" timeout="1500" bottom right color="success" :elevation="2">
      <v-icon small class="mr-1">mdi-check</v-icon> Copied
    </v-snackbar>
  </v-container>
</template>

<script>
import { omModel as BUNDLED_MODEL, omDescriptions as BUNDLED_DESCRIPTIONS, MODEL_REF, DSF_REF_LABEL } from './model-data.js'
import store from '@/store'

// ── Type helpers ───────────────────────────────────────────────
function resolveCollectionType (t) {
  let m = t.match(/ModelCollection<([^>]+)>/)
  if (m) return m[1].replace(/\s*\|\s*null/g, '').trim()
  m = t.match(/ModelDictionary<([^>]+)>/)
  if (m) { const p = m[1].split(','); return p[p.length - 1].replace(/\s*\|\s*null/g, '').trim() }
  m = t.match(/Array<([^>]+)>/)
  if (m) return m[1].trim()
  return null
}
function isCollectionType (t) { return /ModelCollection</.test(t) || /Array</.test(t) || t.endsWith('[]') }
function isDictType (t) { return /ModelDictionary</.test(t) || /Map</.test(t) }
function shortType (t) { return t.replace('ModelCollection', 'Collection').replace('ModelDictionary', 'Dict').replace('ModelSet', 'Set') }
function pascalToCamel (s) { return s ? s.charAt(0).toLowerCase() + s.slice(1) : s }

// Get a human-readable type name for a live value
function liveTypeName (val) {
  if (val === null) return 'null'
  if (Array.isArray(val)) return 'array[' + val.length + ']'
  if (val instanceof Map) return 'map'
  if (typeof val === 'object') return 'object'
  return typeof val
}

// Walk a dot-path like "heat.heaters[0].current" against an object
function resolvePath (obj, path) {
  try {
    let cur = obj
    for (const seg of path.replace(/\[(\d+)\]/g, '.$1').split('.')) {
      if (cur == null || typeof cur !== 'object') return undefined
      cur = cur instanceof Map ? cur.get(seg) : cur[seg]
    }
    return cur
  } catch (e) { return undefined }
}

export default {
  name: 'OmBrowser',

  data () {
    return {
      omModel: BUNDLED_MODEL,
      descriptions: BUNDLED_DESCRIPTIONS,
      modelRef: MODEL_REF,
      dsfLabel: `DSF: ${DSF_REF_LABEL} (${Object.keys(BUNDLED_DESCRIPTIONS).length} types)`,

      // Tree state
      openNodes: {},
      searchTerm: '',
      treeVersion: 0,  // incremented on refresh to force recompute

      // Selection
      selectedNode: null,   // dot-path for live mode, class name for ref mode
      detailMode: null,     // 'live' | 'ref' | 'ref-enum'
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
    liveKeyCount () {
      if (!this.liveModel) return 0
      return Object.keys(this.liveModel).length
    },

    // ── Tree rows ──────────────────────────────────────────────
    treeRows () {
      // eslint-disable-next-line no-unused-expressions
      this.treeVersion  // reactive dependency for manual refresh
      const rows = []
      if (this.hasLiveModel) {
        this.buildLiveRows(rows, this.liveModel, '', 0)
      } else {
        this.buildRefRows(rows)
      }
      return rows
    },

    // ── Detail panel (live mode) ───────────────────────────────
    detailObj () {
      if (this.detailMode !== 'live' || !this.selectedNode || !this.liveModel) return null
      if (this.selectedNode === '') return this.liveModel
      return resolvePath(this.liveModel, this.selectedNode)
    },

    detailLabel () {
      if (!this.selectedNode) return ''
      const parts = this.selectedNode.split('.')
      return parts[parts.length - 1].replace(/\[\d+\]$/, '')
    },

    detailClassDesc () {
      if (this.detailMode !== 'live') return null
      const typeName = this.guessClassName(this.selectedNode)
      if (!typeName) return null
      return (this.descriptions[typeName] || {}).__class__ || null
    },

    detailRows () {
      if (this.detailMode !== 'live' || !this.detailObj || typeof this.detailObj !== 'object') return []
      const obj = this.detailObj
      const basePath = this.selectedNode ? this.selectedNode + '.' : ''
      const typeName = this.guessClassName(this.selectedNode)
      const rows = []

      const entries = obj instanceof Map
        ? Array.from(obj.entries())
        : Object.entries(obj).sort((a, b) => a[0] < b[0] ? -1 : 1)

      for (const [key, val] of entries) {
        const path = basePath + key
        const drillable = val !== null && typeof val === 'object'
        const desc = this.getDescForPath(typeName, String(key))
        // Find TS type info for this property
        const tsProp = typeName ? this.findTsProp(typeName, String(key)) : null
        const enumMembers = tsProp && this.omModel.enums[tsProp.type]
          ? this.omModel.enums[tsProp.type].members.map(m => m.key)
          : null
        rows.push({
          key: String(key),
          value: val,
          path,
          drillable,
          typeName: liveTypeName(val),
          nullable: tsProp ? tsProp.nullable : false,
          desc,
          enumMembers
        })
      }
      return rows
    },

    hasAnyLiveDesc () {
      return this.detailRows.some(r => r.desc)
    },

    // ── Detail panel (ref mode) ────────────────────────────────
    refClass () {
      if (this.detailMode !== 'ref' || !this.selectedNode) return null
      return this.omModel.classes[this.selectedNode] || null
    },
    refEnum () {
      if (this.detailMode !== 'ref-enum' || !this.selectedNode) return null
      return this.omModel.enums[this.selectedNode] || null
    },
    refClassDesc () {
      if (!this.refClass) return null
      return (this.descriptions[this.refClass.name] || {}).__class__ || null
    },
    currentPaths () {
      return this.refClass ? this.findPaths(this.refClass.name) : []
    },
    hasAnyRefDesc () {
      return this.refClass ? (this.refClass.props || []).some(p => this.refPropDesc(p)) : false
    },

    // ── Search ─────────────────────────────────────────────────
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
    // ── Tree building ──────────────────────────────────────────

    buildLiveRows (rows, obj, path, depth) {
      if (!obj || typeof obj !== 'object') return
      const entries = obj instanceof Map
        ? Array.from(obj.entries())
        : Object.entries(obj).sort((a, b) => a[0] < b[0] ? -1 : 1)

      for (const [key, val] of entries) {
        const id = path ? path + '.' + key : String(key)
        const hasChildren = val !== null && typeof val === 'object'
        const typeName = liveTypeName(val)

        rows.push({ id, label: String(key), typeName, hasChildren, depth, isLive: true })

        if (hasChildren && this.openNodes[id]) {
          if (Array.isArray(val)) {
            val.forEach((item, i) => {
              const childId = id + '[' + i + ']'
              const childHasChildren = item !== null && typeof item === 'object'
              rows.push({ id: childId, label: String(i), typeName: liveTypeName(item), hasChildren: childHasChildren, depth: depth + 1, isLive: true })
              if (childHasChildren && this.openNodes[childId]) {
                this.buildLiveRows(rows, item, childId, depth + 2)
              }
            })
          } else {
            this.buildLiveRows(rows, val, id, depth + 1)
          }
        }
      }
    },

    buildRefRows (rows) {
      const root = this.omModel.classes['ObjectModel']
      if (!root) return
      const walk = (cls, propName, depth) => {
        const id = 'ref:' + cls.name + ':' + propName
        rows.push({ id, label: propName, typeName: cls.name, hasChildren: true, depth, isLive: false, className: cls.name })
        if (!this.openNodes[id]) return
        for (const p of cls.props || []) {
          const inner = resolveCollectionType(p.type)
          const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
          const child = this.omModel.classes[t]
          if (child) walk(child, p.name, depth + 1)
        }
      }
      walk(root, 'objectModel', 0)
    },

    refresh () { this.treeVersion++ },

    toggleNode (id) {
      this.$set(this.openNodes, id, !this.openNodes[id])
    },

    expandAll () {
      if (this.hasLiveModel) {
        // Expand top level only to avoid hanging on large models
        const newOpen = {}
        if (this.liveModel) {
          for (const key of Object.keys(this.liveModel)) {
            newOpen[key] = true
          }
        }
        this.openNodes = newOpen
      } else {
        const newOpen = {}
        const walk = (cls, propName, depth) => {
          if (depth > 4) return
          newOpen['ref:' + cls.name + ':' + propName] = true
          for (const p of cls.props || []) {
            const inner = resolveCollectionType(p.type)
            const t = (inner || p.type).replace(/\s*\|\s*null/g, '').trim()
            const child = this.omModel.classes[t]
            if (child) walk(child, p.name, depth + 1)
          }
        }
        const root = this.omModel.classes['ObjectModel']
        if (root) walk(root, 'objectModel', 0)
        this.openNodes = newOpen
      }
    },

    collapseAll () { this.openNodes = {} },

    selectRow (row) {
      this.navStack = []
      if (row.isLive) {
        const val = resolvePath(this.liveModel, row.id)
        if (val !== null && typeof val === 'object' && !Array.isArray(val)) {
          this.selectedNode = row.id
          this.detailMode = 'live'
        } else {
          // For leaf/array, show parent
          const parts = row.id.split('.')
          parts.pop()
          this.selectedNode = parts.join('.') || ''
          this.detailMode = 'live'
        }
      } else {
        this.selectedNode = row.className
        this.detailMode = 'ref'
      }
    },

    selectSearchResult (clsName) {
      this.navStack = []
      this.selectedNode = clsName
      this.detailMode = 'ref'
    },

    // ── Live detail drill-down ─────────────────────────────────

    drillInto (row) {
      if (this.selectedNode !== null) {
        this.navStack = [...this.navStack, { label: this.detailLabel || this.selectedNode, node: this.selectedNode, mode: this.detailMode }]
      }
      this.selectedNode = row.path
      this.detailMode = 'live'
    },

    navBack (idx) {
      const entry = this.navStack[idx]
      if (!entry) return
      this.navStack = this.navStack.slice(0, idx)
      this.selectedNode = entry.node
      this.detailMode = entry.mode
    },

    // ── Reference mode navigation ──────────────────────────────

    refNavigate (type, name) {
      if (this.selectedNode) {
        this.navStack = [...this.navStack, { label: this.selectedNode, node: this.selectedNode, mode: this.detailMode }]
      }
      this.selectedNode = name
      this.detailMode = type === 'enum' ? 'ref-enum' : 'ref'
    },

    // ── Description helpers ────────────────────────────────────

    guessClassName (path) {
      if (!path) return 'ObjectModel'
      // Walk the TS model to find the class at this path
      const segs = path.replace(/\[\d+\]/g, '').split('.').filter(Boolean)
      let cls = this.omModel.classes['ObjectModel']
      for (const seg of segs) {
        if (!cls) return null
        const prop = (cls.props || []).find(p => p.name === seg)
        if (!prop) return null
        const inner = resolveCollectionType(prop.type)
        const t = (inner || prop.type).replace(/\s*\|\s*null/g, '').trim()
        cls = this.omModel.classes[t] || null
      }
      return cls ? cls.name : null
    },

    findTsProp (className, propName) {
      const cls = this.omModel.classes[className]
      if (!cls) return null
      return (cls.props || []).find(p => p.name === propName) || null
    },

    getDescForPath (className, propName) {
      if (!className) return null
      return this.getPropDesc(className, propName)
    },

    getPropDesc (className, propName) {
      const d = this.descriptions[className]
      if (!d) return null
      return d[propName] || d[propName.charAt(0).toUpperCase() + propName.slice(1)] || null
    },
    refPropDesc (p) {
      if (!this.refClass) return null
      return this.getPropDesc(this.refClass.name, p.name)
    },
    enumMemberDesc (enumName, memberName) {
      const d = this.descriptions[enumName]
      if (!d) return null
      const e = d[memberName] || d[pascalToCamel(memberName)]
      return e ? e.summary || null : null
    },

    // ── Reference type helpers ─────────────────────────────────

    refDrillTarget (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner && this.omModel.classes[inner]) return inner
      if (isDictType(p.type) && inner && this.omModel.classes[inner]) return inner
      if (this.omModel.classes[p.type]) return p.type
      return null
    },
    refTypeDisplay (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner) return inner + '[]'
      if (isDictType(p.type) && inner) return inner + '{}'
      return shortType(p.type)
    },
    refTypeLink (p) {
      const inner = resolveCollectionType(p.type)
      if (isCollectionType(p.type) && inner && (this.omModel.classes[inner] || this.omModel.enums[inner])) {
        return { name: inner, kind: this.omModel.classes[inner] ? 'class' : 'enum' }
      }
      if (isDictType(p.type) && inner && this.omModel.classes[inner]) return { name: inner, kind: 'class' }
      if (this.omModel.classes[p.type]) return { name: p.type, kind: 'class' }
      if (this.omModel.enums[p.type]) return { name: p.type, kind: 'enum' }
      return null
    },
    refPropPath (p) {
      const paths = this.findPaths(this.refClass.name)
      const base = paths.length > 0 ? paths[0] : pascalToCamel(this.refClass.name)
      const isColOrDict = isCollectionType(p.type) || isDictType(p.type)
      return (base ? base + (isColOrDict ? '[0].' : '.') : '') + p.name
    },
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
          for (const g of getAll(child)) subMap[name].add(g)
        }
        return subMap[name]
      }
      for (const n of Object.keys(direct)) getAll(n)
      const results = new Set()
      const visited = new Set()
      const walk = (cls, soFar) => {
        const key = cls.name + '|' + soFar
        if (visited.has(key)) return
        visited.add(key)
        if (cls.name === targetClassName && soFar) { results.add(soFar); return }
        for (const prop of cls.props || []) {
          const inner = resolveCollectionType(prop.type)
          const isCol = isCollectionType(prop.type)
          const isDct = isDictType(prop.type)
          const t = (inner || prop.type).replace(/\s*\|\s*null/g, '').trim()
          if (!this.omModel.classes[t]) continue
          const suffix = isCol ? '[]' : isDct ? '{}' : ''
          const seg = soFar ? soFar + '.' + prop.name + suffix : prop.name + suffix
          const toWalk = new Set([t])
          for (const sub of (subMap[t] || [])) toWalk.add(sub)
          for (const n of toWalk) { const c = this.omModel.classes[n]; if (c) walk(c, seg) }
        }
      }
      walk(root, '')
      return [...results].sort()
    },

    // ── Live value formatting ──────────────────────────────────

    fmtLive (val) {
      if (val === undefined) return '—'
      if (val === null) return 'null'
      if (typeof val === 'boolean') return String(val)
      if (typeof val === 'number') return String(val)
      if (typeof val === 'string') return '"' + val + '"'
      if (Array.isArray(val)) return '[' + val.length + ' items]'
      if (val instanceof Map) return '{map ' + val.size + '}'
      return '{object}'
    },
    liveValClass (val) {
      if (val === null || val === undefined) return 'live-val--null'
      if (val === true) return 'live-val--true'
      if (val === false) return 'live-val--false'
      return ''
    },

    // ── Clipboard ─────────────────────────────────────────────

    copyPath (path) {
      if (navigator.clipboard) {
        navigator.clipboard.writeText(path).then(() => { this.copiedSnackbar = true }).catch(() => {})
      }
    },

    // ── Resize ────────────────────────────────────────────────

    startResize (e) {
      this.resizing = true
      this.resizeStartX = e.clientX
      this.resizeStartW = this.$refs.treePanel ? this.$refs.treePanel.offsetWidth : 340
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
  min-height: 26px;
}
.tree-row:hover { background: rgba(255,255,255,0.06); }
.tree-row--selected { background: rgba(137,180,250,0.15); }

.tree-toggle { width: 14px; flex-shrink: 0; font-size: 9px; color: #7f849c; text-align: center; }
.tree-name { font-weight: 500; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tree-type { font-size: 11px; color: #7f849c; margin-left: auto; padding-left: 8px; flex-shrink: 0; white-space: nowrap; }

.resizer { width: 5px; background: transparent; cursor: col-resize; flex-shrink: 0; }
.resizer:hover { background: rgba(137,180,250,0.4); }

.om-path-code {
  font-family: monospace; font-size: 12px; color: #f9e2af;
  background: rgba(249,226,175,0.08); border: 1px solid rgba(249,226,175,0.2);
  border-radius: 4px; padding: 2px 8px;
}

.class-desc {
  font-size: 13px; line-height: 1.6;
  background: rgba(137,180,250,0.06); border-left: 3px solid #89b4fa;
  padding: 8px 12px; border-radius: 0 4px 4px 0;
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
  border-radius: 3px; padding: 1px 5px;
}

.live-val { font-family: monospace; font-size: 12px; }
.live-val--null { color: #7f849c; }
.live-val--true { color: #a6e3a1; }
.live-val--false { color: #f38ba8; }
</style>
