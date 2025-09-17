<template>
  <div ref="container" class="graph-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import G6 from '@antv/g6'

function randomLightColor() {
  const r = Math.floor(150 + Math.random() * 70)  // 150~220
  const g = Math.floor(150 + Math.random() * 70)
  const b = Math.floor(150 + Math.random() * 70)
  return `rgb(${r},${g},${b})`
}



const groups = {
  A: { name: '抖音', nodes: ['A1', 'A21', 'A22', 'A23'] },
  B: { name: '微博', nodes: ['B1', 'B21', 'B22', 'B23'] },
  C: { name: '小红书', nodes: ['C1', 'C21', 'C22', 'C23'] }
}

// 给每个族群分配颜色
for (const key in groups) {
  groups[key].color = randomLightColor()
}

// 构造节点数据，带comboId表示所属族群
const nodes = []
for (const key in groups) {
  const group = groups[key]
  group.nodes.forEach(id => {
    const isPrimary = id.length === 2
    nodes.push({
      id,
      label: id,
      comboId: key,
      group: key,
      isPrimary,
      size: isPrimary ? 60 : 20, // 一级节点大，二级节点小
      style: {
        fill: groups[key].color,
        stroke: '#ccc',
        lineWidth: 1,
        cursor: 'pointer'
      },
      labelCfg: {
        style: {
          fill: '#000',
          fontWeight: isPrimary ? 'bold' : 'normal', // 一级加粗，二级正常
          fontSize: isPrimary ? 16 : 12
        },
        position: isPrimary ? 'center' : 'right',
        offset: isPrimary ? 0 : 10
      }
    })
  })
}

const edges = [
  { source: 'A1', target: 'B1' },
  { source: 'B1', target: 'C1' },
  { source: 'A1', target: 'A21' },
  { source: 'A1', target: 'A22' },
  { source: 'A1', target: 'A23' },
  { source: 'B1', target: 'B21' },
  { source: 'B1', target: 'B22' },
  { source: 'B1', target: 'B23' },
  { source: 'C1', target: 'C21' },
  { source: 'C1', target: 'C22' },
  { source: 'C1', target: 'C23' }
]

const combos = Object.entries(groups).map(([key, group]) => ({
  id: key,
  label: group.name,
  style: {
    fill: groups[key].color,
    opacity: 0.1,
    stroke: groups[key].color,
    lineWidth: 2,
    shadowColor: groups[key].color,
    shadowBlur: 30,
    shadowOffsetX: 0,
    shadowOffsetY: 0,
    filter: `drop-shadow(0 0 10px ${groups[key].color})`
  },
  labelCfg: {
    style: {
      fill: '#000',
      fontWeight: 'bold',
      fontSize: 16
    }
  }
}))

const container = ref(null)
let graph = null

onMounted(() => {
  graph = new G6.Graph({
    container: container.value,
    width: window.innerWidth - 64,
    height: 600,
    modes: {
      default: ['drag-canvas', 'zoom-canvas', 'drag-node']
    },
    defaultNode: {
      type: 'circle',
      style: {
        lineWidth: 1,
        cursor: 'pointer'
      },
      labelCfg: {
        style: {
          fill: '#000'
        }
      }
    },
    defaultEdge: {
      type: 'quadratic', // 弧线
      style: {
        lineWidth: 2,
        endArrow: {
          path: G6.Arrow.triangle(10, 15, 5),
          fill: '#000'
        }
      }
    },
    defaultCombo: {
      type: 'circle',
      padding: [40, 40, 40, 40],
      style: {
        cursor: 'default'
      }
    },
    combos,
    layout: {
      type: 'comboForce',
      preventOverlap: true,
      nodeSpacing: 50,
      comboSpacing: 200,
      nodeStrength: -30,
      edgeStrength: 0.1,
      comboGravity: 0.1,
      center: [(window.innerWidth - 64) / 2, 300]
    }
  })

  // 设置边颜色等
  edges.forEach(edge => {
    const sourceNode = nodes.find(n => n.id === edge.source)
    if (sourceNode) {
      edge.style = {
        stroke: sourceNode.style.fill,
        endArrow: {
          path: G6.Arrow.triangle(10, 15, 5),
          fill: sourceNode.style.fill
        },
        lineWidth: 2
      }
    }
  })

  graph.data({ nodes, edges, combos })
  graph.render()
  // 等布局完成后fitView
  graph.on('afterlayout', () => {
    graph.fitView(20)
  })


  // 节点点击打印信息
  graph.on('node:click', evt => {
    const node = evt.item
    console.log('点击节点信息:', node.getModel())
  })

  // 绑定 resize 事件监听
  window.addEventListener('resize', onResize)
})

function onResize() {
  if (!graph) return
  const width = window.innerWidth - 64
  const height = 600
  graph.changeSize(width, height)
  graph.updateLayout({
    center: [width / 2, height / 2]
  })
  graph.fitView(20)
}

onBeforeUnmount(() => {
  // 移除事件监听，销毁图实例
  window.removeEventListener('resize', onResize)
  if (graph) {
    graph.destroy()
    graph = null
  }
})
</script>

<style scoped>
.graph-container {
  width: calc(100vw - 64px);
  height: 600px;
  margin: 0 auto;
  border: 1px solid #ddd;
  background: #fafafa;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
}
</style>
