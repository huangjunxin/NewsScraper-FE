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
              default-opened
              label="Fetcher List 請選擇你想搜尋的站點"
            >
              <q-card>
                <q-card-section>
                  <q-checkbox v-model="isfetcherListAllSelect" @input="fetcherListAllSelect" label="All 全選" />
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
            <q-btn label="Submit" type="submit" color="primary"/>
          </div>
        </q-form>
      </div>
    </div>
  </q-page>
</template>

<script>
export default {
  data () {
    return {
      keywords: '',
      isfetcherListAllSelect: false,
      fetcherListAccepted: [],
      fetcherListOptions: [
        {
          label: 'news_01.js',
          value: 'news_01.js'
        },
        {
          label: 'news_apple_daily.js',
          value: 'news_apple_daily.js'
        },
        {
          label: 'news_binhduong.js',
          value: 'news_binhduong.js'
        },
        {
          label: 'news_cctime.js',
          value: 'news_cctime.js'
        },
        {
          label: 'news_chinapress.js',
          value: 'news_chinapress.js'
        },
        {
          label: 'news_cna.js',
          value: 'news_cna.js'
        },
        {
          label: 'news_govhk.js',
          value: 'news_govhk.js'
        },
        {
          label: 'news_govviet.js',
          value: 'news_govviet.js'
        },
        {
          label: 'news_guangming.js',
          value: 'news_guangming.js'
        },
        {
          label: 'news_hhua.js',
          value: 'news_hhua.js'
        },
        {
          label: 'news_jianhua.js',
          value: 'news_jianhua.js'
        },
        {
          label: 'news_khmer.js',
          value: 'news_khmer.js'
        },
        {
          label: 'news_kwongwah.js',
          value: 'news_kwongwah.js'
        },
        {
          label: 'news_ltn.js',
          value: 'news_ltn.js'
        },
        {
          label: 'news_medcom.js',
          value: 'news_medcom.js'
        },
        {
          label: 'news_mhwmm.js',
          value: 'news_mhwmm.js'
        },
        {
          label: 'news_mmgpmedia.js',
          value: 'news_mmgpmedia.js'
        },
        {
          label: 'news_orientaldaily.js',
          value: 'news_orientaldaily.js'
        },
        {
          label: 'news_people_cn.js',
          value: 'news_people_cn.js'
        },
        {
          label: 'news_sanwah.js',
          value: 'news_sanwah.js'
        },
        {
          label: 'news_sggp.js',
          value: 'news_sggp.js'
        },
        {
          label: 'news_shinmin.js',
          value: 'news_shinmin.js'
        },
        {
          label: 'news_singtao.js',
          value: 'news_singtao.js'
        },
        {
          label: 'news_thaicn.js',
          value: 'news_thaicn.js'
        },
        {
          label: 'news_thaiembbeij.js',
          value: 'news_thaiembbeij.js'
        },
        {
          label: 'news_thaizhonghua.js',
          value: 'news_thaizhonghua.js'
        },
        {
          label: 'news_tkb.js',
          value: 'news_tkb.js'
        },
        {
          label: 'news_udn.js',
          value: 'news_udn.js'
        },
        {
          label: 'news_udnbkk.js',
          value: 'news_udnbkk.js'
        },
        {
          label: 'news_vietnamplus.js',
          value: 'news_vietnamplus.js'
        },
        {
          label: 'news_wanbao.js',
          value: 'news_wanbao.js'
        },
        {
          label: 'news_zaobao.js',
          value: 'news_zaobao.js'
        }
      ],
      timeLimitModel: null,
      timeLimitOptions: [
        'Any', 'Day', 'Week', 'Year'
      ]
    }
  },
  methods: {
    onSubmit () {
      const res = {
        fetcherListAccepted: this.fetcherListAccepted,
        keywords: this.keywords,
        timeLimitModel: this.timeLimitModel
      }
      console.log(res)
    },
    fetcherListAllSelect (value, evt) {
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
    }
  },
  mounted () {
    this.$http.get('/function/listAllNews', {
      params: {
      }
    })
      .then(res => {
        if (res.data.code === -1) {
          console.error(res.data.data.msg)
        } else {
          this.fetcherListOptions = res.data
        }
      })
  }
}
</script>
