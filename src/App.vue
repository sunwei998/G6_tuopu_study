<template>
  <div ref="container" class="graph-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import G6 from '@antv/g6'

// 生成浅色随机色，避免太白也不太暗
function randomLightColor() {
   const h = Math.floor(Math.random() * 360)
  const s = Math.floor(40 + Math.random() * 30)  // 40%-70% 饱和度，避免过艳
  const l = Math.floor(50 + Math.random() * 20)  // 50%-70% 亮度，避免过浅或过暗
  return `hsl(${h}, ${s}%, ${l}%)`
}


// 后端返回的节点数据示例
const nodes = [
  { id: 'A1', label: '爱搞机的王老吉', groupName: '小红书', level: 1 },
  { id: 'A21', label: '爱搞机的张老吉', groupName: '小红书', level: 2 },
  { id: 'A22', label: '华为老黑马', groupName: '小红书', level: 2 },
  { id: 'A23', label: '华为小白鼠', groupName: '小红书', level: 2 },
  { id: 'B1', label: '校长的梦', groupName: '抖音', level: 1 },
  { id: 'B21', label: '苹果中国', groupName: '抖音', level: 2 },
  { id: 'B22', label: '小鬼的苹果全家桶', groupName: '抖音', level: 2 },
  { id: 'B23', label: '挥霍无度回家', groupName: '抖音', level: 2 },
  { id: 'C1', label: '华为问界', groupName: '微博', level: 1 },
  { id: 'C21', label: '老丁搞机', groupName: '微博', level: 2 },
  { id: 'C22', label: '华强北一哥', groupName: '微博', level: 2 },
  { id: 'C23', label: '张某玩手机', groupName: '微博', level: 2 }
]

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

const container = ref(null)
let graph = null

onMounted(() => {
  // 1. 提取所有族群名，生成 groups 和 combos
  const groupNames = [...new Set(nodes.map(n => n.groupName))]
  const groups = {}
  groupNames.forEach(name => {
    groups[name] = {
      color: randomLightColor(),
      name
    }
  })

  const combos = groupNames.map(name => ({
    id: name,
    label: name,
    style: {
      fill: groups[name].color,
      opacity: 0.1,
      stroke: groups[name].color,
      lineWidth: 2,
      shadowColor: groups[name].color,
      shadowBlur: 30,
      shadowOffsetX: 0,
      shadowOffsetY: 0,
      filter: `drop-shadow(0 0 10px ${groups[name].color})`
    },
    labelCfg: {
      style: {
        fill: '#000',
        fontWeight: 'bold',
        fontSize: 16
      }
    }
  }))

  // 2. 处理节点数据，补充 comboId、大小、样式、标签配置
  const processedNodes = nodes.map(node => {
    const isPrimary = node.level === 1
    const groupColor = groups[node.groupName].color
    return {
      ...node,
      comboId: node.groupName,
      size: isPrimary ? 60 : 20,
      style: {
        fill: groupColor,
        stroke: '#ccc',
        lineWidth: 1,
        cursor: 'pointer'
      },
      labelCfg: {
        style: {
          fill: '#000',
          fontWeight: isPrimary ? '700' : '600',
          fontSize: isPrimary ? 16 : 12
        },
        position: isPrimary ? 'center' : 'right',
        offset: isPrimary ? 0 : 10
      }
    }
  })

  // 3. 设置边颜色和箭头颜色与源节点颜色一致
  edges.forEach(edge => {
    const sourceNode = processedNodes.find(n => n.id === edge.source)
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

  // 4. 初始化图实例
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

  graph.data({ nodes: processedNodes, edges, combos })
  graph.render()

  // 布局完成后自动缩放并居中
  graph.on('afterlayout', () => {
    graph.fitView(20)
  })

  // 节点点击事件
  graph.on('node:click', evt => {
    const node = evt.item
    console.log('点击节点信息:', node.getModel())
  })

  // 监听窗口大小变化，动态调整画布和布局
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
  window.addEventListener('resize', onResize)

  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize)
    if (graph) {
      graph.destroy()
      graph = null
    }
  })
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
