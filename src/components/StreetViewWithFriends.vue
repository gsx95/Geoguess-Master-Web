<template>
  <div id="game-page">
    <HeaderGame
      ref="header"
      :score="scoreHeader"
      :round="round"
      :remainingTime="remainingTime" />
    <div id="street-view-container">
      <div id="street-view">
      </div>
      <MapsWithFriends
        ref="map"
        :randomLatLng="randomLatLng"
        :roomName="roomName"
        :playerNumber="playerNumber"
        :isReady="isReady"
        :round="round"
        :score="score"
        @calculateDistance="updateScore"
        @showResult="showResult"
        @goToNextRound="goToNextRound"
        @finishGame="finishGame" />
    </div>
    <div id="back-btn-container">
      <button role="button" id="back-btn">Back to Start</button>
      </div>
    <v-overlay 
      :value="overlay"
      opacity="0.8" 
      z-index="2"/>
    <DialogMessage 
      :dialogMessage="dialogMessage"
      :dialogTitle="dialogTitle"
      :dialogText="dialogText" />
  </div>
</template>

<script>

  import firebase from 'firebase/app'
  import 'firebase/database'

  import HeaderGame from '@/components/HeaderGame'
  import MapsWithFriends from '@/components/MapsWithFriends'
  import DialogMessage from '@/components/DialogMessage'

  import { randomPoint, randomPolygon } from '@turf/random';
  import _ from '@turf/boolean-point-in-polygon';
  import { polygon } from '@turf/helpers'
  import booleanPointInPolygon from '@turf/boolean-point-in-polygon'

  export default {
    props: [
      'roomName',
      'playerNumber',
      'areas'
    ],
    components: {
      HeaderGame,
      MapsWithFriends,
      DialogMessage,
    },
    data() {
      return {
        randomLatLng: null,
        randomLat: null,
        randomLng: null,
        score: 0,
        scoreHeader: 0,
        round: 1,
        timeLimitation: 0,
        remainingTime: 0,
        hasTimerStarted: false,
        hasLocationSelected: false,
        overlay: false,
        room: null,
        isReady: false,
        dialogMessage: true,
        dialogTitle: 'Waiting for other players...',
        searchRadius: 10,
        dialogText: '',
      }
    },
    methods: {
      loadStreetView() {
        var service = new google.maps.StreetViewService()
        service.getPanorama({
          location: this.getRandomLatLng(this.areas),
          preference: 'nearest',
          radius: this.searchRadius,
          source: 'outdoor',
        }, this.checkStreetView)
      },
      loadDecidedStreetView() {
        // Other players load the decided streetview the first player loaded
        var service = new google.maps.StreetViewService()
        
        console.log("try:   " + this.randomLat + "  " + this.randomLng)
        service.getPanorama({
          location: {
            lat: this.randomLat,
            lng: this.randomLng,
          },
          preference: 'nearest',
          radius: 5,
          source: 'outdoor',
        }, this.checkStreetView)        
      },
      getRandomLatLng(selectedAreas) {
        if(selectedAreas == null || selectedAreas == undefined || !selectedAreas.length || selectedAreas.length <= 0) {
          var lat = (Math.random() * 170) - 85
          var lng = (Math.random() * 360) - 180
          return new google.maps.LatLng(lat, lng)
        }
        let randomAreaIndex = this.getRandomInt(0, selectedAreas.length - 1)
        let area = selectedAreas[randomAreaIndex]
        let boundBox = this.getAreaBoundBox(area)
        let turfPoly = this.toTurfPoly(area)
        
        let pointInsidePolygon = null;

        while(pointInsidePolygon == null || pointInsidePolygon == undefined){
          let rPoints = randomPoint(10, {bbox: boundBox}).features
          for(let i = 0; i < rPoints.length; i++) {
            let testPoint = rPoints[i]
            let isInside = booleanPointInPolygon(testPoint, turfPoly)
            if(isInside) {
              pointInsidePolygon = testPoint
              break;
            }
          }
        }
        
        var lat = pointInsidePolygon.geometry.coordinates[0]
        var lng = pointInsidePolygon.geometry.coordinates[1]
        console.log("lat/lon:      "+ lat + "  " + lng)
        return new google.maps.LatLng(lat, lng)
      },
      toTurfPoly(area) {
        let points = this.pointsOfArea(area)
        let pArr = []
        for(let i = 0;i < points.length; i++) {
          let point = points[i]
          let lat = point.lat()
          let lon = point.lng()
          let coords = [lat, lon]
          pArr.push(coords)
        }

        let firstPoint = points[0]
        let coords = [firstPoint.lat(), firstPoint.lng()]
        pArr.push(coords)        
        return polygon([pArr])
      },
      getAreaBoundBox(area) {
        let points = this.pointsOfArea(area)
        let minLat = 500
        let maxLat = -500
        let minLon = 500
        let maxLon = -500
        for(let i = 0;i < points.length; i++) {
          let point = points[i]
          let lat = point.lat()
          let lon = point.lng()

          minLat = (lat < minLat) ? lat : minLat
          maxLat = (lat > maxLat) ? lat : maxLat
          minLon = (lon < minLon) ? lon : minLon
          maxLon = (lon > maxLon) ? lon : maxLon
        }
        return [
          minLat,
          minLon,
          maxLat,
          maxLon,
        ]
      },
      pointsOfArea(area) {
        return area.getPaths().getArray()[0].i
      },
      getRandomInt(min, max) {
        min = Math.ceil(min);
        max = Math.floor(max);
        return Math.floor(Math.random() * (max - min + 1)) + min;
      },
      checkStreetView(data, status) {
        // Generate random streetview until the valid one is generated
        if (status == 'OK') {
          var panorama = new google.maps.StreetViewPanorama(document.getElementById('street-view'))
          panorama.setOptions({
            addressControl: false,
            fullscreenControl: false,
            motionTracking: false,
            motionTrackingControl: false,
            showRoadLabels: false,
          })
          panorama.setPano(data.location.pano)
          panorama.setPov({
            heading: 270,
            pitch: 0,
          })

          // Save the location's latitude and longitude
          this.randomLatLng = data.location.latLng
          this.searchRadius = 10
          // Put the streetview's location into firebase
          console.log("decided on:  " + this.randomLatLng.lat() + "   " + this.randomLatLng.lng())
          this.randomLat = this.randomLatLng.lat();
          this.randomLng = this.randomLatLng.lng();
          this.room.child('streetView/round' + this.round).set({
            latitude: this.randomLatLng.lat(),
            longitude:this.randomLatLng.lng()
          })

        } else {
          console.log(status)
          this.searchRadius = this.searchRadius * 2
          this.loadStreetView()
        }
      },
      startTimer() {
        if (!this.hasLocationSelected) {
          if (this.remainingTime > 0) {
            setTimeout(() => {
              this.remainingTime -= 1
              this.startTimer()
            }, 1000)
          } else {
            // Set a random location if the player didn't select a location in time
            this.$refs.map.selectRandomLocation(this.getRandomLatLng())
          }
        }
      },
      updateScore(distance) {
        // Update the score and save it into firebase
        this.hasLocationSelected = true
        this.score += distance
        this.room.child('finalScore/player' + this.playerNumber).set(this.score)

        // Wait for other players to guess locations
        this.dialogTitle = 'Waiting for other players...'
        this.dialogMessage = true
      },
      showResult() {
        this.scoreHeader = this.score  // Update the score on header after every players guess locations
        this.dialogMessage = false
        this.overlay = true
      },
      goToNextRound() {
        // Reset
        this.randomLatLng = null
        this.overlay = false
        this.isReady = false  // Turn off the flag so the click event can be added in the next round
        this.dialogMessage = true  // Show the dialog while waiting for other players
        this.hasTimerStarted = false
        this.hasLocationSelected = false

        // Update the round
        this.round += 1

        if (this.playerNumber == 1) {
          this.loadStreetView()
        } else {
          // Trigger listener and load the next streetview
          this.room.child('trigger/player' + this.playerNumber).set(this.round)
          this.$refs.map.startNextRound()
        }

      },
      backToStart() {
        this.loadDecidedStreetView();
      },
      exitGame() {
        // Disable the listener and force the players to exit the game
        this.dialogTitle = 'Redirect to Home Page...'
        this.dialogText = 'You are forced to exit the game. Redirect to home page after 5 seconds...'
        this.dialogMessage = true
        this.room.off()
        this.room.remove()
        
        setTimeout(() => {
          this.$router.push({ name: 'home' })
        }, 5000)
      },
      finishGame() {
        // Open the dialog while waiting for other players to finsih the game
        this.dialogTitle = 'Waiting for other players to finish the game...'
        this.dialogText = ''
        this.dialogMessage = true
      }
    },
    mounted() {
      if (this.playerNumber == 1) {
        this.loadStreetView()
      }
      // Set a room name if it's null to detect when the user refresh the page
      if (this.roomName == null) {
        this.roomName = 'defaultRoomName'
      }
      this.room = firebase.database().ref(this.roomName)
      this.room.child('active').set(true)
      this.room.on('value', (snapshot) => {
        // Check if the room is already removed
        if (snapshot.hasChild('active')) {

          // Put the player into the current round node if the player is not put yet
          if (!snapshot.child('round' + this.round).hasChild('player' + this.playerNumber)) {

            this.room.child('round' + this.round).child('player' + this.playerNumber).set(0)

            // Other players load the streetview the first player loaded earlier
            if (this.playerNumber != 1) {
              this.randomLat = snapshot.child('streetView/round' + this.round + '/latitude').val()
              this.randomLng = snapshot.child('streetView/round' + this.round + '/longitude').val()
              console.log("1:   " + this.randomLat + "  " + this.randomLng)
              this.loadDecidedStreetView()
            }
          }
          let self = this;
          // Enable guess button when every players are put into the current round's node
          if (snapshot.child('round' + this.round).numChildren() == snapshot.child('size').val()) {
            document.getElementById("back-btn").addEventListener("click", function(){
              self.backToStart()
            })
            // Close the dialog when evryone is ready
            if (this.isReady == false) {
              this.dialogMessage = false
            }

            this.isReady = true
            this.$refs.map.startNextRound()

            // Countdown timer starts
            this.timeLimitation = snapshot.child('timeLimitation').val() * 60

            if (this.timeLimitation != 0) {
              if (!this.hasTimerStarted) {
                this.remainingTime = this.timeLimitation
                this.startTimer()
                this.hasTimerStarted = true
              }
            }
          }

          // Delete the room when everyone finished the game
          if (snapshot.child('isGameDone').numChildren() == snapshot.child('size').val()) {
            this.room.child('active').remove()
          }

        } else {
          // Force the players to exit the game when 'Active' is removed
          this.exitGame()
        }
      })

      window.addEventListener('popstate', (event) => {
        // Remove the room when the player pressed the back button on browser
        this.room.child('active').remove()
        this.room.off()
      })

      window.addEventListener('beforeunload', (event) => {
        // Remove the room when the player refreshes the window
        this.room.child('active').remove()
      })

      // Force to exit the game if it's still the name this is set programmatically
      if (this.roomName == 'defaultRoomName') {
        this.room.child('active').remove()
      }
    }
  }
</script>

<style scoped>
  #game-page {
    position: relative;
    height: 100%; 
    width: 100%; 
    top: 0; 
    left: 0;
  }

  #back-btn-container {
    bottom: 0;
    height: 40px;
    width: 150px;
    z-index: 3;
    position: absolute;
    left: 40%;
  }

  #back-btn {
    width:100%; 
    height:100%;
    background-color: #313131;
    border-radius: 20px 20px 0px 0px;
    color: white;
  }

  #street-view-container {
    position: absolute;
    height: 100%; 
    width: 100%; 
    top: 0; 
    left: 0; 
  }

  #street-view {
    position: relative;
    min-height: 100%;
    width: 100%;
  } 

  @media (max-width: 450px) {
    #game-page {
      position: fixed;
      height: 92%;
    }
  }
</style>
