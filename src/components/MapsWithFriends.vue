<template>
  <div>
    <div
      id="map"
      @mouseover="mouseOverMap"
      @mouseleave="mouseOutMap">
    </div>
    <button
      id="guess-button"
      :disabled="selectedLatLng == null || isClicked == true || isReady == false"
      v-if="isNextButtonVisible == false && isSummaryButtonVisible == false"
      @click="selectLocation"
      >GUESS
    </button>
    <button
      id="next-button"
      :disabled="!isNextButtonEnabled"
      :style="{ backgroundColor: isNextButtonEnabled ? '#F44336' : '#B71C1C' }"
      v-if="isNextButtonVisible == true"
      @click="goToNextRound"
      >NEXT ROUND
    </button>
    <button
      id="summary-button"
      v-if="isSummaryButtonVisible == true"
      @click="dialogSummary = true"
      >VIEW SUMMARY
    </button>
    <DialogSummaryWithFriends
      :dialogSummary="dialogSummary"
      :summaryTexts="summaryTexts"
      :score="score"
      @finishGame="finishGame" />
  </div>
</template>

<script>
  import firebase from 'firebase/app'
  import 'firebase/database'

  import DialogSummaryWithFriends from '@/components/DialogSummaryWithFriends'

  export default {
    props: [
      'randomLatLng',
      'roomName',
      'playerNumber',
      'isReady',
      'round',
      'score',
    ],
    components: {
      DialogSummaryWithFriends,
    },
    data() {
      return {
        markers: [],
        polylines: [],
        summaryTexts: [],
        strokeColors: [
          '#F44336',
          '#76FF03',
          '#FFEB3B',
          '#FF4081',
          '#18FFFF',
        ],
        map: null,
        room: null,
        selectedLatLng: null,
        distance: null,
        isClicked: false,
        isSelected: false,
        isNextStreetViewReady: false,
        isNextButtonVisible: false,
        isSummaryButtonVisible: false,
        dialogSummary: false,
      }
    },
    computed: {
      isNextButtonEnabled() {
        if (this.playerNumber == 1) {
          return true
        } else {
          if (this.isNextStreetViewReady == true) {
            return true
          } else {
            return false
          }
        }
      }
    },
    methods: {
      selectLocation() {
        this.calculateDistance()

        // Save the selected location into database
        // So that it uses for putting the markers and polylines
        this.room.child('guesses/Player' + this.playerNumber).set({
            latitude: this.selectedLatLng.lat(),
            longitude:this.selectedLatLng.lng()
        })

        // Clear the event
        google.maps.event.clearListeners(this.map, 'click')

        // Diable guess button and opacity of the map
        this.isClicked = true
        this.isSelected = true

        // Turn off the flag before the next button appears
        this.isNextStreetViewReady = false

        // Focus on the map
        this.mouseOverMap()
      },
      selectRandomLocation(randomLatLng) {
        // set a random location if the player didn't select in time
        this.selectedLatLng = randomLatLng
        this.removeMarkers()
        this.putMarker(this.selectedLatLng)
        this.selectLocation()
      },
      putMarker(position) {
        var marker = new google.maps.Marker({
          position: position,
          map: this.map,
        })
        this.markers.push(marker)
      },
      removeMarkers() {
        for (var i = 0; i < this.markers.length; i++) {
          this.markers[i].setMap(null)
        }
        this.markers = []
      },
      calculateDistance() {
        this.distance = Math.floor(google.maps.geometry.spherical.computeDistanceBetween(this.randomLatLng, this.selectedLatLng) / 1000)

        // Save the distance into firebase
        this.room.child('round' + this.round + '/Player' + this.playerNumber).set(this.distance)

        this.$emit('calculateDistance', this.distance)
      },
      setInfoWindow(playerName, distance) {
        var infoWindow = new google.maps.InfoWindow({
          content: '<b>' + playerName + '</b>' + ' is <b>' + distance + '</b> km away!'
        })
        infoWindow.open(this.map, this.markers[this.markers.length - 1])
      },
      drawPolyline(selectedLatLng, i) {
        var polyline = new google.maps.Polyline({
          path: [selectedLatLng, this.randomLatLng],
          strokeColor: this.strokeColors[i],
        })
        polyline.setMap(this.map)
        this.polylines.push(polyline)
      },
      removePolylines() {
        for (var i = 0; i < this.polylines.length; i++) {
          this.polylines[i].setMap(null)
        }
      },
      startNextRound() {
        const that = this
        this.map.addListener('click', function(e) {
          // Clear the previous marker when clicking the map
          that.removeMarkers()

          // Show the new marker
          that.putMarker(e.latLng)

          // Save latLng
          that.selectedLatLng = e.latLng
        })   
      },
      goToNextRound() {
        // Reset
        this.selectedLatLng = null
        this.isClicked = false
        this.isSelected = false
        this.isNextButtonVisible = false

        this.removeMarkers()
        this.removePolylines()

        this.mouseOutMap()

        // Replace the streetview with the next one
        this.$emit('goToNextRound')
      },
      finishGame() {
        this.dialogSummary = false
        this.room.child('is_game_done/Player' + this.playerNumber).set(true)
        this.$emit('finishGame')
      },
      mouseOverMap() {
        // Focus on map
        document.getElementById('map').style.opacity = 1.0
      },
      mouseOutMap() {
        // Focus on map while the player selected a location
        // Otherwise set the opacity of the map
        if (this.isSelected == false) {
          document.getElementById('map').style.opacity = 0.7
        }
      },
    },
    mounted() {
      this.map = new google.maps.Map(document.getElementById('map'), {
          center: {lat: 37.869260, lng: -122.254811},
          zoom: 1,
          fullscreenControl: false,
          mapTypeControl: false,
          streetViewControl: false,        
      })

      const that = this
      this.room = firebase.database().ref(this.roomName)
      this.room.on('value', function(snapshot) {
        // Check if the room is already removed
        if (snapshot.hasChild('Active')) {

          // Allow players to move on to the next round when every players guess locations
          if (snapshot.child('guesses').numChildren() == snapshot.child('size').val()) {
            that.$emit('showResult')

            // Put markers and draw polylines on the map
            var i = 0
            var j = 1
            snapshot.child('guesses').forEach(function(childSnapshot) {
              var lat = childSnapshot.child('latitude').val()
              var lng = childSnapshot.child('longitude').val()
              var latLng = new google.maps.LatLng({lat: lat, lng: lng});

              var playerName = snapshot.child('player_name').child(childSnapshot.key).val()
              var distance = snapshot.child('round' + that.round + '/Player' + j).val()

              that.drawPolyline(latLng, i)
              that.putMarker(latLng)
              that.setInfoWindow(playerName, distance)
              i++
              j++
            })
            that.putMarker(that.randomLatLng)

            // Remove guess node every time the round is done
            that.room.child('guesses').remove()

            if (that.round >= 5) {
              // Show summary button
              snapshot.child('final_score').forEach(function(childSnapshot) {
                var playerName = snapshot.child('player_name').child(childSnapshot.key).val()
                var finalScore = childSnapshot.val()
                that.summaryTexts.push({
                  playerName: playerName,
                  finalScore: finalScore,
                })
              })

              that.isSummaryButtonVisible = true

            } else {
              // Show next button
              that.isNextButtonVisible = true
            }
          }

          // Allow other players to move on to the next round when the next street view is set
          if (snapshot.child('street_views').numChildren() == that.round + 1) {
            that.isNextStreetViewReady = true
          }
        }
      })
    },
  }
</script>

<style scoped>
  button {
    position: absolute;
    bottom: 10px;
    left: 10px;
    z-index: 3;
    border: none;
    border-radius: 5px;
    opacity: 0.8;
    color: white;
    font-size: 16px;
    text-decoration: none;
    text-align: center;
    padding: 10px 60px;
    width: 360px;    
  }

  #map {
    position: absolute;
    bottom: 60px;
    left: 10px;
    z-index: 3;
    opacity: 0.7;
    height: 240px;
    width: 360px;
  }

  #guess-button {
    background-color: #212121;
  }

  #guess-button:hover {
    opacity: 1.0;
  }

  #summary-button {
    background-color: #F44336;
  }

  @media (max-width: 900px) {
    button {
      width: 300px;
    }

    #map {
      height: 200px; 
      width: 300px;
    }
  }

  @media (max-width: 450px) {
    button {
      bottom: -260px;
    }

    #map {
      bottom: -210px;
      height: 200px; 
      width: 300px;
    }
  } 
</style>