## Changes
Changes I made on this fork:

* Players can restrict the playing area (the player who creates the room can draw one or multiple areas on a world map; only points within those areas are chosen for the game)
* Let players resize the mini map
* Let players go back to the starting point of this round (useful if you play e.g. in one city, and distances within a meter-radius matter)
* Display the distance to the actual location in meters if appropriate
* Remove Singleplayer

What i still want to do:
* Introduce backend application: 
  * Remove firebase dependecy and store data via backend server
  * Put logic in backend
* Let room creation be one screen
* List available rooms on home
* Rewrite frontend to only use vanilla JS after introducing the backend
  
  
### Build Setup
You need to configure Google Maps Platform and Firebase to make game work. 
See the instructions below. 

- [Google Maps API](https://developers.google.com/maps/documentation/javascript/get-api-key#get-the-api-key)  
- [Firebase](https://firebase.google.com/docs/database/web/start)  
 
Once you get an API key and register the project with Firebase, create files named `.env.development.local` and `.env.production.local` inside this project to put environment variables. 
The files should be like this. 

`.env.production.local`
```
NODE_ENV=production
VUE_APP_API_KEY=YOUR_GOOGLE_MAPS_API_KEY
VUE_APP_FIREBASE_API_KEY=YOUR_FIREBASE_API_KEY
VUE_APP_FIREBASE_PROJECT_ID=YOUR_FIREBASE_PROJECT_ID
VUE_APP_FIREBASE_MESSAGING_SENDER_ID=YOUR_FIREBASE_MESSAGING_SENDER_ID
VUE_APP_FIREBASE_APP_ID=YOUR_FIREBASE_APP_ID
VUE_APP_FIREBASE_MEASUREMENT_ID=YOUR_FIREBASE_MEASUREMENT_ID
```

`.env.development.local`
```
NODE_ENV=development
...
```

Now you can run this game.  
Download Node.js [here](https://nodejs.org/en/download/) if you don't have and make sure you can run `npm` from the terminal.

```
# install dependencies
npm install

# run locally
npm run serve

# build for production
npm run build
```

### License
Licensed under [MIT License](https://github.com/spider-hand/Geoguess-Master-Web/blob/master/LICENSE)
