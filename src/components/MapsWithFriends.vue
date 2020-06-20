<template>
  <div>
    <div
      id="map"
      @mouseover="mouseOverMap"
      @mouseleave="mouseOutMap">
    </div>
    <v-btn
      id="hide-map-button"
      v-if="$viewport.width < 450 && !isGuessButtonClicked && isMakeGuessButtonClicked"
      fab
      x-small
      color="red"
      @click="hideMap"
      >
      <v-icon color="white">mdi-close</v-icon>
    </v-btn>
    <button
      id="make-guess-button"
      v-if="$viewport.width < 450 && !isGuessButtonClicked && !isMakeGuessButtonClicked"
      @click="showMap"
      >
      MAKE GUESS
    </button>
    <button
      id="guess-button"
      :disabled="selectedLatLng == null || isGuessButtonClicked || !isReady"
      v-if="!isNextButtonVisible && !isSummaryButtonVisible && ($viewport.width > 450 || isMakeGuessButtonClicked)"
      @click="selectLocation"
      >GUESS
    </button>
    <button
      id="next-button"
      :disabled="!isNextButtonEnabled"
      :style="{ backgroundColor: isNextButtonEnabled ? '#F44336' : '#B71C1C' }"
      v-if="isNextButtonVisible"
      @click="goToNextRound"
      >NEXT ROUND
    </button>
    <button
      id="summary-button"
      v-if="isSummaryButtonVisible"
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
        mHeight: 0,
        mWidth: 0,
        mX: 0,
        mY: 0,
        mDX: 0,
        mDY: 0,
        room: null,
        selectedLatLng: null,
        distance: null,
        isGuessButtonClicked: false,
        isMakeGuessButtonClicked: false,
        isSelected: false,
        isNextStreetViewReady: false,
        isNextButtonVisible: false,
        isSummaryButtonVisible: false,
        dialogSummary: false,
        drewPolygon1: false,
        drewPolygon1: false,
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
      showMap() {
        document.getElementById('map').style.transform = "translateY(-300px)"
        this.isMakeGuessButtonClicked = true
      },
      hideMap() {
        document.getElementById('map').style.transform = "translateY(300px)"
        this.isMakeGuessButtonClicked = false
      },
      selectLocation() {
        this.calculateDistance()
        // Save the selected location into database
        // So that it uses for putting the markers and polylines
        this.room.child('guess/player' + this.playerNumber).set({
            latitude: this.selectedLatLng.lat(),
            longitude:this.selectedLatLng.lng()
        })

        // Clear the event
        google.maps.event.clearListeners(this.map, 'click')

        // Diable guess button and opacity of the map
        this.isGuessButtonClicked = true
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
        this.distance = google.maps.geometry.spherical.computeDistanceBetween(this.randomLatLng, this.selectedLatLng)

        // Save the distance into firebase
        this.room.child('round' + this.round + '/player' + this.playerNumber).set(this.distance)

        this.$emit('calculateDistance', this.distance)
      },
      setInfoWindow(playerName, distance) {
        
        let text =  '<b>' + playerName + '</b>' + ' is <b>{%DIST%}</b> away!'
        if(distance < 1000) {
          text = text.replace("{%DIST%}", "" + distance.toFixed(2) + "m", 1)
        }else{
          text = text.replace("{%DIST%}", "" + ((distance/1000).toFixed(2)) + "km", 1)
        }
        var infoWindow = new google.maps.InfoWindow({
          content: text
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

        this.map.addListener('click', (e) => {
          // Clear the previous marker when clicking the map
          this.removeMarkers()

          // Show the new marker
          this.putMarker(e.latLng)

          // Save latLng
          this.selectedLatLng = e.latLng

          this.$emit('locationSelected', e.latLng);
        })   
      },
      goToNextRound() {
        // Reset
        this.selectedLatLng = null
        this.isGuessButtonClicked = false
        this.isSelected = false
        this.isNextButtonVisible = false
        this.map.setCenter(new google.maps.LatLng(37.869260, -122.254811));
        this.map.setZoom(1);

        if (this.$viewport.width < 450) {
          // Hide the map if the player is on mobile
          this.hideMap()
        } else {
          // Set the opacity of the map again if the player is on pc
          this.mouseOutMap()
        }

        this.removeMarkers()
        this.removePolylines()

        // Replace the streetview with the next one
        this.$emit('goToNextRound')
      },
      finishGame() {
        this.dialogSummary = false
        this.room.child('isGameDone/player' + this.playerNumber).set(true)
        this.$emit('finishGame')
      },
      mouseOverMap() {
        // Focus on map
        if (this.$viewport.width > 450) {
          document.getElementById('map').style.opacity = 1.0
        }
      },
      mouseOutMap() {
        // Focus on map while the player selected a location
        // Otherwise set the opacity of the map
        if (this.isSelected == false && this.$viewport.width > 450) {
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

      this.room = firebase.database().ref(this.roomName);

        var outerbounds = [ // covers the (mercator projection) world
              new google.maps.LatLng(-85.1054596961173, -180),
              new google.maps.LatLng(85.1054596961173, -180),
              new google.maps.LatLng(85.1054596961173, 180),
              new google.maps.LatLng(-85.1054596961173, 180),
              new google.maps.LatLng(-85.1054596961173, 0)
          ];

      let self = this;
      this.room.on('value', (snapshot) => {
        if(self.drewPolygon1 && this.playerNumber == 1) {
          return;
        }
        if(self.drewPolygon2 && this.playerNumber == 2) {
          return;
        }
        if (snapshot.hasChild('active')) {
          let areasData = snapshot.child('areas').val()
          if(areasData == undefined || areasData == null || areasData.length == undefined) {
            return;
          }
          let polyData = [];
          polyData.push(outerbounds);

          for(let i = 0;i<areasData.length;i++) {
            let area = areasData[i];
            let ar = [];
            for(let j = 0;j<area.length;j++) {
              let p = area[j];
              ar.push(new google.maps.LatLng(p.lat, p.lng));
            }
            if(google.maps.geometry.spherical.computeSignedArea(ar) >= 0 ){
                polyData.push(ar);
            } else {
                polyData.push(ar.reverse());
            }
          }
          if(self.playerNumber == 1) {
            self.drewPolygon1 = true;
          }else {
            self.drewPolygon2 = true;
          }
          let polygon = new google.maps.Polygon({
                            paths: polyData,
                            strokeColor: "#000000",
                            strokeOpacity: 0.6,
                            strokeWeight: 1,
                            map:self.map
           });
          }
      })


      let mapE = document.getElementById('map')
      if (!mapE.className.includes("reszw")) {
        mapE.className += " reszw"
        var rz = document.createElement('div')
        rz.id = 'mrz-parent'
        rz.style = "width: 20px;height: 20px;position: absolute;right: 0;top: 0;cursor: ne-resize; margin-right:-10px; margin-top:-10px;"
        mapE.appendChild(rz)
        var rs = document.createElement('div')
        rs.id = 'mrz'
        rs.style = "width: 20px; height: 20px; position:fixed; "
        rz.appendChild(rs)
        this.mWidth = mapE.clientWidth
        this.mHeight = mapE.clientHeight

        var startResize = function(evt) {
          self.mX = evt.screenX;
          self.mY = evt.screenY;
        };

        var resize = function(evt) {
          self.mDX = evt.screenX - self.mX;
          self.mDY = evt.screenY - self.mY;
          self.mX = evt.screenX;
          self.mY = evt.screenY;
          self.mWidth += self.mDX;
          self.mHeight -= self.mDY;
          mapE.style.width = self.mWidth + "px";
          mapE.style.height = self.mHeight + "px";
        };

        rs.addEventListener("mousedown", function(evt) {
          startResize(evt);
          document.body.addEventListener("mousemove", resize);
          document.body.addEventListener("mouseup", function() {
            document.body.removeEventListener("mousemove", resize);
          });
        });
      }

      this.room.on('value', (snapshot) => {
        // Check if the room is already removed
        if (snapshot.hasChild('active')) {

          // Allow players to move on to the next round when every players guess locations
          if (snapshot.child('guess').numChildren() == snapshot.child('size').val()) {
            this.$emit('showResult')

            // Put markers and draw polylines on the map
            var i = 0
            var j = 1
            snapshot.child('guess').forEach((childSnapshot) => {
              var lat = childSnapshot.child('latitude').val()
              var lng = childSnapshot.child('longitude').val()
              var latLng = new google.maps.LatLng({lat: lat, lng: lng});

              var playerName = snapshot.child('playerName').child(childSnapshot.key).val()
              var distance = snapshot.child('round' + this.round + '/player' + j).val()

              this.drawPolyline(latLng, i)
              this.putMarker(latLng)
              this.setInfoWindow(playerName, distance)
              i++
              j++
            })
            this.putMarker(this.randomLatLng)

            // Remove guess node every time the round is done
            this.room.child('guess').remove()

            if (this.round >= 5) {
              // Show summary button
              snapshot.child('finalScore').forEach((childSnapshot) => {
                var playerName = snapshot.child('playerName').child(childSnapshot.key).val()
                var finalScore = childSnapshot.val()

                var scoreText = finalScore.toFixed(2) + "m";
                if(finalScore > 1000) {
                  scoreText = (finalScore/1000).toFixed(2) + "km";
                }
                this.summaryTexts.push({
                  playerName: playerName,
                  finalScore: scoreText,
                })
              })

              this.isSummaryButtonVisible = true

            } else {
              // Show next button
              this.isNextButtonVisible = true
            }
          }

          // Allow other players to move on to the next round when the next street view is set
          if (snapshot.child('streetView').numChildren() == this.round + 1) {
            this.isNextStreetViewReady = true
          }
        }
      })
    },
  }
</script>

<style scoped>
  #make-guess-button, #guess-button, #next-button, #summary-button {
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
  }

  #make-guess-button, #guess-button {
    width: 360px;
  }

  #next-button, #summary-button {
    width: 450px;
  }

  #map {
    position: absolute;
    bottom: 60px;
    left: 10px;
    z-index: 3;
    opacity: 0.7;
    height: 240px;
    width: 360px;
    transform-origin: bottom left;
    transition: transform 0.3s;
  }

  #make-guess-button, #guess-button {
    background-color: #212121;
  }

  #guess-button:hover {
    opacity: 1.0;
  }

  #summary-button {
    background-color: #F44336;
  }

  @media (max-width: 900px) {
    #make-guess-button, #guess-button {
      width: 300px;
    }

    #next-button, #summary-button {
      width: 375px;
    }

    #map {
      height: 200px; 
      width: 300px;
    }
  }

  @media (max-width: 450px) {
    #make-guess-button, #guess-button, #next-button, #summary-button {
      bottom: -50px;
    }

    #next-button, #summary-button {
      width: 300px;
    }

    #map {
      bottom: -280px;
      height: 200px; 
      width: 300px;
      opacity: 1.0;
      transition: transform 1s;
    }

    #hide-map-button {
      position: absolute;
      bottom: 210px;
      left: 300px;
      z-index: 3;
    }
  } 
</style>
