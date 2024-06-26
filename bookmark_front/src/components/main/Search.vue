<template>
  <div class="search">
    <div :class="{ listShow: focused && list.length > 0 }" class="newSearch">
      <input ref="searchInput" class="input" type="text" v-model="value" @keydown="keyPress" @focus="inputFocus"
             @blur="inputBlur" />
      <div class="action">
        <a-dropdown :trigger="['click']">
          <a-tooltip title="点击切换网页搜索">
            <my-icon class="icon" style="margin-right: 0.5em" :type="checkedSearchEngine.icon"
                     @click="searchIconClick" />
          </a-tooltip>
          <a-menu slot="overlay" @click="searchEngineChange">
            <a-menu-item v-for="item in searchEngineList" :key="item.id">{{ item.name }}</a-menu-item>
          </a-menu>
        </a-dropdown>
        <a-icon class="icon" type="search" @click="submit(true)" />
      </div>
    </div>
    <div v-if="focused && list.length > 0" class="searchContent">
      <div
        class="listItem"
        :class="{ itemActive: index == selectIndex }"
        v-for="(item, index) in list"
        :key="item.bookmarkId"
        @mousedown="itemClick(index)"
        :id="'bookmark:' + item.bookmarkId"
      >
        <div class="name" target="_blank" :title="item.url.substr(0, 50)">
          {{ item.name }}
        </div>
        <div class="icons">
          <a-tooltip title="定位到书签树中" v-if="showLocation">
            <my-icon style="color: white; font-size: 1.3em" type="icon-et-location"
                     @mousedown="location($event, item)" />
          </a-tooltip>
          <a-tooltip title="复制链接">
            <a-icon
              type="copy"
              style="color: white; font-size: 1.3em; padding-left: 0.5em"
              class="search-copy-to-board"
              @mousedown.prevent="copy($event, item)"
              :data="item.url"
            />
          </a-tooltip>
          <a-icon
            v-if="!homePinBookmarkIdMap[item.bookmarkId]"
            style="color: white; font-size: 1.3em; padding-left: 0.5em"
            type="pushpin"
            title="固定到首页"
            @mousedown.prevent="pinBookmark($event, item)"
          />
        </div>
      </div>
    </div>
    <a ref="targetA" style="left: 1000000px" />
  </div>
</template>

<script>
import HttpUtil from "@/util/HttpUtil";
import { mapState } from "vuex";
import ClipboardJS from "clipboard";
import { GLOBAL_CONFIG, USER_INFO } from "@/store/modules/globalConfig";
import { TREE_DATA, refreshHomePinList, HOME_PIN_BOOKMARK_ID_MAP } from "@/store/modules/treeData";

export default {
  name: "Search",
  props: {
    showLocation: Boolean //是否显示定位等按钮
  },
  data() {
    return {
      value: "",
      focused: false,
      list: [],
      //上下选中
      selectIndex: null,
      copyBoard: null, //剪贴板对象
      searchEngineList: [],
      checkedSearchEngine: { icon: "icon-baidu", name: "百度", url: "https://www.baidu.com/s?ie=UTF-8&wd=%s" }
    };
  },
  async mounted() {
    //初始化clipboard
    this.copyBoard = new ClipboardJS(".search-copy-to-board", {
      text: function(trigger) {
        return trigger.attributes.data.nodeValue;
      }
    });
    this.copyBoard.on("success", (e) => {
      this.$message.success("复制成功");
      e.clearSelection();
    });
    if (this.$store.state.globalConfig.token != null) {
      this.searchEngineList = await HttpUtil.get("/searchEngine/list");
      this.checkedSearchEngine = this.searchEngineList.find(item => item.checked === 1);
    }
  },
  destroyed() {
    if (this.copyBoard != null) {
      this.copyBoard.destroy();
    }
  },
  computed: {
    ...mapState("treeData", ["totalTreeData", HOME_PIN_BOOKMARK_ID_MAP]),
    ...mapState("globalConfig", ["userInfo"])
  },
  watch: {
    value(newVal, oldVal) {
      this.search(newVal);
    }
  },
  methods: {
    searchIconClick() {
      if (this.userInfo == null) {
        this.searchEngineList = [];
        this.$message.warning("未登录，请登录后操作");
      }
    },
    search(content) {
      console.log(content);
      let val = content.toLocaleLowerCase().trim();
      if (val === "" || this.userInfo == null) {
        this.list = [];
      } else {
        this.list = this.dealSearch(val);
      }
      this.selectIndex = null;
    },
    //下方列表点击
    itemClick(index) {
      this.selectIndex = index;
      this.submit();
    },
    //搜索或者跳转到书签
    submit(forceSearch) {
      let url;
      if (forceSearch || this.selectIndex == null) {
        //说明使用网页搜索
        url = this.checkedSearchEngine.url.replace("%s", encodeURIComponent(this.value));
      } else {
        //说明跳转到书签
        let bookmark = this.list[this.selectIndex];
        HttpUtil.post("/bookmark/visitNum", { id: bookmark.bookmarkId });
        url = bookmark.url;
      }
      let a = this.$refs["targetA"];
      a.href = url;
      a.click();
    },
    inputBlur() {
      this.focused = false;
    },
    inputFocus() {
      this.focused = true;
    },
    //定位到书签树中
    location(event, item) {
      this.$emit("location", item);
      event.stopPropagation();
    },
    /**
     * 键盘事件处理
     */
    keyPress(e) {
      if (e.key === "Enter") {
        this.submit();
        return this.stopDefault();
      } else if (this.list.length > 0) {
        if (e.key === "ArrowUp") {
          this.selectIndex = this.selectIndex == null ? this.list.length - 1 : this.selectIndex == 0 ? null : this.selectIndex - 1;
          return this.stopDefault();
        } else if (e.key === "ArrowDown" || e.key === "Tab") {
          this.selectIndex = this.selectIndex == null ? 0 : this.selectIndex == this.list.length - 1 ? null : this.selectIndex + 1;
          return this.stopDefault();
        }
      }
    },
    //修改默认搜索引擎
    async searchEngineChange(item) {

      let target = this.searchEngineList.find(one => one.id === item.key);
      await HttpUtil.post("/searchEngine/setChecked", null, { id: item.key });
      this.checkedSearchEngine = target;
    },
    //固定书签到首页
    async pinBookmark(event, { bookmarkId }) {
      this.stopDefault(event);
      await HttpUtil.put("/home/pin", null, { bookmarkId });
      await this.$store.dispatch(TREE_DATA + "/" + refreshHomePinList);
    },
    dealSearch(content) {
      let res = [];
      let arrs = Object.values(this.totalTreeData);
      for (let i1 = 0, length1 = arrs.length; i1 < length1; i1++) {
        for (let i2 = 0, length2 = arrs[i1].length; i2 < length2; i2++) {
          let item = arrs[i1][i2];
          if (item.type === 1) {
            continue;
          }
          if (item.searchKey.indexOf(content) > -1) {
            res.push(item);
            if (res.length >= 12) {
              return res;
            }
          }
        }
      }
      return res;
    },
    //复制
    async copy(event) {
      return this.stopDefault(event);
    },
    /**
     * 阻止默认事件
     */
    stopDefault(e) {
      //阻止默认浏览器动作(W3C)
      if (e && e.preventDefault) {
        e.preventDefault();
        e.stopPropagation();
      } else {
        window.event.returnValue = false;
      }
      console.log("阻止成功");
      return false;
    }
  }
};
</script>

<style lang="less" scoped>
@inputBgColor: rgb(74, 74, 74);
@listBgColor: #2b2b2b;
@textColor: white;
@listActiveBgColor: #454545;
.search {
  position: relative;

  .listShow {
    border-bottom-left-radius: 0 !important;
    border-bottom-right-radius: 0 !important;
  }

  .newSearch {
    display: flex;
    align-items: center;
    background-color: @inputBgColor;
    border-radius: 0.18rem;
    overflow: hidden;
    font-size: 1.2em;
    color: @textColor;

    .input {
      flex: 1;
      border: 0;
      background-color: @inputBgColor;
      padding: 0.1rem;
      padding-left: 0.19rem;
      outline: none;
    }

    .action {
      padding: 0.1rem;
      padding-right: 0.19rem;
      display: flex;
      align-items: center;

      .icon {
        color: @textColor;
        cursor: pointer;
        font-size: 1.3em;
      }
    }
  }

  .searchContent {
    position: absolute;
    width: 100%;
    background: @listBgColor;
    z-index: 1;
    border-top: 1px solid white;
    border-bottom-left-radius: 0.18rem;
    border-bottom-right-radius: 0.18rem;
    overflow: hidden;

    .listItem {
      font-size: 0.16rem;
      display: flex;
      align-items: center;
      height: 0.3rem;
      line-height: 0.3rem;
      color: @textColor;
      margin: 0.05rem 0 0.05rem 0;
      padding: 0 0.19rem 0 0.19rem;
      cursor: pointer;

      .name {
        padding-right: 1em;
        max-width: calc(100% - 2em);
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      .icons {
        display: none;
        align-items: center;
      }
    }

    .listItem:hover {
      background-color: @listActiveBgColor;
    }

    .itemActive {
      background-color: @listActiveBgColor;
    }

    .listItem:hover .icons {
      display: flex;
    }
  }
}
</style>
