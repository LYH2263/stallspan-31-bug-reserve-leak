<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { api } from '../api'
const data = ref<any>(null)
const vendors = ref<any[]>([])
async function run() { data.value = await api('/allocate/run?segment_id=1', { method: 'POST' }) }
onMounted(async () => {
  vendors.value = await api('/vendors')
  await run()
})
const colors = ['#e8a87c','#85dcb8','#e27d60','#c38d9e','#41b3a3','#f4a261','#e76f51']
// 所有图形坐标都取自同一次运行快照（引擎挖带结果）：
// 应急留白、挡柱、摊位起止共用同一套米数，主图不得另算一套。
const cells = computed(() => {
  if (!data.value) return []
  const width = data.value.segment.width_m
  const pct = (m: number) => (m / width) * 100
  const out: any[] = []
  // 留白与挡柱一律取自本次运行快照的 blocked_spans——即引擎先挖应急带、
  // 再并挡柱后输出的同一套禁入区，主图不再自行按登记值或挡柱另算
  for (const b of data.value.blocked_spans || []) {
    let label: string
    if (b.kind === 'emergency') {
      label = b.start_m <= 1e-9 ? `起点应急 ${b.end_m}m` : `终点应急 ${width - b.start_m}m`
    } else {
      const mid = (b.start_m + b.end_m) / 2
      const p = (data.value.pillars || []).find((q: any) =>
        mid >= q.position_m - q.thickness_m / 2 - 1e-9 && mid <= q.position_m + q.thickness_m / 2 + 1e-9)
      label = p?.label || '挡柱'
    }
    out.push({ type: b.kind, start: b.start_m, w: b.end_m - b.start_m, label })
  }
  for (const [i, p] of (data.value.placements || []).entries()) {
    out.push({ type: 'stall', start: p.start_m, w: p.width_m, label: p.vendor_name, color: colors[i % colors.length] })
  }
  return out.map(c => ({ ...c, left: pct(c.start), pct: Math.max(pct(c.w), 1.5) }))
})
</script>
<template>
  <div class="ss-street-wrap">
    <h1>街段分配带</h1>
    <p class="sub">沿街一维开间 · 两端为应急留白（摊位起止不得进入）· 挡柱为竖直阻断 · 底部为摊主排队</p>
    <button class="btn" @click="run">重新分配</button>
    <div class="ss-band-ruler" v-if="data">
      <span>{{ (data.emergency?.start_m || 0) > 0 ? `应急 ${data.emergency.start_m}m ▏0 m` : '0 m' }}</span>
      <span>{{ data.segment.name }} · 共 {{ data.segment.width_m }} m · 可用开间从应急带内侧起算</span>
      <span>{{ data.segment.width_m }} m ▕{{ (data.emergency?.end_m || 0) > 0 ? ` 应急 ${data.emergency.end_m}m` : '' }}</span>
    </div>
    <div class="ss-street-band" v-if="data">
      <div class="ss-street-inner ss-street-abs">
        <div
          v-for="(c,i) in cells" :key="i"
          class="ss-band-cell"
          :class="{ 'ss-pillar': c.type === 'pillar', 'ss-emergency': c.type === 'emergency' }"
          :style="{ left: c.left + '%', width: c.pct + '%', background: c.type === 'stall' ? c.color : undefined }"
        >{{ c.type === 'stall' ? c.label : '' }}<span v-if="c.type !== 'stall'" class="ss-zone-label">{{ c.label }}</span></div>
      </div>
    </div>
    <div class="ss-vendor-queue">
      <div v-for="v in vendors" :key="v.id" class="ss-vendor-chip">
        <strong>{{ v.name }}</strong>
        <span>需 {{ v.stall_width_m }} m · 优先 {{ v.priority }}</span>
      </div>
    </div>
    <div class="card" v-if="data">
      <table>
        <thead><tr><th>摊主</th><th>起点</th><th>终点</th><th>宽度</th></tr></thead>
        <tbody>
          <tr v-for="p in data.placements" :key="p.vendor_id">
            <td>{{ p.vendor_name }}</td><td>{{ p.start_m }}</td><td>{{ p.end_m }}</td><td>{{ p.width_m }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>
