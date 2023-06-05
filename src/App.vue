<script setup>
import gpx from "./gpx-parser-builder";
</script>

<template>
  <main>
    <h1>
      OpenCPN GPX to GP32 gpx converter
      <a href="https://github.com/PepinNucleaire/gpx-gp32" aria-label="Repo source" class="footer-octicon" title="GitHub">
        <svg aria-hidden="true" class="octicon octicon-mark-github" height="24" version="1.1" viewBox="0 0 16 16"
          width="24">
          <path fill="white" fill-rule="evenodd"
            d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0 0 16 8c0-4.42-3.58-8-8-8z">
          </path>
        </svg>
      </a>
    </h1>

    <p>Upload your GPX file from OpenCPN</p>
    <p>
      It will automatically transform and export to gp32 using serial connection
    </p>
    <input type="file" id="gpxFile" @change="readFile" />
    <!-- <button @click="submitFile">Upload!</button> -->
    <hr v-if="errors.display" />
    <p class="notice" v-if="errors.display">ERROR<br />{{ errors.msg }}</p>

    <hr v-if="hasSerial && serialDisplay && !errors.display" />
    <p class="notice" v-if="!errors.display">ERROR<br />"Your GPX looks good, good job !"</p>
    <div v-if="hasSerial && serialDisplay && !errors.display">
      <button v-if="connected" style="background-color: green"></button>
      <button v-if="!connected" style="background-color: red"></button>
      <button @click="selectSerial">Connect to the GPS</button>
      <button v-if="connected" @click="disconnect">Reset</button><br />
      <button v-if="connected" @click="writeNmea">Write to GPS</button>
      <hr />
      <p>
        Avancement {{ lineNumber }} / {{ nmeaToWrite.length - 1 }} waypoints
      </p>
    </div>
    <div v-if="!hasSerial">
      You must have Google Chrome or Edge to use this website as Web Serial API
      is only supported on these browsers
    </div>
    <hr />
    <h2>Notice</h2>
    <ul>
      <li>Select your gpx file exported from OpenCPN</li>
      <li>Connect the GPS via the Serial USB</li>
      <li>On the GPS, Go to I/O menu, Select "Load data GPS PC"</li>
      <li>Click on "Write to GPS" and wait for the finish bip</li>
    </ul>
    <h2>Debug</h2>
    <input type="checkbox" v-model="debug.checkNMEA" />
    <label for="checkbox">NMEA in console</label>
    <br />
    <input type="checkbox" v-model="debug.checkErrors" />
    <label for="checkbox">Errors in console</label>
    <br />

  </main>
</template>

<script>
export default {
  data() {
    return {
      MSG: {
        SAME_NAME: "More than 2 waypoints with same name",
        TOO_MANY: "Too many waypoints (Max 1000)",
        NAME_LONG: "1 or more waypoints has more than 6 characters"
      },
      debug: {
        checkNMEA: false,
        checkErrors: true,
      },
      errors: {
        display: false,
        msg: "",
        tooManyWpts: false,
        wptsWithSameName: [],
        wptsWithNameTooLong: []
      },
      serialDisplay: false,
      filegpx: null,
      gpxParsed: null,
      hasSerial: false,
      connected: false,
      port: null,
      reader: null,
      baudRate: 4800,
      connectFailed: false,
      inputField: null,
      inputStream: null,
      inputDone: null,
      outputStream: null,
      outputDone: null,
      running: true,
      nmeaToWrite: [],
      lineNumber: 0,
    };
  },
  created() {
    if ("serial" in navigator) {
      this.hasSerial = true;
    }
  },
  methods: {
    async disconnect() {
      this.running = false;
      // I have been unable to get a cancel and close port to work.
      // Instead I am using a brute force solution and forcing a browser
      // refresh.
      // console.log("wait for readloop close");
      // await this.closed;
      location.reload();
    },
    async selectSerial() {
      this.connected = false;
      this.connected = false;
      console.log("in connect");
      console.log("baud rate: ", this.baudRate);
      this.port = await navigator.serial.requestPort();
      // - Wait for the port to open.
      try {
        await this.port.open({ baudRate: this.baudRate });
      } catch (error) {
        console.log("*** open port error ***");
        console.log(error);
        this.connectFailed = true;
      }
      // Check for success and continue
      if (!this.connectFailed) {
        this.connected = true;
        // writeFailed is set if we try to write before connecting
        console.log("Open");

        let decoder = new TextDecoderStream();
        this.inputDone = this.port.readable.pipeTo(decoder.writable);
        this.inputStream = decoder.readable;
        // eslint-disable-next-line no-undef
        const encoder = new TextEncoderStream();
        this.outputDone = encoder.readable.pipeTo(this.port.writable);
        this.outputStream = encoder.writable;
        // this.reader = this.inputStream.getReader();
        // this.closed = this.readLoop();
      }
    },
    writeNmea() {
      this.lineNumber = 0;
      this.nmeaToWrite.forEach((el, i) => {
        setTimeout(() => {
          this.lineNumber = i;
          if (this.debug.checkNMEA) {
            console.log(el);
          }
          this.writeToSerial(el);
        }, i * 300);
      });
    },
    writeToSerial(text) {
      let writer;
      try {
        writer = this.outputStream.getWriter();
      } catch (error) {
        this.writeFailed = true;
      }
      let textToSend = text;
      const result = text + "\r\n";
      writer.write(result);
      // console.log("Text written : ", text);
      writer.releaseLock();
    },
    async readLoop() {
      console.log("Readloop");

      while (this.running) {
        const { value, done } = await this.reader.read();
        if (value) {
          console.log("raw: ", value);
        }
      }
    },
    readFile(e) {
      this.filegpx = e.target.files[0];
      this.errors.display = false
      this.errors.msg = ""
      if (this.filegpx) {
        if (this.filegpx.name.includes(".gpx")) {
          this.serialDisplay = true;
        }
        let reader = new FileReader();

        reader.readAsText(this.filegpx);
        reader.onload = (e) => {
          this.gpxParsed = this.parseGPX(e.target.result);
          this.fromGPXtoText();
        };
      }
    },
    parseGPX(e) {
      let text = e;
      const gpxParsed = gpx.parse(text);
      let newwpts = [];
      this.checkErrors(gpxParsed.wpt)
      gpxParsed.wpt.map((wpt) => {
        const lon = wpt["$"].lon;
        const lat = wpt["$"].lat;
        const newwpt = {
          lon: lon,
          lat: lat,
          furunoLon:
            Math.round(
              1000 *
              (Math.floor(Math.abs(lon)) * 100 +
                (Math.abs(lon) - Math.floor(Math.abs(lon))) * 60)
            ) / 1000,
          furunoLat:
            Math.round(
              1000 *
              (Math.floor(Math.abs(lat)) * 100 +
                (Math.abs(lat) - Math.floor(Math.abs(lat))) * 60)
            ) / 1000,
          ns: wpt["$"].lon > 0 ? "S" : "N",
          ew: wpt["$"].lat > 0 ? "W" : "E",
          name: wpt.name.toUpperCase().slice(0, 6),
        };
        newwpts.push(newwpt);
      });
      if (this.debug.checkNMEA) {
        console.log("**** GPX PARSED ****");
        console.log(newwpts);
      }
      return newwpts;
    },
    checkErrors(listParsedWaypoints) {
      const moreThan6Letters = this.checkNamesWithMoreThan6Letters(listParsedWaypoints);
      const sameName = this.checkWptWithSameName(listParsedWaypoints)
      const tooManyWaypoints = this.checkNumbersOfWaypoints(listParsedWaypoints)

      if (!moreThan6Letters && !sameName && !tooManyWaypoints) {
        if (this.checkErrors) { console.info("**** NO ERROR DETECTED ****") }

        this.errors.display = false
        return
      }
      if (this.checkErrors) {
        console.info("**** ERRORS DETECTED ****")

        console.log(moreThan6Letters)
        console.log(sameName)
        console.log(tooManyWaypoints)
      }
      this.errors.msg = `${moreThan6Letters.error} : ${moreThan6Letters.wpts.toString()}\n`
      this.errors.msg = `${sameName.error} : ${sameName.wpts.toString()}\n`
      this.errors.msg = `${tooManyWaypoints.error}`
    },
    checkNumbersOfWaypoints(listParsedWaypoints) {
      if (listParsedWaypoints.length > 1000) {

        this.errors.display = true
        this.errors.tooManyWpts = true

        return { error: this.MSG.TOO_MANY }
      }
    },
    checkWptWithSameName(listParsedWaypoints) {
      var arr = listParsedWaypoints.map(x => x.name)

      const map = arr.reduce((acc, e) => acc.set(e, (acc.get(e) || 0) + 1), new Map());
      const errors = [...map.entries()].filter(x => { return x[1] > 1 })
      if (errors.length > 0) {
        this.errors.display = true
        return {
          error: this.MSG.SAME_NAME, wpts: errors.map(x => x[0])
        }
      }
    },
    checkNamesWithMoreThan6Letters(listParsedWaypoints) {
      const listWithErrors = listParsedWaypoints.filter(wpt => wpt.name.length > 6)
      if (listWithErrors.length == 0) { return }
      this.errors.display = true
      return {
        error: this.MSG.NAME_LONG,
        wpts: listWithErrors.map(x => x.name)
      }
    },
    fromGPXtoText() {
      const listNmea = [];
      this.gpxParsed.map((wpt) =>
        listNmea.push(
          `$PFEC, GPwpl, ${wpt.furunoLat},${wpt.ns},${wpt.furunoLon},${wpt.ew},${wpt.name}, 0,, V,,,, `
        )
      );
      listNmea.push("$PFEC,GPxfr,CTL,E");
      if (this.debug.checkNMEA) {
        console.info("****** NMEA TO WRITE ******");
        console.info(listNmea);
      }
      this.nmeaToWrite = listNmea;
    },
  },
};
</script>
<style>
@import url("simpledotcss/simple.min.css");

#app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;

  font-weight: normal;
}
</style>
