<template lang="pug">
  #extractor
    nav#nav(class='uk-navbar-container uk-navbar-transparent' uk-navbar)
      div(class='uk-navbar-left')
        ul(class='uk-navbar-nav')
          li: router-link(to='/' uk-icon='icon: arrow-left; ratio: 1.4')
      div(class='uk-navbar-right')
        ul(class='uk-navbar-nav')
          li: img(:src='userProfile')
    main#main
      h3.warning *글만 백업되며, 페이지와 공지사항은 미포함입니다. (티스토리 오픈 API 미지원)
      table(class='uk-table uk-table-middle uk-table-divider')
        thead
          tr
            th
            th 블로그 정보
            th 주소
            th 대표 블로그
        tbody
          tr(v-for="blog in blogs")
            td: checkbox(class='uk-checkbox' type='checkbox' :val='blog.name' v-model='checkedNames')
            td {{ blog.title }}
            td {{ blog.url }}
            td {{ blog.default }}
      button#extract(ref='extract' @click='zip(checkedNames)') 시작하기
      a#download(href='#' ref='download') 다운받기
      #metainfo
        .url {{ url }}
        .message {{ message }}
        .errors
          .e(v-for='error in errors') {{ error }}
</template>

<script>

import tistory from 'tistory'
import JSZip from 'jszip'
import moment from 'moment'
import Swal from 'sweetalert2'

import Checkbox from '../Checkbox'

const electron = require('electron').remote

export default {
  name: 'extractor',
  async mounted () {
    this.isValidAccessToken().then(async () => {
      const blogInfo = await tistory.blog.info(this.$store.state.tistory.accessToken).catch(reason => {
        Swal.fire({ icon: 'error', title: '이런!', text: '블로그 정보를 불러올 수 없습니다.' })
        this.errors.push('블로그 정보를 불러올 수 없습니다.')
      })

      this.blogs = blogInfo.data.tistory.item.blogs
      this.userProfile = this.blogs.filter(blog => blog.default === 'Y')[0].profileImageUrl
    })
  },
  data () {
    return {
      userProfile: null,
      blogs: [],
      checkedNames: [],
      url: null,
      message: null,
      errors: []
    }
  },
  methods: {
    async images (doc, postFolder, pathname) {
      for (const img of doc.querySelectorAll('img')) {
        const sources = img.getAttribute('src').split(',')
        const imgRegex = /kage@(.*)/

        for (let i = 0; i < sources.length; i++) {
          if (imgRegex.test(sources[i])) {
            const imageSourceName = imgRegex.exec(sources[i])[1]

            const response = await fetch(`https://blog.kakaocdn.net/dn/${imageSourceName}`)
            switch (response.status) {
              case 404: {
                this.errors.push(`https://blog.kakaocdn.net/dn/${imageSourceName} 를 가져오는데 실패했습니다.`)
              }
            }

            let imageFilename = imageSourceName.replace(/\//g, '_')
            imageFilename = imageFilename.substring(0, imageFilename.length - 8)
            imageFilename = imageFilename + '.' + response.headers.get('Content-Type').substring(6)

            postFolder.file(imageFilename, response.blob())

            if (i > 0) {
              const newImg = doc.createElement('img')
              newImg.setAttribute('src', './' + imageFilename)
              img.parentNode.appendChild(newImg)
            } else {
              img.setAttribute('src', './' + imageFilename)
            }

            this.message = `${pathname}/${imageFilename}`
          }
        }
      }
    },
    html (doc, postFolder, pathname) {
      postFolder.file('index.html', new XMLSerializer().serializeToString(doc))
      this.message = `${pathname}/index.html`
    },
    async collectZipContents (html, postFolder) {
      const parser = new DOMParser()
      const doc = parser.parseFromString(html, 'text/html')
      const pathname = this.message

      await this.images(doc, postFolder, pathname)
      this.html(doc, postFolder, pathname)
    },
    async buildZip (rootFolder, blogName, posts) {
      for (const post of posts) {
        const postDetail = await tistory.post.read(this.$store.state.tistory.accessToken, { blogName, postId: post.id }).catch(reason => {
          Swal.fire({ icon: 'error', title: '이런!', text: `${post.postUrl} 에 해당하는 포스트를 찾을 수 없습니다.` })
          this.errors.push(`${post.postUrl} 에 해당하는 포스트를 찾을 수 없습니다.`)
        })
        const postFolder = rootFolder.folder(post.title)

        this.url = post.postUrl
        this.message = `${blogName}/${post.title}`

        this.collectZipContents(postDetail.data.tistory.item.content, postFolder)
      }
    },
    generateZip (zip) {
      zip.generateAsync({ type: 'blob' }).then(blob => {
        const file = new Blob([ blob ], { type: 'application/zip' })
        const fileURL = URL.createObjectURL(file)

        this.$refs.extract.style.display = 'none'

        this.$refs.download.setAttribute('href', fileURL)
        this.$refs.download.setAttribute('download', moment().format('X_YYYY_MM_DD'))
        this.$refs.download.style.display = 'inline-block'

        Swal.fire({ icon: 'success', title: '백업 성공!', text: '티스토리 블로그의 백업이 완료되었습니다. 다운로드 버튼을 눌러 결과를 확인해보세요.' })
      })
    },
    async zip (checkedNames) {
      if (checkedNames.length <= 0) {
        Swal.fire({ icon: 'error', title: '이런!', text: '티스토리 블로그를 백업하려면 블로그 선택해야합니다.' })
        return this.errors.push('티스토리 블로그를 백업하려면 블로그 선택해야합니다.')
      }
      this.isValidAccessToken().then(async () => {
        const zip = new JSZip()

        for (const blogName of checkedNames) {
          const rootFolder = zip.folder(blogName)
          this.message = blogName

          let page = 1

          while (true) {
            const postList = await tistory.post.list(this.$store.state.tistory.accessToken, { blogName, page: page++ }).catch(reason => {
              Swal.fire({ icon: 'error', title: '이런!', text: `https://${blogName}.tistory.com/${page - 1} 에 해당하는 글 목록을 불러올 수 없습니다.` })
              this.errors.push(`https://${blogName}.tistory.com/${page - 1} 에 해당하는 글 목록을 불러올 수 없습니다.`)
            })

            if (postList.data.tistory.item.hasOwnProperty('posts')) {
              await this.buildZip(rootFolder, blogName, postList.data.tistory.item.posts)
            } else break
          }
        }
        this.generateZip(zip)
      })
    },
    isValidAccessToken () {
      return tistory.blog.info(this.$store.state.tistory.accessToken).catch(reason => Swal.fire({ icon: 'error', title: '이런!', text: '티스토리 세션이 만료되어 인증을 다시해야 합니다.' }).then(() => this.$router.push('/')))
    }
  },
  components: {
    Checkbox
  }
}
</script>

<style lang="stylus">
  #extractor
    #nav
      position fixed
      top 0
      right 0
      width 100%
      padding 10px 20px
      box-sizing border-box
      a
        padding 0
      .uk-navbar-left
        li
          margin-right 12px
      .uk-navbar-right
        img
          width 40px
          height 40px
          border-radius 50%
    #main
      width 820px
      .warning
        font-size .92rem
        font-weight 600
        color black
      #extract, #download
        color white
        padding 15px 30px
        border-radius 5px
        box-shadow 0 0 0 1px rgba(0, 0, 0, .1), 0 2px 5px rgba(0, 0, 0, .1)
        font-weight 500
        border none
        cursor pointer
        font-size 1rem
      #extract
        background-color #ed5207
        display inline-block
      #download
        background-color black
        display none
      table
        td
          &:nth-child(2)
            font-weight 600
            color black
      #metainfo
        margin 15px 0
        *
          word-break break-all
        .url
          margin-bottom 3px
          font-size .9rem
        .message
          color black
          font-weight 500
        .errors
          margin-top 10px
          .e
            color red
            margin 5px 0
</style>
