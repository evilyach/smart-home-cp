<template>
  <div>
    <div class="text-h4 q-mb-md">Климатконтроль</div>

    <q-input
      v-model="name"
      filled
      label="Имя устройства"
      hint="Введите имя устройства"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-select
      v-model="real_estate_picker"
      filled
      label="Помещение"
      hint="Выберите помещение, где находится устройство"
      :options="options"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-input
      v-model="ip"
      filled
      label="IP-адрес"
      hint="Введите сетевой адрес устройства"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-input
      v-model="password"
      filled
      label="Пароль"
      hint="Введите пароль доступа. Он должен содержать как минимум 1 строчный символ,
            1 заглавный символ, 1 цифру и быть не менее 8 символов в длину."
      :type="isPwd ? 'password' : 'text'"
      style="width: 99%"
    >
      <template v-slot:append>
        <q-icon
          :name="isPwd ? 'visibility_off' : 'visibility'"
          class="cursor-pointer"
          @click="isPwd = !isPwd"
        />
      </template>
    </q-input>

    <div style="height: 20px" />

    <q-checkbox
      v-model="to_alert"
      filled
      right-label
      label="Уведомлять об инцидентах?"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-input
      v-model="color"
      filled
      label="Цвет"
      hint="Выберете цвет подсветки устройства"
      :rules="['anyColor']"
      style="width: 99%"
    >
      <template v-slot:append>
        <q-icon name="colorize" class="cursor-pointer">
          <q-popup-proxy transition-show="scale" transition-hide="scale">
            <q-color v-model="color" />
          </q-popup-proxy>
        </q-icon>
      </template>
    </q-input>

    <div style="height: 20px" />

    <q-input
      v-model="temperature"
      filled
      label="Температура"
      hint="Введите температуру"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-input
      v-model="humidity"
      filled
      label="Влажность"
      hint="Введите влажность"
      style="width: 99%"
    />

    <div style="height: 20px" />

    <q-btn
      outline
      rounded
      color="primary"
      label="Добавить"
      @click="buttonClick"
    />
  </div>
</template>

<script>
import axios from "axios";
import passwordHash from "password-hash";
import router from "../../router/routes-backend";
import { addLogRecord } from "../../lib/log";

export default {
  data() {
    return {
      name: "",

      real_estate_picker: null,
      options: [],
      full_options: [],

      ip: "192.168.1.221",

      password: "",
      isPwd: true,

      to_alert: false,
      temperature: null,
      humidity: null,
      color: "#027be3",

      getData() {
        axios
          .get(router.getRealEstates())
          .then(res => {
            const data = res["data"][0];

            data.forEach(element => {
              this.options.push(element[0][1]);

              this.full_options.push({
                id: element[0][0],
                name: element[0][1]
              });
            });
          })
          .catch(e => {
            this.$q.notify({
              message: "Не удалось получить доступ к базе данных: " + e,
              color: "negative"
            });
          });
      },

      pushData() {
        // Check name to protect from SQLi
        if (/^[a-zA-Zа-яА-Я0-9\-_ ]+$/.test(this.name) === false) {
          this.$q.notify({
            message: "Введенное имя некорректно!",
            color: "warning"
          });
          return;
        }

        // Check if real estate is picked
        if (this.real_estate_picker === null) {
          this.$q.notify({
            message: "Выберите помещение!",
            color: "warning"
          });
          return;
        }

        // Get real estate id by name
        let real_estate_id;
        this.full_options.forEach(element => {
          if (element["name"] === this.real_estate_picker) {
            real_estate_id = element["id"];
          }
        });

        // Check if IP is valid
        if (
          /^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(
            this.ip
          ) === false
        ) {
          this.$q.notify({
            message: "Введенный IP-адрес некорректен!",
            color: "warning"
          });
          return;
        }

        // Check if password is strong
        if (
          /^(?=.*\d)(?=.*[a-z])(?=.*[A-Z])[0-9a-zA-Z]{8,}$/.test(
            this.password
          ) === false
        ) {
          this.$q.notify({
            message: "Пароль не удовлетворяет политике безопасности паролей!",
            color: "warning"
          });
          return;
        }

        // Check if temperature is valid
        if (
          this.temperature === null ||
          isNaN(this.temperature) === true ||
          this.temperature < 0 ||
          this.temperature > 100
        ) {
          this.$q.notify({
            message: "Введена неправильная температура!",
            color: "warning"
          });
          return;
        }

        // Check if humidity is valid
        if (
          isNaN(this.humidity) === true ||
          this.humidity < 0 ||
          this.humidity > 100
        ) {
          this.$q.notify({
            message: "Введена неправильная влажность!",
            color: "warning"
          });
          return;
        }

        // Get current time
        const date = new Date();
        const time =
          date.getFullYear() +
          "-" +
          (date.getMonth() + 1) +
          "-" +
          date.getDate() +
          " " +
          date.getHours() +
          ":" +
          date.getMinutes() +
          ":" +
          date.getSeconds();

        axios
          .post(router.addDevice(), {
            device_type_id: 10,
            real_estate_id: real_estate_id,
            status: 0,
            time_activated: time,
            time_deactivated: time,
            name: this.name,
            ip: this.ip,
            to_alarm: this.to_alert,
            temperature: this.temperature,
            humidity: null,
            speed: null,
            color: this.color,
            max_value: null,
            power: null,
            password: passwordHash.generate(this.password)
          })
          .then(() => {
            addLogRecord({
              log_level: "info",
              message: `Добавлено устройство "Климатконтроль" с названием "${this.name}"`
            });

            this.$q.notify({
              message: "Устройство успешно добавлено!",
              color: "positive"
            });
          })
          .catch(e => {
            addLogRecord({
              log_level: "error",
              message: `Неудачная попытка добавить устройство "Климатконтроль" с названием "${this.name}"`
            });

            this.$q.notify({
              message: "Не удалось получить доступ к базе данных: " + e,
              color: "negative"
            });
          });
      }
    };
  },
  methods: {
    buttonClick() {
      this.pushData();
    }
  },
  created() {
    this.getData();
  }
};
</script>
