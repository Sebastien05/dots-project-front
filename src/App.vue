<template>
  <v-app>
    <v-app-bar
      app
      color="teal"
      dark
    >
      <v-toolbar>
        <v-toolbar-title>
          Dot project 
          <v-icon> blur_circular</v-icon>
        </v-toolbar-title>
        <v-spacer></v-spacer>
          by Sebastien Lefevre
      </v-toolbar>
    </v-app-bar>
    
    <v-content>
      <v-fade-transition mode="out-in">
        <div key="home" v-if="!started" class="bg">
          <div class="bg-img">
            <v-hover v-slot:default="{ hover }">
              <v-card raised :elevation="hover ? 15: 6">
                <v-btn v-if ="!started"
                  outlined light text
                  width="150" height="60"
                  @click="started = true">
                  Start a demo
                </v-btn>
              </v-card>
            </v-hover>
          </div>
        </div>
        <div key="canvas" v-else class="bg bg-column">
          <v-hover v-slot:default="{ hover }">
            <v-card raised :elevation="hover ? 9 : 6">
              <canvas 
                ref="canvas" 
                width="800"
                height="550"
                >
              </canvas>
            </v-card>
          </v-hover>
          <div style="margin-top:40px">
            <div class="d-flex justify-space-between">
              <v-select
                style="max-width:300px; margin-top:2px;"
                v-model="selectedRandom"
                :items="randomType"
                label="Select Random Type"
                outlined
              ></v-select>
              <v-btn outlined light text
                style="margin-left:10px;margin-right:10px"
                color="primary"
                height="60"
                @click="dotGeneration">
                Generate
              </v-btn>
              <div style="440px">
                <v-slider
                  v-model="slider"
                  class="align-center"
                  style="width: 280px;"
                  :max="computedMax"
                  :min="min"
                  hide-details
                >
                  <template v-slot:append>
                    <v-text-field
                      v-model="slider"
                      hide-details
                      type="number"
                      style="width: 50px;"
                    ></v-text-field>
                  </template>
                </v-slider>
              </div>
            </div>
          </div>
          <div class="d-flex justify-space-between">
            <v-select
              style="max-width:250px;height:70px"
              v-model="selectedAlgo"
              :items="algorithms"
              label="Select Algorithm"
              filled
            ></v-select>
            <v-btn outlined light text
                color="primary"
                height="60"
                filled
                :disabled="disabledApply"
                @click="applyAlgo">
                Apply
              </v-btn>
          </div>
        </div>
      </v-fade-transition>
    </v-content>
  </v-app>
</template>

<script>

export default {
  name: 'App',

  components: {
  },

  data: () => ({
    started: false,
    randomType: ["Classic Random", "Gaussian Random"],
    algorithms: ["Sort by pixel", "Convex Graham","Ritter"],
    selectedRandom: "Classic Random",
    selectedAlgo: "",
    pointsList: {},
    canvasCtx: null,
    min: 4,
    max: 100,
    slider: 7,
    applied: false,
    applied2: false,
    applied3: false
  }),

  computed: {
    computedMax () {
      return this.selectedRandom != "Classic Random" ? 5000: 100;
    },
    disabledApply () {
      return (this.selectedAlgo == "Ritter" && this.applied)
          || (this.selectedAlgo == "Convex Graham" && this.applied2)
          || (this.selectedAlgo == "Sort by pixel" && this.applied3);
    }
  },

  methods: {
    dotGeneration () {
      let path = "gen";
      if (this.selectedRandom == "Gaussian Random")
        path = "generate-gaussian";
      fetch("http://localhost:3000/dot/"+path, {
          method: 'post',
          body: JSON.stringify({
            number: this.slider,
            width: 800,
            height: 500
          }),
          headers: { "Content-Type": "application/json" }
        })
        .then(res => res.json())
        .then(json => {
          if (!this.canvasCtx) {
            this.canvasCtx = this.$refs.canvas.getContext("2d")
          }
          let ctx = this.canvasCtx
          this.pointsList = json;
          ctx.fillStyle = "black";
          ctx.strokeStyle = "black"
          ctx.clearRect(0, 0, 800, 550);
          for (var x=0; x < this.pointsList.length; x++){
            let p = this.pointsList[x]
            ctx.beginPath();
            ctx.moveTo(p.x-1, p.y-1)
            ctx.arc(p.x, p.y, 1, 0, 2 * Math.PI, true);
            ctx.stroke();
          }
          this.applied = false;
          this.applied2 = false;
          this.applied3 = false;

          ctx.closePath();
        });
    },

    applyAlgo () {
      if (this.selectedAlgo != ""  && this.pointsList.length>0) {
        if (this.selectedAlgo == "Ritter"){
          this.circleRitter();
        } else if (this.selectedAlgo == "Convex Graham"){
          this.graham();
        } else if (this.selectedAlgo == "Sort by pixel") {
          this.sortByPixel();
        }
      }
    },

    circleRitter () {
      fetch("http://localhost:3000/dot/circle-min-ritter", {
        method: 'post',
          body: JSON.stringify({
            list: this.pointsList
          }),
          headers: { "Content-Type": "application/json" }
        })
        .then(res => res.json())
        .then(json => {
          let p = json.point;
          let rad = json.rad;
          let text = "Runtime: "+json.timeExec+"ms"
          let ctx = this.canvasCtx
          ctx.beginPath();
          ctx.strokeStyle = "blue";
          ctx.arc(p.x, p.y, rad, 0, 2 * Math.PI);
          ctx.stroke();
          ctx.clearRect(0, 500, 120, 800);
          ctx.font = '14px Comic Sans MS';
          ctx.fillText(text, 20, 535);
          ctx.closePath();
          ctx.fillStyle = "black";
          this.applied = true;
        });
    },
    graham () {
      fetch("http://localhost:3000/dot/graham", {
        method: 'post',
          body: JSON.stringify({
            list: this.pointsList
          }),
          headers: { "Content-Type": "application/json" }
        })
        .then(res => res.json())
        .then(json => {
          let points = json.points;
          let text = "Runtime: "+json.timeExec+"ms"
          let ctx = this.canvasCtx
          ctx.beginPath();
          ctx.strokeStyle = "blue";
          ctx.moveTo(points[0].x, points[0].y)
          for (var i = 1; i<points.length; i++) {
            let p = points[i]
            ctx.lineTo(p.x, p.y);
          }
          ctx.lineTo(points[0].x, points[0].y);
          ctx.stroke();
          ctx.fillStyle = "black";
          ctx.font = '14px Comic Sans MS';
          ctx.fillText(text, 20, 535);
          ctx.closePath();
          this.applied2 = true;
        });  
    },

    sortByPixel () {
      fetch("http://localhost:3000/dot/sort-by-pixel", {
          method: 'post',
          body: JSON.stringify({
            list: this.pointsList
          }),
          headers: { "Content-Type": "application/json" }
        })
        .then(res => res.json())
        .then(json => {
          let resH = json.resH
          let resD = json.resD
          let ctx = this.canvasCtx
          for (var i=0; i < resH.length; i++){
            let p = resH[i]
            ctx.beginPath();
            ctx.fillStyle = "blue";
            ctx.strokeStyle = "blue"
            ctx.moveTo(p.x, p.y)
            ctx.arc(p.x, p.y, 6, 0, 2 * Math.PI, true);
            ctx.fill();
            ctx.stroke();
          }
          for (var j=0; j < resD.length; j++){
            let p = resD[j]
            ctx.beginPath();
            ctx.fillStyle = "red";
            ctx.strokeStyle = "red";
            ctx.moveTo(p.x, p.y)
            ctx.arc(p.x, p.y, 4, 0, 2 * Math.PI, true);
            ctx.fill();
            ctx.stroke();
          }
          ctx.closePath();
          ctx.fillStyle = "black";
          ctx.strokeStyle = "black"
          this.applied3 = true;
        });
    }
  }
};
</script>

<style>

.bg-column {
  flex-direction: column;
}

.bg {
  display: flex;
  align-items: center;
  justify-content: center;
  width:100%;
  height:100%;
  background-color: white !important;
}

.bg-img {
  /* Add the blur effect */
  background: url("https://upload.wikimedia.org/wikipedia/commons/b/bc/ConvexHull.png");

  display: flex;
  align-items: center;
  justify-content: center;

  min-width: 700px;
  min-height: 600px;
  background-size: cover;
}

.bg-img:before {
  content: "";
  position: absolute;
  min-width: 750px;
  min-height: 600px;
  background: inherit;
  z-index: 0;

  filter        : blur(10px);
  -moz-filter   : blur(10px);
  -webkit-filter: blur(10px);
  -o-filter     : blur(10px);
}
</style>