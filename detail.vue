<template>
  <el-container class="detail-view">
    <el-page-header @back="$router.go(-1)"></el-page-header>
    <el-main class="detail-view-main">
      <h3 class="center">{{articleTitle}}</h3>
      <div class="detail-subtitle">
        <div class="clearfix detail-subtitle-item">
          <label class="fl">原文链接:</label>
          <a :href="form.link" target="_blank">{{form.link}}</a>
        </div>
        <div class="clearfix detail-subtitle-item">
          <label class="fl">推荐人:</label>
          <p>{{form.referrer}}</p>
        </div>
        <div class="clearfix detail-subtitle-item">
          <label class="fl">文章分类:</label>
          <p>{{form.category}}</p>
        </div>
        <div class="clearfix detail-subtitle-item">
          <label class="fl">推荐理由:</label>
          <p>{{form.description}}</p>
        </div>
      </div>
      <viewer :value="content"
              :locale="zhHans"
              :plugins="plugins"></viewer>
    </el-main>
    <el-aside class="detail-view-aside" width="300px">
      <nav class="article-catalog" v-sticky="{ zIndex: 3, stickyTop: 20 }">
        <div class="catalog-title">目录</div>
        <div class="catalog-body">
          <ul class="catalog-list" ref="catalog">
            <li class="item" v-for="(h, i) in hast" :key="i" :class="[`d${h.level - minLevel}`, { 'active': headNum === i }]">
              <div class="a-container">
                <a @click="locationTxt(h, i)" :title="h.text" class="catalog-aTag">{{h.text}}</a>
              </div>
            </li>
          </ul>
        </div>
      </nav>
      <el-backtop target=".home-layout-main"></el-backtop>
    </el-aside>
  </el-container>
</template>

<script>
import visit from 'unist-util-visit';
import VueSticky from 'vue-sticky';
import { findIndex } from 'lodash';
import throttle from 'throttle-debounce/throttle';
/* eslint-disable */
import { getArticleDetail } from '@/api/editor';
import 'bytemd/dist/index.min.css';
// 引入高亮css
import 'highlight.js/styles/vs.css';
// import 'github-markdown-css/github-markdown.css';
import { Viewer } from '@bytemd/vue';
import gfm from '@bytemd/plugin-gfm';
// import highlight from '@bytemd/plugin-highlight'
import breaks from '@bytemd/plugin-breaks';
import frontmatter from '@bytemd/plugin-frontmatter';
import mediumZoom from '@bytemd/plugin-medium-zoom';
import highlight from '@bytemd/plugin-highlight-ssr';
import math from '@bytemd/plugin-math';
import mermaid from '@bytemd/plugin-mermaid';
import footnotes from '@bytemd/plugin-footnotes';
import { getProcessor } from 'bytemd'
import zhHans from 'bytemd/lib/locales/zh_Hans.json';
import gfmLocale from '@bytemd/plugin-gfm/lib/locales/zh_Hans.json';
import mathLocale from '@bytemd/plugin-math/lib/locales/zh_Hans.json';
import mermaidLocale from '@bytemd/plugin-mermaid/lib/locales/zh_Hans.json';
import themes from '@/utils/theme/theme';

const plugins = [
  gfm({ locale: gfmLocale }),
  highlight(),
  breaks(),
  mediumZoom(),
  frontmatter(),
  math({ locale: mathLocale }),
  mermaid({ locale: mermaidLocale }),
  footnotes(),
  {
    viewerEffect() {
      const $style = document.createElement('style');
      $style.innerHTML = themes.juejin.style;
      document.head.appendChild($style);
      return () => {
        $style.remove();
      };
    },
  },
  // Add more plugins here
];
export default {
  name: 'detail',
  components: {
    Viewer,
  },
  directives: {
    'sticky': VueSticky,
  },
  data() {
    return {
      content: '',
      plugins,
      zhHans,
      articleTitle: '', // 文章标题
      form: {
        link: '', // 链接
        referrer: '',
        description: '',
        category: '',
      },
      hast: [],
      minLevel: 1, //
      headNum: 0,
      itemOffsetTop: [], // 各个id距离顶部的高度
    };
  },
  created() {
    this.getArticleDetail();
  },
  methods: {
    getArticleDetail() {
      const params = {
        id: this.$route.params.id,
      };
      getArticleDetail(params).then((res) => {
        const { data } = res;
        this.content = data.content;
        this.articleTitle = data.title;
        this.form.link = data.link;
        this.form.referrer = data.referrer;
        this.form.description = data.description;
        this.form.category = data.category;
        this.getProcessor()
      }, () => {
      });
    },
    getProcessor() {
      const stringifyHeading = function (e) {
        let result = ''
        visit(e, (node) => {
          if (node.type === 'text') {
            result += node.value
          }
        })
        return result
      }
      getProcessor({
        plugins: [
          {
            rehype: (p) => p.use(() => (tree) => {
              console.log(tree)
              if (tree && tree.children.length) {
                const items = [];
                tree.children.filter(v => v.type === 'element').forEach((node) => {
                  if (node.tagName[0] === 'h' && !!node.children.length) {
                    const i = Number(node.tagName[1])
                    this.minLevel = Math.min(this.minLevel, i)
                    items.push({
                      level: i,
                      text: stringifyHeading(node),
                    })
                  }
                })
                this.hast = items.filter(v => v.level === 1 || v.level === 2 || v.level === 3);
                this.$nextTick(() => {
                  this.transformToId();
                  this.calculateOffTop();
                })
              }
            }),
          },
        ],
      }).processSync(this.content);
    },
    // 计算dom高度
    calculateOffTop() {
      this.throttledScrollHandler = throttle(300, this.onScroll);
      document.querySelector('.home-layout-main').addEventListener('scroll', this.throttledScrollHandler);
      this.hast.forEach((val, i) => {
        const firstHead = document.querySelector(`#head-${i}`);
        this.itemOffsetTop.push({
          key: i,
          top: firstHead.offsetTop,
        });
      });
    },
    onScroll() {
      this.itemOffsetTop = []
      this.hast.forEach((val, i) => {
        const firstHead = document.querySelector(`#head-${i}`);
        this.itemOffsetTop.push({
          key: i,
          top: firstHead.offsetTop,
        });
      });
      const scrollTop = document.querySelector('.home-layout-main').scrollTop;
      let num = 0;
      for (let n = 0; n < this.itemOffsetTop.length; n += 1) {
        if (scrollTop >= this.itemOffsetTop[n].top) {
          num = this.itemOffsetTop[n].key;
        }
      }
      this.headNum = num;
    },
    /* 定位 */
    transformToId() {
      const dom = document.querySelector('.markdown-body');
      let children = Array.from(dom.children);
      if (children.length > 0){
        // current element has children, look deeper
        for (let i = 0; i < children.length; i += 1) {
          const tagName = children[i].tagName;
          if (tagName === 'H1' || tagName === 'H2' || tagName === 'H3') {
            const index = findIndex(this.hast, v => v.text === children[i].textContent);
            if (index >= 0) {
              children[i].setAttribute('id', `head-${index}`);
            }
          }
        }
      }
    },
    /* 定位 */
    locationTxt(item, index) {
      this.headNum = index;
      this.$router.replace({
        query: {
          heading: index,
        }
      });
      this.$nextTick(() => {
        document.querySelector(`#head-${index}`).scrollIntoView({
          behavior: 'smooth',
        });
      });
    },
  },
  mounted() {
  },
};
</script>

<style scoped>

</style>
