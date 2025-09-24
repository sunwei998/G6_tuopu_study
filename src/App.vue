<template>
  <div ref="container" class="graph-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import G6 from '@antv/g6'

// 生成浅色随机色，避免太白也不太暗
// 通过 HSL 色彩空间生成，色相随机，饱和度和亮度控制在一定范围内，保证颜色柔和且明亮
function randomLightColor() {
  const h = Math.floor(Math.random() * 360)
  const s = Math.floor(40 + Math.random() * 30)  // 饱和度 40%-70%
  const l = Math.floor(50 + Math.random() * 20)  // 亮度 50%-70%
  return `hsl(${h}, ${s}%, ${l}%)`
}

// 截断文本函数，超过最大长度时用省略号替代
function truncateText(text, maxLength) {
  if (text.length <= maxLength) return text
  return text.slice(0, maxLength) + '…'
}

// 后端返回的节点数据示例，包含节点id、显示标签、所属群组和层级信息
const nodes = [
  { id: 'A1', label: '爱搞机的王老吉', groupName: '小红书', level: 1 },
  { id: 'A21', label: '爱搞机的张老吉', groupName: '小红书', level: 2 },
  { id: 'A22', label: '华为老黑马', groupName: '小红书', level: 2 },
  { id: 'A23', label: '华为小白鼠', groupName: '小红书', level: 2 },
  { id: 'B1', label: '校长的梦', groupName: '抖音', level: 1 },
  { id: 'B21', label: '苹果中国', groupName: '抖音', level: 2 },
  { id: 'B22', label: '小鬼的苹果全家桶都坏了', groupName: '抖音', level: 2 },
  { id: 'B23', label: '挥霍无度回家', groupName: '抖音', level: 2 },
  { id: 'C1', label: '华为问界', groupName: '微博', level: 1 },
  { id: 'C21', label: '老丁搞机', groupName: '微博', level: 2 },
  { id: 'C22', label: '华强北一哥', groupName: '微博', level: 2 },
  { id: 'C23', label: '张某玩手机', groupName: '微博', level: 2 }
]

// 后端返回的边数据，定义节点之间的连接关系
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

const container = ref(null) // 容器 DOM 引用
let graph = null // G6 图实例

// Tooltip DOM 元素，用于显示节点完整名称
let tooltipDiv = null

// 创建 tooltip DOM，设置样式并添加到 body
function createTooltip() {
  tooltipDiv = document.createElement('div')
  tooltipDiv.style.position = 'fixed' // 固定定位，方便跟随鼠标
  tooltipDiv.style.padding = '6px 10px'
  tooltipDiv.style.background = 'rgba(0,0,0,0.75)' // 半透明黑色背景
  tooltipDiv.style.color = '#fff' // 白色文字
  tooltipDiv.style.borderRadius = '4px'
  tooltipDiv.style.fontSize = '12px'
  tooltipDiv.style.pointerEvents = 'none' // 不阻止鼠标事件
  tooltipDiv.style.zIndex = 1000 // 保证在最上层
  tooltipDiv.style.visibility = 'hidden' // 初始隐藏
  document.body.appendChild(tooltipDiv)
}

// 显示 tooltip，内容为 content，位置根据鼠标 x,y 偏移显示
function showTooltip(content, x, y) {
  tooltipDiv.innerText = content
  tooltipDiv.style.top = y + 10 + 'px'
  tooltipDiv.style.left = x + 10 + 'px'
  tooltipDiv.style.visibility = 'visible'
}

// 隐藏 tooltip
function hideTooltip() {
  tooltipDiv.style.visibility = 'hidden'
}

onMounted(() => {
  createTooltip() // 初始化 tooltip

  // 1. 提取所有唯一的群组名称，生成 groups 对象，给每个群组分配随机颜色
  const groupNames = [...new Set(nodes.map(n => n.groupName))]
  const groups = {}
  groupNames.forEach(name => {
    groups[name] = {
      color: randomLightColor(),
      name
    }
  })

  // 2. 根据群组生成 combos（族群遮罩），设置样式和标签样式
  const combos = groupNames.map(name => ({
    id: name,
    label: name,
    style: {
      fill: groups[name].color,
      opacity: 0.05, // 半透明
      stroke: groups[name].color,
      lineWidth: 2,
      shadowColor: groups[name].color,
      shadowBlur: 30,
      shadowOffsetX: 0,
      shadowOffsetY: 0,
      filter: `drop-shadow(0 0 10px ${groups[name].color})` // 阴影效果
    },
    labelCfg: {
      style: {
        fill: groups[name].color,
        fontWeight: 'bold',
        fontSize: 24
      }
    }
  }))

  // 3. 处理节点数据，补充 comboId 关联群组，设置大小、样式、标签配置
  //    同时对 label 进行截断，保留完整 label 用于 tooltip 显示
  const processedNodes = nodes.map(node => {
    const isPrimary = node.level === 1
    const groupColor = groups[node.groupName].color
    const maxLen = isPrimary ? 5 : 8 // 一级节点最多5个字，非一级最多8个字
    const truncatedLabel = truncateText(node.label, maxLen)
    return {
      ...node,
      comboId: node.groupName,
      size: isPrimary ? 80 : 20, // 一级节点大，二级节点小
      label: truncatedLabel, // 显示截断后的标签
      fullLabel: node.label, // 用于 tooltip 显示完整名称
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
          fontSize: isPrimary ? 14 : 12
        },
        position: isPrimary ? 'center' : 'right', // 一级节点文字居中，二级节点文字右侧
        offset: isPrimary ? 0 : 10
      }
    }
  })

  // 4. 设置边的样式，颜色和箭头颜色与源节点颜色一致
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

  // 5. 初始化 G6 图实例，配置容器、尺寸、交互模式、默认节点、边、combo、布局等
  graph = new G6.Graph({
    container: container.value,
    width: window.innerWidth - 64, // 宽度为视口宽度减64px
    height: 600, // 固定高度600px
    modes: {
      default: [
        'drag-canvas', // 允许拖动画布
        'zoom-canvas', // 允许缩放画布
        {
          type: 'drag-node', // 允许拖拽节点
          shouldBegin: (e) => {
            const node = e.item
            if (!node) return false
            const model = node.getModel()
            // 禁止拖拽 legend 节点，id 以 'legend-' 开头
            if (model.id && model.id.startsWith('legend-')) {
              return false
            }
            return true
          }
        }
      ]
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
      type: 'quadratic', // 弧线边
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
      padding: [40, 40, 40, 40], // combo 内边距
      style: {
        cursor: 'default'
      }
    },
    combos,
    layout: {
      type: 'comboForce', // 族群力导向布局
      preventOverlap: true, // 防止节点重叠
      nodeSpacing: 50, // 节点间距
      comboSpacing: 200, // 族群间距
      nodeStrength: -30, // 节点斥力
      edgeStrength: 0.1, // 边吸引力
      comboGravity: 0.1, // 族群引力
      center: [(window.innerWidth - 64) / 2, 300] // 布局中心点
    }
  })

  // 6. 载入数据并渲染图
  graph.data({ nodes: processedNodes, edges, combos })
  graph.render()

  // 7. 添加图例节点，显示群组颜色和名称
  addLegend()

  // 8. 布局完成后自动缩放并居中，保证所有内容完整显示
  graph.on('afterlayout', () => {
    graph.fitView(20) // 20px 边距
  })

  // 9. 节点点击事件，打印节点信息
  graph.on('node:click', evt => {
    const node = evt.item
    console.log('点击节点信息:', node.getModel())
  })

  // 10. 节点鼠标进入事件，显示 tooltip（完整名称）
  graph.on('node:mouseenter', evt => {
    const node = evt.item
    const model = node.getModel()
    // 只有截断过的才显示 tooltip
    if (model.fullLabel && model.fullLabel !== model.label) {
      showTooltip(model.fullLabel, evt.clientX, evt.clientY)
    }
  })

  // 11. 鼠标移动时更新 tooltip 位置
  graph.on('node:mousemove', evt => {
    if (tooltipDiv && tooltipDiv.style.visibility === 'visible') {
      showTooltip(tooltipDiv.innerText, evt.clientX, evt.clientY)
    }
  })

  // 12. 鼠标离开节点时隐藏 tooltip
  graph.on('node:mouseleave', () => {
    hideTooltip()
  })

  // 13. 监听窗口大小变化，动态调整画布尺寸和布局，并重新添加图例
  function onResize() {
    if (!graph) return
    const width = window.innerWidth - 64
    const height = 600
    graph.changeSize(width, height) // 改变画布尺寸
    graph.updateLayout({
      center: [width / 2, height / 2] // 更新布局中心
    })
    graph.fitView(20) // 缩放居中

    // 重新添加图例，先移除旧图例
    removeLegend()
    addLegend()
  }

  // 14. 添加图例节点函数，图例显示在画布右上角，圆形颜色块 + 群组名称
  function addLegend() {
    const legendNodes = []
    const legendX = graph.getWidth() - 100 // 距离画布右边100px
    let legendY = 40 // 距离顶部40px

    groupNames.forEach(name => {
      const color = groups[name].color
      legendNodes.push({
        id: `legend-${name}`, // 唯一id，方便后续查找移除
        x: legendX,
        y: legendY,
        label: name,
        style: {
          fill: color,
          stroke: '#ccc',
          lineWidth: 1,
          cursor: 'default' // 鼠标样式默认
        },
        size: 12, // 圆形大小
        labelCfg: {
          position: 'right', // 标签在圆形右侧
          offset: 10, // 标签偏移量
          style: {
            fill: '#000',
            fontSize: 14,
            fontWeight: 'bold'
          }
        },
        draggable: false, // 禁止拖拽
        event: false // 禁止事件响应，避免影响交互
      })
      legendY += 30 // 每个图例项垂直间距30px
    })

    // 将图例节点添加到图中
    legendNodes.forEach(node => {
      graph.addItem('node', node)
    })
  }

  // 15. 移除图例节点函数，方便窗口缩放时刷新图例
  function removeLegend() {
    groupNames.forEach(name => {
      const item = graph.findById(`legend-${name}`)
      if (item) graph.removeItem(item)
    })
  }

  // 16. 绑定窗口 resize 事件，调用 onResize 处理
  window.addEventListener('resize', onResize)

  // 17. 组件卸载时清理资源，移除事件监听和销毁图实例、tooltip DOM
  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize)
    if (graph) {
      graph.destroy()
      graph = null
    }
    if (tooltipDiv) {
      document.body.removeChild(tooltipDiv)
      tooltipDiv = null
    }
  })
})
</script>

<style scoped>
.graph-container {
  width: calc(100vw - 64px); /* 宽度为视口宽度减64px */
  height: 600px; /* 固定高度 */
  margin: 0 auto; /* 居中 */
  border: 1px solid #ddd; /* 边框 */
  background: #fafafa; /* 背景色 */
  border-radius: 8px; /* 圆角 */
  box-shadow: 0 4px 12px rgb(0 0 0 / 0.1); /* 阴影 */
}
</style>
