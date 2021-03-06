<template>
  <flexbox v-loading="loading"
           class="main-container"
           direction="column"
           align="stretch">
    <div class="handle-bar">
      <el-date-picker v-model="dateSelect"
                      type="year"
                      :clearable="false"
                      value-format="yyyy"
                      :picker-options="pickerOptions"
                      placeholder="选择年">
      </el-date-picker>
      <el-select v-model="structuresSelectValue"
                 @change="structuresValueChange"
                 placeholder="选择部门">
        <el-option v-for="item in deptList"
                   :key="item.id"
                   :label="item.name"
                   :value="item.id">
        </el-option>
      </el-select>
      <el-select v-model="userSelectValue"
                 :clearable="true"
                 placeholder="选择员工">
        <el-option v-for="item in userOptions"
                   :key="item.id"
                   :label="item.realname"
                   :value="item.id">
        </el-option>
      </el-select>
      <el-button @click.native="handleClick('search')"
                 type="primary">搜索</el-button>
    </div>
    <div class="content">
      <div class="axis-content">
        <div id="axismain"
             style="width: 850px;height:400px;"></div>
      </div>
      <div class="box"
           v-if="showType !== 0">
        <el-table :data="list"
                  :height="tableHeight"
                  stripe
                  @row-click="handleRowClick"
                  highlight-current-row>
          <el-table-column v-for="(item, index) in fieldList"
                           :key="index"
                           align="center"
                           header-align="center"
                           show-overflow-tooltip
                           :formatter="fieldFormatter"
                           :prop="item.prop"
                           :label="item.label">
          </el-table-column>
        </el-table>
      </div>
    </div>
    <contract-detail v-if="showContractDview"
                     :id="rowID"
                     append-to-body
                     @hide-view="showContractDview=false"
                     class="d-view"></contract-detail>
    <money-detail v-if="showMoneyDview"
                  :id="rowID"
                  append-to-body
                  @hide-view="showMoneyDview=false"
                  class="d-view"></money-detail>
  </flexbox>
</template>

<script>
import echarts from 'echarts'
import { adminStructuresSubIndex, getSubUserByStructrue } from '@/api/common'
import moment from 'moment'
import {
  biReceivablesStatistics,
  biReceivablesStatisticList
} from '@/api/businessIntelligence/bi'
import { filedGetField } from '@/api/customermanagement/common'
import { getDateFromTimestamp } from '@/utils'
import ContractDetail from '@/views/customermanagement/contract/ContractDetail'
import MoneyDetail from '@/views/customermanagement/money/MoneyDetail'

export default {
  /** 回款统计 */
  name: 'money-statistics',
  components: {
    ContractDetail,
    MoneyDetail
  },
  data() {
    return {
      pickerOptions: {
        disabledDate(time) {
          return time.getTime() > Date.now()
        }
      },
      loading: false,
      tableHeight: 400,

      dateSelect: '', // 选择的date
      /** 部门选择解析数据 */
      structuresProps: {
        children: 'children',
        label: 'label',
        value: 'id'
      },
      deptList: [], // 部门列表
      structuresSelectValue: '',
      /** 用户列表 */
      userOptions: [],
      userSelectValue: '',

      list: [],
      showType: 0, // 	1：合同 2:回款 0 不展示
      /** 格式化规则 */
      formatterRules: {},
      /** 合同列表 */
      fieldContractList: [],
      /** 回款字段列表 */
      fieldReceivablesList: [],
      /** 详情查看 */
      rowID: '', // 行信息
      showContractDview: false,
      showMoneyDview: false
    }
  },
  computed: {
    fieldList() {
      // 1：合同 2:回款 0 不展示
      if (this.showType == 1) {
        return this.fieldContractList
      } else if (this.showType == 2) {
        return this.fieldReceivablesList
      }
      return []
    }
  },
  mounted() {
    this.dateSelect = moment(new Date())
      .year()
      .toString()
    this.initAxis()

    this.getDeptList()
  },
  methods: {
    /**
     * 获取部门列表
     */
    getDeptList() {
      this.loading = true
      adminStructuresSubIndex()
        .then(res => {
          this.deptList = res.data
          if (res.data.length > 0) {
            this.structuresSelectValue = res.data[0].id
            this.getUserList() // 更新员工列表
            this.getAxisData()
          }
          this.loading = false
        })
        .catch(() => {
          this.loading = false
        })
    },
    /** 部门更改 */
    structuresValueChange(data) {
      this.userSelectValue = ''
      this.userOptions = []
      this.getUserList() // 更新员工列表
    },
    /** 部门下员工 */
    getUserList() {
      var params = {}
      params.structure_id = this.structuresSelectValue
      getSubUserByStructrue(params)
        .then(res => {
          this.userOptions = res.data
        })
        .catch(() => {})
    },
    /** 顶部操作 */
    handleClick(type) {
      if (type === 'search') {
        this.getAxisData()
      }
    },
    /** 获取部门业绩完成信息 */
    getAxisData() {
      this.loading = true
      this.showType = 0 // table 不展示
      biReceivablesStatistics({
        year: this.dateSelect,
        structure_id: this.structuresSelectValue,
        user_id: this.userSelectValue
      })
        .then(res => {
          var self = this
          var receivablesMoneys = []
          var contractMoneys = []
          for (let index = 1; index < 13; index++) {
            const element = res.data.charMonthArr[index]
            receivablesMoneys.push(element.receivablesMoney)
            contractMoneys.push(element.contractMoney)
            this.list.push(element)
          }

          this.axisOption.series[0].data = receivablesMoneys
          this.axisOption.series[1].data = contractMoneys
          this.axisChart.setOption(this.axisOption, true)

          this.loading = false
        })
        .catch(() => {
          this.loading = false
        })
    },
    /** 柱状图 */
    initAxis() {
      var axisChart = echarts.init(document.getElementById('axismain'))

      var option = {
        color: ['#6ca2ff', '#ff7474'],
        tooltip: {
          trigger: 'axis',
          axisPointer: {
            // 坐标轴指示器，坐标轴触发有效
            type: 'shadow' // 默认为直线，可选为：'line' | 'shadow'
          }
        },
        legend: {
          data: ['回款金额', '合同金额'],
          bottom: '0px',
          itemWidth: 14
        },
        grid: {
          top: '40px',
          left: '20px',
          right: '20px',
          bottom: '40px',
          containLabel: true,
          borderColor: '#fff'
        },
        xAxis: [
          {
            type: 'category',
            data: [
              '1月',
              '2月',
              '3月',
              '4月',
              '5月',
              '6月',
              '7月',
              '8月',
              '9月',
              '10月',
              '11月',
              '12月'
            ],
            axisTick: {
              alignWithLabel: true,
              lineStyle: { width: 0 }
            },
            axisLabel: {
              color: '#BDBDBD'
            },
            /** 坐标轴轴线相关设置 */
            axisLine: {
              lineStyle: { color: '#BDBDBD' }
            },
            splitLine: {
              show: false
            }
          }
        ],
        yAxis: [
          {
            type: 'value',
            name: '合同金额',
            axisTick: {
              alignWithLabel: true,
              lineStyle: { width: 0 }
            },
            axisLabel: {
              color: '#BDBDBD',
              formatter: '{value} 元'
            },
            /** 坐标轴轴线相关设置 */
            axisLine: {
              lineStyle: { color: '#BDBDBD' }
            },
            splitLine: {
              show: false
            }
          }
        ],
        series: [
          {
            name: '回款金额',
            type: 'bar',
            yAxisIndex: 0,
            data: []
          },
          {
            name: '合同金额',
            type: 'bar',
            yAxisIndex: 0,
            data: []
          }
        ]
      }

      axisChart.setOption(option, true)
      var self = this
      axisChart.on('click', function(params) {
        // 	1：合同 2:回款
        if (params.seriesIndex === 0) {
          // 回款金额
          if (self.fieldReceivablesList.length === 0) {
            self.getFieldList('receivables', () => {
              self.showType = 2
              self.getDataList(params.dataIndex + 1, 2)
            })
          } else {
            self.showType = 2
            self.getDataList(params.dataIndex + 1, 2)
          }
        } else if (params.seriesIndex === 1) {
          // 合同金额
          if (self.fieldContractList.length === 0) {
            self.getFieldList('contract', () => {
              self.showType = 1
              self.getDataList(params.dataIndex + 1, 1)
            })
          } else {
            self.showType = 1
            self.getDataList(params.dataIndex + 1, 1)
          }
        }
      })
      this.axisOption = option
      this.axisChart = axisChart
    },
    /** 获取列表信息 */
    getDataList(month, type) {
      this.loading = true
      biReceivablesStatisticList({
        year: this.dateSelect,
        structure_id: this.structuresSelectValue,
        month: month,
        type: type,
        user_id: this.userSelectValue
      })
        .then(res => {
          this.loading = false
          this.list = res.data
        })
        .catch(() => {
          this.loading = false
        })
    },
    /** 获取字段 */
    getFieldList(crmType, success) {
      this.loading = true
      filedGetField({
        types: 'crm_' + crmType,
        module: 'crm',
        controller: crmType,
        action: 'index'
      })
        .then(res => {
          for (let index = 0; index < res.data.length; index++) {
            const element = res.data[index]
            /** 获取需要格式化的字段 和格式化的规则 */
            if (element.form_type === 'date') {
              function fieldFormatter(time) {
                if (time == '0000-00-00') {
                  time = ''
                }
                return time
              }
              this.formatterRules[element.field] = {
                formatter: fieldFormatter
              }
            } else if (element.form_type === 'datetime') {
              function fieldFormatter(time) {
                if (time == 0 || !time) {
                  return ''
                }
                return moment(getDateFromTimestamp(time)).format(
                  'YYYY-MM-DD HH:mm:ss'
                )
              }
              this.formatterRules[element.field] = {
                formatter: fieldFormatter
              }
            } else if (element.field === 'create_user_id') {
              function fieldFormatter(info) {
                return info ? info.realname : ''
              }
              this.formatterRules[element.field] = {
                type: 'crm',
                formatter: fieldFormatter
              }
            } else if (element.form_type === 'user') {
              function fieldFormatter(info) {
                if (info) {
                  var content = ''
                  for (let index = 0; index < info.length; index++) {
                    const element = info[index]
                    content =
                      content +
                      element.realname +
                      (index === info.length - 1 ? '' : ',')
                  }
                  return content
                }
                return ''
              }
              this.formatterRules[element.field] = {
                type: 'crm',
                formatter: fieldFormatter
              }
            } else if (element.form_type === 'structure') {
              function fieldFormatter(info) {
                if (info) {
                  var content = ''
                  for (let index = 0; index < info.length; index++) {
                    const element = info[index]
                    content =
                      content +
                      element.name +
                      (index === info.length - 1 ? '' : ',')
                  }
                  return content
                }
                return ''
              }
              this.formatterRules[element.field] = {
                type: 'crm',
                formatter: fieldFormatter
              }
              /** 联系人 客户 商机 */
            } else if (
              element.field === 'contacts_id' ||
              element.field === 'customer_id' ||
              element.field === 'business_id' ||
              element.field === 'contract_id'
            ) {
              function fieldFormatter(info) {
                return info ? info.name : ''
              }
              this.formatterRules[element.field] = {
                type: 'crm',
                formatter: fieldFormatter
              }
            } else if (
              element.field === 'status_id' ||
              element.field === 'type_id' ||
              element.field === 'category_id'
            ) {
              function fieldFormatter(info) {
                return info ? info : ''
              }
              this.formatterRules[element.field] = {
                type: 'crm',
                formatter: fieldFormatter
              }
            }

            var width = 0
            if (!element.width) {
              if (element.name.length <= 6) {
                width = element.name.length * 15 + 45
              } else {
                width = 140
              }
            } else {
              width = element.width
            }
            if (crmType === 'contract') {
              this.fieldContractList.push({
                prop: element.field,
                label: element.name,
                width: width
              })
            } else if (crmType === 'receivables') {
              this.fieldReceivablesList.push({
                prop: element.field,
                label: element.name,
                width: width
              })
            }
          }
          // 获取好字段开始请求数据
          success() // 回调成功
        })
        .catch(() => {
          this.loading = false
        })
    },
    /** 格式化字段 */
    fieldFormatter(row, column) {
      // 如果需要格式化
      var aRules = this.formatterRules[column.property]
      if (aRules) {
        if (aRules.type === 'crm') {
          if (column.property) {
            return aRules.formatter(row[column.property + '_info'])
          } else {
            return ''
          }
        } else {
          return aRules.formatter(row[column.property])
        }
      }
      return row[column.property]
    },
    /** 查看详情 */
    handleRowClick(row, column, event) {
      // 	1：合同 2:回款
      if (this.showType == 1) {
        this.rowID = row.contract_id
        this.showContractDview = true
      } else if (this.showType == 2) {
        this.rowID = row.receivables_id
        this.showMoneyDview = true
      }
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
@import './styles/detail.scss';
@import '@/views/customermanagement/styles/detailview.scss';

.handle-bar {
  padding: 15px 20px 5px 20px;
  .el-date-editor {
    width: 130px;
    margin-right: 15px;
  }
  .el-cascader {
    width: 130px;
    margin-right: 15px;
  }
  .el-select {
    width: 120px;
    margin-right: 15px;
  }
}
.axis-content {
  padding-left: 50px;
  padding: 20px 0 40px 0;
}

.content {
  flex: 1;
  overflow-y: auto;
  padding: 10px 20px;
  .box {
    border: 1px solid #e6e6e6;
  }
}
.content /deep/ .el-table {
}

.detail-bar {
  padding: 10px 20px;
  font-size: 14px;
  color: #3e84e9;
  background-color: #fff3e8;
  div {
    flex: 1;
  }
  div:first-child {
    flex: 5;
  }
}
</style>
