<template>
  <div class="root">
    <header>
      <div class="reader-nav">
        <a href="/#/cts/">CTS API (Perseids)</a>
        &bull;
        <a href="/#/cite-services/">cite-services</a>
        &bull;
        <a href="/#/morphgnt/">MorphGNT API</a>
      </div>
      <h1>DeepReader (MorphGNT)</h1>
    </header>
    <div class="grid-wrapper">
      <div class="left">
        <book-select :books="books"></book-select>
        <book-info v-if="book"></book-info>
        <verse-lookup></verse-lookup>
        <bookmark-list v-if="user"></bookmark-list>
      </div>

      <div class="main">
        <template v-if="passage">

          <pagination :prev="passageLink(query, passage.prev)" :next="passageLink(query, passage.next)" :title="passage.title"></pagination>

          <div id="text" :class="'textSize-' + this.textSize + (this.colour ? ' colour-' + this.colour : '')">
            <p>
              <div class="word unit" v-for="(word, index) in passage.words" @click="handleWordSelect(word)">
                <span class="verse-num" v-if="word['@id'].slice(-8, -5) == '001'">{{ parseInt(word['@id'].slice(-11, -8)) }}</span>
                <span :id="'w' + index" :class="'txt pos-' + word.pos + ' case-' + word.case">{{ word.text }}</span>
                <br><template v-if="interlinear"><span class="gls">
                  <span class="pos">{{ word.pos }}</span><span v-if="word.mood">.{{ word.tense }}{{ word.voice }}{{ word.mood }}</span><span v-if="word.number">.{{ word.person }}{{ word.case }}{{ word.number }}{{ word.gender }}</span>
                  <br>{{ word.lemma }}
                </span></template>
              </div>
            </p>
          </div>

          <pagination :prev="passageLink(query, passage.prev)" :next="passageLink(query, passage.next)" :title="passage.title"></pagination>

        </template>
        <template v-else>
          <p>
            DeepReader is a highly modular, Vue.js-based online reading
            environment designed for deep reading of texts with integrated
            learning tools.
          </p>

          <p>
            It is particulary intended for the study of classical languages
            such as Ancient Greek but could be applied to any texts with rich
            annotations. What is here is an early prototype using the MorphGNT
            API and the CTS protocol but we plan to support other text services
            as well.
          </p>

          <p>
            If you hover over the reader, you'll see various pluggable widgets
            on the left and right. Those on the left are used to choose what
            passage to read, and those on the right are used to present
            additional information about the passage and its individual words,
            and to control the appearance of the passage.
          </p>

          <p>
            You can expand or collapse any widget by clicking on its title. You
            can use the arrow keys on your keyboard to pagination between
            passages in a work.
          </p>

          <p>
            Each widget is a separate Vue.js component. We are working to make
            it as simple as possible to develop new widgets that interact and
            engage with the current passage, optionally calling out to external
            APIs.
          </p>

          <p>
            We are also experimenting with Firebase for persistence. Offline
            use is also planned as is packaging DeepReader up as an app for
            mobile use.
          </p>
        </template>
      </div>
      <div class="right">
        <text-formatting></text-formatting>
        <text-colouring></text-colouring>
        <interlinear></interlinear>
        <frequency></frequency>
        <word-info></word-info>
        <morpheus></morpheus>
        <kwic></kwic>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import Morpheus from '@/components/Morpheus';
import Pagination from '@/components/Pagination';
import TextFormatting from '@/components/TextFormatting';
import BookmarkList from '@/components/BookmarkList';

import morphgnt from '@/morphgnt';
import BookSelect from '@/morphgnt/components/BookSelect';
import WordInfo from '@/morphgnt/components/WordInfo';
import BookInfo from '@/morphgnt/components/BookInfo';
import VerseLookup from '@/morphgnt/components/VerseLookup';
import Interlinear from '@/morphgnt/components/Interlinear';
import TextColouring from '@/morphgnt/components/TextColouring';
import Frequency from '@/morphgnt/components/Frequency';
import Kwic from '@/morphgnt/components/Kwic';

export default {
  name: 'reader',
  created() {
    this.fetchData();
  },
  mounted() {
    window.addEventListener('keyup', this.handleKeyUp);
  },
  beforeDestroy() {
    window.removeEventListener('keyup', this.handleKeyUp);
  },
  data() {
    return {
      query: {},
      books: [],
    };
  },
  computed: mapGetters(['user', 'book', 'passage', 'textSize', 'interlinear', 'colour']),
  watch: {
    $route: 'fetchData',
  },
  methods: {
    fetchData() {
      this.asyncData({ query: this.$route.query }).then(({ query, books, book, passage }) => {
        if (book && !passage) {
          this.$router.push(this.passageLink(query, book.first_paragraph));
        } else {
          this.query = query;
          this.books = books;
          this.$store.commit('setReader', { book, passage });
        }
      });
    },
    async asyncData({ query }) {
      let book = null;
      let passage = null;
      const books = await morphgnt.books();
      if (query.passage) {
        passage = await morphgnt.resource(query.passage);
        book = await morphgnt.resource(passage.book);
      } else if (query.book) {
        book = await morphgnt.resource(query.book);
      }
      return { query, books, book, passage };
    },
    passageLink(query, passage) {
      if (passage) {
        delete query.book;
        return {
          query: Object.assign({}, query, { passage }),
        };
      }
      return null;
    },
    handleKeyUp(e) {
      if (e.key === 'ArrowLeft') {
        if (this.passage.prev) {
          this.$router.push(this.passageLink(this.query, this.passage.prev));
        }
      } else if (e.key === 'ArrowRight') {
        if (this.passage.next) {
          this.$router.push(this.passageLink(this.query, this.passage.next));
        }
      }
    },
    handleWordSelect(word) {
      this.$emit('word-select', word);
      this.$emit('word-select2', word);
    },
  },
  components: {
    Pagination,
    BookSelect,
    WordInfo,
    BookInfo,
    Morpheus,
    BookmarkList,
    VerseLookup,
    TextFormatting,
    Interlinear,
    TextColouring,
    Frequency,
    Kwic,
  },
};
</script>

<style lang="scss">

  /* variables */

  $main-font-family: "Skolar";
  $widget-font-family: "PT Sans", $main-font-family;

  /* hover opacity */

  .widget, .root > header, .page-nav-1 {
    opacity: 0;
    transition: opacity 0.5s ease-in-out;
  }

  body:hover {
    .widget, .root > header, .page-nav-1 {
      opacity: 1;
    }
  }

  /* grid */

  .grid-wrapper {
    display: grid;
    grid-template-columns: 2fr 6fr 4fr;
    grid-column-gap: 10px;
    margin: 10px;
    > * {
      min-width: 200px;
    }
  }

  /* header */

  .root > header {
    background: #F7F7F7;
    padding: 10px 20px;
    > h1 {
      font-size: 24pt;
      font-family: $main-font-family;
      margin: 0;
      font-weight: normal;
      color: #444;
    }
    .reader-nav {
      float: right;
      font-family: $widget-font-family;
      color: #999;
      a {
        text-decoration: none;
        color: inherit;
        cursor: pointer;
        &:hover {
          color: #000;
        }
      }
    }
  }

  /* main column */

  .main {
    font-family: $main-font-family;
    height: 500px;
    margin: 20px 50px;
    > p:first-of-type {
      margin-top: 0;
    }
  }

  /* text */

  #text {
    clear: both;
    word-spacing: 0.3em;
    color: #333;
    .word {
      cursor: pointer;
      .verse-num {
        color: #999;
        font-family: $widget-font-family;
        font-size: 60%;
      }
    }
    &.textSize-small {
      font-size: 14pt;
    }
    &.textSize-normal {
      font-size: 16pt;
    }
    &.textSize-large {
      font-size: 20pt;
    }

    div.unit {
      display: inline-block;
      margin-bottom: 0.5em;
    }
    .txt {
      display: inline-block;
    }
    .gls {
      display: inline-block;
      font-size: 75%;
      color: gray;
    }

    &.colour-pos {
      .pos-N, .pos-A {
        color: #C00;
      }
      .pos-RA, .pos-RD, .pos-RI, .pos-RP, .pos-RR {
        color: #C50;
      }
      .pos-V {
        color: #00C;
      }
    }
    &.colour-case {
      .case-N {
        color: #C00;
      }
      .case-G {
        color: #9C6;
      }
      .case-D {
        color: #6CC;
      }
      .case-A {
        color: #FC0;
      }
    }

    .freq-0 {
      background: #fff;
    }
    .freq-1 {
      background: #ffe;
    }
    .freq-2 {
      background: #ffd;
    }
    .freq-3 {
      background: #ffc;
    }
    .freq-4 {
      background: #ffb;
    }
    .freq-5 {
      background: #ffa;
    }
    .freq-6 {
      background: #ff9;
    }
    .freq-7 {
      background: #ff8;
    }
    .freq-8 {
      background: #ff7;
    }
    .freq-9 {
      background: #ff6;
    }
    .freq-10 {
      background: #ff5;
    }
  }

  /* widgets */

  .widget {
    margin-bottom: 10px;
    font-family: $widget-font-family;
    color: #666;
    transition: color 0.2s ease-in-out, opacity 0.5s ease-in-out;
    background: #F7F7F7;
    font-size: 9pt;
    > header {
      cursor: pointer;
      background: #EEE;
      transition: background 0.2s ease-in-out, opacity 0.5s ease-in-out;
      padding: 5px 10px;
      font-weight: bold;
      .summary {
        float: right;
        font-weight: normal;
      }
    }
    > section {
      padding: 10px 10px;
      ul {
        list-style-type: none;
        margin: 0;
        padding: 0;
      }
      .textSize-small {
        font-family: $main-font-family;
        font-size: 14pt;
      }
      .textSize-normal {
        font-family: $main-font-family;
        font-size: 16pt;
      }
      .textSize-large {
        font-family: $main-font-family;
        font-size: 20pt;
      }
    }
    &:hover {
      color: #000;
      > header {
        background: #DDD;
      }
    }
  }

  /* pagination */

  div.page-nav-1 {

    display: flex;
    justify-content: space-between;

    overflow: auto;

    font-family: $widget-font-family;

    > div {
      &.prev {
        width: 30px;  // fix width so takes up space even without link
        float: left;
        text-align: left;
      }
      &.next {
        width: 30px;  // fix width so takes up space even without link
        float: right;
        text-align: right;
      }
      line-height: 20pt;
    }

    .title {
      text-align: center;
    }

    a {
      font-weight: bold;
      font-size: 20pt;
      text-decoration: none;
      color: #999;
      cursor: pointer;
    }

    a:hover {
      color: #000;
    }
  }
</style>
