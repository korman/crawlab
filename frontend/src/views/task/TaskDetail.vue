<template>
  <div class="app-container">
    <!--tour-->
    <v-tour
      name="task-detail"
      :steps="tourSteps"
      :callbacks="tourCallbacks"
      :options="$utils.tour.getOptions(true)"
    />
    <!--./tour-->

    <!--tabs-->
    <el-tabs v-model="activeTabName" @tab-click="onTabClick" type="border-card">
      <el-tab-pane :label="$t('Overview')" name="overview">
        <task-overview @click-log="activeTabName = 'log'"/>
      </el-tab-pane>
      <el-tab-pane :label="$t('Log')" name="log">
        <log-view/>
      </el-tab-pane>
      <el-tab-pane :label="$t('Results')" name="results">
        <div class="button-group">
          <el-button size="small" class="btn-download" type="primary" icon="el-icon-download" @click="downloadCSV">
            {{$t('Download CSV')}}
          </el-button>
        </div>
        <general-table-view :data="taskResultsData"
                            :columns="taskResultsColumns"
                            :page-num="resultsPageNum"
                            :page-size="resultsPageSize"
                            :total="taskResultsTotalCount"
                            @page-change="onResultsPageChange"/>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script>
import {
  mapState,
  mapGetters
} from 'vuex'
import TaskOverview from '../../components/Overview/TaskOverview'
import GeneralTableView from '../../components/TableView/GeneralTableView'
import LogView from '../../components/ScrollView/LogView'

export default {
  name: 'TaskDetail',
  components: {
    LogView,
    GeneralTableView,
    TaskOverview
  },
  data () {
    return {
      activeTabName: 'overview',
      handle: undefined,

      // tutorial
      tourSteps: [
        // overview
        {
          target: '.task-info-overview-wrapper',
          content: this.$t('This is the info of the task detail.'),
          params: {
            placement: 'right'
          }
        },
        {
          target: '.task-info-spider-wrapper',
          content: this.$t('This is the spider info of the task.'),
          params: {
            placement: 'left'
          }
        },
        {
          target: '.spider-title',
          content: this.$t('You can click to view the spider detail for the task.'),
          params: {
            placement: 'left'
          }
        },
        {
          target: '.task-info-node-wrapper',
          content: this.$t('This is the node info of the task.'),
          params: {
            placement: 'left'
          }
        },
        {
          target: '.node-title',
          content: this.$t('You can click to view the node detail for the task.'),
          params: {
            placement: 'left'
          }
        },
        // log
        {
          target: '#tab-log',
          content: this.$t('Here you can view the log<br> details for the task. The<br> log is automatically updated.')
        },
        // results
        {
          target: '#tab-results',
          content: this.$t('Here you can view the results scraped by the spider.<br><br><strong>Note:</strong> If you find your results here are empty, please refer to the <a href="https://docs.crawlab.cn/Integration/" target="_blank" style="color: #409EFF">Documentation (Chinese)</a> about how to integrate your spider into Crawlab.')
        },
        {
          target: '.btn-download',
          content: this.$t('You can download your results as a CSV file by clicking this button.')
        }
      ],
      tourCallbacks: {
        onStop: () => {
          this.$utils.tour.finishTour('task-detail')
        },
        onPreviousStep: (currentStep) => {
          if (currentStep === 5) {
            this.activeTabName = 'overview'
          } else if (currentStep === 6) {
            this.activeTabName = 'log'
          }
          this.$utils.tour.prevStep('task-detail', currentStep)
        },
        onNextStep: (currentStep) => {
          if (currentStep === 4) {
            this.activeTabName = 'log'
          } else if (currentStep === 5) {
            this.activeTabName = 'results'
          }
          this.$utils.tour.nextStep('task-detail', currentStep)
        }
      }
    }
  },
  computed: {
    ...mapState('task', [
      'taskForm',
      'taskResultsData',
      'taskResultsTotalCount',
      'taskLog'
    ]),
    ...mapGetters('task', [
      'taskResultsColumns'
    ]),
    ...mapState('file', [
      'currentPath'
    ]),
    ...mapState('deploy', [
      'deployList'
    ]),
    resultsPageNum: {
      get () {
        return this.$store.state.task.resultsPageNum
      },
      set (value) {
        this.$store.commit('task/SET_RESULTS_PAGE_NUM', value)
      }
    },
    resultsPageSize: {
      get () {
        return this.$store.state.task.resultsPageSize
      },
      set (value) {
        this.$store.commit('task/SET_RESULTS_PAGE_SIZE', value)
      }
    },
    isRunning () {
      return ['pending', 'running'].includes(this.taskForm.status)
    }
  },
  methods: {
    onTabClick (tab) {
      this.$st.sendEv('任务详情', '切换标签', tab.name)
    },
    onSpiderChange (id) {
      this.$router.push(`/spiders/${id}`)
    },
    onResultsPageChange (payload) {
      const { pageNum, pageSize } = payload
      this.resultsPageNum = pageNum
      this.resultsPageSize = pageSize
      this.$store.dispatch('task/getTaskResults', this.$route.params.id)
    },
    downloadCSV () {
      this.$store.dispatch('task/getTaskResultExcel', this.$route.params.id)
      this.$st.sendEv('任务详情', '结果', '下载CSV')
    },
    getTaskLog () {
      this.$store.dispatch('task/getTaskLog', this.$route.params.id)
    }
  },
  created () {
    this.$store.dispatch('task/getTaskData', this.$route.params.id)
    this.$store.dispatch('task/getTaskResults', this.$route.params.id)

    this.getTaskLog()
    this.handle = setInterval(() => {
      if (!this.isRunning) return
      this.$store.dispatch('task/getTaskData', this.$route.params.id)
      this.$store.dispatch('task/getTaskResults', this.$route.params.id)
      this.getTaskLog()
    }, 5000)
  },
  mounted () {
    if (!this.$utils.tour.isFinishedTour('task-detail')) {
      this.$utils.tour.startTour(this, 'task-detail')
    }
  },
  destroyed () {
    clearInterval(this.handle)
  }
}
</script>

<style scoped>

  .selector {
    display: flex;
    align-items: center;
    position: absolute;
    right: 20px;
    /*float: right;*/
    z-index: 999;
    margin-top: -7px;
  }

  .selector .el-select {
    padding-left: 10px;
  }

  .log-view {
    margin: 20px;
    height: 640px;
  }

  .log-view pre {
    height: 100%;
    overflow-x: auto;
    overflow-y: auto;
  }

  .button-group {
    margin-bottom: 10px;
    text-align: right;
  }
</style>
