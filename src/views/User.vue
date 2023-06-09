<template>
  <v-card class="mx-auto">
    <v-container>
      <v-card-title v-if="retrievedUser.steam_id != 0">
        {{ retrievedUser.name }}
        <a
          :href="
            `https://steamcommunity.com/profiles/${retrievedUser.steam_id}`
          "
          target="_blank"
        >
          <v-icon>mdi-steam</v-icon>
        </a>
        <v-spacer />
        <img style="border-radius: 15px;" :src="retrievedUser.medium_image" />
      </v-card-title>
      <v-card-title v-else>
        {{ $t("User.ClaimUser") }}
        <v-spacer />
      </v-card-title>
      <v-card-subtitle>
        <v-chip
          class="ma-2"
          color="primary"
          v-if="retrievedUser.super_admin === 1"
          small
        >
          {{ $t("User.SuperAdmin") }}
        </v-chip>
        <v-chip
          class="ma-2"
          color="orange"
          v-if="retrievedUser.admin === 1"
          small
        >
          {{ $t("User.Admin") }}
        </v-chip>
      </v-card-subtitle>
    </v-container>
    <v-container v-if="retrievedUser.api_key != undefined">
      <v-card-title class="headline">
        {{ $t("User.API") }}
      </v-card-title>
      <v-row>
        <v-col cols="12" md="11" sm="11" lg="6">
          <v-text-field
            v-model="retrievedUser.api_key"
            :append-icon="showAPI ? 'mdi-eye' : 'mdi-eye-off'"
            :type="showAPI ? 'text' : 'password'"
            name="apiKeyInput"
            readonly
            @click:append="showAPI = !showAPI"
          ></v-text-field>
        </v-col>
        <v-col cols="12" md="11" sm="11" lg="6">
          <v-btn
            class="ma-2"
            :loading="apiResetLoading"
            :disabled="apiResetLoading"
            name="apiReset"
            @click="resetApiKey"
            color="secondary"
          >
            {{ $t("User.ResetAPI") }}
          </v-btn>
        </v-col>
      </v-row>
      <v-row>
        <v-col cols="12" md="12" sm="12" lg="12">
          <v-card-title class="headline">
            {{ $t("User.Challonge") }}
          </v-card-title>
        </v-col>
        <v-col cols="12" md="11" sm="11" lg="6">
          <v-text-field
            v-model="retrievedUser.challonge_api_key"
            :append-icon="showChallongeAPI ? 'mdi-eye' : 'mdi-eye-off'"
            :type="showChallongeAPI ? 'text' : 'password'"
            name="apiKeyInput"
            @click:append="showChallongeAPI = !showChallongeAPI"
          ></v-text-field>
        </v-col>
        <v-col cols="12" md="11" sm="11" lg="6">
          <v-btn
            class="ma-2"
            :loading="challongeApiResetLoading"
            :disabled="challongeApiResetLoading"
            name="challongeApiSave"
            @click="saveChallongeApiKey"
            color="secondary"
          >
            {{ $t("User.ChallongeSave") }}
          </v-btn>
        </v-col>
      </v-row>
      <v-row>
        <v-col cols="12" md="11" sm="11" lg="6">
          <v-btn
            class="ma-2"
            :loading="apiResetLoading"
            :disabled="apiResetLoading"
            name="apiReset"
            @click="passwordResetDialog = true"
            color="secondary"
          >
            {{ $t("User.ResetPass") }}
          </v-btn>
        </v-col>
      </v-row>
    </v-container>
    <v-container>
      <v-card-title class="headline">
        {{ $t("User.Stats") }}
      </v-card-title>
      <PlayerStats v-if="userStats != null" :statArray="userStats" />
    </v-container>
    <v-container>
      <v-card-title class="headline">
        {{ $t("User.Past5") }}
      </v-card-title>
      <MatchesTable v-if="retrievedUser.steam_id != ''" :user="retrievedUser" />
    </v-container>
    <v-container v-if="retrievedUser.id == user.id || IsAnyAdmin(user)">
      <v-card-title class="headline">
        {{ $t("User.UserMaps", { players: retrievedUser.name }) }}
      </v-card-title>
      <MapList v-if="retrievedUser.id > 0" :user="retrievedUser" />
    </v-container>
    <PasswordResetDialog
      v-model="passwordResetDialog"
      :title="$t('User.ResetPass')"
    />
  </v-card>
</template>
<script>
import MatchesTable from "@/components/MatchesTableNoLimits";
import PlayerStats from "@/components/PlayerStatInfo";
import MapList from "@/components/NewMap";
import PasswordResetDialog from "@/components/PasswordResetDialog";

export default {
  name: "User",
  components: {
    MatchesTable,
    PlayerStats,
    MapList,
    PasswordResetDialog
  },
  data() {
    return {
      user: {
        admin: false,
        steam_id: "",
        id: -1,
        super_admin: false,
        name: "",
        small_image: "",
        medium_image: "",
        large_image: ""
      },
      retrievedUser: {
        admin: false,
        steam_id: "",
        id: -1,
        super_admin: false,
        name: "",
        small_image: "",
        medium_image: "",
        large_image: ""
      }, // should be object from JSON response
      userStats: [],
      showAPI: false,
      showChallongeAPI: false,
      apiResetLoading: false,
      showPass: false,
      passwordResetDialog: false,
      challongeApiResetLoading: false
    };
  },
  async created() {
    this.user = await this.IsLoggedIn();
    if (this.$route.params.id == undefined) this.retrievedUser = this.user;
    else this.retrievedUser = await this.GetUserData(this.$route.params.id);

    if (this.retrievedUser.id === 0) {
      this.userStats = await this.GetUserPlayerStats(this.$route.params.id);
      this.retrievedUser.name = this.userStats[0].name;
      this.retrievedUser.steam_id = this.userStats[0].steam_id;
    } else
      this.userStats = await this.GetUserPlayerStats(
        this.retrievedUser.steam_id
      );
    if (typeof this.userStats == "string") this.userStats = [];
  },
  methods: {
    async resetApiKey() {
      this.apiResetLoading = true;
      let renewKey = [
        {
          steam_id: this.retrievedUser.steam_id,
          new_api: 1
        }
      ];
      await this.UpdateUserInfo(renewKey);
      this.retrievedUser = await this.GetUserData(this.retrievedUser.id);
      this.apiResetLoading = false;
    },
    async saveChallongeApiKey() {
      this.challongeApiResetLoading = true;
      let updateData = [
        {
          steam_id: this.retrievedUser.steam_id,
          id: this.retrievedUser.id,
          challonge_api_key: this.retrievedUser.challonge_api_key
        }
      ];
      await this.UpdateUserInfo(updateData);
      this.retrievedUser = await this.GetUserData(this.retrievedUser.id);
      this.challongeApiResetLoading = false;
    }
  }
};
</script>
