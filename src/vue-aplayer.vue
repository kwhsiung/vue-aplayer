<template>
  <div
    class="aplayer"
    :class="{
      'aplayer-narrow': isMiniMode,
      'aplayer-withlist' : !isMiniMode && musicList.length > 0,
      'aplayer-withlrc': !isMiniMode && (!!$slots.display || shouldShowLrc),
      'aplayer-float': isFloatMode,
      'aplayer-loading': isPlaying && isLoading
    }"
    :style="floatStyleObj"
  >
    <div class="aplayer-body">
      <thumbnail
        :pic="currentMusic.pic"
        :playing="isPlaying"
        :enable-drag="isFloatMode"
        :theme="currentTheme"
        @toggleplay="toggle"
        @dragbegin="onDragBegin"
        @dragging="onDragAround"
      />
      <div class="aplayer-info" v-show="!isMiniMode">
        <div class="aplayer-music">
          <span class="aplayer-title">{{ currentMusic.title || 'Untitled' }}</span>
          <span class="aplayer-author">{{ currentMusic.artist || currentMusic.author || 'Unknown' }}</span>
        </div>
        <slot name="display" :current-music="currentMusic" :play-stat="playStat">
          <lyrics :current-music="currentMusic" :play-stat="playStat" v-if="shouldShowLrc" />
        </slot>
        <controls
          :shuffle="shouldShuffle"
          :repeat="repeatMode"
          :stat="playStat"
          :volume="audioVolume"
          :muted="isAudioMuted"
          :theme="currentTheme"
          @toggleshuffle="shouldShuffle = !shouldShuffle"
          @togglelist="showList = !showList"
          @togglemute="toggleMute"
          @setvolume="setAudioVolume"
          @dragbegin="onProgressDragBegin"
          @dragend="onProgressDragEnd"
          @dragging="onProgressDragging"
          @nextmode="setNextMode"
        />
      </div>
    </div>
    <audio ref="audio"></audio>
    <music-list
      :show="showList && !isMiniMode"
      :current-music="currentMusic"
      :music-list="musicList"
      :play-index="playIndex"
      :listmaxheight="listmaxheight || listMaxHeight"
      :theme="currentTheme"
      @selectsong="onSelectSong"
    />
  </div>
</template>
<script type="text/babel">
  import Vue from 'vue'
  import Thumbnail from './components/aplayer-thumbnail.vue'
  import MusicList from './components/aplayer-list.vue'
  import Controls from './components/aplayer-controller.vue'
  import Lyrics from './components/aplayer-lrc.vue'
  import { deprecatedProp, versionCompare, warn } from './utils'
  const debug = require('debug')('VUE:vue-aplayer.vue')

  // NOTE: done
  // 瀏覽器 console log 絢麗的版本提示顯示過了沒
  let versionBadgePrinted = false

  // NOTE: done
  // 版本檢查器
  const canUseSync = versionCompare(Vue.version, '2.3.0') >= 0

  // NOTE: done
  // cache color-thief.js 抓來的顏色資料
  /**
   * memorize self-adapting theme for cover image urls
   * @type {Object.<url, rgb()>}
   */
  const picThemeCache = {}

  // NOTE: mutual exclusion
  // 作用: Pause other players when this player is playing
  /* 原理: vue component 其實是透過 export default export 出去的 module
  **      所以此 module 內的 global variable 會共用
  **      利用這個特性，我們可以讓所有 imported components 共享變數
  */
  // mutex playing instance
  let activeMutex = null

  // TODO: 一隻半解
  // 存放 mappings 的地方
  const REPEAT = {
    NONE: 'none',
    MUSIC: 'music',
    LIST: 'list',
    NO_REPEAT: 'no-repeat',
    REPEAT_ONE: 'repeat-one',
    REPEAT_ALL: 'repeat-all',
  };

  const VueAPlayer = {
    name: 'APlayer',

    // NOTE: done
    // 控制瀏覽器 console log 絢麗的版本提示的開關
    disableVersionBadge: false,
    components: {
      Thumbnail,
      Controls,
      MusicList,
      Lyrics,
    },
    props: {
      // Props
      // https://github.com/SevenOutman/vue-aplayer/blob/develop/docs/README.md#props

      // Music info for current playing music, see Music info
      music: {
        type: Object,
        required: true,
        validator (song) {
          if (song.url) {
            deprecatedProp('music.url', '1.4.0', 'music.src')
          }
          if (song.author) {
            deprecatedProp('music.author', '1.4.1', 'music.artist')
          }
          return song.src || song.url
        },
      },

      // Music list to play and display. If list is not empty, music list panel will be shown, even if the only song in the list is identical to music prop.
      list: {
        type: Array,
        default () {
          return []
        },
      },

      // Mini mode
      mini: {
        type: Boolean,
        default: false,
      },

      // Whether to show lyrics or not
      showLrc: {
        type: Boolean,
        default: false,
      },

      // Pause other players when this player is playing
      mutex: {
        type: Boolean,
        default: true,
      },

      // Theme color, will be overridden by current music's theme if set
      theme: {
        type: String,
        default: '#41b883',
      },

      // Max height of play list
      listMaxHeight: String,

      // Fold playlist initially
      /**
       * @since 1.4.1
       * Fold playlist initially
       */
      listFolded: {
        type: Boolean,
        default: false,
      },

      // Float mode, in which you can drag the player around and leave it anywhere on your page
      /**
       * @since 1.2.0 Float mode
       */
      float: {
        type: Boolean,
        default: false,
      },

      // Audio attributes as props
      // since 1.4.0
      // autoplay controls muted preload volume
      // autoplay is not observable
      // https://github.com/SevenOutman/vue-aplayer/blob/develop/docs/README.md#audio-attributes-as-props

      // Starts playing as soon as loaded. If more than one mutex player are set autoplay, only the first one will play.
      /**
       * @since 1.4.0
       * not observable
       */
      autoplay: {
        type: Boolean,
        default: false,
      },

      // Shows native audio controls below the player, or between player and playlist. This only works in develop environment, and doesn't work in Mini mode.
      /**
       * @since 1.4.0
       * whether to show native audio controls below Vue-APlayer
       * only work in development environment and not mini mode
       *
       * observable
       */
      controls: {
        type: Boolean,
        default: false,
      },

      // Player is muted.
      /**
       * @since 1.4.0
       * observable, sync
       */
      muted: {
        type: Boolean,
        default: false,
      },

      // The way to load music, can be 'none' 'metadata' or 'auto'
      /**
       * @since 1.4.0
       * observable
       */
      preload: String,

      // Playback volume.
      /**
       * @since 1.4.0
       * observable, sync
       */
      volume: {
        type: Number,
        default: 0.8,
        validator (value) {
          return value >= 0 && value <= 1
        },
      },

      // play order control
      // since 1.5.0

      /**
       * @since 1.5.0
       * @see https://support.apple.com/en-us/HT207230
       * twoWay
       */
      shuffle: {
        type: Boolean,
        default: false,
      },
      /**
       * @since 1.5.0
       * @see https://support.apple.com/en-us/HT207230
       * twoWay
       */
      repeat: {
        type: String,
        default: REPEAT.NO_REPEAT,
      },


      // deprecated props

      /**
       * @deprecated since 1.1.2, use listMaxHeight instead
       */
      listmaxheight: {
        type: String,
        validator (value) {
          if (value) {
            deprecatedProp('listmaxheight', '1.1.2', 'listMaxHeight')
          }
          return true
        },
      },
      /**
       * @deprecated since 1.1.2, use mini instead
       */
      narrow: {
        type: Boolean,
        default: false,
        validator (value) {
          if (value) {
            deprecatedProp('narrow', '1.1.2', 'mini')
          }
          return true
        },
      },
      /**
       * @deprecated since 1.2.2
       */
      showlrc: {
        type: Boolean,
        default: false,
        validator (value) {
          if (value) {
            deprecatedProp('showlrc', '1.2.2', 'showLrc')
          }
          return true
        },
      },
      /**
       * @deprecated and REMOVED since 1.5.0
       */
      // mode: {
      //   type: String,
      //   default: 'circulation',
      //   validator (value) {
      //     if (value) {
      //       deprecatedProp('mode', '1.5.0', 'shuffle and repeat')
      //     }
      //     return true
      //   }
      // },
    },
    data () {
      return {
        internalMusic: this.music,
        isPlaying: false,
        isSeeking: false,
        wasPlayingBeforeSeeking: false,
        isMobile: /mobile/i.test(window.navigator.userAgent),
        playStat: {
          duration: 0,
          loadedTime: 0,
          playedTime: 0,
        },
        showList: !this.listFolded,

        // handle Promise returned from audio.play()
        // @see https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/play
        audioPlayPromise: Promise.resolve(),


        // @since 1.2.0 float mode

        floatOriginX: 0,
        floatOriginY: 0,
        floatOffsetLeft: 0,
        floatOffsetTop: 0,


        // @since 1.3.0 self adapting theme
        selfAdaptingTheme: null,


        // @since 1.4.0
        // sync muted, volume

        internalMuted: this.muted,
        internalVolume: this.volume,

        // @since 1.4.1
        // Loading indicator
        isLoading: false,


        // @since 1.5.1
        // sync shuffle, repeat
        internalShuffle: this.shuffle,
        internalRepeat: this.repeat,
        // for shuffling
        shuffledList: [],
      }
    },
    computed: {
      // alias for $refs.audio
      audio () {
        return this.$refs.audio
      },

      // sync music
      currentMusic: {
        get () {
          return this.internalMusic
        },
        set (val) {
          canUseSync && this.$emit('update:music', val)
          this.internalMusic = val
        },
      },
      // compatible for deprecated props
      isMiniMode () {
        return this.mini || this.narrow
      },
      shouldShowLrc () {
        return this.showLrc || this.showlrc
      },

      // props wrappers

      currentTheme () {
        return this.selfAdaptingTheme || this.currentMusic.theme || this.theme
      },
      isFloatMode () {
        return this.float && !this.isMobile
      },
      shouldAutoplay () {
        if (this.isMobile) return false
        return this.autoplay
      },
      musicList () {
        return this.list
      },
      shouldShowNativeControls () {
        return process.env.NODE_ENV !== 'production' &&
          this.controls &&
          !this.isMiniMode
      },

      // useful

      floatStyleObj () {
        // transform: translate(floatOffsetLeft, floatOffsetY)
        return {
          transform: `translate(${this.floatOffsetLeft}px, ${this.floatOffsetTop}px)`,
          webkitTransform: `translate(${this.floatOffsetLeft}px, ${this.floatOffsetTop}px)`,
        }
      },
      currentPicStyleObj () {
        if (this.currentMusic && this.currentMusic.pic) {
          return {
            backgroundImage: `url(${this.currentMusic.pic})`,
          }
        }
        return {}
      },
      loadProgress () {
        if (this.playStat.duration === 0) return 0
        return this.playStat.loadedTime / this.playStat.duration
      },
      playProgress () {
        if (this.playStat.duration === 0) return 0
        return this.playStat.playedTime / this.playStat.duration
      },
      playIndex: {
        get () {
          return this.shuffledList.indexOf(this.currentMusic)
        },
        set (val) {
          this.currentMusic = this.shuffledList[val % this.shuffledList.length]
        },
      },
      shouldRepeat () {
        return this.repeatMode !== REPEAT.NO_REPEAT
      },

      // since 1.4.0
      // sync muted, volume

      isAudioMuted: {
        get () {
          return this.internalMuted
        },
        set (val) {
          canUseSync && this.$emit('update:muted', val)
          this.internalMuted = val
        },
      },
      audioVolume: {
        get () {
          return this.internalVolume
        },
        set (val) {
          canUseSync && this.$emit('update:volume', val)
          this.internalVolume = val
        },
      },


      // since 1.5.0
      // sync shuffle, repeat
      shouldShuffle: {
        get () {
          return this.internalShuffle
        },
        set (val) {
          canUseSync && this.$emit('update:shuffle', val)
          this.internalShuffle = val
        },
      },
      repeatMode: {
        get () {
          switch (this.internalRepeat) {
            case REPEAT.NONE:
            case REPEAT.NO_REPEAT:
              return REPEAT.NO_REPEAT
            case REPEAT.MUSIC:
            case REPEAT.REPEAT_ONE:
              return REPEAT.REPEAT_ONE
            default:
              return REPEAT.REPEAT_ALL
          }
        },
        set (val) {
          canUseSync && this.$emit('update:repeat', val)
          this.internalRepeat = val
        },
      },
    },
    methods: {
      // Float mode

      onDragBegin () {
        this.floatOriginX = this.floatOffsetLeft
        this.floatOriginY = this.floatOffsetTop
      },
      onDragAround ({ offsetLeft, offsetTop }) {
        this.floatOffsetLeft = this.floatOriginX + offsetLeft
        this.floatOffsetTop = this.floatOriginY + offsetTop
      },

      // functions

      setNextMode () {
        if (this.repeatMode === REPEAT.REPEAT_ALL) {
          this.repeatMode = REPEAT.REPEAT_ONE
        } else if (this.repeatMode === REPEAT.REPEAT_ONE) {
          this.repeatMode = REPEAT.NO_REPEAT
        } else {
          this.repeatMode = REPEAT.REPEAT_ALL
        }
      },
      thenPlay () {
        this.$nextTick(() => {
          this.play()
        })
      },

      // controls

      // play/pause

      toggle () {
        if (!this.audio.paused) {
          this.pause()
        } else {
          this.play()
        }
      },
      play () {
        if (this.mutex) {
          if (activeMutex && activeMutex !== this) {
            activeMutex.pause()
          }
          activeMutex = this
        }
        // handle .play() Promise
        const audioPlayPromise = this.audio.play()
        if (audioPlayPromise) {
          return this.audioPlayPromise = new Promise((resolve, reject) => {
            // rejectPlayPromise is to force reject audioPlayPromise if it's still pending when pause() is called
            this.rejectPlayPromise = reject
            audioPlayPromise.then((res) => {
              this.rejectPlayPromise = null
              resolve(res)
            }).catch(warn)
          })
        }
      },
      pause () {
        this.audioPlayPromise
          .then(() => {
            this.audio.pause()
          })
          // Avoid force rejection throws Uncaught
          .catch(() => {
            this.audio.pause()
          })

        // audioPlayPromise is still pending
        if (this.rejectPlayPromise) {
          // force reject playPromise
          this.rejectPlayPromise()
          this.rejectPlayPromise = null
        }
      },

      // progress bar

      onProgressDragBegin (val) {
        this.wasPlayingBeforeSeeking = this.isPlaying
        this.pause()
        this.isSeeking = true

        // handle load failures
        if (!isNaN(this.audio.duration)) {
          this.audio.currentTime = this.audio.duration * val
        }
      },
      onProgressDragging (val) {
        if (isNaN(this.audio.duration)) {
          this.playStat.playedTime = 0
        } else {
          this.audio.currentTime = this.audio.duration * val
        }
      },
      onProgressDragEnd (val) {
        this.isSeeking = false

        if (this.wasPlayingBeforeSeeking) {
          this.thenPlay()
        }
      },

      // volume

      toggleMute () {
        this.setAudioMuted(!this.audio.muted)
      },
      setAudioMuted (val) {
        this.audio.muted = val
      },
      setAudioVolume (val) {
        this.audio.volume = val
        if (val > 0) {
          this.setAudioMuted(false)
        }
      },

      // playlist

      getShuffledList () {
        // 只有一首，那就是只有一首
        if (!this.list.length) {
          return [this.internalMusic]
        }

        // clone 一個新的 list 叫做 unshuffled
        let unshuffled = [...this.list]
        if (!this.internalShuffle || unshuffled.length <= 1) {
          return unshuffled
        }

        let indexOfCurrentMusic = unshuffled.indexOf(this.internalMusic)
        if (unshuffled.length === 2 && indexOfCurrentMusic !== -1) {
          if (indexOfCurrentMusic === 0) {
            return unshuffled
          } else {
            return [this.internalMusic, unshuffled[0]]
          }
        }
        // shuffle list
        // @see https://stackoverflow.com/questions/6274339/how-can-i-shuffle-an-array
        for (let i = unshuffled.length - 1; i > 0; i--) {
          const j = Math.floor(Math.random() * (i + 1))
          const tmp = unshuffled[i]
          unshuffled[i] = unshuffled[j]
          unshuffled[j] = tmp
        }

        // take currentMusic to first
        if (indexOfCurrentMusic !== -1) {
          indexOfCurrentMusic = unshuffled.indexOf(this.internalMusic)
          if (indexOfCurrentMusic !== 0) {
            [unshuffled[0], unshuffled[indexOfCurrentMusic]] = [unshuffled[indexOfCurrentMusic], unshuffled[0]]
          }
        }

        return unshuffled
      },

      onSelectSong (song) {
        if (this.currentMusic === song) {
          this.toggle()
        } else {
          this.currentMusic = song
          this.thenPlay()
        }
      },

      // event handlers
      // for keeping up with audio states

      onAudioPlay () {
        this.isPlaying = true
      },
      onAudioPause () {
        this.isPlaying = false
      },
      onAudioWaiting () {
        this.isLoading = true
      },
      onAudioCanplay () {
        this.isLoading = false
      },
      onAudioDurationChange () {
        if (this.audio.duration !== 1) {
          this.playStat.duration = this.audio.duration
        }
      },
      onAudioProgress () {
        if (this.audio.buffered.length) {
          this.playStat.loadedTime = this.audio.buffered.end(this.audio.buffered.length - 1)
        } else {
          this.playStat.loadedTime = 0
        }
      },
      onAudioTimeUpdate () {
        this.playStat.playedTime = this.audio.currentTime
      },
      onAudioSeeking () {
        this.playStat.playedTime = this.audio.currentTime
      },
      onAudioSeeked () {
        this.playStat.playedTime = this.audio.currentTime
      },
      onAudioVolumeChange () {
        this.audioVolume = this.audio.volume
        this.isAudioMuted = this.audio.muted
      },
      // TODO: fuguring out: 一知半解
      // 當某首音檔播放完後觸發的事件
      onAudioEnded () {

        // determine next song according to shuffle and repeat
        debug('onAudioEnded')
        // 全部循環模式
        if (this.repeatMode === REPEAT.REPEAT_ALL) {
          debug('this.shouldShuffle:', this.shouldShuffle)
          debug('this.playIndex:', this.playIndex)
          debug('this.shuffledList.length:', this.shuffledList.length)
          debug('this.playIndex === this.shuffledList.length - 1', this.playIndex === this.shuffledList.length - 1)
          
          // 如果目前是「隨機播放」模式
          // 當某首音檔播放完後，判斷這首音檔是否已經是隨機播放列表的最後一首
          // 如果是，則重新計算 shuffledList
          // 然後再次播放
          if (this.shouldShuffle && this.playIndex === this.shuffledList.length - 1) {
            this.shuffledList = this.getShuffledList()
          }
          this.playIndex++
          this.thenPlay()

        // 單曲循環模式，直接播放原曲
        } else if (this.repeatMode === REPEAT.REPEAT_ONE) {
          this.thenPlay()
        } else {
          this.playIndex++
          if (this.playIndex !== 0) {
            this.thenPlay()
          } else if (this.shuffledList.length === 1) {
            this.audio.currentTime = 0
          }
        }
      },

      initAudio () {

        // since 1.4.0 Audio attributes as props

        this.audio.controls = this.shouldShowNativeControls
        this.audio.muted = this.muted
        this.audio.preload = this.preload
        this.audio.volume = this.volume


        // since 1.4.0 Emit as many native audio events
        // @see https://developer.mozilla.org/en-US/docs/Web/Guide/Events/Media_events

        const mediaEvents = [
          'abort',
          'canplay', 'canplaythrough',
          'durationchange',
          'emptied', 'encrypted', 'ended', 'error',
          'interruptbegin', 'interruptend',
          'loadeddata', 'loadedmetadata', 'loadstart',
          'mozaudioavailable',
          'pause', 'play', 'playing', 'progress',
          'ratechange',
          'seeked', 'seeking', 'stalled', 'suspend',
          'timeupdate',
          'volumechange',
          'waiting',
        ]
        mediaEvents.forEach(event => {
          this.audio.addEventListener(event, e => this.$emit(event, e))
        })


        // event handlers
        // they don't emit native media events

        this.audio.addEventListener('play', this.onAudioPlay)
        this.audio.addEventListener('pause', this.onAudioPause)
        this.audio.addEventListener('abort', this.onAudioPause)
        this.audio.addEventListener('waiting', this.onAudioWaiting)
        this.audio.addEventListener('canplay', this.onAudioCanplay)
        this.audio.addEventListener('progress', this.onAudioProgress)
        this.audio.addEventListener('durationchange', this.onAudioDurationChange)
        this.audio.addEventListener('seeking', this.onAudioSeeking)
        this.audio.addEventListener('seeked', this.onAudioSeeked)
        this.audio.addEventListener('timeupdate', this.onAudioTimeUpdate)
        this.audio.addEventListener('volumechange', this.onAudioVolumeChange)
        this.audio.addEventListener('ended', this.onAudioEnded)


        if (this.currentMusic) {
          this.audio.src = this.currentMusic.src || this.currentMusic.url
        }
      },

      setSelfAdaptingTheme () {
        // auto theme according to current music cover image
        if ((this.currentMusic.theme || this.theme) === 'pic') {
          const pic = this.currentMusic.pic
          // use cache
          if (picThemeCache[pic]) {
            this.selfAdaptingTheme = picThemeCache[pic]
          } else {
            try {
              new ColorThief().getColorAsync(pic, ([r, g, b]) => {
                picThemeCache[pic] = `rgb(${r}, ${g}, ${b})`
                this.selfAdaptingTheme = `rgb(${r}, ${g}, ${b})`
              })
            } catch (e) {
              warn('color-thief is required to support self-adapting theme')
            }
          }
        } else {
          this.selfAdaptingTheme = null
        }
      },
    },
    watch: {
      music (music) {
        this.internalMusic = music
      },

      currentMusic: {
        handler (music) {
          // async
          this.setSelfAdaptingTheme()

          const src = music.src || music.url
          // HLS support
          if (/\.m3u8(?=(#|\?|$))/.test(src)) {
            if (this.audio.canPlayType('application/x-mpegURL') || this.audio.canPlayType('application/vnd.apple.mpegURL')) {
              this.audio.src = src
            } else {
              try {
                const Hls = require('hls.js')
                if (Hls.isSupported()) {
                  if (!this.hls) {
                    this.hls = new Hls()
                  }
                  this.hls.loadSource(src)
                  this.hls.attachMedia(this.audio)
                } else {
                  warn('HLS is not supported on your browser')
                  this.audio.src = src
                }
              } catch (e) {
                warn('hls.js is required to support m3u8')
                this.audio.src = src
              }
            }
          } else {
            this.audio.src = src
          }
          // self-adapting theme color
        },
      },

      // since 1.4.0
      // observe controls, muted, preload, volume

      shouldShowNativeControls (val) {
        this.audio.controls = val
      },
      isAudioMuted (val) {
        this.audio.muted = val
      },
      preload (val) {
        this.audio.preload = val
      },
      audioVolume (val) {
        this.audio.volume = val
      },

      // sync muted, volume

      muted (val) {
        this.internalMuted = val
      },
      volume (val) {
        this.internalVolume = val
      },


      // sync shuffle, repeat
      shuffle (val) {
        this.internalShuffle = val
      },
      repeat (val) {
        this.internalRepeat = val
      },
    },
    // NOTE: figuring out: done
    beforeCreate () {
      // 瀏覽器 console log 絢麗的版本提示
      if (!VueAPlayer.disableVersionBadge && !versionBadgePrinted) {
        // version badge
        console.log(`\n\n %c Vue-APlayer ${VERSION} %c vue-aplayer.js.org \n`, 'color: #fff; background:#41b883; padding:5px 0;', 'color: #fff; background: #35495e; padding:5px 0;')
        versionBadgePrinted = true
      }
    },
    // TODO: fuguring out: 一知半解，skipped
    created () {
      // shuffled: 下一首歌隨機播放
      // 得到一個隨機打散的 playlist
      this.shuffledList = this.getShuffledList()
    },
    // NOTE: figuring out: done
    mounted () {
      // init eventEmitters and eventHandlers
      this.initAudio()

      // 可透過 color-thief.js 來取得單曲封面的主題色
      this.setSelfAdaptingTheme()


      if (this.autoplay) this.play()
    },
    // NOTE: fuguring out: done
    // 將目前的 player instance 從 activeMutex 解除
    // 將 hls lib 銷毀
    beforeDestroy () {
      if (activeMutex === this) {
        activeMutex = null
      }
      if (this.hls) {
        this.hls.destroy()
      }
    },
  }

  export default VueAPlayer

</script>

<style lang="scss">
  @import "./scss/variables";

  .aplayer {
    font-family: Arial, Helvetica, sans-serif;
    color: #000;
    background-color: #fff;
    margin: 5px;
    box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.07), 0 1px 5px 0 rgba(0, 0, 0, 0.1);
    border-radius: 2px;
    overflow: hidden;
    user-select: none;
    line-height: initial;

    * {
      box-sizing: content-box;
    }

    .aplayer-lrc-content {
      display: none;
    }

    .aplayer-body {
      display: flex;

      position: relative;
      .aplayer-info {
        flex-grow: 1;
        display: flex;
        flex-direction: column;

        text-align: start;
        padding: 14px 7px 0 10px;
        height: $aplayer-height;
        box-sizing: border-box;
        overflow: hidden;

        .aplayer-music {
          flex-grow: 1;

          overflow: hidden;
          white-space: nowrap;
          text-overflow: ellipsis;
          margin-left: 5px;
          user-select: text;
          cursor: default;
          padding-bottom: 2px;

          .aplayer-title {
            font-size: 14px;
          }

          .aplayer-author {
            font-size: 12px;
            color: #666;
          }
        }

        .aplayer-lrc {
          z-index: 0;
        }
      }
    }

    audio[controls] {
      display: block;
      width: 100%;
    }

    // Mini mode
    &.aplayer-narrow {
      width: $aplayer-height;
    }

    &.aplayer-withlrc {
      .aplayer-body {
        .aplayer-pic {
          height: $aplayer-height-lrc;
          width: $aplayer-height-lrc;
        }

        .aplayer-info {
          height: $aplayer-height-lrc;
        }

        .aplayer-info {
          padding: 10px 7px 0 7px;
        }
      }
    }

    &.aplayer-withlist {
      .aplayer-body {
        .aplayer-info {
          border-bottom: 1px solid #e9e9e9;
        }

        .aplayer-controller .aplayer-time .aplayer-icon.aplayer-icon-menu {
          display: block;
        }
      }
    }

    /* floating player on top */
    position: relative;
    &.aplayer-float {
      z-index: 1;
    }
  }

  @keyframes aplayer-roll {
    0% {
      left: 0
    }
    100% {
      left: -100%
    }
  }
</style>