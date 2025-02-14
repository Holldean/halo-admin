<template>
  <a-card
    title="流量统计"
    :loading="loading"
    :body-style="{ padding: '20px 16px' }"
    :bordered="false"
  >
    <a-dropdown slot="extra">
      <a-menu
        slot="overlay"
        @click="handleOptionClick"
      >
        <a-menu-item key="1">近七天</a-menu-item>
        <a-menu-item key="2">近三十天</a-menu-item>
        <a-menu-item key="3">近一年</a-menu-item>
        <a-menu-item key="4">访问记录</a-menu-item>
      </a-menu>
      <a-icon type="ellipsis" />
    </a-dropdown>
    <div
      id="container"
      style="height: 308px; width: 100%;"
    ></div>
  </a-card>
</template>
<script>
import elementResizeDetectorMaker from 'element-resize-detector'
import { Chart } from '@antv/g2'

import visitLogApi from '@/api/visitorLog'

const ONE_DAY = 1000 * 60 * 60 * 24
const WEEKDAY_ABBR = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat']
const MONTH_ABBR = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sept', 'Oct', 'Nov', 'Dec']
const VIEW_TYPE = {
  WEEK: 1,
  MONTH: 2,
  YEAR: 3
}

export default {
  name: 'VisitChart',
  props: {
    loading: {
      type: Boolean,
      required: false,
      default: false
    },
    visitCountToday: {
      type: Object,
      required: false,
      default: null
    },
    visitCountCurrentMonth: {
      type: Object,
      required: false,
      default: null
    }
  },
  data() {
    return {
      visitData: [],
      chart: null,
      current: null,
      weekData: [],
      monthData: [],
      yearData: [],
      dateString: [],
      view: VIEW_TYPE.WEEK
    }
  },
  watch: {
    visitCountToday(newVal, oldVal) {
      const date = new Date(newVal.date)
      // if current date is not same with server's current date (e.g. after 24PM)
      if (date.getTime() !== this.current.getTime()) {
        // update current date and reload the chart data
        this.current = new Date()
        this.current.setHours(0, 0, 0, 0)
        this.initData()
        this.loadData()
      } else {
        // if the value sent from server is different from current chartdata, update it
        if (this.view !== VIEW_TYPE.YEAR && this.chartData[this.chartData.length - 1].visit !== newVal.count) {
          this.chartData[this.chartData.length - 1].visit = newVal.count
          this.chart.changeData(this.chartData)
        }
      }
    },
    visitCountCurrentMonth(newVal, oldVal) {
      if (this.current.getMonth() + 1 !== newVal.month) {
        this.current = new Date()
        this.current.setHours(0, 0, 0, 0)
        this.initData()
        this.loadData()
      } else {
        if (this.view === VIEW_TYPE.YEAR && this.chartData[this.chartData.length - 1].visit !== newVal.count) {
          this.chartData[this.chartData.length - 1].visit = newVal.count
          this.chart.changeData(this.chartData)
        }
      }
    }
  },
  computed: {
    chartData() {
      switch (this.view) {
        case VIEW_TYPE.WEEK:
          return this.weekData
        case VIEW_TYPE.MONTH:
          return this.monthData
        case VIEW_TYPE.YEAR:
          return this.yearData
        default:
          return []
      }
    }
  },
  created() {
    this.current = new Date()
    this.current.setHours(0, 0, 0, 0)
    this.initData()
  },
  mounted() {
    this.chart = new Chart({
      container: 'container',
      autoFit: true,
      height: 308
    })
    this.chart
      .data(this.chartData)
      .scale({
        date: {
          nice: true
        },
        visit: {
          alias: '访问量',
          nice: true,
          min: 0
        }
      })
      .axis('visit', {
        label: {
          formatter: val => {
            return val
          }
        }
      })
      .tooltip({
        shared: true,
        showCrosshairs: true,
        title: 'dateString'
      })
    this.chart
      .line()
      .position('date*visit')
    this.chart
      .point()
      .position('date*visit')
    const that = this
    const erd = elementResizeDetectorMaker()
    erd.listenTo(document.querySelector('#container'), (element) => {
      that.$nextTick(() => {
        const e = document.createEvent('Event')
        e.initEvent('resize', true, true)
        window.dispatchEvent(e)
      })
    })
    this.chart.render()
    this.loadData()
  },
  methods: {
    handleOptionClick(event) {
      if (event.key === '4') {
        this.$emit('showLog')
      } else {
        if (event.key === '1') {
          this.view = VIEW_TYPE.WEEK
          this.weekData = []
          this.initWeekData()
        } else if (event.key === '2') {
          this.view = VIEW_TYPE.MONTH
          this.monthData = []
          this.initMonthData()
        } else if (event.key === '3') {
          this.view = VIEW_TYPE.YEAR
          this.yearData = []
          this.initYearData()
        }
        this.loadData()
      }
    },
    initData() {
      this.weekData = []
      this.monthData = []
      this.yearData = []
      this.initDateString()
      this.initWeekData()
      this.initMonthData()
      this.initYearData()
    },
    initDateString() {
      const current = new Date()
      for (let i = 0; i < 30; i++) {
        const year = current.getFullYear()
        const month = current.getMonth() + 1
        const day = current.getDate()
        const dateString =
          `${year}-${month < 10 ? '0' : ''}${month}-${day < 10 ? '0' : ''}${day}`
        this.dateString.push(dateString)
        current.setDate(current.getDate() - 1)
      }
    },
    initWeekData() {
      let start = this.current.getDay()
      const current = new Date()
      for (let i = 0; i < 7; i++) {
        start = (start + 1) % 7
        this.weekData.push({
          date: WEEKDAY_ABBR[start],
          dateString: this.dateString[6 - i],
          visit: 0
        })
        current.setDate(current.getDate() - 1)
      }
    },
    initMonthData() {
      for (let i = 0; i < 30; i++) {
        this.monthData.push({
          date: this.dateString[29 - i],
          dateString: this.dateString[29 - i],
          visit: 0
        })
      }
    },
    initYearData() {
      let start = this.current.getMonth()
      const current = new Date()
      current.setMonth(current.getMonth() - 11)
      for (let i = 0; i < 12; i++) {
        start = (start + 1) % 12
        const year = current.getFullYear()
        const month = current.getMonth() + 1
        this.yearData.push({
          date: MONTH_ABBR[start],
          dateString: `${year}-${month < 10 ? '0' : ''}${month}`,
          visit: 0
        })
        current.setMonth(current.getMonth() + 1)
      }
    },
    loadData() {
      if (this.view === VIEW_TYPE.WEEK) {
        this.loadDataByDay(7)
      } else if (this.view === VIEW_TYPE.MONTH) {
        this.loadDataByDay(30)
      } else {
        this.loadDataByMonth()
      }
    },
    loadDataByDay(interval) {
      visitLogApi.getVisitsByDay(interval - 1).then(reponse => {
        const visitCount = reponse.data.data
        for (const data of visitCount) {
          const date = new Date(data.date)
          let diff = Math.round(this.current.getTime() - date.getTime()) / (ONE_DAY)
          diff = diff.toFixed(0)
          this.chartData[interval - 1 - diff].visit = data.count
        }
        this.chart.data(this.chartData)
        this.chart.render()
      })
    },
    loadDataByMonth() {
      visitLogApi.getVisitsByMonth(11).then(reponse => {
        const visitCount = reponse.data.data
        const month = this.current.getMonth() + 1
        for (const data of visitCount) {
          let diff = 0
          if (data.month > month) {
            diff = month + 12 - data.month
          } else {
            diff = month - data.month
          }
          this.chartData[11 - diff].visit = data.count
        }
        this.chart.data(this.chartData)
        this.chart.render()
      })
    }
  }
}
</script>
