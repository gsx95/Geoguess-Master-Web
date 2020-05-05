<template>
  <div>
    <div id="section-top">
      <v-img
        id="image-earth"
        src="../assets/earth.jpg" />
      <v-container fluid>
        <v-row
          id="row-buttons" 
          justify="center">
          <DialogRoom />
        </v-row>
      </v-container>
    </div>
        <v-footer
      id="footer"
      absolute
      color="#061422"
      height="210">
      <v-container>
        <v-row justify="center">
          <v-btn
            class="ml-4 mr-4" 
            icon
            color="#FFFFFF"
            href="https://github.com/spider-hand/Geoguess-Master-Web">
            <v-icon size="30">mdi-github-circle</v-icon>
          </v-btn>
        </v-row>
        <v-row
          class="mt-8" 
          justify="center">
          <span id="copyright-text">All rights reserved. Copyright © {{ new Date().getFullYear() }} <strong>Spider Hand</strong></span>
        </v-row>
        <v-row
          class="mt-1"
          justify="center">
          <span id="credit-text">
            Photo by <a href="https://unsplash.com/@nasa">NASA</a> on <a href="https://unsplash.com/">Unsplash</a>, 
            Design by <a href="http://www.pauloxgomes.com/">Paulo Gomes</a>
          </span>
        </v-row>
        <v-row
          class="mt-3"
          justify="center">
          <v-btn
            text
            color="#FFFFFF"
            @click="$router.push('privacy-policy')">
            <span id="privacy-policy">Privacy Policy</span>
          </v-btn>
        </v-row>
      </v-container>
    </v-footer>
  </div>
</template>

<script>
  import firebase from 'firebase/app'
  import 'firebase/database'

  import DialogRoom from '@/components/DialogRoom'

  export default {
    components: {
      DialogRoom,
    },
    data() {
      return {
        record: localStorage.getItem('record')
      }
    }
  }
</script>

<style scoped>
  span, .section-header {
    font-family: Montsetrrat;
  }

  #section-top {
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0px;
    top: 0px;
    background: linear-gradient(#071D3E, #06121D);
  }

  #image-earth {
    position: absolute;
    width: 100%;
    height: 100%;
    opacity: 0.4;
  }

  #row-record {
    position: absolute;
    width: 100%;
    top: 200px;
  }

  #record {
    font-size: 26px;
    font-weight: 700;
    color: #FFFFFF;
  }

  #row-buttons {
    position: absolute;
    width: 100%;
    top: 300px;
  }



  #copyright-text, #credit-text {
    font-size: 12px;
    color: #7D7D7D;
  }

  #privacy-policy {
    font-size: 12px;
  }


  @media (max-width: 450px) {
    #row-buttons {
      top: 200px;
    }


    .section-description {
      top: 96px;
      padding: 0 12%;
    }

  }
</style>
