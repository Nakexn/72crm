<template>
  <div v-loading="loading"
       class="main-container">
    <filtrate-handle-view class="filtrate-bar"
                          :showUserSelect="false"
                          moduleType="ranking"
                          @load="loading=true"
                          @change="getDataList">
    </filtrate-handle-view>
    <div class="content">
      <div class="content-title">回款金额排行（按实际回款时间）</div>
      <div class="axis-content"
           v-empty="list.length === 0"
           xs-empty-text="暂无排行">
        <div id="axismain"></div>
      </div>
      <div class="table-content">
        <el-table :data="list"
                  max-height="400"
                  stripe
                  border
                  highlight-current-row>
          <el-table-column align="center"
                           header-align="center"
                           show-overflow-tooltip
                           label="公司总排名">
            <template slot-scope="scope">
              {{scope.$index + 1}}
            </template>
          </el-table-column>
          <el-table-column v-for="(item, index) in fieldList"
                           :key="index"
                           align="center"
                           header-align="center"
                           show-overflow-tooltip
                           :prop="item.field"
                           :label="item.name">
          </el-table-column>
        </el-table>
      </div>
    </div>
  </div>
</template>

<script>
import rankingMixins from '../mixins/ranking'
import echarts from 'echarts'
import { biRankingReceivablesAPI } from '@/api/businessIntelligence/ranking'

export default {
  /** 回款金额排行 */
  name: 'ranking-receivables-statistics',
  data() {
    return {}
  },
  mixins: [rankingMixins],
  computed: {},
  mounted() {
    this.fieldList = [
      { field: 'user_name', name: '签订人' },
      { field: 'structure_name', name: '归属人' },
      { field: 'money', name: '回款金额（元）' }
    ]
    this.initAxis()
  },
  methods: {
    getDataList(params) {
      this.loading = true
      biRankingReceivablesAPI(params)
        .then(res => {
          this.loading = false
          this.list = res.data || []

          let showData = []
          let yAxis = []

          let rankingIndex = res.data.length > 10 ? 10 : res.data.length
          for (let index = 0; index < rankingIndex; index++) {
            const element = res.data[index]
            showData.splice(0, 0, parseFloat(element.money))
            yAxis.splice(0, 0, element.user_name)
          }
          this.axisOption.yAxis[0].data = yAxis
          this.axisOption.series[0].data = showData
          this.axisChart.setOption(this.axisOption, true)
        })
        .catch(() => {
          this.loading = false
        })
    },
    /** 柱状图 */
    initAxis() {
      this.axisChart = echarts.init(document.getElementById('axismain'))
      this.axisChart.setOption(this.axisOption, true)
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
@import '../styles/detail.scss';
</style>
