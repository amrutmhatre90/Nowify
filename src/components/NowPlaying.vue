<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
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
    <div v-else class="now-playing" :class="getNowPlayingClass()">
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weather and Time</title>

    <!-- Link to Iconvault or another icon library (replace with actual Iconvault link) -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">

    <style>
        body {
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            height: 100vh;
            margin: 0;
            background-color: #000000;  /* Set background to black */
            color: white;
            font-family: Tahoma, sans-serif;  /* Set font to Tahoma */
        }
        #datetime {
            font-size: 36px;
            text-align: center;
            display: flex;
            flex-direction: column;
            justify-content: center;
            height: 60%;  /* Take 60% of the page */
        }
        .date-line, .time-line {
            font-size: 36px;  /* Font size for both date and time */
        }
        .time-line {
            margin-top: 10px;
        }
        .now-playing__idle-heading {
            font-size: 40px;  /* Slightly larger font size for the "No music" message */
            margin-top: 20px;  /* Add margin for spacing */
            color: white;      /* Make the text white */
        }
        .weatherwidget-io {
            width: 100%;  /* Full width for weather widget */
            height: 40%;  /* Set to take up 40% of the page */
            display: block;
        }
        footer {
            width: 100%;
        }
        .icon {
            font-size: 24px;  /* Adjust icon size */
            margin-right: 10px;
        }
    </style>
</head>
<body>

    <!-- Date and Time Section with Day and Time -->
    <div id="datetime">
        <div class="date-line">
            <i class="fas fa-calendar-alt icon"></i> <!-- Example Icon -->
            <span id="day-date"></span>
        </div>

        <div class="time-line">
            <i class="fas fa-clock icon"></i> <!-- Example Icon -->
            <span id="time"></span>
        </div>

        <!-- Now Playing Message -->
        <h1 class="now-playing__idle-heading">😔 No music is playing!!!</h1>
    </div>

    <!-- Weather Widget in Footer -->
    <footer>
        <a class="weatherwidget-io" href="https://forecast7.com/en/19d0872d88/mumbai/" 
           data-label_1="MUMBAI" data-label_2="WEATHER" data-theme="dark">MUMBAI WEATHER</a>
        <script>
            !function(d,s,id){
                var js,fjs=d.getElementsByTagName(s)[0];
                if(!d.getElementById(id)){
                    js=d.createElement(s);
                    js.id=id;
                    js.src='https://weatherwidget.io/js/widget.min.js';
                    fjs.parentNode.insertBefore(js,fjs);
                }
            }(document,'script','weatherwidget-io-js');
        </script>
    </footer>

    <!-- JavaScript to Display Date, Day, and Time -->
    <script>
        function updateDateTime() {
            const now = new Date();

            // Format day and date
            const dayOptions = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
            const formattedDayDate = now.toLocaleDateString('en-US', dayOptions);

            // Format time
            const timeOptions = { hour: 'numeric', minute: 'numeric', second: 'numeric', hour12: true };
            const formattedTime = now.toLocaleTimeString('en-US', timeOptions);

            // Set the content of the day-date and time elements
            document.getElementById('day-date').textContent = formattedDayDate;
            document.getElementById('time').textContent = formattedTime;
        }

        // Update date and time every second
        setInterval(updateDateTime, 1000);
        updateDateTime();  // Initialize the time immediately on page load
    </script>

</body>
</html>

    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

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
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        data = await response.json()
        this.playerResponse = data
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      /**
       * No image (rare).
       */
      if (!this.player.trackAlbum?.image) {
        return
      }

      /**
       * Run node-vibrant to get colours.
       */
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
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
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse.error?.status === 401 ||
        this.playerResponse.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (this.playerResponse.is_playing === false) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * The newly fetched track is the same as our stored
       * one, we don't want to update the DOM yet.
       */
      if (this.playerResponse.item?.id === this.playerData.trackId) {
        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours

      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },
  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
