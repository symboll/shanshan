<template>
  <div class="home">
    <button @click="getMapHandler()">get map</button>
    <div class="image">
      <canvas class="bg-canvas" ref="bg-canvas"></canvas>
      <canvas class="canvas" ref="canvas"></canvas>
    </div>
  </div>
</template>

<script>
const CANVAS_WIDTH = 1920
const CANVAS_HIGHT = 1080

const CARD_WIDTH = 300
const CARD_HIGHT = 60

const LINE_X_POSITION = 100
const LINE_X_GAP = 60
const LINE_Y_GAP = 100

const DEFAULT_TIMES = 3 // 画布缩放尺寸  1920 / 640

const BACKGROUND = '#ffffff'
const COLOR = '#42b983'
export default {
  name: 'HomeView',
  data () {
    return {
      ctx: null,
      bgCtx: null,
      tree: [
        {
          id: 'zs-1',
          name: 'zhangsan',
          child: [
            {
              id: 'zs-1-1',
              name: 'zhangsan-1'
            },
            {
              id: 'zs-1-2',
              name: 'zhansgan-2'
            }
          ]
        },
        {
          id: 'ls-2',
          name: 'lisi'
        },
        {
          id: 'ww-3',
          name: 'wangwu',
          child: [
            {
              id: 'ww-3-1',
              name: 'wangwu - 1'
            },
            {
              id: 'ww-3-2',
              name: 'wangwu - 2',
              child: [
                {
                  id: 'ww-3-2-1',
                  name: 'wangwu - 2 - 1'
                }
              ]
            }
          ]
        }
      ],
      list: [
        {
          id: '1',
          label: '工程师'
        },
        {
          id: '2',
          label: '设计师'
        }
      ],

      treeToList: [],
      map: {
        'ww-3-2-1': ['1']
      }, // treeItem -> listItem
      mapWithStartAndEndPoint: {},
      currentFlag: '', // 'treeItem', 'listItem', ''
      currentPosition: {},
      firstId: '',
      lastId: ''
    }
  },
  mounted () {
    this.initCanvasRenderingContext2D()
    this.treeDataTransformList(this.tree)

    this.drawTreeData()
    this.drawListData()

    this.initMapWithStartEndPoint()

    this.ctx.drawImage(this.$refs['bg-canvas'], 0, 0, CANVAS_WIDTH, CANVAS_HIGHT)

    this.drawMapLine()
  },
  methods: {
    initCanvasRenderingContext2D () {
      this.$refs['bg-canvas'].width = CANVAS_WIDTH
      this.$refs['bg-canvas'].height = CANVAS_HIGHT
      this.bgCtx = this.$refs['bg-canvas'].getContext('2d')
      this.bgCtx.fillStyle = BACKGROUND
      this.bgCtx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HIGHT)

      this.$refs.canvas.width = CANVAS_WIDTH
      this.$refs.canvas.height = CANVAS_HIGHT
      this.ctx = this.$refs.canvas.getContext('2d')

      this.$refs.canvas.addEventListener('mousedown', (event) => this.mousedownHandler(event))
      this.$refs.canvas.addEventListener('mousemove', (event) => this.mousemoveHandler(event))
      this.$refs.canvas.addEventListener('mouseup', (event) => this.mouseupHandler(event))
    },

    mousedownHandler (event) {
      const { offsetX, offsetY } = event
      const mouseX = offsetX * DEFAULT_TIMES; const mouseY = offsetY * DEFAULT_TIMES

      for (let i = 0; i < this.treeToList.length; i++) {
        const { position: { x, y, w, h }, id } = this.treeToList[i]
        if (mouseX >= x && mouseX <= x + w && mouseY >= y && mouseY <= y + h) {
          this.currentFlag = 'treeItem'
          this.currentPosition = { x, y, w, h }
          this.firstId = id
        }
      }

      for (let i = 0; i < this.list.length; i++) {
        const { position: { x, y, w, h }, id } = this.list[i]
        if (mouseX >= x && mouseX <= x + w && mouseY >= y && mouseY <= y + h) {
          this.currentFlag = 'listItem'
          this.currentPosition = { x, y, w, h }
          this.firstId = id
        }
      }
    },
    mousemoveHandler (event) {
      if (!this.currentFlag) {
        return
      }
      const { offsetX, offsetY } = event
      const mouseX = offsetX * DEFAULT_TIMES; const mouseY = offsetY * DEFAULT_TIMES
      this.drawLine({ mouseX, mouseY })
    },
    mouseupHandler (event) {
      const { offsetX, offsetY } = event
      const endX = offsetX * DEFAULT_TIMES; const endY = offsetY * DEFAULT_TIMES

      let mouseX, mouseY
      if (this.currentFlag === 'treeItem') {
        for (let i = 0; i < this.list.length; i++) {
          const { position: { x, y, w, h }, id } = this.list[i]
          if (endX >= x && endX <= x + w && endY >= y && endY <= y + h) {
            this.lastId = id

            if (!this.map[this.firstId]) {
              this.map[this.firstId] = []
            }
            this.map[this.firstId].push(this.lastId)

            mouseX = x
            mouseY = y + CARD_HIGHT / 2
          }
        }
      }
      if (this.currentFlag === 'listItem') {
        for (let i = 0; i < this.treeToList.length; i++) {
          const { position: { x, y, w, h }, id } = this.treeToList[i]
          if (endX >= x && endX <= x + w && endY >= y && endY <= y + h) {
            this.lastId = id
            if (!this.map[this.lastId]) {
              this.map[this.lastId] = []
            }
            this.map[this.lastId].push(this.firstId)

            mouseX = x + CARD_WIDTH
            mouseY = y + CARD_HIGHT / 2
          }
        }
      }

      this.mapWithStartAndEndPoint = {}
      this.initMapWithStartEndPoint()

      // 鼠标抬起的时候，落入另一个卡片之中，需要画线连接
      if (this.lastId) {
        this.drawLine({ mouseX, mouseY })
      } else {
        this.ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HIGHT)
        this.ctx.drawImage(this.$refs['bg-canvas'], 0, 0, CANVAS_WIDTH, CANVAS_HIGHT)
        this.drawMapLine()
      }

      this.firstId = ''
      this.lastId = ''
      this.currentFlag = ''
      this.currentPosition = {}
    },

    treeDataTransformList (tree, parentId = '0', level = 0) {
      for (let i = 0; i < tree.length; i++) {
        const { child = [], id, name } = tree[i]
        this.treeToList.push({
          id,
          name,
          parentId,
          level,
          children: child.length
        })
        if (child.length) {
          this.treeDataTransformList(child, id, level + 1)
        }
      }
    },
    drawTreeData () {
      this.bgCtx.moveTo(LINE_X_POSITION, 0)
      this.bgCtx.lineTo(LINE_X_POSITION, CANVAS_HIGHT)
      this.bgCtx.strokeStyle = COLOR
      this.bgCtx.stroke()

      this.treeToList = this.treeToList.map((item, index) => {
        const position = {
          x: LINE_X_POSITION + (item.level + 1) * LINE_X_GAP,
          y: (index + 1) * LINE_Y_GAP,
          w: CARD_WIDTH,
          h: CARD_HIGHT
        }

        return {
          ...item,
          position
        }
      })
      this.treeToList.map((item, index) => this.drawTreeItem(item, index))
    },
    drawTreeItem (data, index) {
      const { position: { x, y, w, h }, name } = data
      this.bgCtx.beginPath()
      this.bgCtx.save()
      this.bgCtx.strokeStyle = COLOR
      this.bgCtx.strokeRect(x, y, w, h)

      if (data.children) {
        this.bgCtx.moveTo(LINE_X_POSITION + (data.level + 1) * LINE_X_GAP, (index + 1) * LINE_Y_GAP + LINE_X_GAP)
        this.bgCtx.lineTo(LINE_X_POSITION + (data.level + 1) * LINE_X_GAP, (index + 1) * LINE_Y_GAP + LINE_X_GAP + data.children * LINE_Y_GAP - LINE_X_GAP / 2)
      }

      this.bgCtx.moveTo(LINE_X_POSITION + data.level * LINE_X_GAP, (index + 1) * LINE_Y_GAP + LINE_X_GAP / 2)
      this.bgCtx.lineTo(LINE_X_POSITION + (data.level + 1) * LINE_X_GAP, (index + 1) * LINE_Y_GAP + LINE_X_GAP / 2)
      this.bgCtx.strokeStyle = COLOR
      this.bgCtx.stroke()

      this.bgCtx.font = '60px sans-serif'
      this.bgCtx.fillStyle = COLOR
      this.bgCtx.textBaseline = 'top'
      this.bgCtx.fillText(name, x, y, w)

      this.bgCtx.restore()
      this.bgCtx.closePath()
    },
    drawListData () {
      this.list = this.list.map((item, index) => {
        const position = {
          x: 1200,
          y: (index + 1) * 100,
          w: CARD_WIDTH,
          h: CARD_HIGHT
        }
        return {
          ...item,
          position
        }
      })

      this.list.map(item => this.drawListItem(item))
    },
    drawListItem (data) {
      const { position: { x, y, w, h }, label } = data
      this.bgCtx.beginPath()
      this.bgCtx.save()
      this.bgCtx.strokeStyle = COLOR
      this.bgCtx.strokeRect(x, y, w, h)
      this.bgCtx.font = '60px sans-serif'
      this.bgCtx.fillStyle = COLOR
      this.bgCtx.textBaseline = 'top'
      this.bgCtx.fillText(label, x, y)
      this.bgCtx.restore()
      this.bgCtx.closePath()
    },

    drawLine ({ mouseX, mouseY }) {
      this.ctx.clearRect(0, 0, CANVAS_WIDTH, CANVAS_HIGHT)
      this.ctx.drawImage(this.$refs['bg-canvas'], 0, 0, CANVAS_WIDTH, CANVAS_HIGHT)
      this.drawMapLine()

      let startX, startY
      if (this.currentFlag === 'treeItem') {
        startX = this.currentPosition.x + CARD_WIDTH
        startY = this.currentPosition.y + CARD_HIGHT / 2
      } else {
        startX = this.currentPosition.x
        startY = this.currentPosition.y + CARD_HIGHT / 2
      }
      this.ctx.beginPath()
      this.ctx.save()
      this.ctx.moveTo(startX, startY)
      this.ctx.lineTo(mouseX, mouseY)
      this.ctx.strokeStyle = COLOR
      this.ctx.stroke()
      this.ctx.restore()
      this.ctx.closePath()
    },
    initMapWithStartEndPoint () {
      Object.entries(this.map).forEach(([key, value]) => {
        const keyDetail = this.treeToList.find(item => item.id === key)
        const valueDetailList = this.list.filter(item => value.includes(item.id))

        if (keyDetail && valueDetailList) {
          const startPoint = {
            x: keyDetail.position.x + keyDetail.position.w,
            y: keyDetail.position.y + keyDetail.position.h / 2
          }
          const endPointList = valueDetailList.map(item => {
            return {
              x: item.position.x,
              y: item.position.y + item.position.h / 2
            }
          })

          if (!this.mapWithStartAndEndPoint[key]) {
            this.mapWithStartAndEndPoint[key] = {
              startPoint,
              endPointList: [...endPointList],
              endId: value
            }
          } else {
            this.mapWithStartAndEndPoint[key].endPointList.push(...endPointList)
          }
        }
      })
    },

    drawMapLine () {
      Object.entries(this.mapWithStartAndEndPoint).forEach(([key, value]) => {
        const startPoint = value.startPoint
        value.endPointList.forEach(endPoint => {
          this.drawMapSingleLine(startPoint, endPoint)
        })
      })
    },

    drawMapSingleLine (start, end) {
      this.ctx.beginPath()
      this.ctx.save()
      this.ctx.moveTo(start.x, start.y)
      this.ctx.lineTo(end.x, end.y)
      this.ctx.strokeStyle = COLOR
      this.ctx.stroke()
      this.ctx.restore()
      this.ctx.closePath()
    },

    getMapHandler () {
      console.log(this.map)
    }

  }
}
</script>

<style scoped lang="less">
.home {
  display: flex;
  flex-direction: column;
  align-items: center;

  .image {
    position: relative;
    width: 640px;
    height: 360px;

    .canvas,
    .bg-canvas {
      position: absolute;
      width: 640px;
      height: 360px;
    }

    .canvas {
      z-index: 2;
    }
  }
}
</style>
