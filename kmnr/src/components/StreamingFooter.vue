<template>
    <div>
      <transition name="slide-fade">
        <div v-if="open || stillWatching" class="stream-container">

          <div id="timer">{{ cDur }}</div>
          <div id="duration">{{ totalDuration }}</div>

          <div class="controlsOuter center">
              <div id="playlistBtn" ref="btn"><a @click="openQueue()"><i class="material-icons-round">queue_music</i></a></div>
              <div id="prevBtn"><a @click="prev()"><i class="material-icons-round">skip_previous</i></a></div>
              <div id="playPauseBtn">
                <a @click="playPause(curIndex)">
                  <i class="material-icons-round" :class="{'playing': playing, 'paused': !playing}">
                    {{playing ? 'pause_circle_filled' : 'play_circle_filled'}}
                  </i>
                </a>
              </div>
              <div id="nextBtn"><a @click="next()"><i class="material-icons-round">skip_next</i></a></div>
              <div class="volume-outer"><div id="volume-inner"></div></div>
              <div id="volumeBtn"><a @click="changeVol()"><i class="material-icons-round">volume_up</i></a></div>
          </div>
        </div>
      </transition>
      <div class="progress-outer"><div id="prog-inner"></div></div>
      <div>
      <md-dialog :md-active.sync="queueOpen" style="z-index: 999999; width: 25%">
      <md-dialog-title>Edit Playlist</md-dialog-title>

      <md-tabs md-dynamic-height>
        <md-tab md-label="Playlist" class="md-primary">
          <draggable v-model="queue" ghost-class="ghost" @end="onEnd">
                <transition-group type="transition" name="flip-list">
                    <div class='sortable' v-for="(song, index) in queue" :key="song.file" @click="updateSong(index)">{{ song.title }}</div>
            </transition-group>
            </draggable>
        </md-tab>
      </md-tabs>
        <md-dialog-actions>
        <md-button class="md-primary" @click="closeQueue()">Close</md-button>
      </md-dialog-actions>
    </md-dialog>
    </div>
        <md-snackbar class="snack"
      :md-position="position"
      :md-duration="isInfinity ? Infinity : duration"
      :md-active.sync="showsnackbar"
      md-persistent
    >
      <span>Now Playing: {{this.currentTrack}}</span>
    </md-snackbar>
    </div>
</template>

<script lang="ts">
  import { Vue, Component, Prop, Watch } from 'vue-property-decorator'
  import * as V from 'vue'
  import StreamQueue from './StreamQueue.vue'
  import { Howl } from 'howler'
  import draggable from 'vuedraggable'
  import {mapState} from 'vuex'

  let handleOutsideClick

  @Component({
    components: {
      draggable
    },
    directives: {
      closable: {
        bind (el, binding, vnode) {
          handleOutsideClick = (e) => {
            e.stopPropagation()
            const { handler, exclude } = binding.value
            let clickedOnExcludedEl = false
            exclude.forEach(refName => {
              if (!clickedOnExcludedEl) {
                const excludedEl = (vnode.context as any).$refs[refName]
                clickedOnExcludedEl = excludedEl.contains(e.target)
              }
            })
            if (!el.contains(e.target) && !clickedOnExcludedEl) {
              (vnode.context as any)[handler]()
            }
          }
          document.addEventListener('click', handleOutsideClick)
          document.addEventListener('touchstart', handleOutsideClick)
        },
        unbind () {
          document.removeEventListener('click', handleOutsideClick)
          document.removeEventListener('touchstart', handleOutsideClick)
        }
      }
    }
  })
  export default class StreamingFooter extends Vue {
      @Prop({type: Number, default: 0}) newIndex
      @Prop({type: Boolean, default: false}) open

      queueOpen = false

      playing: boolean = false
      curIndex: number = 0

      cDur: string = '0:00'
      queue: any = []
      curVol = 1
      muted = false
      currentTrack = ''
      position ="left"
      duration= 7000
      isInfinity= false
      stillWatching = false
      showsnackbar = false
      clickedInProgressBar = false
      clickedInVolumeSlider = false
      oIndex = ''
      nIndex = ''
      index: number = -1

      dragging = false

      get statefulQueue() {
        return mapState(['queue'])
      }

      @Watch('statefulQueue')
      newState(newQueue, oldQueue) {
        for (let i in newQueue) {
          if (this.queue.map(e => {return e.file}).indexOf(newQueue[i].file) !== -1) {
            /**
            newQueue[i].song = new Howl({
                src: [this.queue[i].file],
                onend: () => {
                    this.next()
                },
                onseek: () => {
                  this.curDuration()
                },
                onplay: () => {
                  this.curDuration()
                },
                onplayerror: () => {
                  this.queue[i].song.once('unlock', () => {
                    this.queue[i].song.play()
                  })
                },
                html5: true,
                buffer: true
            })  */
            this.queue.push(newQueue[i])
          }
        }
      }

      created() {
        this.queue = this.$store.state.queue

        this.loadAllSongs()
      }

      loadAllSongs() {
        for (let i in this.queue) {
          this.queue[i].song = new Howl({
                  src: [this.queue[i].file],
                  onend: () => {
                      this.next()
                  },
                  onseek: () => {
                    this.curDuration()
                  },
                  onplay: () => {
                    this.curDuration()
                  },
                  onplayerror: () => {
                    this.queue[i].song.once('unlock', () => {
                      this.queue[i].song.play()
                    })
                  },
                  html5: true,
                  buffer: true
              })
        }
      }

      updateSong(index) {
        this.queue[this.curIndex].song.stop()
        this.curIndex = index
        this.queue[this.curIndex].song.play()
        this.playing = true
      }

      onEnd(evt){
        let oldIdx = evt.oldIndex
        console.log(evt.oldIndex, evt.newIndex)
        if (evt.oldIndex === this.curIndex && evt.newIndex !== this.curIndex) {
          this.curIndex = evt.newIndex
        } else if (evt.newIndex >= this.curIndex && oldIdx < this.curIndex) {
          this.curIndex--
        } else if (evt.newIndex <= this.curIndex && oldIdx > this.curIndex) {
          this.curIndex++
        }
      }
      mousedownListener = (e) => {
        e.preventDefault()
        this.clickedInProgressBar = true
      }

      mousedownListenerVolume = (e) => {
        e.preventDefault()
        this.clickedInVolumeSlider = true
      }

      mouseupListener = (ev) => {
        if (this.clickedInProgressBar) {
          this.clickedInProgressBar = false

          let percentage = ev.pageX / window.screen.width

          if (percentage < 0) {
            percentage = 0
          } else if (percentage > 1) {
            percentage = 1
          }

          this.seek(percentage)
        } else if (this.clickedInVolumeSlider) {
          this.clickedInVolumeSlider = false

          const ele = (document.getElementsByClassName('volume-outer')[0] as HTMLElement)
          const eleIn = (document.getElementById('volume-inner') as HTMLElement)
          const newPageX = ev.pageX - ele.offsetLeft

          let vol = newPageX / ele.clientWidth

          if (vol < 0) {
            vol = 0
          } else if (vol > 1) {
            vol = 1
          }

          eleIn.style.width = (vol * ele.clientWidth).toString() + "%"
          this.setVol(vol)

          this.changeVol(vol)
        }
      }

      mouseMove = (e) => {
        if (this.clickedInProgressBar || this.clickedInVolumeSlider) {
          e.preventDefault()
        }
      }

      @Watch('open')
      opened(openFirst, newOpen) {
        if (!newOpen) {
          this.stillWatching = true;
          this.$nextTick(() => {
            (document.getElementsByClassName('volume-outer')[0] as HTMLElement).addEventListener('mousedown', this.mousedownListenerVolume);
          })
        } else {
          this.$nextTick(() => {
            (document.getElementsByClassName('volume-outer')[0] as HTMLElement).removeEventListener('mousedown', this.mousedownListenerVolume);
          })
          this.stillWatching = false;
        }
      }

      mounted() {
        (document.getElementsByClassName('progress-outer')[0] as HTMLElement).addEventListener('mousedown', this.mousedownListener);
        (document.getElementsByClassName('full-view')[0] as HTMLElement).addEventListener('mouseup', this.mouseupListener);
        (document.getElementsByClassName('full-view')[0] as HTMLElement).addEventListener('mousemove', this.mouseMove);

        this.queue[this.curIndex].song = new Howl({
          src: [this.queue[this.curIndex].file],
          onend: () => {
            this.next()
          },
          onseek: () => {
            this.curDuration()
          },
          onplay: () => {
            this.curDuration()
          },
          onplayerror: () => {
            this.queue[this.curIndex].song.once('unlock', () => {
              this.queue[this.curIndex].song.play()
            })
          },
          html5: true,
          buffer: true
        })
      }

      beforeDestroy() {
        (document.getElementsByClassName('progress-outer')[0] as HTMLElement).removeEventListener('mousedown', this.mousedownListener);
        (document.getElementsByClassName('volume-outer')[0] as HTMLElement).removeEventListener('mousedown', this.mousedownListener);

        (document.getElementsByClassName('full-view')[0] as HTMLElement).removeEventListener('mouseup', this.mouseupListener);
        (document.getElementsByClassName('full-view')[0] as HTMLElement).removeEventListener('mousemove', this.mouseMove);
      }

      get totalDuration() {
        return this.secondsToHms(this.queue[this.curIndex].song.duration())
      }

      setVol(vol) {
        this.curVol = vol
      }

      curDuration() {
        // Get the Howl we want to manipulate.
        (document.getElementById('prog-inner') as HTMLElement).style.width = ((this.queue[this.curIndex].song.seek() / this.queue[this.curIndex].song.duration()) * 100).toString() + "%"

        this.cDur = this.secondsToHms(this.queue[this.curIndex].song.seek())
        if (this.playing) {
          setTimeout(() => {
            this.curDuration()
          }, 10)
        }
      }

      secondsToHms(d) {
        d = Number(d);
        const h = Math.floor(d / 3600);
        const m = Math.floor(d % 3600 / 60);
        const s = Math.floor(d % 3600 % 60);

        const hDisplay = h > 0 ? h + ":" : "";
        const mDisplay = m > 0 ? m + ":" : "0:";
        const sDisplay = s > 0 ? (s >= 10 ? s : "0" + s) : "00";
        return hDisplay + mDisplay + sDisplay; 
      }

      playPause(index) {
        if (!this.playing){
          this.showsnackbar = true
          this.currentTrack = this.queue[this.curIndex].title
          this.play(index)
        }
        else
          this.pause()
      }

      play(index) {
        this.curIndex = index
        if (this.queue[this.curIndex].song) {
            this.queue[this.curIndex].song.play()
        }
        else {
            this.queue[this.curIndex].song = new Howl({
                src: [this.queue[this.curIndex].file],
                onend: () => {
                    this.next()
                },
                onseek: () => {
                  this.curDuration()
                },
                onplay: () => {
                  this.curDuration()
                },
                onplayerror: () => {
                  this.queue[this.curIndex].song.once('unlock', () => {
                    this.queue[this.curIndex].song.play()
                  })
                },
                html5: true,
                buffer: true
            })
            this.queue[this.curIndex].song.play()
        }
        this.playing = true
      }

      pause() {
          this.queue[this.curIndex].song.pause()
          this.playing = false
      }

      next() {
        this.queue[this.curIndex].song.stop()
        this.showsnackbar = true

        if ((this.curIndex + 1) > this.queue.length - 1) {
          this.curIndex = 0
        } else {
          this.curIndex++
        }
        this.currentTrack = this.queue[this.curIndex].title
        this.play(this.curIndex)
      }

      prev() {
        this.showsnackbar = true
        const pastFive = this.queue[this.curIndex].song.seek()
        this.queue[this.curIndex].song.stop()

        if (!(pastFive > 5)) {
          if ((this.curIndex - 1) < 0) {
            this.curIndex = this.queue.length - 1
          } else {
            this.curIndex--
          }
        } else {
          
        }
       this.currentTrack = this.queue[this.curIndex].title
        this.play(this.curIndex)
      }

      seek(percentage) {
        this.queue[this.curIndex].song.seek(percentage * this.queue[this.curIndex].song.duration())
      }

      changeVol(vol?) {
        const ele = (document.getElementsByClassName('volume-outer')[0] as HTMLElement)
        const eleIn = (document.getElementById('volume-inner') as HTMLElement)

        if (vol) {
          this.queue[this.curIndex].song.volume(vol)
          this.muted = false
        } else if (this.muted) {
          this.queue[this.curIndex].song.volume(this.curVol)
          console.log(this.curVol)
          eleIn.style.width = (this.curVol * ele.clientWidth).toString() + "%"
          this.muted = false
        } else {
          this.queue[this.curIndex].song.volume(0)
          eleIn.style.width = "0%"
          this.muted = true
        }
      }

      openQueue() {
        this.queueOpen = true
      }

      closeQueue() {
        this.queueOpen = false
        this.$store.commit('setQueue', this.queue)
      }
  }
</script>

<style lang="scss" scoped>
.snack{
  height: auto;
  margin-bottom: 6.3em;
  z-index: 99999999;
}
.queue {
  position:fixed;
}

.volume-outer {
  cursor: pointer;
  margin-top: 1em;
  height: 40px;
  background-image:linear-gradient(white, white);
  background-size: 100% 5px;
  background-repeat: no-repeat;
  background-position: center;
  width: 100px;
  position: fixed;
  bottom: 0px;
  margin-bottom: 2.5em;
  left: 86%;

  #volume-inner {
    height: 100%;
    background-image:linear-gradient(to right, #f12711, #f5af19);
    background-size: 100% 5px;
    background-repeat: no-repeat;
    background-position: center;
    width: 100%;
  }
}

.progress-outer {
  cursor: pointer;

  position: fixed;
  background-image:linear-gradient(white, white);
  background-size: 100% 5px;
  background-repeat: no-repeat;
  background-position: bottom;
  height: 15px;
  width: 100%;
  bottom: 0px;
  margin-bottom: 0px;

  #prog-inner {
    width: 0%;
    background: linear-gradient(to left, #DC2424, #4A569D);
    background-size: 100% 5px;
    background-repeat: no-repeat;
    height: 100%;
    background-position: bottom;
  }

  &:hover {
    background-size: 100% 8px;
    transition: background-size .2s;

    #prog-inner {
      background-size: 100% 8px;
      transition: background-size .15s;
    }
  }
}

.slide-fade-enter-active {
  transition: all .3s ease;
}
.slide-fade-leave-active {
  transition: all .25s cubic-bezier(1.0, 0.5, 0.8, 1.0);
}
.slide-fade-enter, .slide-fade-leave-to
/* .slide-fade-leave-active below version 2.1.8 */ {
  transform: translateY(10px);
  opacity: 0;
}

.stream-container{
    position: fixed;
    bottom: 0px;
    background-color: black;
    width: 100%;
    height: 7.5em;
}
#timer {
  position: absolute;
  bottom: 10%;
  text-align: left;
  font-size: 26px;
  font-weight: 300;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.33);
}
#duration {
  position: absolute;
  bottom: 10%;
  width: 100%;
  text-align: right;
  font-size: 26px;
  font-weight: 300;
  color: #fff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.33);
}
.controlsOuter {
  position: absolute;
  width: 100%;
  height: 70px;
  bottom: 3%;
}


#playPauseBtn, #nextBtn, #prevBtn {
  display: inline-block;
  cursor: pointer;
  opacity: 0.9;
  -webkit-filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.33));
  filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.33));
  
  width: 5em;

  bottom: 1em;
  
  -webkit-user-select: none; /* Safari */        
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* IE10+/Edge */
  user-select: none; /* Standard */

  position: fixed;

  a { text-decoration: none; }

  &:hover {
    opacity: 1;
  }

  .material-icons-round {
    font-size: 5rem;
    color: grey;
  }
}

#playPauseBtn {
  left: 50%;

  .material-icons-round {
    background-image: url('../assets/playing.png');
    -webkit-background-clip: text;
    -moz-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color:transparent;
  }
}

.playing {
  background-size: 150% !important;
  transition: all 1s ease;
}

.paused {
  background-size: 300%;
  transition: all .5s ease;
}

#prevBtn {
  left: 45%;
}

#nextBtn {
  left: 55%;
}

#playlistBtn, #volumeBtn {
  display: inline-block;
  cursor: pointer;
  opacity: 0.9;
  -webkit-filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.33));
  filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.33));

  width: 5em;
  margin-top: 1em;

  position: fixed;

  bottom: 2em;

  -webkit-user-select: none; /* Safari */        
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* IE10+/Edge */
  user-select: none; /* Standard */

  a { text-decoration: none; }

  &:hover {
    opacity: 1;
  }

  .material-icons-round {
    font-size: 3rem;
    color: grey !important
  }
}

#volumeBtn {
  left: 92%;
}

#playlistBtn {
  left: 0%;
}
.sortable{
    width: 100%;
    background: white;
    padding: 1em;
    cursor: move;
    margin-bottom: 2px;;
}

.sortable-drag{
    opacity: 0
}

.flip-list-move{
    transition: transform 0.5s;
}

.ghost{
    border-left: 6px solid black;
    box-shadow: 10px 10px 5px -1px rgba(0, 0, 0, 0.17);
    opacity: .7;
}
</style>