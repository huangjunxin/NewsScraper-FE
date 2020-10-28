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
          row-key="link-href"
        >
          <template v-slot:top-right>
          <q-input
            v-model="outputlimit"
            label="導出條目限制 if 0 output all"
            placeholder="0"
          ></q-input>
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
      nameNews: '',
      title: '',
      content: '',
      locale: '',
      outputlimit: 0,
      isSubmitDisabled: false,
      isResultShow: false,
      resultData: [],
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
      const header = [this.resultColumns.map(col => wrapCsvValue(col.label))]
      const data = this.resultData.map(row => this.resultColumns.map(col => wrapCsvValue(
        typeof col.field === 'function'
          ? col.field(row)
          : row[col.field === undefined ? col.name : col.field],
        col.format
      )).join(','))
      const totalLength = data.length
      const today = new Date()
      const nowDateTime = `${today.getFullYear()}${today.getMonth()}${today.getDate()}-${today.getHours()}${today.getMinutes()}${today.getSeconds()}`
      let count = 0
      while (data.length > 0) {
        let content = []
        for (let i = 0; data.length > 0 && i < Math.floor(totalLength / this.outputlimit); ++i) { content.push(data.shift()) }
        if (this.outputlimit === 0) {
          content = header.concat(data).join('\r\n')
        } else {
          content = header.concat(content).join('\r\n')
        }
        const status = exportFile(
          `news-crawler-storage-export-${count++}-${nowDateTime}.csv`,
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
        if (this.outputlimit === 0) { break }
      }
    }
  },
  mounted () {
    // 仅调试时启用
    // window.vue = this
  }
}
</script>
