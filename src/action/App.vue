<template>
<div id="app" v-show="dataLoaded">
  <div class="header">
    <div class="title">
      {{ getText('extensionName') }}
    </div>
    <div class="header-buttons">
      <img class="contribute-icon"
          src="/src/contribute/assets/heart.svg"
          @click="showContribute">
      <img class="settings-icon" src="/src/icons/misc/settings.svg"
          @click="showSettings = !showSettings"/>
    </div>
  </div>

  <transition name="settings"
      @after-enter="handleSizeChange"
      @after-leave="handleSizeChange">
    <div class="settings" v-show="showSettings">
      <v-textfield v-model="customUrl"
          :placeholder="getText('inputPlaceholder_customURL')"
          :fullwidth="true">
      </v-textfield>
    </div>
  </transition>

  <div class="list-padding-top"></div>
  <ul class="mdc-list list list-bulk-button" v-if="searchAllEngines">
    <li class="mdc-list-item list-item ripple-surface"
        @click="selectEngine('allEngines')">
      <img class="mdc-list-item__graphic list-item-icon"
          :src="getEngineIcon('allEngines')">
      {{ getText('engineName_allEngines_full') }}
    </li>
  </ul>
  <ul class="mdc-list list list-separator"
      v-if="searchAllEngines || hasScrollBar">
    <li role="separator" class="mdc-list-divider"></li>
  </ul>
  <div class="list-items-wrap" ref="items" :class="listClasses">
    <resize-observer @notify="handleSizeChange"></resize-observer>
    <ul class="mdc-list list list-items">
      <li class="mdc-list-item list-item ripple-surface"
          v-for="engine in engines"
          :key="engine.id"
          @click="selectEngine(engine)">
        <img class="mdc-list-item__graphic list-item-icon"
            :src="getEngineIcon(engine)">
        {{ getText(`engineName_${engine}_short`) }}
      </li>
    </ul>
  </div>

</div>
</template>

<script>
import browser from 'webextension-polyfill';
import {ResizeObserver} from 'vue-resize';
import {TextField} from 'ext-components';

import storage from 'storage/storage';
import {
  getEnabledEngines,
  showNotification,
  validateUrl,
  showContributePage
} from 'utils/app';
import {getText, isAndroid} from 'utils/common';
import {targetEnv} from 'utils/config';

export default {
  components: {
    [TextField.name]: TextField,
    [ResizeObserver.name]: ResizeObserver
  },

  data: function() {
    return {
      dataLoaded: false,

      showSettings: false,
      hasScrollBar: false,
      isPopup: false,

      engines: [],
      searchAllEngines: false,
      customUrl: ''
    };
  },

  computed: {
    listClasses: function() {
      return {
        'list-items-max-height': this.isPopup
      };
    }
  },

  methods: {
    getText,

    getEngineIcon: function(engine) {
      if (engine === 'googleText') {
        engine = 'google';
      }
      let ext = 'svg';
      if (['gigablast', 'megalodon'].includes(engine)) {
        ext = 'png';
      }
      return `/src/icons/engines/${engine}.${ext}`;
    },

    selectEngine: async function(engine) {
      let customUrl = this.customUrl;
      if (customUrl) {
        customUrl = customUrl.trim();
        if (!validateUrl(customUrl)) {
          showNotification({messageId: 'error_invalidUrl'});
          return;
        }
      }
      browser.runtime.sendMessage({
        id: 'actionPopupSubmit',
        customUrl,
        engine
      });

      this.closeAction();
    },

    showContribute: async function() {
      await showContributePage();
      this.closeAction();
    },

    closeAction: async function() {
      if (!this.isPopup) {
        browser.tabs.remove((await browser.tabs.getCurrent()).id);
      } else {
        window.close();
      }
    },

    handleSizeChange: function() {
      const items = this.$refs.items;
      this.hasScrollBar = items.scrollHeight > items.clientHeight;
    }
  },

  created: async function() {
    const currentTab = await browser.tabs.getCurrent();
    this.isPopup = !currentTab || currentTab instanceof Array;
    if (!this.isPopup) {
      document.documentElement.style.height = '100%';
      document.body.style.minWidth = 'initial';
    }

    const options = await storage.get(
      ['engines', 'disabledEngines', 'searchAllEnginesAction'],
      'sync'
    );

    const enEngines = await getEnabledEngines(options);

    if (
      targetEnv === 'firefox' &&
      (await isAndroid()) &&
      (enEngines.length <= 1 || options.searchAllEnginesAction === 'main')
    ) {
      // Removing the action popup has no effect on Android
      showNotification({messageId: 'error_optionsNotApplied'});
      return;
    }

    this.searchAllEngines = options.searchAllEnginesAction === 'sub';
    this.engines = enEngines;

    this.dataLoaded = true;
  }
};
</script>

<style lang="scss">
$mdc-theme-primary: #1abc9c;

@import '@material/list/mdc-list';
@import '@material/theme/mixins';
@import '@material/typography/mixins';
@import '@material/ripple/mixins';

@import 'vue-resize/dist/vue-resize';

body,
#app {
  height: 100%;
}

#app {
  display: flex;
  flex-direction: column;
}

body {
  margin: 0;
  min-width: 413px;
  overflow: hidden;
  @include mdc-typography-base;
  font-size: 100%;
  background-color: #ffffff;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  white-space: nowrap;
  padding-top: 16px;
  padding-left: 16px;
  padding-right: 16px;
}

.title {
  overflow: hidden;
  text-overflow: ellipsis;
  @include mdc-typography('title');
  @include mdc-theme-prop('color', 'text-primary-on-light');
}

.header-buttons {
  display: flex;
  align-items: center;
  margin-left: 56px;
  @media (max-width: 412px) {
    margin-left: 32px;
  }
}

.contribute-icon {
  margin-right: 16px;
  cursor: pointer;
}

.settings-icon {
  cursor: pointer;
}

.settings {
  padding: 16px;
}

.settings-enter-active,
.settings-leave-active {
  max-height: 100px;
  padding-top: 16px;
  padding-bottom: 16px;
  transition: max-height 0.3s ease, padding-top 0.3s ease,
    padding-bottom 0.3s ease, opacity 0.2s ease;
}

.settings-enter,
.settings-leave-to {
  max-height: 0;
  padding-top: 0;
  padding-bottom: 0;
  opacity: 0;
}

.list {
  padding: 0 !important;
}

.list-padding-top {
  margin-bottom: 8px;
}

.list-bulk-button {
  height: 48px;
}

.list-separator {
  height: 1px;
}

.list-items-wrap {
  overflow-y: auto;
}

.list-items-max-height {
  max-height: 392px;
}

.list-items {
  padding-bottom: 8px !important;
}

.list-item {
  padding-left: 16px;
  padding-right: 48px;
  cursor: pointer;
}

.list-item-icon {
  margin-right: 16px !important;
}

.ripple-surface {
  @include mdc-ripple-surface;
  @include mdc-ripple-radius-bounded;
  @include mdc-states;

  position: sticky;
  outline: none;
  overflow: hidden;
}
</style>
