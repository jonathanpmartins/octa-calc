<template>
  <q-page>
    <div v-if="enabled">
      <q-list v-if="this.list.length > 0" bordered separator>
        <q-item
          clickable
          v-ripple
          v-for="item in this.list" :key="item.id"
          @click="connectTo(item)"
        >
          <q-item-section>
            <q-item-label><b>Name: </b>{{ item.name }}</q-item-label>
            <q-item-label caption>{{ item.id }}</q-item-label>
          </q-item-section>
        </q-item>
      </q-list>
      <q-list v-else bordered separator>
        <q-item>
          <q-item-section>
            <q-item-label><b>Scanning...</b></q-item-label>
          </q-item-section>
        </q-item>
      </q-list>
    </div>
    <q-banner
      v-else
      dense
      inline-actions
      class="text-white bg-red"
    >
      Bluetooth is not enabled.
      <template v-slot:action>
        <q-btn
          flat color="white"
          label="Turn ON Bluetooth"
          @click="doEnable()"
        />
      </template>
    </q-banner>
    <code v-if="error">
      {{ error }}
    </code>
  </q-page>
</template>

<script>
import { defineComponent } from 'vue';
import { Plugins } from '@capacitor/core';

const { BluetoothSerial } = Plugins;

export default defineComponent({
  name: 'IndexPage',
  data() {
    return {
      enabled: false,
      connected: false,
      address: null,
      list: [],
      error: null,
    };
  },
  async mounted() {
    await this.setup();
  },
  methods: {
    async setup() {
      await this.doEnable();

      await this.scanAndSetList();

      setInterval(async () => {
        await this.scanAndSetList();
      }, 2000);
    },
    async doEnable() {
      let isEnabled = await this.isEnabled();
      if (!isEnabled) {
        isEnabled = await this.enable();
      }
      this.enabled = isEnabled;
    },
    async enable() {
      return new Promise((resolve, reject) => {
        BluetoothSerial.enable().then((response) => {
          resolve(!!response.enabled);
        }).catch((error) => {
          reject(error);
        });
      });
    },
    async isEnabled() {
      return new Promise((resolve, reject) => {
        BluetoothSerial.isEnabled().then((response) => {
          resolve(!!response.enabled);
        }).catch((error) => {
          reject(error);
        });
      });
    },
    async scan() {
      return new Promise((resolve, reject) => {
        BluetoothSerial.scan().then((result) => {
          resolve(result.devices);
        }).catch((error) => {
          reject(error);
        });
      });
    },
    async scanAndSetList() {
      if (!this.enabled || this.connected) {
        // console.warn('Is not enabled! dont scan!');
        return;
      }
      try {
        const data = [];
        const list = await this.scan();
        console.log('list', list);
        list.forEach((device) => data.push(device));
        this.list = data;
      } catch (e) {
        console.log('e', e);
        if ((`${e}`).includes('desabilitado')) {
          this.enabled = false;
          this.list = [];
        }
      }
    },
    async connectTo(item) {
      try {
        await this.connect(item.address);
        this.connected = true;
        this.address = item.address;
        console.log('--------------- connected!!!');

        BluetoothSerial.enableNotifications({
          address: '00:11:22:33:44:55',
          delimiter: '\n',
        }).then((result) => {
          console.warn('result', result);
          console.warn('result.eventName', result.eventName);
          BluetoothSerial.addListener(result.eventName, (payload) => {
            console.warn('payload', payload);
            console.warn('payload.data', payload.data);
            console.warn('payload.data.value', payload.data.value);
          });
        }).catch(() => {
          console.log('Error enabling listener for device');
        });
      } catch (e) {
        console.log('e', e);
      }
    },
    async connect(address) {
      return new Promise((resolve, reject) => {
        BluetoothSerial.connect({ address }).then(() => {
          resolve();
        }).catch(() => {
          reject();
        });
      });
    },
  },
});
</script>
