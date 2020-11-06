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
          <q-input
            v-model="nameNews"
            label="News Name 報刊名稱"
            placeholder=""
          >
            <template v-slot:append v-if="nameNews.length !== 0">
              <q-icon name="close" @click="nameNews = ''" class="cursor-pointer" />
            </template>
          </q-input>
          <q-input
            v-model="title"
            label="Title 報道標題 可使用正則表達式「Regex Available」"
            placeholder=""
          >
            <template v-slot:append v-if="title.length !== 0">
              <q-icon name="close" @click="title = ''" class="cursor-pointer" />
            </template>
          </q-input>
          <q-input
            v-model="content"
            label="Content 報道正文 可使用正則表達式「Regex Available」"
            placeholder=""
          >
            <template v-slot:append v-if="content.length !== 0">
              <q-icon name="close" @click="content = ''" class="cursor-pointer" />
            </template>
          </q-input>
          <q-input
            v-model="locale"
            label="Locale 報道區域"
            placeholder=""
          >
            <template v-slot:append v-if="locale.length !== 0">
              <q-icon name="close" @click="locale = ''" class="cursor-pointer" />
            </template>
          </q-input>
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
      <div class="q-ma-md" v-if="isResultShow">
        <q-table
          title="Result Data"
          dense
          :data="resultData"
          :columns="resultColumns"
          :visible-columns="visibleColumns"
          row-key="link-href"
        >
          <template v-slot:top-right>
            <div class="row">
              <div class="col">
                <q-item-label header>Select Output / Display columns Below</q-item-label>
                <q-toggle v-model="visibleColumns" val="date" label="Date" />
                <q-toggle v-model="visibleColumns" val="name" label="News Name" />
                <q-toggle v-model="visibleColumns" val="locale" label="Locale" />
                <q-toggle v-model="visibleColumns" val="description" label="Description" />
                <q-toggle v-model="visibleColumns" val="title" label="Title" />
                <q-toggle v-model="visibleColumns" val="content" label="Content" />
                <q-toggle v-model="visibleColumns" val="url" label="Url" />
              </div>
              <div class="col-5">
                <q-item-label header class="q-gutter-xs">Select Output Type</q-item-label>
                <q-btn-toggle
                  v-model="outputType"
                  no-caps
                  rounded
                  toggle-color="green"
                  color="white"
                  text-color="primary"
                  :options="[
                    {label: 'CSV', value: 'csv'},
                    {label: 'TEXT', value: 'txt'}
                  ]"
                />
                <div class="q-gutter-lg">
                  <q-input
                    v-model="outputlimit"
                    label="導出條目限制 if 0 output all"
                    placeholder="0"
                  />
                  <q-btn
                    color="primary"
                    icon-right="archive"
                    label="Export to csv"
                    no-caps
                    @click="exportTable"
                  />
                  <q-circular-progress
                      v-if="isShowProgress"
                      show-value
                      font-size="14px"
                      :value="progress"
                      size="40px"
                      :thickness="0.32"
                      color="teal"
                      track-color="grey-3"
                  />
                </div>
              </div>
            </div>
          </template>
        </q-table>
      </div>
    </div>
  </q-page>
</template>

<script>
import { exportFile } from 'quasar'
import JSZip from 'jszip'
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
      nameNews: '',
      title: '',
      content: '',
      locale: '',
      outputlimit: 0,
      isSubmitDisabled: false,
      isResultShow: false,
      resultData: [],
      outputType: 'csv',
      progress: 0,
      isShowProgress: false,
      visibleColumns: ['date', 'name', 'locale', 'description', 'title', 'content', 'url'],
      resultColumns: [
        {
          name: 'date',
          label: 'Date',
          align: 'left',
          field: 'date',
          sortable: true
        },
        {
          name: 'name',
          label: 'News Name',
          align: 'left',
          field: 'name',
          sortable: true
        },
        {
          name: 'locale',
          label: 'Locale',
          align: 'left',
          field: 'locale',
          sortable: true
        },
        {
          name: 'description',
          label: 'Description',
          align: 'left',
          field: 'description',
          sortable: true
        },
        {
          name: 'title',
          label: 'Title',
          align: 'left',
          field: 'title',
          sortable: true
        },
        {
          name: 'content',
          label: 'Content',
          align: 'left',
          field: 'content',
          sortable: true
        },
        {
          name: 'url',
          label: 'Url',
          align: 'left',
          field: 'url',
          sortable: false
        }
      ]
    }
  },
  methods: {
    // 單擊 Submit 提交時
    async onSubmit () {
      console.info('[methods][onSubmit]')
      const bar = this.$refs.bar
      this.isSubmitDisabled = true
      bar.start()
      await this.$http.get('/newsData', {
        params: {
          name: this.nameNews,
          title: this.title,
          content: this.content,
          locale: this.locale
        }
      })
        .then(res => {
          if (res.data === '[]') {
            console.error('error')
          } else {
            bar.stop()
            this.isSubmitDisabled = false
            this.isResultShow = true
            this.resultData = res.data
          }
        })
    },
    // 導出函數
    exportTable () {
      this.isShowProgress = true
      const zip = new JSZip()
      const header = [this.visibleColumns.map(col => wrapCsvValue(col))]
      const data = this.resultData.map(row => this.visibleColumns.map(col => wrapCsvValue(row[col])).join(','))
      const totalLength = data.length
      const today = new Date()
      const nowDateTime = `${today.getFullYear()}${today.getMonth()}${today.getDate()}-${today.getHours()}${today.getMinutes()}${today.getSeconds()}`
      let count = 0
      const jump = Math.floor(totalLength / this.outputlimit)
      while (data.length > 0) {
        let content = data.splice(0, jump)
        switch (this.outputType) {
          case 'csv':
            content = header.concat(content).join('\r\n')
            break
          case 'txt':
            content = content.join('\r\n')
            break
          default:
            content = header.concat(content).join('\r\n')
            break
        }
        zip.file(`news-data-${count++}-${nowDateTime}.` + this.outputType, content)
        this.progress = (1 - data.length / this.resultData.length).toFixed(2) * 80
      }
      this.progress = 90
      zip.generateAsync({
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: {
          level: 5
        }
      }).then(content => {
        const status = exportFile(
          `news-crawler-storage-export-${nowDateTime}.zip`,
          content,
          'blob'
        )
        this.progress = 100
        setTimeout(() => {
          this.isShowProgress = false
        }, 2000)
        if (status !== true) {
          this.$q.notify({
            message: 'Browser denied file download...',
            color: 'negative',
            icon: 'warning'
          })
        }
      })
    }
  },
  mounted () {
    // 仅调试时启用
    // window.vue = this
  }
}
</script>
