<template>
  <main class="main-container main-container--fullheight">
    <div class="container">
      <ConfigEditor :fields="fields" :categories="displayCategories ? categories : null" :model="model"></ConfigEditor>

      <div class="form-item">
        <button class="button button--confirm" @click="save">{{ $t('save') }}</button>
      </div>
    </div>
  </main>
</template>

<script>
  import { mapGetters } from 'vuex';
  import ConfigEditor from '../components/Config/Editor.vue';
  import { uiCategories } from '../utils/configCategories';
  import delay from '../utils/delay';

  export default {
    name: 'UiConfig',
    metaInfo() {
      return {
        title: this.$t('ui-config'),
      };
    },
    components: { ConfigEditor },
    data() {
      const fields = [
        {
          param: this.$t('default-page'),
          paramName: 'defaultView',
          type: 'enum',
          defaultValue: 'bots',
          values: {
            [this.$t('bots')]: 'bots',
            [this.$t('commands')]: 'commands',
            [this.$t('releases')]: 'releases',
            [this.$t('plugins')]: 'plugins',
            [this.$t('log')]: 'log',
            [this.$t('asf-config')]: 'asf-config',
            [this.$t('ui-config')]: 'ui-config',
            [this.$t('mass-editor')]: 'mass-editor',
            [this.$t('last-visited-page')]: '_last-visited-page',
          },
          description: this.$t('default-page-description'),
        },
        {
          param: this.$t('notification-position'),
          paramName: 'notificationPosition',
          type: 'enum',
          defaultValue: 'rightBottom',
          values: {
            [this.$t('notification-position-left-top')]: 'leftTop',
            [this.$t('notification-position-left-bottom')]: 'leftBottom',
            [this.$t('notification-position-right-top')]: 'rightTop',
            [this.$t('notification-position-right-bottom')]: 'rightBottom',
            [this.$t('notification-position-center-top')]: 'centerTop',
            [this.$t('notification-position-center-bottom')]: 'centerBottom',
          },
          description: this.$t('notification-position-description'),
        },
        {
          param: this.$t('notify-release'),
          paramName: 'notifyRelease',
          type: 'boolean',
          description: this.$t('notify-release-description'),
        },
        {
          param: this.$t('display-categories'),
          paramName: 'displayCategories',
          type: 'boolean',
          description: this.$t('display-categories-description'),
        },
        {
          param: this.$t('timestamps'),
          paramName: 'timestamps',
          type: 'boolean',
          description: this.$t('timestamps-description'),
        },
        {
          param: this.$t('bot-nicknames'),
          paramName: 'nicknames',
          type: 'boolean',
          description: this.$t('bot-nicknames-description'),
        },
        {
          param: this.$t('bot-game-name'),
          paramName: 'gameName',
          type: 'boolean',
          description: this.$t('bot-game-name-description'),
        },
        {
          param: this.$t('bot-order-numeric'),
          paramName: 'orderBotsNumeric',
          type: 'boolean',
          description: this.$t('bot-order-numeric-description'),
        },
        {
          param: this.$t('bot-order-disabled'),
          paramName: 'orderDisabledBotsLast',
          type: 'boolean',
          description: this.$t('bot-order-disabled-description'),
        },
        {
          param: this.$t('bot-fav-buttons'),
          paramName: 'favButtons',
          type: 'flag',
          defaultValue: 0,
          values: {
            [this.$t('none')]: 0,
            [this.$t('bot-fav-buttons-2fa')]: 1 << 0,
            [this.$t('bot-fav-buttons-bgr')]: 1 << 1,
            [this.$t('bot-fav-buttons-config')]: 1 << 2,
            [this.$t('bot-fav-buttons-pause')]: 1 << 3,
            [this.$t('all')]: 15,
          },
          description: this.$t('bot-fav-buttons-description'),
        },
        {
          param: this.$t('tooltip-delay'),
          paramName: 'tooltipDelay',
          type: 'enum',
          defaultValue: 800,
          values: {
            [this.$t('tooltip-instant')]: 0,
            [this.$t('tooltip-delayed')]: 800,
            [this.$t('tooltip-hide')]: 600_000,
          },
          description: this.$t('tooltip-delay-description'),
        },
        {
          param: this.$t('log-previous-amount'),
          paramName: 'previousAmount',
          type: 'enum',
          defaultValue: 50,
          values: {
            25: 25,
            50: 50,
            75: 75,
            100: 100,
          },
          description: this.$t('log-previous-amount-description'),
        },
        {
          param: this.$t('log-information'),
          paramName: 'logInformation',
          type: 'flag',
          defaultValue: 13,
          values: {
            [this.$t('none')]: 0,
            [this.$t('log-information-time')]: 1 << 0,
            [this.$t('log-information-process')]: 1 << 1,
            [this.$t('log-information-level')]: 1 << 2,
            [this.$t('log-information-logger')]: 1 << 3,
            [this.$t('all')]: 15,
          },
          description: this.$t('log-information-description'),
        },
        {
          param: this.$t('log-timestamp'),
          paramName: 'logTimestamp',
          type: 'enum',
          defaultValue: 'timeOnlyEu',
          values: {
            [this.$t('log-timestamp-time-only-eu')]: 'timeOnlyEu',
            [this.$t('log-timestamp-time-only-locale')]: 'timeOnlyLocale',
            [this.$t('log-timestamp-time-only-us')]: 'timeOnlyUs',
            [this.$t('log-timestamp-time-date-eu')]: 'timeDateEu',
            [this.$t('log-timestamp-time-date-locale')]: 'timeDateLocale',
            [this.$t('log-timestamp-time-date-us')]: 'timeDateUs',
          },
          description: this.$t('log-timestamp-description'),
        },
      ];

      return {
        fields,
        categories: uiCategories,
        model: {
          defaultView: this.$store.getters['settings/defaultView'],
          notificationPosition: this.$store.getters['settings/notificationPosition'],
          notifyRelease: this.$store.getters['settings/notifyRelease'],
          displayCategories: this.$store.getters['settings/displayCategories'],
          timestamps: this.$store.getters['settings/timestamps'],
          nicknames: this.$store.getters['settings/nicknames'],
          gameName: this.$store.getters['settings/gameName'],
          favButtons: this.$store.getters['settings/favButtons'],
          tooltipDelay: this.$store.getters['settings/tooltipDelay'],
          orderDisabledBotsLast: this.$store.getters['settings/orderDisabledBotsLast'],
          previousAmount: this.$store.getters['settings/previousAmount'],
          logInformation: this.$store.getters['settings/logInformation'],
          logTimestamp: this.$store.getters['settings/logTimestamp'],
          orderBotsNumeric: this.$store.getters['settings/orderBotsNumeric'],
        },
      };
    },
    computed: {
      ...mapGetters({
        displayCategories: 'settings/displayCategories',
      }),
    },
    methods: {
      async save() {
        this.$store.dispatch('settings/setDefaultView', this.model.defaultView);
        this.$store.dispatch('settings/setNotificationPosition', this.model.notificationPosition);
        this.$store.dispatch('settings/setNotifyRelease', this.model.notifyRelease);
        this.$store.dispatch('settings/setDisplayCategories', this.model.displayCategories);
        this.$store.dispatch('settings/setTimestamps', this.model.timestamps);
        this.$store.dispatch('settings/setNicknames', this.model.nicknames);
        this.$store.dispatch('settings/setGameName', this.model.gameName);
        this.$store.dispatch('settings/setFavButtons', this.model.favButtons);
        this.$store.dispatch('settings/setTooltipDelay', this.model.tooltipDelay);
        this.$store.dispatch('settings/setOrderDisabledBotsLast', this.model.orderDisabledBotsLast);
        this.$store.dispatch('settings/setPreviousAmount', this.model.previousAmount);
        this.$store.dispatch('settings/setLogInformation', this.model.logInformation);
        this.$store.dispatch('settings/setLogTimestamp', this.model.logTimestamp);
        this.$store.dispatch('settings/setOrderBotsNumeric', this.model.orderBotsNumeric);

        this.$snotify.setDefaults({
          toast: {
            position: this.model.notificationPosition,
          },
        });

        this.$success(this.$t('settings-saved'));

        // The vue2 version of v-tooltip does not support changing tooltip-options in runtime
        // Ref issue: https://github.com/Akryum/v-tooltip/pull/773
        //
        // So we have to do it the ugly way:
        await delay(1000);
        window.location.reload();
      },
    },
  };
</script>
