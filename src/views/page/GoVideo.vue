<template>
  <div class="content g2-content">
    <div v-if="options && options.api" class="video-content">
      <iframe
        width="100%"
        height="100%"
        :src="apiVideoUrl"
        frameborder="0"
        border="0"
        marginwidth="0"
        marginheight="0"
        scrolling="no"
        allowtransparency="true"
        allowfullscreen="true"
      ></iframe>
    </div>
    <div v-else>
      <vue-plyr ref="plyr" :options="options">
        <video controls crossorigin playsinline>
          <source :src="videoUrl" type="video/mp4" />
          <track
            v-if="subtitle"
            kind="captions"
            label="Subtitles"
            srclang="en"
            :src="subtitle"
            default
          />
        </video>
      </vue-plyr>
    </div>
    <div class="card">
      <header class="card-header">
        <p class="card-header-title">
          <span class="icon">
            <i class="fa fa-play-circle" aria-hidden="true"></i>
          </span>
          {{ $t("page.video.play") }} /
          <span class="icon">
            <i class="fa fa-download" aria-hidden="true"></i>
          </span>
          {{ $t("page.video.download") }}
        </p>
      </header>
      <div class="card-content">
        <div class="content">
          <div class="field">
            <label class="label">
              {{ $t("page.video.link") }}
              <a class="button is-text index-button-copy" @click="copy">
                {{ $t("copy.text") }}
              </a>
            </label>
            <div class="control">
              <input class="input" type="text" :value="videoUrl" />
            </div>
          </div>
          <div class="columns is-mobile is-multiline has-text-centered">
            <div
              class="column"
              v-for="(item, index) in players"
              v-bind:key="index"
            >
              <p class="heading">
                <a :href="item.scheme">
                  <figure class="image is-48x48" style="margin: 0 auto;">
                    <img class="icon" :src="item.icon" />
                  </figure>
                </a>
              </p>
              <p class>{{ item.name }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { decode64 } from "@utils/AcrouUtil";
import VuePlyr from "vue-plyr";
export default {
  comments: {
    VuePlyr,
  },
  data: function() {
    return {
      apiVideoUrl: "",
      videoUrl: "",
      subtitle: "",
      blobUrl: null, // Store blob URL for cleanup
    };
  },
  mounted() {
    this.render();
  },
  beforeDestroy() {
    // Clean up blob URL when component is destroyed
    if (this.blobUrl) {
      URL.revokeObjectURL(this.blobUrl);
    }
  },
  methods: {
    render() {
      let path = encodeURI(this.url);
      let index = path.lastIndexOf(".");
      this.suffix = path.substring(index + 1, path.length);
      this.loadSub(path, index);
      this.videoUrl = window.location.origin + path;
      this.apiVideoUrl = this.options.api + this.videoUrl;
      if (!this.options.api) {
        let options = {
          src: this.videoUrl,
          autoplay: this.options.autoplay,
          media: this.player.media,
        };
        if (this.suffix === "m3u8") {
          this.loadHls(options);
        } else if (this.suffix === "flv") {
          this.loadFlv(options);
        }
      }
    },
    loadSub(path, index) {
      // Try to load SRT subtitle first, then fallback to VTT
      this.subtitle = path.substring(0, index) + ".srt";
      this.loadSrtSubtitle(this.subtitle);
    },
    async loadSrtSubtitle(srtUrl) {
      try {
        const response = await fetch(srtUrl);
        if (response.ok) {
          const srtContent = await response.text();
          const vttContent = this.convertSrtToVtt(srtContent);
          this.subtitle = this.createVttBlobUrl(vttContent);
        } else {
          // Fallback to VTT if SRT not found
          this.subtitle = srtUrl.replace('.srt', '.vtt');
        }
      } catch (error) {
        console.log('SRT subtitle not found, trying VTT...');
        // Fallback to VTT if SRT not found
        this.subtitle = srtUrl.replace('.srt', '.vtt');
      }
    },
    convertSrtToVtt(srtContent) {
      // Convert SRT format to VTT format
      let vttContent = 'WEBVTT\n\n';
      
      // Split by double newlines to get subtitle blocks
      const blocks = srtContent.trim().split(/\n\s*\n/);
      
      blocks.forEach(block => {
        const lines = block.trim().split('\n');
        if (lines.length >= 3) {
          // Skip the sequence number (first line)
          const timeLine = lines[1];
          const textLines = lines.slice(2);
          
          // Convert SRT time format (00:00:00,000 --> 00:00:00,000) to VTT format (00:00:00.000 --> 00:00:00.000)
          const vttTimeLine = timeLine.replace(/,/g, '.');
          
          vttContent += vttTimeLine + '\n';
          vttContent += textLines.join('\n') + '\n\n';
        }
      });
      
      return vttContent;
    },
    createVttBlobUrl(vttContent) {
      // Clean up previous blob URL if exists
      if (this.blobUrl) {
        URL.revokeObjectURL(this.blobUrl);
      }
      
      // Create a blob URL for the VTT content
      const blob = new Blob([vttContent], { type: 'text/vtt' });
      this.blobUrl = URL.createObjectURL(blob);
      return this.blobUrl;
    },
    loadHls(options) {
      import("@/plugin/vplayer/hls").then((res) => {
        var Hls = res.default;
        Hls({
          ...options,
          callback: (hls) => {
            // Handle changing captions
            this.player.on("languagechange", () => {
              // Caption support is still flaky. See: https://github.com/sampotts/plyr/issues/994
              setTimeout(
                () => (hls.subtitleTrack = this.player.currentTrack),
                50
              );
            });
          },
        });
      });
    },
    loadFlv(options) {
      import("@/plugin/vplayer/flv").then((res) => {
        var Flv = res.default;
        Flv(options);
      });
    },
    copy() {
      this.$copyText(this.videoUrl);
    },
  },
  computed: {
    options() {
      var options = window.themeOptions.video;
      return {
        autoplay: false,
        invertTime: false,
        settings: ["quality", "speed", "loop"],
        ratio: "16:9",
        controls: [
          "play-large",
          "progress",
          "play",
          "current-time",
          "duration",
          "mute",
          "captions",
          "settings",
          "pip",
          "airplay",
          "download",
          "fullscreen",
        ],
        // Control layout options
        controlsLayout: "bottom",
        progressBar: {
          enabled: true,
          buffered: true,
          played: true,
          hover: true,
        },
        // Mobile-specific options
        seekTime: 10,
        volume: 1,
        muted: false,
        clickToPlay: true,
        hideControls: false,
        resetOnEnd: false,
        disableContextMenu: false,
        // Ensure progress bar is always visible
        progress: {
          enabled: true,
          buffered: true,
          played: true,
          hover: true,
        },
        // Subtitle/caption options
        captions: { 
          active: true, 
          language: "en", 
          update: false,
          ...options.captions 
        },
        ...options,
      };
    },
    player() {
      return this.$refs.plyr.player;
    },
    url() {
      if (this.$route.params.path) {
        return decode64(this.$route.params.path);
      }
      return "";
    },
    players() {
      return [
        {
          name: "IINA",
          icon: this.$cdnpath("images/player/iina.png"),
          scheme: "iina://weblink?url=" + this.videoUrl,
        },
        {
          name: "PotPlayer",
          icon: this.$cdnpath("images/player/potplayer.png"),
          scheme: "potplayer://" + this.videoUrl,
        },
        {
          name: "VLC",
          icon: this.$cdnpath("images/player/vlc.png"),
          scheme: "vlc://" + this.videoUrl,
        },
        {
          name: "Thunder",
          icon: this.$cdnpath("images/player/thunder.png"),
          scheme: "thunder://" + this.getThunder,
        },
        {
          name: "Aria2",
          icon: this.$cdnpath("images/player/aria2.png"),
          scheme: 'javascript:alert("暂未实现")',
        },
        {
          name: "nPlayer",
          icon: this.$cdnpath("images/player/nplayer.png"),
          scheme: "nplayer-" + this.videoUrl,
        },
        {
          name: "MXPlayer(Free)",
          icon: this.$cdnpath("images/player/mxplayer.png"),
          scheme:
            "intent:" +
            this.videoUrl +
            "#Intent;package=com.mxtech.videoplayer.ad;S.title=" +
            this.title +
            ";end",
        },
        {
          name: "MXPlayer(Pro)",
          icon: this.$cdnpath("images/player/mxplayer.png"),
          scheme:
            "intent:" +
            this.videoUrl +
            "#Intent;package=com.mxtech.videoplayer.pro;S.title=" +
            this.title +
            ";end",
        },
      ];
    },
    getThunder() {
      return Buffer.from("AA" + this.videoUrl + "ZZ").toString("base64");
    },
  },
};
</script>

<style lang="scss" scoped>
/* Custom styles for better mobile video player experience */
</style>

<style lang="scss">
/* Global styles for Plyr video player - progress bar positioning */
.plyr {
  /* Make progress bar more prominent */
  .plyr__progress {
    height: 6px !important;
    margin: 6px 0 !important;
    border-radius: 3px !important;
    
    @media (max-width: 768px) {
      height: 4px !important;
      margin: 6px 0 !important;
      border-radius: 2px !important;
    }
  }
  
  .plyr__progress__buffer {
    height: 100% !important;
    border-radius: 3px !important;
  }
  
  .plyr__progress__played {
    height: 100% !important;
    border-radius: 3px !important;
  }

  /* Shrink progress knob across browsers */
  .plyr__progress input[type="range"]::-webkit-slider-thumb { width: 12px !important; height: 12px !important; }
  .plyr__progress input[type="range"]::-moz-range-thumb { width: 12px !important; height: 12px !important; }
  .plyr__progress input[type="range"]::-ms-thumb { width: 12px !important; height: 12px !important; }

  /* Also reduce the track height where supported */
  .plyr__progress input[type="range"]::-webkit-slider-runnable-track { height: 6px !important; }
  .plyr__progress input[type="range"]::-moz-range-track { height: 6px !important; }
  .plyr__progress input[type="range"]::-ms-track { height: 6px !important; }
  
  /* Reorganize controls to put progress bar above other controls */
  .plyr__controls {
    display: flex !important;
    flex-direction: row !important;
    flex-wrap: wrap !important;
    align-items: center !important;
    justify-content: flex-start !important;
    padding: 10px !important;
    opacity: 1 !important;
    visibility: visible !important;
  }
  
  /* Progress bar container - move to top and span full width */
  .plyr__progress__container {
    order: 0 !important;
    flex: 0 0 100% !important;
    width: 100% !important;
    margin-bottom: 10px !important;
    padding: 0 !important;
  }
  
  /* Control buttons row - arranged horizontally below progress bar */
  .plyr__controls__item {
    order: 1 !important;
  }
  
  /* Time display - keep with controls */
  .plyr__time {
    order: 1 !important;
  }
  
  /* Volume control */
  .plyr__volume {
    order: 1 !important;
  }
  
  /* Remove any full-width forcing on other controls (was causing column layout) */
  /* .plyr__controls > *:not(.plyr__progress__container) { ... } removed */
  
  /* Mobile-specific improvements */
  @media (max-width: 768px) {
    /* Keep progress on its own row */
    .plyr__progress__container {
      margin-bottom: 8px !important;
      flex: 0 0 100% !important;
    }

    /* Make the controls compact and keep them on one row where possible (allow wrap if needed) */
    .plyr__controls {
      column-gap: 4px !important;
      row-gap: 0 !important;
      flex-wrap: wrap !important;
      opacity: 1 !important;
      visibility: visible !important;
    }

    .plyr__controls__item { 
      flex: 0 0 auto !important;
    }

    .plyr__control {
      padding: 2px !important;
      min-width: 28px !important;
      min-height: 28px !important;
    }

    .plyr__time {
      font-size: 11px !important;
      padding: 0 3px !important;
    }

    /* If volume exists in other devices, keep it compact */
    .plyr__volume input[type="range"],
    .plyr__volume .plyr__slider {
      width: 56px !important;
      max-width: 56px !important;
    }
  }
  
  /* Ensure progress bar is always visible and accessible */
  .plyr__progress__container {
    position: relative !important;
    z-index: 10 !important;
    background: transparent !important;
    border-radius: 0 !important;
    padding: 0 !important;
  }
  
  /* Make progress bar thumb more visible on mobile */
  @media (max-width: 768px) {
    .plyr__progress__seek {
      height: 20px !important;
      margin-top: -6px !important;
    }
    
    .plyr__progress__seek::before {
      width: 16px !important;
      height: 16px !important;
      margin-top: -8px !important;
    }
  }
}
</style>
