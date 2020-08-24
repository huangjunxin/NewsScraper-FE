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
          <q-select v-model="timeLimitModel" :options="timeLimitOptions" label="Time Limit 搜尋結果時間限制" />
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
      timeLimitModel: null,
      timeLimitOptions: [
        'Any', 'Day', 'Week', 'Year'
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
        }
      ],
      isFetchJobStarted: false,
      fetchJobQueue: {}
    }
  },
  methods: {
    // 單擊 Submit 提交時
    async onSubmit () {
      console.info('[methods][onSubmit]')
      const bar = this.$refs.bar
      bar.start()
      this.isSubmitDisabled = true
      for (const oneNews of this.fetcherListAccepted) {
        await this.$http.get('/urlLists', {
          params: {
            news: oneNews,
            keyword: this.keywords,
            timeLimit: this.timeLimitModel
          }
        })
          .then(res => {
            if (res.data === '[]') {
              console.error('error')
            } else {
              bar.stop()
              this.isFetcherListExpanded = false
              this.isSubmitDisabled = false
              this.isResultUrlsShow = true
              for (const row of res.data) {
                const temp = {
                  ...row,
                  status: 'waiting',
                  newsName: oneNews
                }
                this.resultUrlsData.push(temp)
              }
            }
          })
      }
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
      while (this.isFetchJobStarted) {
        // 當任務隊列數量少於5時，新增一個分發任務
        if (Object.keys(this.fetchJobQueue).length < 5) {
          for (const row of this.resultUrlsData) {
            if (row.status === 'waiting') {
              // 分發任務
              await this.$http.post('/fetchJob', JSON.stringify({
                url: row['link-href'],
                newsName: row.newsName
              }))
                .then(res => {
                  if (res.data.code === -1) {
                    console.error(res.data.data.msg)
                  } else {
                    // 更改當前任務在列表的顯示狀態
                    row.status = 'running'
                    this.fetchJobQueue[row['link-href']] = row.status
                  }
                })
            }
          }
        } else if (Object.keys(this.fetchJobQueue).length === 5) {
          // 當任務隊列數量等於5時，每隔一秒發送請求更新狀態
          for (const item in this.fetchJobQueue) {
            await this.$http.get('/statusJob', {
              params: {
                url: item
              }
            })
              .then(res => {
                if (res.data.code === -1) {
                  console.error(res.data.data.msg)
                } else {
                  if (res.status === 'completed') {
                    delete this.fetchJobQueue[item]
                    for (const row of this.resultUrlsData) {
                      if (row.url === item.toString()) {
                        row.status = res.status
                      }
                    }
                  }
                }
              })
          }
          // 等待一秒
          setTimeout(() => {
            console.log('Wait a second')
          }, 1000)
        }
      }
    }
  },
  mounted () {
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
