<template>
  <q-page class="">
    <div class="" style="max-width: 900px; margin-left: auto; margin-right: auto">
      <div style="height: 50px">
      </div>
      <div class="q-ma-md">
        <q-form
          @submit="onSubmit"
          class="q-gutter-md"
        >
          <div class="">
            <q-expansion-item
              expand-separator
              v-model="isFetcherListExpanded"
              label="Fetcher List 請選擇你想搜尋的站點"
            >
              <q-card>
                <q-card-section>
                  <q-checkbox v-model="isFetcherListAllSelect" @input="fetcherListAllSelect" label="All 全選" />
                  <q-option-group
                    name="accepted_genres"
                    v-model="fetcherListAccepted"
                    :options="fetcherListOptions"
                    type="checkbox"
                    color="primary"
                    inline
                  />
                </q-card-section>
              </q-card>
            </q-expansion-item>
          </div>
          <q-input
            v-model="keywords"
            label="Keyword 關鍵字"
            placeholder=""
          >
            <template v-slot:append v-if="keywords.length !== 0">
              <q-icon name="close" @click="keywords = ''" class="cursor-pointer" />
            </template>
          </q-input>
            <q-input
              v-model="resultLimit"
              label="Search engine result limit 搜尋結果數限制"
              placeholder=""
            > </q-input>
          <q-select v-model="timeLimitModel" :options="timeLimitOptions" label="Time Limit 搜尋文章的時間範圍" />
          <q-select v-model="engineModel" :options="engineOptions" label="Search Engine" />
          <q-select v-model="concurrencyModel" :options="concurrencyOptions" label="Parallel Jobs 並行工作數 注：數值越高越耗費系統資源" />
          <div>
            <q-btn label="Submit" type="submit" color="primary" :disable="isSubmitDisabled"/>
            <q-ajax-bar
              ref="bar"
              position="bottom"
              color="primary"
              size="10px"
              skip-hijack
            />
          </div>
        </q-form>

      </div>
      <div class="q-ma-md" v-if="isResultUrlsShow">
        <q-table
          title="Result Urls"
          :data="resultUrlsData"
          :columns="resultUrlsColumns"
          row-key="link-href"
          :rows-per-page-options="[0]"
        >
          <template v-slot:top-right>
            <q-btn
              color="primary"
              icon-right="archive"
              label="Export to csv"
              no-caps
              @click="exportTable"
            />
          </template>
        </q-table>
      </div>
    </div>
  </q-page>
</template>

<script>
import { exportFile } from 'quasar'
// 格式化Csv值
const wrapCsvValue = (val, formatFn) => {
  let formatted = formatFn !== undefined ? formatFn(val) : val
  formatted = formatted === undefined || formatted === null ? '' : String(formatted)
  formatted = formatted.split('"').join('""')
  return `"${formatted}"`
}
export default {
  data () {
    return {
      keywords: '',
      resultLimit: 100,
      isFetcherListAllSelect: false,
      isFetcherListExpanded: true,
      fetcherListAccepted: [],
      fetcherListOptions: [],
      timeLimitModel: 'Any',
      timeLimitOptions: [
        'Any', 'Day', 'Week', 'Month', 'Year'
      ],
      concurrencyModel: 1,
      concurrencyOptions: [
        1, 2, 3, 5, 8, 10, 15, 20, 25, 30
      ],
      engineModel: 'bing',
      engineOptions: [
        'duckduckgo', 'bing'
      ],
      isSubmitDisabled: false,
      isResultUrlsShow: false,
      resultUrlsData: [],
      resultUrlsColumns: [
        {
          name: 'status',
          label: 'Status',
          align: 'left',
          field: 'status',
          sortable: true
        },
        {
          name: 'name',
          label: 'Title',
          align: 'left',
          field: 'link',
          sortable: true
        },
        {
          name: 'snippet',
          label: 'Snippet',
          align: 'left',
          field: 'snippet',
          sortable: true
        },
        {
          name: 'url',
          label: 'Url',
          align: 'center',
          field: 'link-href',
          sortable: false
        }
      ],
      isFetchJobStarted: false,
      fetchJobQueue: [],
      workingJobQueue: []
    }
  },
  methods: {
    // 單擊 Submit 提交時
    async onSubmit () {
      console.info('[methods][onSubmit]')
      const bar = this.$refs.bar
      this.isSubmitDisabled = true
      for (const oneNews of this.fetcherListAccepted) {
        bar.start()
        await this.$http.get('/urlLists', {
          params: {
            news: oneNews,
            keyword: this.keywords,
            timeLimit: this.timeLimitModel,
            resultLimit: this.resultLimit,
            engine: this.engineModel
          }
        })
          .then(res => {
            bar.stop()
            if (res.data === '[]') {
              console.error('error')
            } else {
              this.isFetcherListExpanded = false
              this.isResultUrlsShow = true
              // 限制結果的debug option
              let cnt = 0
              for (const row of res.data) {
                if (cnt++ >= this.resultLimit) break
                const temp = {
                  ...row,
                  status: 'waiting',
                  newsName: oneNews
                }
                this.fetchJobQueue.push(temp)
                this.resultUrlsData.push(temp)
              }
            }
          })
      }
      this.isSubmitDisabled = false
      // 自動開始進行fetch工作
      if (this.resultUrlsData.length > 0) {
        if (!this.isFetchJobStarted) {
          this.fetchJob()
        }
      }
    },
    // Fetcher List 全選操作
    fetcherListAllSelect (value, evt) {
      console.info('[methods][fetcherListAllSelect]')
      this.isfetcherListAllSelect = value
      const allList = []
      for (const item of this.fetcherListOptions) {
        allList.push(item.value)
      }
      if (this.isfetcherListAllSelect) {
        this.fetcherListAccepted = allList
      } else {
        this.fetcherListAccepted = []
      }
    },
    // 導出函數
    exportTable () {
      console.info('[methods][exportTable]')
      const content = [this.resultUrlsColumns.map(col => wrapCsvValue(col.label))].concat(
        this.resultUrlsData.map(row => this.resultUrlsColumns.map(col => wrapCsvValue(
          typeof col.field === 'function'
            ? col.field(row)
            : row[col.field === undefined ? col.name : col.field],
          col.format
        )).join(','))
      ).join('\r\n')

      const today = new Date()
      const nowDateTime = `${today.getFullYear()}${today.getMonth()}${today.getDate()}-${today.getHours()}${today.getMinutes()}${today.getSeconds()}`

      const status = exportFile(
        `news-crawler-home-export-${nowDateTime}.csv`,
        content,
        'text/csv'
      )

      if (status !== true) {
        this.$q.notify({
          message: 'Browser denied file download...',
          color: 'negative',
          icon: 'warning'
        })
      }
    },
    // Fetch Job 任務分發
    async fetchJob () {
      console.info('[methods][fetchJob]')
      this.isFetchJobStarted = true
      this.dispatcher = setInterval(() => {
        console.info('[method][pushingWorkingQueue]')
        while (this.workingJobQueue.length < this.concurrencyModel && this.fetchJobQueue.length > 0 && this.isFetchJobStarted) {
          this.workingJobQueue.push(this.fetchJobQueue.shift())
        }
        if (this.fetchJobQueue.length <= 0 && this.workingJobQueue.length <= 0) {
          console.info('[method][fetchJob]: Fetching Done!')
          this.isFetchJobStarted = false
          clearInterval(this.dispatcher)
        }
        setTimeout(this.jobHandler, 3000)
      }, 5000)
    },
    async jobHandler () {
      console.info('[method][getStatus&postJob]')
      if (this.isFetchJobStarted) {
        const readyToPost = []
        for (const working of this.workingJobQueue) {
          if (working.status === 'waiting') {
            working.status = 'running'
            readyToPost.push(working)
          }
        }
        await this.postJob(readyToPost)
        if (this.workingJobQueue.length > 0) {
          this.checkStatus(this.workingJobQueue)
        }
      }
    },
    async postJob (jobs) {
      const request = []
      for (const job of jobs) {
        request.push(job)
      }
      if (request.length > 0) {
        await this.$http.post('/fetchJob', {
          jobs: request
        })
          .then(res => {
            if (res.data.code === -1) {
              console.error(res.data.data.msg)
            }
          })
      }
    },
    async checkStatus (job) {
      const query = []
      for (const each of job) {
        query.push(each['link-href'])
        each.status = 'checking'
      }
      await this.$http.get('/statusJob', {
        params: {
          url: query
        }
      })
        .then(res => {
          // 若得到返回值為完成或失敗，則更新running隊列
          const data = res.data
          for (const each of job) {
            const eachStatus = data[each['link-href']]
            const status = eachStatus.status
            if (!status) {
              continue
            }
            if (status === 'completed' || status === 'failed') {
              each.status = status
            } else if (status === 'undefined') {
              each.status = 'failed'
            } else {
              each.status = 'running'
            }
          }
          this.workingJobQueue = this.workingJobQueue.filter((ele, _index, _arr) => {
            return !(ele.status === 'completed' || ele.status === 'failed')
          })
        })
    }
  },
  mounted () {
    // 仅调试时启用
    window.vue = this

    const bar = this.$refs.bar
    bar.start()
    this.$http.get('/function/listAllNews', {
      params: {
      }
    })
      .then(res => {
        if (res.data.code === -1) {
          console.error(res.data.data.msg)
        } else {
          bar.stop()
          this.fetcherListOptions = res.data
        }
      })
  }
}
</script>
