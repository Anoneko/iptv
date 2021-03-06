<template>
  <div class="page">
    <ul id="scale-menu" class="dropdown-content">
      <li @click="ratio=null"><a>自动</a></li>
      <li @click="ratio=0"><a>铺满</a></li>
      <li v-for="ar in allowedAspectRatio" @click="ratio=ar[1]/ar[0]">
        <a>{{ar[0]}}:{{ar[1]}}</a>
      </li>
    </ul>
    <nav>
      <div class="nav-wrapper">
        <router-link :to="backLink" class="button-collapse show-on-large">
          <i class="material-icons">arrow_back</i>
        </router-link>
        <a class="brand-logo channel-title" :class="{center: notMobile}">
          <span class="hide-on-small-only">{{currentChannel.Category}}/</span>{{currentChannel.Name}}
        </a>
        <ul class="right">
          <li v-if="engine" class="hide-on-small-only icon"><a><i class="zmdi" :class="[engineIcon]"></i></a></li>
          <li class="icon" @click="star"><a><i class="material-icons">{{starIcon}}</i></a></li>
          <li v-if="currentEPG" class="hide-on-small-only icon"><a class="modal-trigger" href="#epg-modal" id="epg"><i class="material-icons">playlist_play</i></a></li>
          <li class="hide-on-small-only icon"><a class="modal-trigger" href="#help-modal" id="help"><i class="material-icons">keyboard</i></a></li>
          <li class="icon"><a href="#" class="dropdown-button" data-activates="scale-menu"><i class="material-icons">settings_overscan</i></a></li>
        </ul>
      </div>
    </nav>
    <div class="valign-wrapper player-container grey darken-3">
      <div class="player fp-mute center-align"></div>
    </div>
    <div id="help-modal" class="modal">
      <div class="modal-content">
        <h4>键盘绑定</h4>
        <table class="centered highlight">
          <thead>
            <tr>
              <th data-field="key">按键</th>
              <th data-field="function">功能</th>
            </tr>
          </thead>

          <tbody>
            <tr>
              <td>ESC</td>
              <td>返回列表</td>
            </tr>
            <tr>
              <td>+/-</td>
              <td>增减音量</td>
            </tr>
            <tr>
              <td>m</td>
              <td>切换静音</td>
            </tr>
            <tr>
              <td>左右</td>
              <td>切换频道</td>
            </tr>
            <tr>
              <td>上下</td>
              <td>切换分类</td>
            </tr>
            <tr>
              <td>f</td>
              <td>切换全屏</td>
            </tr>
            <tr>
              <td>?</td>
              <td>查看帮助</td>
            </tr>
            <tr>
              <td>p</td>
              <td>查看节目列表</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <div id="epg-modal" class="modal">
      <div class="modal-content">
        <h4>节目列表</h4>
        <table class="centered responsive-table">
          <thead>
            <tr>
              <th data-field="date">日期</th>
              <th data-field="time">时间</th>
              <th data-field="title">标题</th>
            </tr>
          </thead>

          <tbody>
            <tr v-for="program in currentEPG" :class="{'current-program': program.now}">
              <td>{{program.date}}</td>
              <td>{{program.start}} - {{program.stop}}</td>
              <td>{{program.title}}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
// To activate lite hlsjs plugin of flowplayer
import Hls from 'hls.js';
window.Hls = Hls;

import flowplayer from 'flowplayer';
import jQuery from 'jquery';
import Modernizr from 'modernizr';
import strftime from 'strftime';
import {mapState} from 'vuex';

import swf from 'flowplayer/dist/flowplayer.swf';
import swfHls from 'flowplayer/dist/flowplayerhls.swf';

import {categoryLink, channelLink} from '../route/link.js';

import config from '../../config.json5';

/**
 * whether on mobile devices
 * @return {bool}
 **/
function notMobile() {
  return window.innerWidth > 600;
}

export default {
  props: {
    channel: String,
    from: {
      type: String,
      default: 'channel',
    },
  },
  name: 'hls-player',
  data() {
    return {
      allowedAspectRatio: [
        [16, 9],
        [3, 2],
        [4, 3],
      ],
      engine: '',
      player: null,
      ratio: null,
      notMobile: notMobile(),
    };
  },
  methods: {
    resizeHandler() {
      this.notMobile = notMobile();
    },
    star() {
      this.$store.commit('toggleCollection', this.channel);
    },
    applyRatio() {
      const player = this.$el.querySelector('.fp-player');
      const container = this.$el.querySelector('.player-container');
      const fullscreen = this.player.isFullscreen;
      const containerWidth = container.clientWidth;
      const containerHeight = container.clientHeight;

      if (player === null) return;

      // clear
      Object.assign(player.style, {
        'width': null,
        'height': null,
        'margin-top': null,
        'margin-left': null,
      });
      this.$el.querySelector('video').style['object-fit'] = null;

      if (this.ratio != null) {
        if (this.ratio === 0) {
          player.style.width = player.style.height = '100%';
        } else if (containerHeight / containerWidth > this.ratio) {
          player.style.width = '100%';
          player.style['margin-top'] =
            (containerHeight - containerWidth * this.ratio) / 2
            + 'px';
          player.style.height = containerWidth * this.ratio + 'px';
        } else {
          player.style.height = '100%';
          if (!fullscreen) {
            player.style['margin-left'] =
              (containerWidth - containerHeight / this.ratio) / 2
              + 'px';
          }
          player.style.width = containerHeight / this.ratio + 'px';
        }
        this.$el.querySelector('video').style['object-fit'] = 'fill';
      }
    },
    keyHandler(event) {
      let captured = true;
      // workaround for safari
      switch (event.key || event.keyIdentifier) {
        case 'Esc': // keyIdentifier
        case 'U+001B': // keyIdentifier
        case 'Escape':
          if (!this.player.isFullscreen) {
            this.$router.push(this.backLink);
          }
          break;
        case 'Left': // keyIdentifier
        case 'H': // keyIdentifier
        case 'U+0048': // keyIdentifier
        case 'ArrowLeft':
        case 'h':
          this.switchChannel(-1);
          break;
        case 'Right': // keyIdentifier
        case 'L': // keyIdentifier
        case 'U+004C': // keyIdentifier
        case 'ArrowRight':
        case 'l':
          this.switchChannel(1);
          break;
        case 'Up': // keyIdentifier
        case 'K': // keyIdentifier
        case 'U+004B': // keyIdentifier
        case 'ArrowUp':
        case 'k':
          this.switchCategory(-1);
          break;
        case 'Down': // keyIdentifier
        case 'J': // keyIdentifier
        case 'U+004A': // keyIdentifier
        case 'ArrowDown':
        case 'j':
          this.switchCategory(1);
          break;
        case 'Plus': // keyIdentifier
        case 'U+002B': // keyIdentifier
        case '+':
          this.player.volume(Math.min(this.player.volumeLevel + 0.1, 1));
          break;
        case 'HyphenMinus': // keyIdentifier
        case 'U+002D': // keyIdentifier
        case '-':
          this.player.volume(Math.max(this.player.volumeLevel - 0.1, 0));
          break;
        case 'F': // keyIdentifier
        case 'U+0046': // keyIdentifier
        case 'f':
          this.player.fullscreen();
          break;
        case 'P': // keyIdentifier
        case 'U+0050': // keyIdentifier
        case 'p': {
          const $epgModel = jQuery('#epg-modal');
          $epgModel.modal($epgModel.hasClass('open') ? 'close' : 'open');
          break;
        }
        case 'M': // keyIdentifier
        case 'U+004D': // keyIdentifier
        case 'm': {
          this.player.mute(!this.player.muted);
          break;
        }
        case 'QuestionMark': // KeyIdentifier
        case 'U+003F': // KeyIdentifier
        case '?': {
          const $helpModel = jQuery('#help-modal');
          $helpModel.modal($helpModel.hasClass('open') ? 'close' : 'open');
          break;
        }
        default:
          captured = false;
          console.warn(`Unkown key event: ${event.key}(${event.keyIdentifier})`,
                       event);
      }
      if (captured) {
        event.preventDefault();
      }
    },
    switchChannel(offset) {
      if (this.from === 'channel') {
        const currentCategory = this.channels.Categories[this.categoryIndex];
        const nextChannelIndex = this.channelIndex + offset;
        if (nextChannelIndex >= 0 &&
              nextChannelIndex < currentCategory.Channels.length) {
          this.$router.replace(
            channelLink(
              currentCategory.Channels[nextChannelIndex].Vid, this.from));
        }
      }
    },
    switchCategory(offset) {
      if (this.from === 'channel') {
        const categories = this.channels.Categories;
        const nextCategoryIndex = this.categoryIndex + offset;
        if (nextCategoryIndex >= 0 &&
              nextCategoryIndex < categories.length) {
          const category = categories[nextCategoryIndex];
          this.$router.replace(
            channelLink(category.Channels[0].Vid, this.from));
        }
      }
    },
  },
  computed: {
    starred() {
      return this.$store.getters.inCollection(this.channel);
    },
    currentEPG() {
      const current = this.$store.state.epg[this.channel];
      if (current) {
        return current.map((program) => {
          return {
            date: strftime('%F', new Date(program.start * 1000)),
            start: strftime('%R', new Date(program.start * 1000)),
            stop: strftime('%R', new Date(program.stop * 1000)),
            title: program.title,
            now: program.start < this.now && program.stop > this.now,
          };
        });
      } else {
        return [];
      }
    },
    currentChannel() {
      return this.$store.getters.getChannel(this.channel) || {};
    },
    existedChannel() {
      return this.$store.getters.hasChannel(this.channel);
    },
    categoryIndex() {
      return this.channels.Categories.findIndex((category) => {
        return category.Name === this.currentChannel.Category;
      });
    },
    channelIndex() {
      const category = this.channels.Categories[this.categoryIndex];
      return category.Channels.findIndex((channel) => {
        return channel['Vid'] === this.channel;
      });
    },
    backLink() {
      if (this.from != 'channel') {
        return {
          name: this.from,
        };
      } else {
        const category = this.channels.Categories[this.categoryIndex];
        return category ? categoryLink(category) : {};
      }
    },
    fallbackUrl() {
      if (!this.$store.getters.defaultCategory) {
        // no category to fallback
        return;
      }
      if (this.$store.getters.hasChannel(this.channel)) {
        // no need to fallback
        return;
      }
      return categoryLink(this.$store.getters.defaultCategory);
    },
    clip() {
      return {
        sources: [{
          type: 'application/x-mpegurl',
          src: `${config.hlsUrl}/${this.channel}.m3u8`,
        }],
      };
    },
    starIcon() {
      if (this.starred) {
        return 'star';
      } else {
        return 'star_border';
      }
    },
    engineIcon() {
      return 'zmdi-' + {
        'flash': 'flash',
        'hlsjs': 'language-javascript',
        'hlsjs-lite': 'language-javascript',
        'html5': 'language-html5',
      }[this.engine];
    },
    ...mapState([
      'now',
      'channels',
    ]),
  },
  created() {
    flowplayer((api) => {
      api.on('ready', (e, api, video) => {
        const engineName = api.engine.engineName;
        this.engine = engineName;
      });
      api.on('error', (e, api, error) => {
        if (error.code === 4) {
          // TODO: how about a 404 response?
          this.$emit('unauth');
        }
      });
      api.on('fullscreen', () => {
        if (Modernizr.lockorientation) {
          try {
            Modernizr.prefixed('lockOrientation', screen)('landscape');
          } catch (e) {
            console.warn('lockOrientation is not supported on this browser.');
            // the device does not support rotation
          }
        } else if (Modernizr.orientationlock) {
          screen.orientation.lock('landscape').catch(() => {
            console.warn('orientation.lock is not supported on this chrome.');
          });
        }
        this.applyRatio();
      });
      api.on('fullscreen-exit', () => {
        if (Modernizr.lockorientation) {
          Modernizr.prefixed('unlockOrientation', screen)('landscape');
        } else if (Modernizr.orientationlock) {
          screen.orientation.unlock();
        }
        this.applyRatio();
      });
    });

    if (config.googleAnalytics) {
      flowplayer.conf.analytics = config.googleAnalytics;
    }
  },
  mounted() {
    this.player = flowplayer(this.$el.getElementsByClassName('player')[0], {
      autoplay: this.existedChannel,
      share: false,
      ratio: false,
      keyboard: false,
      chromecast: false,
      live: true,
      swf,
      swfHls,
      hlsjs: {
        xhrSetup: (xhr) => {
          xhr.withCredentials = true;
        },
      },
      clip: this.clip,
    });
    jQuery('.modal').modal();
    jQuery('.dropdown-button').dropdown();
    window.addEventListener('keydown', this.keyHandler);
    window.addEventListener('resize', this.applyRatio);
    window.addEventListener('resize', this.resizeHandler);

    // as second screen
    if (navigator.presentation && navigator.presentation.receiver) {
      navigator.presentation.receiver.connectionList.then((list) => {
        list.connections.map((conn) => {
          conn.onmessage = (msg) => {
            let data = JSON.parse(msg.data);
            if (data.action === 'channel') {
              this.$router.push(channelLink(data.channel.Vid, this.from));
            }
            conn.send('ack');
          };
        });
      });
    }
  },
  watch: {
    clip(val) {
      this.player.load(val);
    },
    ratio(val) {
      this.applyRatio();
    },
    fallbackUrl(val) {
      if (val) {
        this.$router.push(val);
      }
    },
    starredChannels(val) {
      window.localStorage.starredChannels = JSON.stringify(val);
    },
  },
  beforeDestroy() {
    window.removeEventListener('keydown', this.keyHandler);
    window.removeEventListener('resize', this.applyRatio);
    window.removeEventListener('resize', this.resizeHandler);
    this.player.unload();
  },
};
</script>

<style scoped lang="scss">
@import "~materialize-css/sass/components/_color.scss";
@import "~materialize-css/sass/components/_variables.scss";

.page {
  height: 100vh;

  div.nav-wrapper {
    li.icon {
      width: 50px;
      text-align: center;
    }

    a.channel-title {
      left: 60px;
      transform: none;

      @media #{$small-and-down} {
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        max-width: 10ch;
      }

      &.center {
        left: 50%;
        transform: translateX(-50%);
      }
    }
  }

  .player-container {
    justify-content: center;

    @media #{$small-and-down} {
      height: calc(100% - 56px);
    }

    @media #{$medium-and-up} {
      height: calc(100% - 64px);
    }
  }

  div.modal {
    table {
      tr.current-program {
        background-color: #f2f2f2;
      }
    }
  }
}
</style>

<style>
.player {
  width: 100%;
  height: 100%;
}

.flowplayer {
  border-radius: 0!important;
}
</style>
