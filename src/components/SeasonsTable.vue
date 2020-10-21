<template>
  <v-container fluid>
    <v-data-table
      item-key="name"
      class="elevation-1"
      :loading="isLoading"
      loading-text="Loading... Please wait"
      :headers="headers"
      :items="seasons"
      :sort-by="['id']"
      sort-desc
      ref="SeasonsTable"
    >
      <template v-slot:top>
        <v-toolbar flat>
          Seasons/Tournaments
          <v-spacer />
          <v-btn color="primary" @click="newDialog = true">
            New Season
          </v-btn>
        </v-toolbar>
      </template>
      <template v-slot:item.id="{ item }">
        <router-link :to="{ path: '/season/' + item.id }">
          {{ item.id }}
        </router-link>
      </template>
      <template v-slot:item.owner="{ item }">
        <router-link :to="{ path: '/user/' + item.user_id }">
          {{ item.owner }}
        </router-link>
      </template>
      <template v-slot:item.end_date="{ item }">
        <div v-if="item.end_date == null">
          Ongoing
        </div>
        <div v-else>
          {{ item.end_date }}
        </div>
      </template>
      <template v-slot:item.actions="{ item }" v-if="user.super_admin == 1">
        <v-icon @click="deleteSelectedSeason(item)">
          mdi-delete
        </v-icon>
      </template>
    </v-data-table>
    <v-bottom-sheet v-model="responseSheet" inset persistent>
      <v-sheet class="text-center" height="200px">
        <v-btn
          class="mt-6"
          text
          color="success"
          @click="
            responseSheet = !responseSheet;
            response = '';
          "
        >
          close
        </v-btn>
        <div class="my-3">
          {{ response }}
        </div>
      </v-sheet>
    </v-bottom-sheet>
    <v-dialog v-model="deleteDialog" persistent max-width="600px">
      <v-card>
        <v-card-title>
          <span class="headline">
            Are you sure you wish to delete this season?
          </span>
        </v-card-title>
        <v-card-text>
          This will remove all match associations to the season.
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" text @click="deleteDialog = false">
            No
          </v-btn>
          <v-btn color="red darken-1" text @click="deleteSeasonConfirm()">
            Yes
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <v-dialog shake v-model="newDialog" persistent max-width="800px">
      <v-card color="lighten-4">
        <v-card-title>
          <span class="headline">
            New Season
          </span>
        </v-card-title>
        <v-card-text>
          <v-form ref="newSeasonForm">
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12">
                    <v-text-field
                      v-model="newSeason.name"
                      label="Name"
                    ></v-text-field>
                  </v-col>
                  <v-col cols="6">
                    <div class="text-h4">
                      Start Date
                    </div>
                    <v-date-picker
                      v-model="newSeason.start_date"
                      label="Start Date"
                      required
                    />
                  </v-col>
                  <v-col cols="6">
                    <div class="text-h4">
                      End Date
                    </div>
                    <v-date-picker
                      v-model="newSeason.end_date"
                      label="End Date"
                    />
                  </v-col>
                  <!-- TODO: Add in combobox with chips. Key value is based on space in between. -->
                </v-row>
              </v-container>
            </v-card-text>
          </v-form>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="darken-1" text @click="newDialog = false">
            Cancel
          </v-btn>
          <v-btn color="primary" text @click="saveNewSeason()">
            Save
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
export default {
  props: ["user"],
  data() {
    return {
      headers: [
        {
          text: "ID",
          align: "start",
          sortable: true,
          value: "id"
        },
        {
          text: "Name",
          value: "name"
        },
        {
          text: "Start Date (YYYY/MM/DD)",
          value: "start_date"
        },
        {
          text: "End Date (YYYY/MM/DD)",
          value: "end_date"
        },
        {
          text: "Owner",
          value: "owner"
        },
        {
          text: "",
          value: "actions",
          sortable: false
        }
      ],
      seasons: [],
      isLoading: true,
      deleteDialog: false,
      response: "",
      responseSheet: false,
      removeIndex: -1,
      removeSeason: {},
      newDialog: false,
      newSeason: {
        name: "",
        start_date: new Date().toISOString().substr(0, 10),
        end_date: null,
        cvars: {}
      }
    };
  },
  created() {
    this.GetSeasons();
  },
  methods: {
    async GetSeasons() {
      try {
        let res;
        if (this.$route.path == "/myseasons") res = await this.GetMySeasons();
        else res = await this.GetAllSeasons();
        res.forEach(async season => {
          const ownerRes = await this.GetUserData(season.user_id);
          season.owner = ownerRes.name;
          season.start_date = new Date(season.start_date).toLocaleDateString(
            "en-CA"
          );
          if (season.end_date != null)
            season.end_date = new Date(season.end_date).toLocaleDateString(
              "en-CA"
            );
          this.seasons.push(season);
        });
      } catch (error) {
        console.log(error);
      } finally {
        this.isLoading = false;
      }
      return;
    },
    deleteSelectedSeason(item) {
      if (item != null) {
        this.removeIndex = this.seasons.indexOf(item);
        this.removeSeason = Object.assign({}, item);
      }
      this.deleteDialog = true;
    },
    async deleteSeasonConfirm() {
      let memberData = [
        {
          season_id: this.removeSeason.id
        }
      ];
      this.response = await this.DeleteSeason(memberData);
      this.seasons.splice(this.removeIndex, 1);
      this.closeDelete();
    },
    closeDelete() {
      this.deleteDialog = false;
      this.responseSheet = true;
      this.$nextTick(() => {
        this.removeSeason = {};
        this.removeIndex = -1;
      });
    },
    async saveNewSeason() {
      if (this.$refs.newSeasonForm.validate()) {
        // let serverRes;
        // let serverObj = [
        //   {

        //   }
        // ];
        return true;
      }
    }
  }
};
</script>