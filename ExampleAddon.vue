<script>
import ToolTemplate from "../../../src/modules/tools/ToolTemplate.vue";
import { mapGetters, mapMutations } from "vuex";
import getters from "../store/gettersExampleAddon";
import mutations from "../store/mutationsExampleAddon";
import { fromBlob } from "geotiff";
import GeoTIFF from "ol/source/GeoTIFF.js";
import TileLayer from "ol/layer/WebGLTile.js";
import store from "../../../src/app-store";
import * as math from 'mathjs';

export default {
  name: "ExampleAddon",
  components: {
    ToolTemplate,
  },
  computed: {
    ...mapGetters("Tools/ExampleAddon", Object.keys(getters)),
    ...mapGetters("Maps", ["getView"]),
    ...mapGetters("Maps", ["getCoordinateFromPixel"]),
    ...mapGetters("Maps", ["getEventCoordinate"]),
  },

  data() {
    return {
      map: null,
      tiffsource: null,
      fileSelected: false,
      geoTiffLoaded: false,
      width: null,
      height: null,
      tifFileSelecteded: false,
      urlToPngFile: null,
      clickCounter: 0,
      isPictureCoordinatesNext: true,
      imageCoordinates: [],
      mapCoordinates: [],
      newimageCoordinates: [],
      newmapCoordinates: [],
      countcoord:3,
      dotransform: false,
      tiffElement: null,
      tileLayers:[],
    };
  },
  created() {
    this.$on("close", this.close);
  },
  methods: {
    ...mapMutations("Tools/ExampleAddon", Object.keys(mutations)),

    close() {
      this.setActive(false);
      const model = Radio.request("ModelList", "getModelByAttributes", {
        id: this.$store.state.Tools.ExampleAddon.id,
      });

      if (model) {
        model.set("isActive", false);
      }
    },

    async handleFileSelect(event) {
      const fileInput = event.target;
      const file = fileInput.files[0];

      if (file) {
        if (file.name.endsWith(".tif")) {
          this.fileSelected = false;
          const tiff = await fromBlob(file);
          const image = await tiff.getImage();
          const width = image.getWidth();
          const height = image.getHeight();
          this.width = width;
          this.height = height;
          console.log(width);
          console.log(height);

          const origin = image.getOrigin();
          const resolution = image.getResolution();
          const bbox = image.getBoundingBox();

          console.log(origin, resolution, bbox);

          //Geotiff laden:
          const tiffSource = new GeoTIFF({
            sources: [{ url: URL.createObjectURL(file) }],
          });
          this.tiffSource = tiffSource;
          const coordinateSystemTiff = this.getcoordinateSystemTiff(image);
          //aus dem EPSG einen String machen, damit überprüft wird, ob sie identisch sind.
          const epsgString = `EPSG:${coordinateSystemTiff}`;
          console.log(
            "1 = ",
            epsgString,
            " 2 = ",
            this.getView.getProjection().getCode()
          );
          //Überprüfe ob Koordinatensystem der Geotiff gleich Koordinatensystem von der Map ist.
          if (epsgString == this.getView.getProjection().getCode()) {
            console.log("sind gleich");
            this.initializeMap();
            this.geoTiffLoaded = true;
          } else {
            console.log(
              "Datei hat nicht das gleiche Koordinatensystem",
              this.getView.getProjection().getCode()
            );
            //Wenn Kooridnatensystem nicht gleich, dann wandel um.
          }
        } else {
          console.error("Bitte wählen Sie eine GeoTIFF-Datei aus.");
        }
      }
    },
    //Bild wird geladen, wenn tif ausgewählt.
    async tifFileSelected(event) {
      const fileInput = event.target;
      const file = fileInput.files[0];

      if (file) {
        if (file.name.endsWith(".tif")) {
          this.tifFileSelecteded = true;
          this.urlToPngFile = URL.createObjectURL(file);
         
          const imageElement = new Image();

          imageElement.onload = () => {
            const width = imageElement.width;
            const height = imageElement.height;

            console.log(`Breite: ${width}px, Höhe: ${height}px`);
          };

          imageElement.src = this.urlToPngFile;
        }
      }
    },
    extractPictureCoordinatesonClick(event) {
      if (this.isPictureCoordinatesNext && !this.dotransform) {
        tiffElement = event.target;
        const rect = tiffElement.getBoundingClientRect();
        const x = event.clientX - rect.left;
        //y-Wert muss angepasst sein, da die UTM-Kooridnaten von unten nach oben zählen.
        const y = rect.height - (event.clientY - rect.top);
        //Bildkoordinate einfügen
        this.imageCoordinates.push({ x, y });
        this.isPictureCoordinatesNext = false;
        console.log("Auf Bild wurde geklickt.");
        console.log("Klicke auf Map.");
      } else if (!this.isPictureCoordinatesNext && !this.dotransform) {
        mapCollection.getMap("2D").on("click", (event) => {
          if (!this.isPictureCoordinatesNext) {
            const mapcoordinates = mapCollection
              .getMap("2D")
              .getCoordinateFromPixel(event.pixel);
            const x = mapcoordinates[0];
            const y = mapcoordinates[1];
            this.$forceUpdate(); // Aktualisiert das Template

            // Füge x,y MapListe hinzu
            this.mapCoordinates.push({ x, y });
            console.log("Map Kooridnaten: ", x, y);
            console.log("mapCoordinates:", this.mapCoordinates);
            this.printAllCoordinates();
            this.isPictureCoordinatesNext = true;
            console.log("Auf Map wurde geklickt.");
            if (this.mapCoordinates.length == this.countcoord) {
              this.dotransform = true;
              this.transform();
            }
          } else {
            console.log("Klicke auf das Bild.");
          }
        });
      }
    },

    printAllCoordinates() {
      console.log("Bildkoordinaten:");
      this.imageCoordinates.forEach((coord, index) => {
        console.log(`Punkt ${index}: (x: ${coord.x}, y: ${coord.y})`);
      });

      console.log("Kartenkoordinaten:");
      this.mapCoordinates.forEach((coord, index) => {
        console.log(`Punkt ${index}: (x: ${coord.x}, y: ${coord.y})`);
      });
    },
    transform() {

      // Alle Kooridnaten ausgeben.
      this.printAllCoordinates();
      if (
        this.imageCoordinates.length !== this.countcoord ||
        this.mapCoordinates.length !== this.countcoord
      ) {
        console.log("Es sind nicht genügend Koordinaten vorhanden.");
        return;
      }
      // Berechnen der Durchschnittspunkte
      const meanPointImage = this.calculateMeanPoint(this.imageCoordinates);
      const meanPointMap = this.calculateMeanPoint(this.mapCoordinates);

//Transformationsparameter berechnen:
// Erstellen der Matrix A und des Vektors B
let A = [];
let B = [];

//Test Koordinaten:
// this.imageCoordinates = [
//   {x: 209, y: 383},
//   {x: 410, y: 432},
//   {x: 460, y: 281}
// ];
// this.mapCoordinates = [
//   {x: 633217, y: 5807018},
//   {x: 633823, y: 5807166},
//   {x: 633979, y: 5806721}
// ];

this.imageCoordinates.forEach((coord, index) => {
  const mapCoord = this.mapCoordinates[index];
  A.push([1, 0, 0, 0, coord.x, coord.y]);
  A.push([0, 1, coord.y, -coord.x, 0, 0]);
  B.push(mapCoord.x);
  B.push(mapCoord.y);
});
// Lösung des Gleichungssystems
const params = math.lusolve(A, B);
// Extrahieren der Lösung
const solution = params.map(param => param[0]);
    console.log("Transformationsparameter:", solution);


    // Extrahierte Transformationsparameter
const X_0 = solution[0];
const Y_0 = solution[1];
const a_1 = solution[2];
const a_2 = solution[3];
const a_3 = solution[4];
const a_4 = solution[5];

console.log("Transformationsparameter:");
console.log("X_0:", X_0);
console.log("Y_0:", Y_0);
console.log("a_1:", a_1);
console.log("a_2:", a_2);
console.log("a_3:", a_3);
console.log("a_4:", a_4);
// Berechnung der Skalierungsmaßstäbe
const m_x = Math.sqrt(a_2**2 + a_3**2);
const m_y = Math.sqrt(a_1**2 + a_4**2);

// Berechnung der Scherwinkel in Radiant
const alpha = Math.atan(a_4 / a_1);
const beta = Math.atan(a_2 / a_3);

// Umrechnung der Winkel von Radiant in Grad
const alpha_deg = alpha * (180 / Math.PI);
const beta_deg = beta * (180 / Math.PI);

// Ausgabe der Skalierungsmaßstäbe und Scherwinkel
console.log(`Skalierungsmaßstab in x-Richtung: ${m_x}`);
console.log(`Skalierungsmaßstab in y-Richtung: ${m_y}`);
console.log(`Erster Scherwinkel (α) in Grad: ${alpha_deg}`);
console.log(`Zweiter Scherwinkel (β) in Grad: ${beta_deg}`);


    //Eigene Berechungen nach Skalierung:
      // Berechnen der Skalierungsfaktoren für X- und Y-Achsen
      const scaleX = this.calculateScaleFactor(
        this.imageCoordinates,
        meanPointImage,
        this.mapCoordinates,
        meanPointMap,
        "x"
      );
      const scaleY = this.calculateScaleFactor(
        this.imageCoordinates,
        meanPointImage,
        this.mapCoordinates,
        meanPointMap,
        "y"
      );

      // Berechnen der Drehung
      const rotation = this.calculateRotation(
        this.imageCoordinates,
        meanPointImage,
        this.mapCoordinates,
        meanPointMap
      );

      // Berechnen der Verschiebung
      const translateX =
        meanPointMap.x -
        scaleX * meanPointImage.x * Math.cos(rotation) +
        scaleY * meanPointImage.y * Math.sin(rotation);
      const translateY =
        meanPointMap.y -
        scaleX * meanPointImage.x * Math.sin(rotation) -
        scaleY * meanPointImage.y * Math.cos(rotation);
      console.log("Skalierung X:", scaleX);
      console.log("Skalierung Y:", scaleY);
      console.log("Drehung:", rotation);
      console.log("Verschiebung X:", translateX);
      console.log("Verschiebung Y:", translateY);


      //ungefähre Durchführung zur Erstellung des Bildes
      // Anwenden der Transformation auf das Bild
      const rect = tiffElement.getBoundingClientRect();

      // Berechne die neue Größe des transformierten Bildes basierend auf Skalierung und Rotation
      const newWidth = Math.abs(scaleX) * rect.width;
      const newHeight = Math.abs(scaleY) * rect.height;
      console.log(newWidth, " und ",newHeight);

      // Erstellen eines Canvas zum Zeichnen des transformierten Bildes
      const canvas = document.createElement("canvas");
      canvas.width = newWidth;
      canvas.height = newHeight;
      const context = canvas.getContext("2d");

      // Transformationsmatrix erstellen (Skalierung und Rotation)
      // Hier müssten alpha, beta, m_x und m_y verwendet werden.
      context.setTransform(scaleX, 0, 0, scaleY, newWidth / 2, newHeight / 2);
      context.rotate(-rotation);
      // Verschieben des Zeichenzentrums an die ursprüngliche Position
      context.translate(-rect.width / 2, -rect.height / 2);

      // Zeichnen des Bildes auf das Canvas
      context.drawImage(tiffElement, 0, 0, rect.width, rect.height);
 
      //Download
      canvas.toBlob(function (blob) {
        const a = document.createElement("a");
        const url = window.URL.createObjectURL(blob);
        a.href = url;
        a.download = "transformed_image.tif";
        a.style.display = "none";
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        window.URL.revokeObjectURL(url);
      }, "image/tif");

      // // GeoTIFF erstellen
      this.createGeoTIFF();
      this.dotransform = true;
    },

    createGeoTIFF() {
    },
    // Berechnen des Durchschnittspunktes der Koordinaten
    calculateMeanPoint(points) {
      const meanPoint = { x: 0, y: 0 };
      for (const point of points) {
        meanPoint.x += point.x;
        meanPoint.y += point.y;
      }
      meanPoint.x /= points.length;
      meanPoint.y /= points.length;
      return meanPoint;
    },
    //Berechnung Skalierung
    calculateScaleFactor(points1, meanPoint1, points2, meanPoint2, axis) {
      // Berechnen des Skalierungsfaktors auf einer Achse
      let numerator = 0;
      let denominator = 0;
      for (let i = 0; i < points1.length; i++) {
        const diff1 = points1[i][axis] - meanPoint1[axis];
        const diff2 = points2[i][axis] - meanPoint2[axis];
        numerator += diff1 * diff2;
        denominator += diff1 * diff1;
      }
      return numerator / denominator;
    },
    //Berechnung Rotation
    calculateRotation(points1, meanPoint1, points2, meanPoint2) {
      // Berechnen der Drehung
      let numerator = 0;
      let denominator = 0;
      for (let i = 0; i < points1.length; i++) {
        const diff1x = points1[i].x - meanPoint1.x;
        const diff1y = points1[i].y - meanPoint1.y;
        const diff2x = points2[i].x - meanPoint2.x;
        const diff2y = points2[i].y - meanPoint2.y;
        numerator += diff1x * diff2y - diff1y * diff2x;
        denominator += diff1x * diff2x + diff1y * diff2y;
      }
      return Math.atan2(numerator, denominator);
    },

    //Löschen von Koordinaten
    deleteMapCoordinate(index) {
      this.mapCoordinates.splice(index, 1);
      this.imageCoordinates.splice(index, 1);
      this.dotransform = false;
    },

    // Extrahiere das verwendete Referenzsystem aus den GeoTIFF-Metadaten
    getcoordinateSystemTiff(image) {
      const geoKeyDirectory = image.getGeoKeys();
      if (geoKeyDirectory && geoKeyDirectory.ProjectedCSTypeGeoKey != null) {
        const geogrGTCitationGeoKey = geoKeyDirectory.GTCitationGeoKey;
        const projectedCSTypeGeoKey = geoKeyDirectory.ProjectedCSTypeGeoKey;

        console.log("geographische Koordinatensystem: ", geogrGTCitationGeoKey);
        console.log("EPSG Code: ", projectedCSTypeGeoKey);

        const mapcs = this.getView;
        console.log(mapcs);
        console.log(mapcs.getProjection().getCode());
        console.log(geoKeyDirectory.projectedCSTypeGeoKey);

        return projectedCSTypeGeoKey;
      } else {
        return "Kein Koordinatensystem gefunden";
      }
    },

    //hinzufügen der Layer
    initializeMap() {
      //OpenLayers-Karte
      const tilelayer = new TileLayer({
        source: this.tiffSource,
      });
      tilelayer.setZIndex(1000);
      store.commit("Maps/addLayerToMap", tilelayer);
      Radio.trigger("ModelList", "refreshLightTree");
      this.tileLayers.push(tilelayer);
    },

  //Entfernen der Layer
    removeLayers() {
      this.tileLayers.forEach(layer => {
      store.commit("Maps/removeLayerFromMap", layer);
    });

    // Leeren des Arrays nach dem Entfernen der Layer
    this.tileLayers = [];

    Radio.trigger("ModelList", "refreshLightTree");
    //Referenz muss geschaffen werden, da sonst der handleselectedfile gleich bleibt und somit der das event nicht aktiviert wird
    if (this.$refs.fileInput) {
    this.$refs.fileInput.value = '';
  }
    },
  },
};
</script>

<template lang="html">
  <ToolTemplate
    :title="$t(name)"
    :icon="glyphicon"
    :active="active"
    :render-to-window="renderToWindow"
    :resizable-window="resizableWindow"
    :deactivateGFI="deactivateGFI"
  >
    <template v-slot:toolBody>
      <div v-if="active" id="ExampleAddon">
        <p><strong>GeoTIFF-Datei auswählen:</strong></p>
        <input type="file" @change="handleFileSelect" ref="fileInput" />
        <button v-if="tileLayers.length > 0" @click="removeLayers">Layer entfernen</button>
        <div v-if="tileLayers.length > 0"><p>Breite der GeoTIFF-Datei: {{ width }}</p>
          <p>Höhe der GeoTIFF-Datei: {{ height }}</p></div>
                <p>
          <p><strong>
            Zu Georeferenzierende Tiff-Datei auswählen:</strong></p>
          <input
            v-if="!tifFileSelecteded"
            type="file"
            @change="tifFileSelected"
          />
        </p>
        <div v-if="geoTiffLoaded && !tifFileSelecteded">
          <p>GeoTIFF-Datei wurde ausgewählt.</p>
          <p>Breite der GeoTIFF-Datei: {{ width }}</p>
          <p>Höhe der GeoTIFF-Datei: {{ height }}</p>
        </div>

        <div v-else-if="tifFileSelecteded">
          <p>TIFF wurde ausgewählt.</p>
          <div id="myImage" class="my-image-container">
            <img
              :src="urlToPngFile"
              @click="extractPictureCoordinatesonClick"
              class="my-image"
            />
          </div>
        </div>

        <div v-else>
          <p v-if="fileSelected">
            Bitte warten, während die Datei verarbeitet wird...
          </p>
          <p v-else>Bitte wähle eine GeoTIFF-Datei oder eine PNG-Datei aus.</p>
        </div>
        <div v-if="imageCoordinates.length > 0">
          <p>Bildkoordinaten:</p>
          <ul>
            <li v-for="(coord, index) in imageCoordinates" :key="index">
              Punkt {{ index + 1 }}: (x: {{ coord.x }}, y: {{ coord.y }})
            </li>
          </ul>
        </div>

        <div v-if="mapCoordinates.length > 0">
          <p>Kartenkoordinaten:</p>
          <ul>
            <li v-for="(coord, index) in mapCoordinates" :key="index">
              Punkt {{ index + 1 }}: (x: {{ coord.x }}, y: {{ coord.y }})
              <button @click="deleteMapCoordinate(index)">Löschen</button>
            </li>
          </ul>
        </div>
      </div>
    </template>
  </ToolTemplate>
</template>

<style>
.my-image-container {
  border: 2px solid black;
  display: inline-block;

  overflow: auto;
  max-width: 100%;
  max-height: 80vh;
}

.my-image {
  border: none;
}
</style>
