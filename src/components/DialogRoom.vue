<template>
  <v-dialog 
    persistent
    :fullscreen="$viewport.width < 450"
    v-model="dialogRoom"
    max-width="600">
    <template v-slot:activator="{ on }">
      <v-btn 
        id="multi-player-button"
        class="ml-8 mr-8"
        dark
        color="#43B581"
        v-on="on">
        With Friends
      </v-btn>
    </template>
    <v-card color="#061422">
      <v-card-title>
        <span id="card-title">{{ cardTitle }}</span>
      </v-card-title>
      <v-card-text>
        <v-container>
          <div v-if="cardTitle == 'Set location'">
            <div id="selector-map"></div>
            <br>
            <v-btn depressed @click="fsMap" id="fs-btn">Fullscreen</v-btn>
          </div>
          <v-row>
            <v-col 
              cols="12"
              v-if="cardTitle == 'Type a room name'">
              <v-text-field
                dark
                maxlength="10"
                autofocus
                :error-messages="errorMessage"
                v-model="roomName"></v-text-field>
            </v-col>
            <v-col 
              cols="4"
              v-if="cardTitle == 'Set a room size'">
              <v-select 
                dark
                v-model="roomSize"
                :items="roomSizeItems"></v-select>
            </v-col>
            <v-col 
              cols="6"
              sm="4"
              md="4"
              lg="4"
              xl="4"
              v-if="cardTitle == 'Set a time limitation'">
              <v-select
                dark 
                v-model="timeLimitation"
                :items="timeLimitationItems"></v-select>
            </v-col>
            <v-col 
              cols="12"
              v-if="cardTitle == 'Type a player name'">
              <v-text-field 
                dark
                maxlength="10"
                v-model="playerName"></v-text-field>
            </v-col>
          </v-row>
        </v-container>
      </v-card-text>
      <v-card-actions>
        <div class="flex-grow-1"></div>
        <v-btn
          dark
          depressed
          color="#FF5252"
          @click="clickNext">NEXT</v-btn>
        <v-btn
          dark
          depressed
          color="#43B581"
          @click="cancel">CANCEL</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&libraries=drawing&callback=initMap"
         async defer></script>
<script>
  import firebase from 'firebase/app'
  import 'firebase/database'

  export default {
    data() {
      return {
        dialogRoom: false,
        cardTitle: 'Type a room name',
        errorMessage: '',
        room: null,
        playerNumber: null,
        roomName: '',
        roomSize: 2,
        roomSizeItems: [
          {
            text: '2',
            value: 2,
          },
          {
            text: '3',
            value: 3,
          },
          {
            text: '4',
            value: 4,
          },
          {
            text: '5',
            value: 5,
          },          
        ],
        timeLimitation: 0,
        timeLimitationItems: [
          {
            text: 'Infinite',
            value: 0,
          },
          {
            text: '1',
            value: 1,
          },
          {
            text: '2',
            value: 2,
          },
          {
            text: '3',
            value: 3,
          },
          {
            text: '4',
            value: 4,
          },
          {
            text: '5',
            value: 5,
          }, 
          {
            text: '6',
            value: 6,
          },
          {
            text: '7',
            value: 7,
          },
          {
            text: '8',
            value: 8,
          },
          {
            text: '9',
            value: 9,
          },
          {
            text: '10',
            value: 10,
          },          
        ],
        playerName: '',
      }
    },
    methods: {
      fsMap() {
        var mapDiv = document.getElementById('selector-map')
        if(mapDiv.webkitRequestFullscreen) {
          mapDiv.webkitRequestFullscreen(Element.ALLOW_KEYBOARD_INPUT);
        }
        if(mapDiv.mozRequestFullScreen) {
          mapDiv.mozRequestFullScreen();
        }
        if(mapDiv.msRequestFullscreen) {
          mapDiv.msRequestFullscreen();
        }
        if(mapDiv.webkitRequestFullscreen) {
          mapDiv.requestFullscreen(); // standard
        }

      },
      clickNext() {
        if (this.cardTitle == 'Type a room name') {
          this.searchRoom()
        }
        else if (this.cardTitle == 'Set a room size') {
          this.setRoomSize()
        } else if (this.cardTitle == 'Set a time limitation') {
          this.setTimeLimitation()
        } else if (this.cardTitle == 'Type a player name') {
          this.setPlayerName()
        }else if (this.cardTitle == 'Set location') {
          this.setLocation()
        }
      },
      searchRoom() {
        if (this.roomName == '') {
          this.errorMessage = 'Invalid name. Please try another.'
        } else {
          this.room = firebase.database().ref(this.roomName)

          this.room.once('value', (snapshot) => {
            var numberOfPlayers = snapshot.child('playerName').numChildren()
            this.playerNumber = numberOfPlayers + 1

            if (numberOfPlayers == 0) {
              // Put the tentative player's name into the room node
              // So that other player can't enter as the first player while the player decide the name and room size
              this.room.child('playerName').update({
                player1: 'player1'
              }, (error) => {
                if (!error) {
                  // Put the timestamp the room is created so the expired rooms can be removed by cloud function
                  this.room.update({
                    createdAt: firebase.database.ServerValue.TIMESTAMP,
                  })
                  // Ask the first player to decide the room size
                  this.cardTitle = 'Set a room size'
                }
              })

            } else if (!snapshot.hasChild('size') || !snapshot.hasChild('streetView')) {
              // Prevent other players from getting into the room earlier than the first player
              this.errorMessage = 'The first player is creating the room right now. Please wait and try again.'

            } else if (numberOfPlayers >= snapshot.child('size').val()) {
              // In case the room is already full
              this.errorMessage = 'This room is already full. Please try another.'

            } else {
              // Put other player's tentative name
              this.room.child('playerName/player' + this.playerNumber).set('player' + this.playerNumber, (error) => {
                if (!error) {
                  // Let the other players decide the player name
                  this.cardTitle = 'Type a player name'
                }
              })
            }

          })
        }
      },
      setRoomSize() {
        this.room.update({
          size: this.roomSize
        }, (error) => {
          if (!error) {
            // Ask the first player to set time limitation
            this.cardTitle = 'Set a time limitation'
          }
        })
      },
      setTimeLimitation() {
        this.room.update({
          timeLimitation: this.timeLimitation
        }, (error) => {
          if (!error) {
            this.cardTitle = 'Type a player name'
          }
        })
      },
      setPlayerName() {
        let self = this;
        this.room.child('playerName/player' + this.playerNumber).set(this.playerName, (error) => {
          if (!error) {

            if(this.playerNumber != 1) {
              this.$router.push({
                name: 'with-friends',
                params: { 
                  roomName: this.roomName, 
                  playerNumber: this.playerNumber,
                  areas: this.areas,
                }
              })
              return
            }
            this.cardTitle = 'Set location'
            window.addEventListener('keydown', function(e) {
              if (e.keyCode != 46 && e.keyCode != 8) {
                return
              }
              if(self.selectedPolygon == null || self.selectedPolygon == undefined) {
                return
              }
              self.deleteSelectedPolygon()
            });
            this.$nextTick(function () {
              this.areas = []
              this.map = new google.maps.Map(document.getElementById('selector-map'), {
                  center: {lat: 37.869260, lng: -122.254811},
                  zoom: 1,
                  fullscreenControl: false,
                  mapTypeControl: false,
                  streetViewControl: false,
              })

              this.drawingManager = new google.maps.drawing.DrawingManager({
                drawingMode: google.maps.drawing.OverlayType.POLYGON,
                drawingControl: true,
                drawingControlOptions: {
                  position: google.maps.ControlPosition.TOP_CENTER,
                  drawingModes: ['polygon']
                },
                markerOptions: {icon: 'https://developers.google.com/maps/documentation/javascript/examples/full/images/beachflag.png'},
              });
              google.maps.event.addListener(this.drawingManager, 'overlaycomplete', function(event) {
                if (event.type == 'polygon') {
                  event.overlay.setMap(null);
                  if(event.type == 'polygon') {
                    self.areas.push(new google.maps.Polygon({
                      paths: event.overlay.getPaths().getArray(),
                      strokeColor: '#000000',
                      strokeOpacity: 0.8,
                      strokeWeight: 3,
                      fillColor: '#000000',
                      fillOpacity: 0.35
                    }));
                    self.updateAreasDB();
                  }
                  self.areas[self.areas.length - 1].setMap(self.map);
                  self.areas[self.areas.length - 1].addListener('click', function(e, p1, p2) {
                    self.selectArea(this, self)
                  });
                  console.log(self.areas.length);
                } 
              });
              this.drawingManager.setMap(this.map);

            })
          }
        })
      },
      updateAreasDB() {
        let areasData = [];
        for (let i = 0;i<this.areas.length;i++) {
            let area = this.areas[i];
            let points = area.getPaths().getArray()[0].i;
            let areaData = [];
            for(let j = 0;j<points.length;j++) {
                let point = points[j];
                let lat = point.lat()
                let lon = point.lng()
                areaData.push({"lat": lat, "lng" : lon});
            }
            areasData.push(areaData);
        }
        this.room.update({
          areas: areasData,
        })
      },
      selectArea(polygon, self) {
        for(let i = 0; i < self.areas.length; i++) {
          self.setPolygonColor(self.areas[i], '#000000')
        }
        self.setPolygonColor(polygon, '#ff0000')
        self.selectedPolygon = polygon
      },
      setPolygonColor(polygon, color) {
        var options = {
          strokeColor: color,
          strokeOpacity: 0.8,
          strokeWeight: 3,
          fillColor: color,
          fillOpacity: 0.35
        }
        options.paths = polygon.getPaths().getArray()
        polygon.setOptions(options)
      },
      deleteSelectedPolygon() {
        this.selectedPolygon.setMap(null)
        for(let i = 0;i < this.areas.length; i++) {
          if(this.areas[i] == this.selectedPolygon) {
            this.areas.splice(i, 1);
            break;
          }
        }
        this.selectedPolygon = null
        this.updateAreasDB();
      },
      setLocation() {
      // Start the game
      this.$router.push({
        name: 'with-friends',
        params: { 
          roomName: this.roomName, 
          playerNumber: this.playerNumber,
          areas: this.areas,
        }
      })

      },
      cancel() {
        // Reset
        this.cardTitle = 'Type a room name'
        this.roomName = ''
        this.errorMessage = ''

        // Remove the room
        this.dialogRoom = false
        if (this.room != null) {
          if (this.playerNumber == 1) {
            // Remove the entire node if the player is the first player
            this.room.remove()
          } else {
            // Remove only the player's name node if the player isn't the first player
            this.room.child('playerName/player' + this.playerNumber).remove()
          }
        }
      }
    }
  }
</script>

<style scoped>
  span {
    font-family: Montsetrrat;
  }

  #multi-player-button {
    height: 44px;
    width: 240px;
    border-radius: 40px;
  }

  #card-title {
    font-size: 16px;
    font-weight: 500;
    color: #FFFFFF;
    opacity: 0.9;
  }

  #selector-map {
    height:400px;
    width:500px;
  }

  #fs-btn {
    color: #fff;
    background-color: #ffd96778;
    height:auto;
    padding: 3px 15px 3px 15px;
  }
</style>
