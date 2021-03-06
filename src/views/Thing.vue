<template>
  <v-container v-if="!isLoadingInitialThingShadow">
    <v-row>
      <v-col>
        <span class="text-h2">{{ thingName }}</span>
        <v-btn icon color="secondary" @click="refreshConnection">
          <v-icon>mdi-refresh</v-icon>
        </v-btn>
        <v-icon top color="red" v-if="amplifyPubSubHasError"
          >mdi-wifi-off</v-icon
        >
        <v-icon top color="green" v-if="!amplifyPubSubHasError"
          >mdi-wifi</v-icon
        >
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-card>
          <v-card-title>Lamp status</v-card-title>
          <v-card-subtitle>
            Reported state
            <span class="green--text" v-if="thingShadow.reported.lampState"
              >ON</span
            >
            <span class="red--text" v-if="!thingShadow.reported.lampState"
              >OFF</span
            >
          </v-card-subtitle>

          <v-card-actions>
            <v-btn
              :loading="isUpdatingThingShadow"
              color="secondary"
              @click="publishShadowUpdateLampToggle"
            >
              <span v-if="thingShadow.desired.lampState">Turn off</span>
              <span v-if="!thingShadow.desired.lampState">Turn on</span>
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-card>
          <v-card-title>Pump status</v-card-title>
          <v-card-subtitle>
            Reported state:
            <span class="green--text" v-if="thingShadow.reported.pumpState"
              >ON</span
            >
            <span class="red--text" v-if="!thingShadow.reported.pumpState"
              >OFF</span
            >
          </v-card-subtitle>

          <v-card-actions>
            <v-btn
              :loading="isUpdatingThingShadow"
              color="secondary"
              @click="publishShadowUpdatePumpToggle"
            >
              <span v-if="thingShadow.desired.pumpState">Turn off</span>
              <span v-if="!thingShadow.desired.pumpState">Turn on</span>
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-card>
          <v-card-title>Light threshold</v-card-title>
          <v-card-subtitle>
            Reported threshold:
            <span class="green--text">{{
              thingShadow.reported.lightThreshold
            }}</span>
          </v-card-subtitle>

          <v-card-actions>
            <v-text-field
              v-on:keyup.enter="publishShadowUpdateLightThreshold"
              v-model.number="thingShadow.desired.lightThreshold"
            >
            </v-text-field>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-card>
          <v-card-title>Moisture threshold</v-card-title>
          <v-card-subtitle>
            Reported threshold:
            <span class="green--text">{{
              thingShadow.reported.moistureThreshold
            }}</span>
          </v-card-subtitle>

          <v-card-actions>
            <v-text-field
              v-on:keyup.enter="publishShadowUpdateMoistureThreshold"
              v-model.number="thingShadow.desired.moistureThreshold"
            >
            </v-text-field>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
  <v-progress-linear v-else indeterminate></v-progress-linear>
</template>

<script>
import { PubSub } from "aws-amplify";

export default {
  name: "Thing",
  data() {
    return {
      // MQTT SUBSCRIBERS
      acceptedShadowGetAndUpdateSubscriber: undefined,
      rejectedShadowUpdateSubscriber: undefined,
      // THING'S STATE
      thingShadow: {
        desired: {
          lampState: false,
          pumpState: false,
          lightThreshold: 0,
          moistureThreshold: 0
        },
        reported: {
          lampState: false,
          pumpState: false,
          lightThreshold: 0,
          moistureThreshold: 0
        }
      },

      // DISPLAY CONTROLS
      isLoadingInitialThingShadow: true,
      isUpdatingThingShadow: false,
      amplifyPubSubHasError: false,
      // INTERVALS
      intervalPublishShadowGet: undefined
    };
  },
  computed: {
    thingName() {
      return this.$route.params.thingName;
    }
  },
  beforeMount() {
    this.subscribeAllTopics();

    /**
     * Query shadow after 3 seconds
     */
    setTimeout(async () => {
      this.publishShadowGet();
    }, 3000);

    /**
     * Query shadow every 30 seconds
     */
    this.intervalPublishShadowGet = setInterval(async () => {
      this.publishShadowGet();
    }, 30000);
  },
  beforeDestroy() {
    this.unsubscribeAllTopics();
    clearInterval(this.intervalPublishShadowGet);
  },
  methods: {
    /**
     * Disconnect and start a new connection with AWS
     */
    async refreshConnection() {
      this.isUpdatingThingShadow = false;
      this.amplifyPubSubHasError = false;
      this.unsubscribeAllTopics();
      this.subscribeAllTopics();
      this.publishShadowGet();
    },

    /**
     * Publish the desired lamp state
     */
    async publishShadowUpdateLampToggle() {
      this.isUpdatingThingShadow = true;
      this.publishShadowUpdate({
        desired: {
          lampState: !this.thingShadow.desired.lampState
        }
      });
    },

    async publishShadowUpdatePumpToggle() {
      this.isUpdatingThingShadow = true;
      this.publishShadowUpdate({
        desired: {
          pumpState: !this.thingShadow.desired.pumpState
        }
      });
    },

    async publishShadowUpdateLightThreshold() {
      this.isUpdatingThingShadow = true;
      this.publishShadowUpdate({
        desired: {
          lightThreshold: this.thingShadow.desired.lightThreshold
        }
      });
    },

    async publishShadowUpdateMoistureThreshold() {
      this.isUpdatingThingShadow = true;
      this.publishShadowUpdate({
        desired: {
          moistureThreshold: this.thingShadow.desired.moistureThreshold
        }
      });
    },

    /**
     * Publish the shadow state to the update topic
     */
    async publishShadowUpdate(state) {
      try {
        PubSub.publish(`$aws/things/${this.thingName}/shadow/update`, {
          state
        });
      } catch (err) {
        console.error("Could not publish shadow update request ", err);
      }
    },

    /**
     * Publish an empty message to the shadow get topic
     */
    async publishShadowGet() {
      try {
        PubSub.publish(`$aws/things/${this.thingName}/shadow/get`, {});
      } catch (err) {
        console.error("Could not publish shadow get request ", err);
      }
    },

    subscribeAllTopics() {
      /**
       * These topics report the current state of the shadow, data recevied from these topics
       * are usually used for updating our local data
       */
      this.acceptedShadowGetAndUpdateSubscriber = PubSub.subscribe([
        "$aws/things/" + this.thingName + "/shadow/update/accepted"
      ]).subscribe({
        next: data => {
          if (this.isUpdatingThingShadow) {
            this.isUpdatingThingShadow = false;
          }
          if (data.value.state.desired != undefined) {
            Object.entries(data.value.state.desired).forEach(([k, v]) => {
              this.thingShadow.desired[k] = v;
            });
          }
          if (data.value.state.reported != undefined) {
            Object.entries(data.value.state.reported).forEach(([k, v]) => {
              this.thingShadow.reported[k] = v;
            });
          }
        },
        error: () => {
          this.amplifyPubSubHasError = true;
        }
      });

      this.acceptedShadowGetAndUpdateSubscriber = PubSub.subscribe([
        "$aws/things/" + this.thingName + "/shadow/get/accepted"
      ]).subscribe({
        next: data => {
          if (this.isLoadingInitialThingShadow) {
            this.isLoadingInitialThingShadow = false;
          }

          Object.entries(data.value.state.desired).forEach(([k, v]) => {
            this.thingShadow.desired[k] = v;
          });

          Object.entries(data.value.state.reported).forEach(([k, v]) => {
            this.thingShadow.reported[k] = v;
          });
        },
        error: () => {
          this.amplifyPubSubHasError = true;
        }
      });

      /**
       * These topics report errors
       */
      this.rejectedShadowUpdateSubscriber = PubSub.subscribe(
        "$aws/things/" + this.thingName + "/shadow/update/rejected"
      ).subscribe({
        next: () => {
          if (this.isUpdatingThingShadow) {
            this.isUpdatingThingShadow = false;
          }
        },
        error: () => {
          this.amplifyPubSubHasError = true;
        }
      });
    },

    unsubscribeAllTopics() {
      if (this.acceptedShadowGetAndUpdateSuscriber) {
        this.acceptedShadowGetAndUpdateSuscriber.unsubscribe();
      }
      if (this.rejectedShadowUpdateSubscriber) {
        this.rejectedShadowUpdateSubscriber.unsubscribe();
      }
    }
  }
};
</script>
