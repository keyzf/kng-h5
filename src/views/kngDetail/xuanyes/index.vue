<template>
  <div class="over-hidden" :class="isAddStudy ? 'pb54' : null" v-if="loading">
    <div class="detail-play-container pr">
      <img :src="kngInfo.coverUrl" class="detail_play-coverUrl" v-if="Object.keys(kngInfo).length > 0">
      <div class="detail_mask"></div>
      <template v-if="!isAudit">
        <yxt-icon name="play-circle-o" class="detail_play_icon" v-if="!isShowProgress" @click="playScorm" />
        <div class="detail-play-study" v-if="isShowProgress && kngInfo.schedule === 0">
          <p class="text-white font-size-16">未开始学习</p>
          <yxt-button round size="medium" style="margin-top: 16px;" @click="playScorm">开始学习</yxt-button>
        </div>
        <div class="detail-play-study" v-if="isShowProgress && kngInfo.schedule !== 0">
          <yxt-button round size="medium" @click="playScorm">继续学习</yxt-button>
        </div>
      </template>
      <template v-else>
        <yxt-icon name="play-circle-o" class="detail_play_icon" @click="playScorm" />
      </template>
    </div>
    <yxt-tabs v-model="active" :swipe-threshold="5" sticky swipeable>
      <yxt-tab title="详情" name="0">
        <kng-info :kngInfoDetail="kngInfo" :kngSchedule="schedule" :isShowProgress="isShowProgress"></kng-info>
      </yxt-tab>
      <yxt-tab title="评价" name="1">
        <comment-list @changeScoreNum="changeScoreNum" :kngInfoDetail="kngInfo" :isAddStudy="isAddStudy"></comment-list>
      </yxt-tab>
      <yxt-tab title="笔记" name="2">
        <kngNoteList :kngId="kngInfo.id" :title="kngInfo.title" :addStudy="kngInfo.addStudy" :isAddStudy="isAddStudy"></kngNoteList>
      </yxt-tab>
      <!-- <yxt-tab title="问答">问答</yxt-tab> -->
    </yxt-tabs>
    <!-- 加入学习 -->
    <template v-if="!isAudit">
      <kng-add-study v-if="isAddStudy" :kngId="kngId" @isAdd="getIsAdd"></kng-add-study>
    </template>
  </div>
  <div class="yui-screen-page" v-else>
    <yxt-loading text-size="16px">加载中...</yxt-loading>
  </div>
</template>

<script>
import api from '@/service/kngDetail.service'
import kngInfo from '@/views/kngDetail/components/kngInfo'
import CommentList from '@/views/kngDetail/components/commentList'
import kngNoteList from '@/views/kngDetail/components/kngNoteList'
import KngAddStudy from '@/views/kngDetail/components/kngAddStudy'
export default {
  name: 'xuanyesdetail',
  components: {
    CommentList,
    kngInfo,
    KngAddStudy,
    kngNoteList
  },
  props: {},
  data () {
    return {
      kngId: this.$route.query.id, // 知识id
      courseId: this.$route.query.courseId, // 课程包id
      targetId: this.$route.query.targetId, // 项目或计划id
      targetCode: this.$route.query.targetCode, // 项目或计划名
      type: this.$route.query.type,
      status: this.$route.query.status, // 是否是从审核页面过来
      taskType: this.$route.query.taskType, // 从项目过来审核的权限
      // 其他参数
      schedule: 0,
      lastName: '',
      active: '0',
      isAddStudy: true, // 是否有加入学习按钮
      isShowProgress: false, // 是否显示进度
      kngInfo: {},
      loading: false,
      url: '', // 微课播放地址
      moreEnabled: 0 // 多知识学习
    }
  },
  computed: {
    // 是否是从审核中那边过来的
    isAudit () {
      if (this.status !== '' && this.status !== undefined && this.status !== null) {
        return this.status !== 4
      }
      return false
    }
  },
  mounted () {
    if (!this.isAudit) {
      this.getKngConf()
    } else {
      this.getKngDetail()
    }

    // 监听浏览器关闭事件
    let that = this
    window.onunload = function (ev) {
      that.storeKngStudying()
    }
  },
  methods: {
    // 获取防作弊配置
    getKngConf () {
      api.getKngConf().then(res => {
        this.handlerConf(res)
      }).then(() => {
        this.handleVersion()
      }).catch(err => {
        console.log(err)
      })
    },
    // 处理是否有多知识学习
    handlerConf (res) {
      this.moreEnabled = res.moreEnabled
    },
    // 获取是否加入学习
    getIsAdd (val) {
      this.isShowProgress = !val
      this.isAddStudy = val
    },
    // 改变评分
    changeScoreNum (data) {
      this.kngInfo.avgCommentScore = data
    },
    // 获取知识详情
    getKngDetail () {
      this.show = false
      let bodyParams = {
        id: this.kngId,
        preview: 0,
        targetId: this.targetId,
        targetCode: this.targetCode ? this.targetCode : 'kng'
      }

      if (this.isAudit) {
        bodyParams.preview = 1
      }

      if (this.taskType) {
        bodyParams.preview = 0
      }

      if ((this.taskType !== undefined || this.taskType !== null) && this.targetCode === 'o2o') {
        bodyParams.taskType = this.taskType
      }

      api.postKngDetail(bodyParams).then(res => {
        this.kngInfo = res
        this.isAddStudy = !res.addStudy
        this.isShowProgress = res.addStudy
        this.schedule = res.schedule
        // 设置知识title
        document.title = this.kngInfo.title
        this.loading = true
        this.lastName = res.lastName
        this.postPlayStudy()
        if (!this.isAudit) {
          // 判断是否开启多知识学习
          if (this.moreEnabled) {
            if (this.lastName !== null && this.lastName !== undefined) {
              this.handleChangeStudy()
            }
          }
        }
        return res
      }).catch(err => {
        if (err.error.key) {
          let msg = this.error(err)
          this.Dialog.confirm({
            title: '提示',
            message: msg,
            confirmButtonText: '我知道了',
            closeOnPopstate: true,
            showCancelButton: false
          }).then(() => {
            this.$router.go(-1)
          })
        } else {
          console.log(err)
        }
      })
    },
    // 播放scorm地址
    postPlayStudy () {
      // 获取知识播放地址
      let bodyParams = {
        clientType: 1, // 是否是h5
        id: this.kngInfo.id, // 知识id
        preview: 0, // 是否预览查看 0：否，1：预览
        targetId: this.targetId, // 项目id
        targetCode: this.targetCode // 来源code：目前（kng,o2o）
      }

      if (this.isAudit) {
        bodyParams.preview = 1
      }
      api.postPlayStudy(bodyParams).then(res => {
        let htmlUrl = ''
        if (res.playDetails && res.playDetails.length > 0) {
          res.playDetails.forEach((item, index) => {
            if (item.desc === 'signUrl') {
              htmlUrl = item.url
            }
          })
        }
        this.url = htmlUrl
        return res
      }).catch(err => {
        console.log(err)
      })
    },
    // 点击播放跳转页面
    playScorm () {
      if (this.url === '') {
        this.$toast('播放地址为空！')
        return
      }
      window.location.href = this.url
    },
    // 处理学习改变
    handleChangeStudy () {
      this.Dialog.confirm({
        title: '提示',
        message: '系统在同一时间只记录一个知识的学习进度和学分，如果您选择切换观看新知识，则只会记录新知识的学习进度和学分',
        confirmButtonText: '切换到新知识',
        closeOnPopstate: true
      }).then(() => {
        // 确定操作
        let bodyParams = {
          kngId: this.kngId
        }
        api.postStudyChange(bodyParams).then(res => {
        }).catch(err => {
          console.log(err)
        })
      }).catch(() => {
        // 取消操作
        this.$router.go(-1)
      })
    },
    // 处理版本问题
    handleVersion () {
      let bodyParams = {
        kngId: this.kngId
      }
      api.postKngVersionAlert(bodyParams).then(versionObj => {
        this.deleteDetail()
      }).catch(err => {
        if (err.error.key === 'apis.knowledge.hours.changed') {
          this.hourUpdate(bodyParams)
        } else if (err.error.key === 'apis.knowledge.version.changed') {
          this.tipUpdate(bodyParams)
        }
        console.log(err)
      })
    },
    hourUpdate (bodyParams) {
      this.Dialog.confirm({
        title: '提示',
        message: '本知识（课程）学时被更新，学习进度将被重置',
        confirmButtonText: '我知道了',
        showCancelButton: false
      }).then(() => {
        // 确定操作
        api.putkngVersionAlert(bodyParams).then(() => {
          this.getKngDetail()
        }).catch(err => {
          console.log(err)
        })
      }).catch(() => {
        // 取消操作
        this.$router.go(-1)
      })
    },
    tipUpdate (bodyParams) {
      this.Dialog.confirm({
        title: '提示',
        message: '本知识（课程）更新了新版本，建议您重新学习',
        confirmButtonText: '我知道了',
        showCancelButton: false
      }).then(() => {
        // 确定操作
        api.putkngVersionAlert(bodyParams).then(() => {
          this.getKngDetail()
        }).catch(err => {
          console.log(err)
        })
      }).catch(() => {
        // 取消操作
        this.$router.go(-1)
      })
    },
    // 删除判断之前学的是不是当前新打开的这个
    deleteDetail () {
      if (localStorage.playStudyArr) {
        let playStudyArr = JSON.parse(localStorage.playStudyArr)
        if (playStudyArr.length > 0) {
          let bodyParams = {
            ids: playStudyArr.join(',')
          }
          api.postDeleteStudy(bodyParams).then((res) => {
            localStorage.removeItem('playStudyArr')
          }).then(() => {
            this.getKngDetail()
          }).catch(err => {
            console.log(err)
          })
        }
      } else {
        this.getKngDetail()
      }
    },
    // 存储正在学
    storeKngStudying () {
      let playStudyArr = []
      if (localStorage.playStudyArr) {
        playStudyArr = JSON.parse(localStorage.playStudyArr)
        playStudyArr.push(this.kngId)
      } else {
        playStudyArr.push(this.kngId)
      }
      let newSet = Array.from(new Set([...playStudyArr]))
      localStorage.setItem('playStudyArr', JSON.stringify(newSet))
    }
  },
  beforeDestroy () {
    // 存储当前正在学
    this.storeKngStudying()
  }
}
</script>
