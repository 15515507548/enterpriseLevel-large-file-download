<template>
  <div>
    <el-button @click="handleExport(file)" :loading="file.isShow">文件下载</el-button>
    <el-progress v-show="file.isShow" :percentage="file.percentage"></el-progress>
    <i
      v-show="file.isShow && !file.issmallfile"
      :class="file.iconClass"
      @click.stop="handlePausePlay(file)"
      style="margin-left: 10px; cursor: pointer"
    ></i>
  </div>
</template>

<script>
import { instance } from './request'
import axios from 'axios'
import qs from 'qs'
import localforage from 'localforage'
export default {
  name: 'myAbout',
  data() {
    return {
      fileList: [],
      file: {
        url: 'http://127.0.0.1:7777/api/rangeFile?filename=assddewwws.rar',
        loadSize: 0, //成功上传切片的个数，也是断点下载之后继续下载的i
        percentage: 0, //进度条的值
        isinterfaceFailedfor: false,
        isShow: false, //是否显示进度条
        iconClass: 'el-icon-video-pause', //暂停或上传图标
        controllers: [], //需要取消的接口请求
        terminateRequest: false, //当前的状态是否是暂停
        allBufferLists: [], //切片下载成功的结果
        interfaceFailedLists: [] //切片下载接口失败的索引i
      }
    }
  },
  async created() {
    //断点续载(类似断点续传)通过localStorage和localforage拿到页面刷新之前保存的最新数据
    this.file.interfaceFailedLists = localStorage.getItem('interfaceFailedLists')
      ? JSON.parse(localStorage.getItem('interfaceFailedLists'))
      : []
    this.file.isinterfaceFailedfor = localStorage.getItem('isinterfaceFailedfor') === 'true'
    const allBufferLists = await localforage.getItem('allBufferLists')
    if (allBufferLists) {
      allBufferLists.sort((a, b) => a.idx - b.idx)
      this.file.allBufferLists = allBufferLists
      if (allBufferLists.length >= 1) {
        //通过allBufferLists从小到大排序后获取最后一条数据拿到当前开始下载的loadSize和percentage
        const lastObj = allBufferLists[allBufferLists.length - 1]
        this.file.loadSize = lastObj.loadSize
        this.file.percentage = lastObj.percentage
      }
    }
    console.log(
      this.file.allBufferLists,
      this.file.loadSize,
      this.file.percentage,
      this.file.interfaceFailedLists,
      'created'
    )
  },
  methods: {
    async handlePausePlay(file) {
      if (file.iconClass == 'el-icon-video-pause') {
        file.iconClass = 'el-icon-video-play'
        file.controllers.forEach((controller) => {
          //注意：一旦调用了controller.abort()，这个controller实例对象就不能再用于新的请求，需要创建一个新的AbortController实例。
          controller.abort()
        })
        file.controllers = []
        file.terminateRequest = true
      } else {
        file.terminateRequest = false
        file.iconClass = 'el-icon-video-pause'
        if (file.isinterfaceFailedfor) {
          this.lastFn(file)
        } else {
          const obj = file.allBufferLists.find((item) => item.loadSize === file.loadSize)
          if (obj) {
            this.continueUpload(file, obj.idx + 1)
          }
        }
      }
    },
    async forFn(i, file) {
      const start = i * file.chunkSize
      const end = i + 1 == file.totalChunks ? file.sizeLength - 1 : (i + 1) * file.chunkSize - 1
      const fn = () => {
        const controller = new AbortController()
        const signal = controller.signal
        file.controllers.push(controller)
        return axios({
          url: file.url,
          method: 'get',
          headers: {
            range: `bytes=${start}-${end}`
          },
          responseType: 'arraybuffer',
          signal
        })
      }

      //接口失败重试每个接口总共可以发送4次请求，重试3次
      const sgfd = (i, fn, index = 0, max = 4) => {
        if (index < max) {
          if (file.terminateRequest) {
            return
          }
          const promise = fn()
          file.lists.add(promise)
          promise
            .then((res) => {
              if (+res.status === 206) {
                file.lists.delete(promise)
                file.interfaceFailedLists = file.interfaceFailedLists.filter((nub) => nub != i)
                file.loadSize++
                file.percentage = +((file.loadSize / file.totalChunks) * 100).toFixed(0)
                file.allBufferLists.push({
                  idx: i,
                  buffer: res.data,
                  loadSize: file.loadSize,
                  percentage: file.percentage
                })
                localStorage.setItem(
                  'interfaceFailedLists',
                  JSON.stringify(file.interfaceFailedLists)
                )
                return
              }
              throw '下载失败，请您稍后再试~~'
            })
            .catch((err) => {
              file.lists.delete(promise)
              if (!file.interfaceFailedLists.includes(i)) {
                file.interfaceFailedLists.push(i)
                localStorage.setItem(
                  'interfaceFailedLists',
                  JSON.stringify(file.interfaceFailedLists)
                )
              }
              index++
              sgfd(i, fn, index)
            })
        }
      }
      async function FSG() {
        if (file.lists.size >= 3) {
          // 3代表请求控制最大并发数
          try {
            await Promise.race(file.lists)
            await localforage.setItem('allBufferLists', file.allBufferLists)
            console.log(
              file.allBufferLists,
              file.loadSize,
              file.totalChunks,
              file.percentage,
              '成功'
            )
          } catch (err) {
            await FSG()
          }
        }
      }
      sgfd(i, fn)
      await FSG()
    },
    async continueUpload(file, startIdx) {
      for (let i = startIdx; i < file.totalChunks; i++) {
        const obj = file.allBufferLists.find((item) => item.idx === i)
        if (obj) {
          continue
        }
        if (file.terminateRequest) {
          return
        }
        await this.forFn(i, file)
      }
      await Promise.allSettled(file.lists)
      await this.lastFn(file)
    },
    //所有请求完成后，失败请求接口重试
    async lastFn(file) {
      if (file.interfaceFailedLists.length) {
        for (const i of file.interfaceFailedLists) {
          const obj = file.allBufferLists.find((item) => item.idx === i)
          if (obj) {
            continue
          }
          if (!file.isinterfaceFailedfor) {
            //所有请求完成后，失败接口重试时点击暂停按钮或页面刷新时的操作
            file.isinterfaceFailedfor = true
            localStorage.setItem('isinterfaceFailedfor', true)
          }
          if (file.terminateRequest) {
            return
          }
          await this.forFn(i, file)
        }
      }
      await Promise.allSettled(file.lists)
      console.log(file.allBufferLists, file.loadSize, file.totalChunks, file.percentage, '接口完毕')
      // if (file.iconClass == 'el-icon-video-play') return
      if (file.loadSize != file.totalChunks) {
        this.$message.error('下载失败，请您稍后再试~~')
      } else {
        file.allBufferLists.sort((a, b) => a.idx - b.idx)
        const arrBufferList = file.allBufferLists.map((item) => new Uint8Array(item.buffer))
        const allBuffer = this.concatenate(Uint8Array, arrBufferList)
        const blob = new Blob([allBuffer])
        let blobURL = window.URL.createObjectURL(blob)
        const elink = document.createElement('a')
        elink.download = file.obj.filename
        elink.href = blobURL
        elink.target = '_blank'
        elink.style.display = 'none'
        document.body.appendChild(elink)
        elink.click()
        window.URL.revokeObjectURL(blobURL)
        document.body.removeChild(elink)
      }
      file.isShow = false
      file.loadSize = 0
      file.percentage = 0
      file.controllers = []
      file.isinterfaceFailedfor = false
      localStorage.removeItem('isinterfaceFailedfor')
      file.allBufferLists = []
      file.interfaceFailedLists = []
      localStorage.removeItem('interfaceFailedLists')
      await localforage.clear()
    },
    concatenate(resultConstructor, arrays) {
      let totalLength = 0
      for (let arr of arrays) {
        totalLength += arr.length
      }
      let result = new resultConstructor(totalLength)
      let offset = 0
      for (let arr of arrays) {
        result.set(arr, offset)
        offset += arr.length
      }
      return result
    },
    async handleExport(file) {
      const res = await axios({
        url: file.url,
        method: 'head'
      })
      if (res.status == 200) {
        // 获取长度来进行分割块
        console.time('并发下载')
        file.sizeLength = Number(res.headers['content-length'])
        file.isShow = true
        file.chunkSize = 5 * 1024 * 1024 //切片大小
        const idx = this.file.url.indexOf('?')
        file.obj = qs.parse(this.file.url.slice(idx + 1))
        file.issmallfile = file.sizeLength <= file.chunkSize
        if (file.issmallfile) {
          try {
            const res1 = await axios({
              url: file.url,
              method: 'get',
              responseType: 'blob',
              onDownloadProgress: (e) => {
                const percentage = +((e.loaded / e.total) * 100).toFixed(0)
                file.percentage = percentage
              }
            })
            if (res1.status == 200) {
              let blobURL = window.URL.createObjectURL(res1.data)
              const elink = document.createElement('a')
              elink.download = file.obj.filename
              elink.href = blobURL
              elink.target = '_blank'
              elink.style.display = 'none'
              document.body.appendChild(elink)
              elink.click()
              window.URL.revokeObjectURL(blobURL)
              document.body.removeChild(elink)
              return
            }
            throw '下载失败'
          } catch (err) {
            this.$message.warning(err)
          } finally {
            file.isShow = false
          }
        } else {
          file.lists = new Set()
          file.controllers = []
          file.totalChunks = Math.ceil(file.sizeLength / file.chunkSize)

          if (file.percentage) {
            if (file.isinterfaceFailedfor) {
              this.lastFn(file)
            } else {
              const obj = file.allBufferLists.find((item) => item.loadSize === file.loadSize)
              console.log(
                file.allBufferLists,
                file.loadSize,
                file.totalChunks,
                file.percentage,
                obj,
                '成功'
              )
              this.continueUpload(file, obj.idx + 1)
            }
          } else {
            this.continueUpload(file, 0)
          }
        }
      } else {
        this.$message.warning(`下载失败`)
      }
    }
  }
}
</script>

<style scoped lang="scss">
.hoverRow {
  display: flex;
  justify-content: space-between;
  margin-right: 50px;
  padding: 3px 5px;
  align-items: center;
}
.hoverRow:hover {
  background: #ebeef5;
  cursor: pointer;
  .colorHover {
    color: #409eff;
  }
}
</style>
