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
          <q-select v-model="timeLimitModel" :options="timeLimitOptions" label="Time Limit 搜尋文章的時間範圍" />
          <q-select v-model="concurrencyModel" :options="concurrencyOptions" label="Concurrency 並發數 注：數值越高越耗費系統資源" />
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
          wrap-cells
        />
      </div>
    </div>
  </q-page>
</template>

<script>
export default {
  data () {
    return {
      keywords: '',
      isFetcherListAllSelect: false,
      isFetcherListExpanded: true,
      fetcherListAccepted: [],
      fetcherListOptions: [],
      timeLimitModel: 'Any',
      timeLimitOptions: [
        'Any', 'Day', 'Week', 'Year'
      ],
      concurrencyModel: 1,
      concurrencyOptions: [
        1, 2, 3, 4, 5
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
      workingJobQueue: [],
      rowCnt: 0
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
            timeLimit: this.timeLimitModel
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
              // let cnt = 0
              for (const row of res.data) {
                // if (cnt++ > 2) break
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
      }, 6000)
    },
    jobHandler () {
      console.info('[method][getStatus]')
      if (this.isFetchJobStarted) {
        const tempQueue = []
        while (this.workingJobQueue.length > 0) {
          const job = this.workingJobQueue.shift()
          switch (job.status) {
            case 'waiting':
              this.postJob(job['link-href'], job.newsName)
              job.status = 'running'
              tempQueue.push(job)
              break
            case 'running':
              job.status = 'checking'
              this.checkStatus(job)
              tempQueue.push(job)
              break
            case 'checking':
              tempQueue.push(job)
              break
            default:
              break
          }
        }
        while (tempQueue.length > 0) {
          this.workingJobQueue.push(tempQueue.pop())
        }
      }
    },
    async postJob (url, newsName) {
      await this.$http.post('/fetchJob', {
        url: url,
        newsName: newsName
      })
        .then(res => {
          if (res.data.code === -1) {
            console.error(res.data.data.msg)
          }
        })
    },
    async checkStatus (job) {
      await this.$http.get('/statusJob', {
        params: {
          url: job['link-href']
        }
      })
        .then(res => {
          // 若得到返回值為完成或失敗，則更新running隊列
          if (res.data.status === 'completed' || res.data.status === 'failed') {
            job.status = res.data.status
          } else {
            job.status = 'running'
          }
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
