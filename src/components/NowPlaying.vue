<template>
  <div id="app">
    <!-- Now Playing when music is playing -->
    <div v-if="player.playing" class="now-playing" :class="getNowPlayingClass()">
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
      </div>
    </div>

    <!-- Now Playing Message when no music is playing -->
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <div id="datetime">
        <div class="date-line">
          <i class="fas fa-calendar-alt icon"></i>
          <span id="day-date">{{ dayDate }}</span>
        </div>
        
        <div class="time-line">
          <i class="fas fa-clock icon"></i>
          <span id="time">{{ time }}</span>
        </div>

        <!-- Now Playing Message -->
        <div class="now-playing__idle-heading">No music is playing 😔</div>
      </div>
    </div>
  </div>
</template>

<script>
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      dayDate: '',
      time: '',
      pollPlaying: '',
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    }
  },

  mounted() {
    this.updateDateTime();
    setInterval(this.updateDateTime, 1000);  // Update time every second
    this.setDataInterval();
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying);
  },

  methods: {
    /**
     * Update date and time.
     */
    updateDateTime() {
      const now = new Date();

      // Format day and date
      const dayOptions = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
      this.dayDate = now.toLocaleDateString('en-US', dayOptions);

      // Format time
      const timeOptions = { hour: 'numeric', minute: 'numeric', second: 'numeric', hour12: true };
      this.time = now.toLocaleTimeString('en-US', timeOptions);
    },

    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {};
      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        );
        if (!response.ok) {
          throw new Error(`An error has occurred: ${response.status}`);
        }

        if (response.status === 204) {
          data = this.getEmptyPlayer();
          this.playerData = data;
          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data);
          });
          return;
        }

        data = await response.json();
        this.playerResponse = data;
      } catch (error) {
        this.handleExpiredToken();
        data = this.getEmptyPlayer();
        this.playerData = data;
        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data);
        });
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle';
      return `now-playing--${playerClass}`;
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      };
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying);
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying();
      }, 2500);
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying);
      this.$emit('requestRefreshToken');
    }
  },

  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying);
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying();
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData);
    }
  }
};
</script>

<style scoped>
#datetime {
  font-size: 36px;
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: center;
  height: 60%;
  color: white;  /* Ensure text color is white */
}

.date-line, .time-line {
  font-size: 36px;  /* Same font size for date and time */
}

.time-line {
  margin-top: 10px;
}

.now-playing__idle-heading {
  font-size: 40px;  /* Slightly larger font size for "No music" message */
  margin-top: 20px;
  color: white;
}

.icon {
  font-size: 24px;
  margin-right: 10px;
}
</style>
