<template>
  <el-dialog
    width="80%"
    title="差异化对比"
    :fullscreen="false"
    class="preview-dialog"
    :visible.sync="visible"
    @close="handleClose()"
    append-to-body
    lock-scroll
  >
    <div class="box">
      <div class="left" >
        <div class="title-center">已发布文章</div>
        <div v-html="leftText"></div>
      </div>
      <div class="main">
        <div class="title-center">差异对比</div>
        <div v-html="diffOutput"></div>
      </div>
      <div class="right">
        <div class="title-center">当前文本</div>
        <div v-html="rightText"></div>
      </div>
    </div>
  </el-dialog>
</template>

<script>
import { getArticle } from "@/api/article/article";
import DiffMatchPatch, { diff_match_patch } from "diff-match-patch";
window.diff_match_patch = DiffMatchPatch;

export default {
  name: "ArticleDiffMatchPreview",
  components: {getArticle },

  data() {
    return {
      tagMap: {},
      mapLength: 0,
      unicodeRangeStart: 0xE000,

      visible: false,
      articleId: "",
      leftText: "", //左侧展示文本
      rightText: "", //右侧展示文本
      diffOutput: "",//差异内容描述
    };
  },

  methods: {
    //关闭对比弹窗
    handleClose(done) {
      this.tagMap = {},
      this.mapLength = 0,
      this.leftText = ""; //左侧展示文本
      this.rightText = ""; //右侧展示文本
      return (this.visible = false);
    },
    // 打开对比弹窗
    openPreview(oldContentHtml, contentHtml) {
      this.visible = true;
      this.leftText = oldContentHtml;
      this.rightText = contentHtml;
      // this.getArticle();
      this.dmp = new diff_match_patch();
      this.doDiff(this.leftText, this.rightText);
    },

    doDiff(leftText, rightText) {
      // Go ahead and map nbsp;
      var unicodeCharacter = String.fromCharCode(this.unicodeRangeStart + this.mapLength);
      this.tagMap['&nbsp;'] = unicodeCharacter;
      this.tagMap[unicodeCharacter] = '&nbsp;';
      this.mapLength++;

      var diffableLeft = this.convertHtmlToDiffableString(leftText);
      var diffableRight = this.convertHtmlToDiffableString(rightText);
      var diffs = this.dmp.diff_main(diffableLeft, diffableRight);
      this.dmp.diff_cleanupSemantic(diffs);
      var diffOutput = "";
      for (var x = 0; x < diffs.length; x++) {
        diffs[x][1] = this.insertTagsForOperation(diffs[x][1], diffs[x][0]);
        diffOutput += this.convertDiffableBackToHtml(diffs[x][1]);
      }
      // console.log("diffOutput=======" + diffOutput);
      this.diffOutput = diffOutput;
    },

    convertHtmlToDiffableString(htmlString) {
      var diffableString = "";
      if(htmlString == null || htmlString.length == 0){
        return diffableString;
      }
      htmlString = htmlString.replace(/&nbsp;/g, this.tagMap['&nbsp;']);

      var offset = 0;
      while (offset < htmlString.length) {
        var tagStart = htmlString.indexOf("<", offset);
        if (tagStart < 0) {
          diffableString += htmlString.substr(offset);
          break;
        } else {
          var tagEnd = htmlString.indexOf(">", tagStart);
          if (tagEnd < 0) {
            // Invalid HTML
            // Truncate at the start of the tag
            console.log("Invalid HTML. String will be truncated.");
            diffableString += htmlString.substr(offset, tagStart - offset);
            break;
          }

          var tagString = htmlString.substr(tagStart, tagEnd + 1 - tagStart);

          // Is this tag already mapped?
          var unicodeCharacter = this.tagMap[tagString];
          if (unicodeCharacter === undefined) {
            // Nope, need to map it
            unicodeCharacter = String.fromCharCode(this.unicodeRangeStart + this.mapLength);
            this.tagMap[tagString] = unicodeCharacter;
            this.tagMap[unicodeCharacter] = tagString;
            this.mapLength++;
          }
          // At this point it has been mapped, so now we can use it
          diffableString += htmlString.substr(offset, tagStart - offset);
          diffableString += unicodeCharacter;

          offset = tagEnd + 1;
        }
      }

      return diffableString;
    },

    insertTagsForOperation(diffableString, operation) {
      // Don't insert anything if these are all tags
      var n = -1;
      do {
        n++;
      } while (diffableString.charCodeAt(n) >= this.unicodeRangeStart + 1);
      if (n >= diffableString.length) {
        return diffableString;
      }
      var openTag = "";
      var closeTag = "";
      if (operation === 1) {
        openTag = "<ins style='color: green;'>";
        closeTag = "</ins>";
      } else if (operation === -1) {
        openTag = "<del style='color: red;'>";
        closeTag = "</del>";
      } else {
        return diffableString;
      }
      var outputString = openTag;
      var isOpen = true;
      for (var x = 0; x < diffableString.length; x++) {
        if (diffableString.charCodeAt(x) < this.unicodeRangeStart) {
          // We just hit a regular character. If tag is not open, open it.
          if (!isOpen) {
            outputString += openTag;
            isOpen = true;
          }
          outputString += diffableString[x];
        } else {
          // We just hit one of our mapped unicode characters. Close our tag.
          if (isOpen) {
            outputString += closeTag;
            isOpen = false;
          }
          outputString += diffableString[x];
        }
      }
      if (isOpen) outputString += closeTag;
      return outputString;
    },

    convertDiffableBackToHtml(diffableString) {
      var htmlString = "";
      for (var x = 0; x < diffableString.length; x++) {
        var charCode = diffableString.charCodeAt(x);
        if (charCode < this.unicodeRangeStart) {
          htmlString += diffableString[x];
          continue;
        }
        var tagString = this.tagMap[diffableString[x]];
        if (tagString === undefined) {
          // We somehow have a character that is above our range but didn't map
          // Do we need to add an upper bound or change the range?
          htmlString += diffableString[x];
        } else {
          htmlString += tagString;
        }
      }
      return htmlString;
    },
  },
};
</script>

<style lang="scss">
.box {
  border: 1px solid red($color: #000000);
  width: auto;
  height: auto;
  position: relative;
  overflow: auto;
}
.main {
  position: absolute;
  left: 450px;
  right: 150px;
  top: 0px;
  bottom: 0px;
  width: 400px;
}
.left {
  float: left;
  width: 400px;
  height: 600px;
}
.right {
  float: right;
  width: 400px;
  height: 600px;
}
.title-center {
  text-align: center;
  font-weight: bolder;
  font-size: 20px;
}
</style>