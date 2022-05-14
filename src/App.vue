<template>
  <v-app>
    <v-main>
      <v-app-bar>
        <img src="img/icons/favicon-32x32.png" />

        <v-spacer></v-spacer>

        <v-btn icon @click="this.searchButton = !this.searchButton">
          <v-icon>mdi-magnify</v-icon>
        </v-btn>

        <v-btn icon @click="showAddDialog()">
          <v-icon>mdi-plus</v-icon>
        </v-btn>
        <v-btn icon @click="showQRDialog()">
          <v-icon>mdi-barcode</v-icon>
        </v-btn>
      </v-app-bar>
      <v-dialog v-model="QRDialog.show">
        <div id="qr"></div>
      </v-dialog>

      <v-dialog v-model="addDialog.show" style="width: 100%; max-width: 400px">
        <v-card>
          <v-card-text>
            <v-form ref="addFormRef" v-model="addDialog.valid">
              <v-text-field
                class="text-h5 grey lighten-2"
                label="Name"
                type="text"
                :rules="addDialog.nameRules"
                v-model="addDialog.item.name"
              />
              <v-text-field
                class="text-h5 grey lighten-2"
                label="Barcode (optional)"
                type="text"
                v-model="addDialog.item.barcode"
              />

              <v-text-field
                class="text-h5 grey lighten-1"
                label="Quantity"
                type="number"
                :rules="addDialog.quantityRules"
                v-model="addDialog.item.quantity"
              />

              <v-text-field
                class="text-h5 grey lighten-1"
                label="Unità"
                type="number"
                :rules="addDialog.scadenzaRules"
                v-model="addDialog.item.scadenzaQnt"
              />

              <v-select
                :items="addDialog.scadenzaUnits"
                v-model="addDialog.item.scadenzaUnit"
                label="Periodo"
              ></v-select>
            </v-form>

            <v-alert type="info" icon="mdi-clock-alert">
              Scade il:
              {{ scadenza.format("DD/MM/YYYY") }}</v-alert
            >
          </v-card-text>

          <v-divider></v-divider>

          <v-card-actions>
            <v-spacer></v-spacer>

            <v-btn
              color="success"
              @click="saveElement"
              v-if="addDialog.item.edit"
            >
              <v-icon left> mdi-content-save-edit </v-icon>
              Save
            </v-btn>

            <v-btn color="success" @click="saveElement" v-else>
              <v-icon left> mdi-plus-circle </v-icon>
              Add
            </v-btn>

            <v-btn color="error" @click="closeAddDialog">
              <v-icon left> mdi-close-circle </v-icon>
              Close
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

      <div id="container">
        <v-card>
          <v-card-title>
            <v-text-field
              v-model="searchString"
              class="my-3"
              append-icon="mdi-magnify"
              label="Search"
              single-line
              hide-details
              v-if="this.searchButton"
            >
            </v-text-field>
          </v-card-title>
        </v-card>

        <v-list>
          <v-list-subheader>FRIGO</v-list-subheader>
          <v-list-item
            v-for="(item, i) in filteredElements"
            :key="i"
            :value="item"
            active-color="primary"
          >
            <v-list-item-avatar start>
              <v-icon> mdi-food </v-icon>
            </v-list-item-avatar>
            <v-list-item-content>
              <v-list-item-title v-text="item.name"></v-list-item-title>
              <v-list-item-subtitle>
                Rimanenti: {{ item.quantity }} unità
              </v-list-item-subtitle>
              <v-chip label :color="getColor(item)" class="ms-auto mt-1">
                <span class="mdi mdi-shield-alert me-1"></span>

                {{ remaningDays(item) }}
              </v-chip>
            </v-list-item-content>
            <div class="ms-auto">
              <v-btn
                color="error"
                dark
                icon
                @click="deleteElement(item)"
                class="mx-3"
              >
                <span class="mdi mdi-delete"></span>
              </v-btn>
              <v-btn
                fab
                dark
                large
                color="cyan"
                icon
                @click="editElement(item)"
              >
                <span class="mdi mdi-pencil-outline"></span>
              </v-btn>
            </div>
          </v-list-item>
        </v-list>
      </div>
    </v-main>
  </v-app>
</template>

<script>
// import HelloWorld from "./components/HelloWorld.vue";
import { Html5Qrcode } from "html5-qrcode";
import moment from "moment";

export default {
  name: "App",

  components: {
    // HelloWorld,
  },

  data: () => ({
    html5QrCode: null,
    searchButton: false,
    searchString: "",
    QRDialog: {
      show: false,
    },
    addDialog: {
      show: false,
      valid: true,
      nameRules: [
        (value) => !!value || "Inserisci nome.",
        (value) => (value && value.length >= 3) || "Almeno 3 caratteri",
      ],
      quantityRules: [
        (value) => !!value || "Inserisci quantità.",
        (value) => (value && value >= 0) || "Inserisci valore valido",
      ],
      scadenzaRules: [
        (value) => !!value || "Inserisci quantità.",
        (value) =>
          (value && value > 0 && value <= 12) || "Inserisci valore valido",
      ],
      item: {
        name: "",
        barcode: "",
        quantity: 0,
        scadenzaQnt: "",
        scadenzaUnit: "",
        edit: false,
        scadenzaTimestamp: null,
        insermentoTimestamp: null,
      },
      scadenzaUnits: ["Giorno", "Settimana", "Mese", "Anno"],
    },
    elements: [],
    db_prodotti: [],
  }),

  mounted() {
    let elements = JSON.parse(localStorage.getItem("elements"));
    let db_prodotti = JSON.parse(localStorage.getItem("db_prodotti"));

    if (elements !== null) this.elements = elements;
    if (db_prodotti !== null) this.db_prodotti = db_prodotti;
  },
  computed: {
    scadenza() {
      let startDate = moment();
      if (this.addDialog.item.insermentoTimestamp) {
        startDate = moment.unix(this.addDialog.item.insermentoTimestamp);
      }

      let unit = this.addDialog.item.scadenzaUnit;
      switch (unit) {
        case "Anno":
          unit = "year";
          break;
        case "Mese":
          unit = "month";
          break;
        case "Settimana":
          unit = "week";
          break;
        case "Giorno":
          unit = "day";
          break;

        default:
          break;
      }
      return startDate.add(this.addDialog.item.scadenzaQnt, unit);
    },
    filteredElements() {
      let result = this.elements.filter((element) => {
        return element.name.includes(this.searchString);
      });

      return result.sort((a, b) => {
        return a.scadenzaTimestamp - b.scadenzaTimestamp;
      });
    },
  },
  methods: {
    moment,
    remaningDays(item) {
      let timestamp = item.scadenzaTimestamp;
      console.log({ timestamp });
      let scadenza = moment.unix(timestamp);
      let out = scadenza.fromNow();
      return out;
    },
    closeAddDialog() {
      this.addDialog.show = false;
    },
    showAddDialog() {
      this.addDialog.show = true;
      this.addDialog.item = {
        name: "",
        barcode: "",
        quantity: "",
        scadenzaGiorno: "",
        scadenzaTemporale: "",
        edit: false,
      };
    },
    showQRDialog() {
      this.QRDialog.show = true;

      setTimeout(() => {
        this.html5QrCode = new Html5Qrcode("qr");

        const config = { fps: 5, qrbox: { width: 250, height: 250 } };

        // If you want to prefer back camera
        this.html5QrCode.start(
          { facingMode: "environment" },
          config,
          this.onScanSuccess
        );
      }, 1000);
    },

    onScanSuccess(decodedText) {
      // handle the scanned code as you like, for example:
      this.html5QrCode.stop();
      console.log(decodedText);
      let index = this.elements.findIndex((element) => {
        return element.barcode === decodedText;
      });
      console.log({ index });
      if (index >= 0) {
        //prodotto esiste
        this.editElement(this.elements[index]);
        this.QRDialog.show = false;
      } else {
        //prodotto da aggiungere
        this.showAddDialog();
        this.QRDialog.show = false;
        this.addDialog.item.barcode = decodedText;
        let prodotto = this.db_prodotti.find((e) => {
          return e.barcode === decodedText;
        });
        console.log("hey");
        console.log("prodotto:", prodotto);
        if (prodotto) {
          this.addDialog.item.name = prodotto.name;
        }
        // check database locale
      }
    },

    saveElement() {
      if (
        this.addDialog.item.name === "" ||
        this.addDialog.item.quantity < 1 ||
        this.addDialog.item.scadenzaQnt < 1 ||
        this.addDialog.item.scadenzaUnit === ""
      ) {
        return;
      }

      this.addDialog.item.scadenzaTimestamp = this.scadenza.unix();

      if (this.addDialog.item.edit !== true) {
        this.addDialog.item.insermentoTimestamp = moment().unix();
        if (this.addDialog.item.barcode > 0) {
          //check se esiste già
          this.db_prodotti.push({
            barcode: this.addDialog.item.barcode,
            name: this.addDialog.item.name,
          });
        }
        this.elements.push(this.addDialog.item);
      }
      this.closeAddDialog();
      this.save();
    },

    deleteElement(item) {
      //index of item in elements
      this.elements = this.elements.filter((element) => {
        return item.name !== element.name;
      });
      this.save();
    },

    editElement(item) {
      this.showAddDialog();
      this.addDialog.item = item;
      this.addDialog.item.edit = true;
      this.save();
    },

    getColor(item) {
      let timestamp = item.scadenzaTimestamp;
      let scadenza = moment.unix(timestamp);
      let durata = scadenza.diff(moment(), "days");

      if (durata < 5) return "red";
      else if (durata < 10) return "orange";
      else return "green";
    },

    save() {
      localStorage.setItem("elements", JSON.stringify(this.elements));
      localStorage.setItem("db_prodotti", JSON.stringify(this.db_prodotti));
    },
  },
};
</script>

<style lang="scss">
.v-overlay__content {
  width: 100%;
  max-width: 360px;
}

.v-card-title {
  align-items: center;
  display: flex;
  flex: none;
  font-size: 1.25rem;
  font-weight: 250;
  hyphens: auto;
  letter-spacing: 0.0125em;
  overflow-wrap: normal;
  padding: 0.5rem 1rem;
  text-transform: none;
  word-break: normal;
  word-wrap: break-word;
}
</style>
