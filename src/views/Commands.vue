<template>
  <main class="main-container main-container--fullheight commands">
    <div class="container">
      <div ref="terminal" class="terminal" @click="focusInput">
        <div v-for="({ type, time, message }, i) in log" :key="i" class="terminal-message">
          <span v-if="timestamps" class="terminal-message__time timestamp">[{{ time }}]</span>
          <span class="terminal-message__sign" :class="`terminal-message__sign--${type}`" v-text="type === 'out' ? '>' : '<'"></span>
          <span class="terminal-message__content">{{ message }}</span>
        </div>
        <div class="terminal__input-wrapper">
          <span v-tooltip="$t('commands-send')" class="terminal-message__sign sign-input" @click="sendCommand">></span>
          <input
            ref="terminal-input" type="text" spellcheck="false" :value="command" enterkeyhint="enter" class="terminal__input"
            @input="command = $event.target.value"
            @keydown.enter="sendCommand"
            @keydown.tab.prevent="autocomplete"
            @keydown.up="historyPrevious"
            @keydown.down="historyNext"
            @keydown.ctrl.65.prevent="jumpToStart"
            @keydown.ctrl.75.prevent="removeAfterCursor"
            @keydown.ctrl.76.prevent="clearTerminal"
            @keydown.ctrl.85.prevent="deleteBeforeCursorAndJumpToStart"
          >
          <input v-model="autocompleteSuggestion" type="text" spellcheck="false" class="terminal__input terminal__input--autocomplete">
        </div>
      </div>
    </div>
  </main>
</template>

<script>
  import { mapGetters } from 'vuex';
  import * as storage from '../utils/storage';
  import fetchWiki from '../utils/fetchWiki';
  import getSelectedText from '../utils/getSelectedText';
  import createVirtualDOM from '../utils/createVirtualDOM';

  class CommandsCache {
    constructor(maxLength) {
      this._cache = [];
      this._maxLength = maxLength;
    }

    get length() {
      return this._cache.length;
    }

    get(index) {
      return this._cache[index] || '';
    }

    add(el) {
      if (this.get(0) === el) return;
      this._cache.unshift(el);
      this.save();
    }

    trim() {
      while (this.length > this._maxLength) this._cache.pop();
    }

    save() {
      storage.set('command-history', this._cache);
    }

    load() {
      const commandHistory = storage.get('command-history');
      if (commandHistory && Array.isArray(commandHistory)) this._cache = commandHistory;
      this.trim();
    }

    clear() {
      this._cache = [];
      storage.remove('command-history');
    }
  }

  export default {
    name: 'Commands',
    metaInfo() {
      return {
        title: this.$t('commands'),
      };
    },
    data() {
      return {
        command: '',
        log: [],
        commandHistory: new CommandsCache(20),
        commandHistoryIndex: -1,
        asfCommands: [],
        lastTabPressTime: 0,
      };
    },
    computed: {
      ...mapGetters({
        version: 'asf/version',
        timestamps: 'settings/timestamps',
      }),
      commands() {
        return [
          ...this.asfCommands.filter(({ command }) => command !== 'help'),
          { command: 'oa', description: this.$t('terminal-command-oa') },
          { command: 'r', description: this.$t('terminal-command-r') },
          { command: 'r^', description: this.$t('terminal-command-r-mode') },
          { command: 'sa', description: this.$t('terminal-command-sa') },
        ];
      },
      commandsNames() {
        return this.commands.map(command => command.command.split(' ')[0]).sort();
      },
      uiCommands() {
        return [
          { command: 'commands', description: this.$t('terminal-commands') },
          { command: 'help <Command>', description: this.$t('terminal-help') },
          { command: 'clear', description: this.$t('terminal-command-clear') },
          { command: 'clearhistory', description: this.$t('terminal-command-clear-history') },
        ];
      },
      uiCommandsNames() {
        return this.uiCommands.map(uiCommand => uiCommand.command.split(' ')[0]).sort();
      },
      allCommands() {
        return this.commands.concat(this.uiCommands).sort();
      },
      allCommandsNames() {
        return this.commandsNames.concat(this.uiCommandsNames).sort();
      },
      allCommandsParameters() {
        return this.allCommands.map(({ command }) => command.split(' '))
          .map(([command, ...params]) => ({ command, params }))
          // eslint-disable-next-line no-return-assign, no-sequences
          .reduce((commandParameters, { command, params }) => (commandParameters[command] = params, commandParameters), {});
      },
      autocompleteSuggestion() {
        if (this.suggestedCommand) return this.command.replace(/./g, ' ') + this.suggestedCommand.substr(this.command.length);

        if (this.selectedCommand) {
          if (!this.suggestedParameters || !this.suggestedParameters.length) return;

          if (this.suggestedParameterValue) {
            return [
              this.command.replace(/./g, ' ') + this.suggestedParameterValue.slice(this.currentParameterValue.length),
              ...this.suggestedParameters.slice(this.currentParameterIndex),
            ].join(' ');
          }

          const remainingParameters = this.suggestedParameters.slice((this.currentParameterValue.length) ? this.currentParameterIndex : this.currentParameterIndex - 1);
          if (!remainingParameters.length && this.currentParameter) remainingParameters.push(this.currentParameter);

          return this.command.replace(/./g, ' ')
            + ((this.currentParameterValue.length) ? ' ' : '')
            + remainingParameters.join(' ');
        }
      },
      suggestedCommand() {
        if (!this.command) return;
        return this.allCommandsNames.find(command => command.startsWith(this.command));
      },
      suggestedParameters() {
        if (this.selectedCommand && this.allCommandsParameters[this.selectedCommand]) return this.allCommandsParameters[this.selectedCommand];

        return [];
      },
      currentParameterIndex() {
        return this.command.split(' ').length - 1;
      },
      currentParameter() {
        if (!this.suggestedParameters.length) return;

        const currentParameter = this.suggestedParameters[this.currentParameterIndex - 1];
        if (currentParameter) return currentParameter;

        const lastParameter = this.suggestedParameters[this.suggestedParameters.length - 1];
        if (lastParameter.substr(-2, 1) === 's') return lastParameter;
      },
      currentParameterValue() {
        if (this.currentParameter && this.currentParameter.substr(-2, 1) === 's') {
          const currentParameterValue = this.command.split(' ')[this.currentParameterIndex].split(',');
          return currentParameterValue[currentParameterValue.length - 1];
        }

        return this.command.split(' ')[this.currentParameterIndex];
      },
      suggestedParameterValue() {
        if (!this.currentParameterValue || !this.currentParameterValue.length) return;
        if (!this.currentParameter) return;

        switch (this.currentParameter.toLowerCase()) {
          case '[bot]':
          case '[bots]':
          case '<targetbot>':
            // eslint-disable-next-line no-case-declarations
            const botGroups = ['ASF', '@ALL', '@FARMING', '@IDLE', '@OFFLINE', '@ONLINE'];

            // eslint-disable-next-line no-case-declarations
            const suggestedBot = [...this.$store.getters['bots/bots'].map(bot => bot.name), ...botGroups]
              .find(name => name.startsWith(this.currentParameterValue));

            if (suggestedBot) return suggestedBot;

            return [...this.$store.getters['bots/bots'].map(bot => bot.name), ...botGroups]
              .find(name => name.toLowerCase().startsWith(this.currentParameterValue.toLowerCase()));
          case '<command>':
            return this.allCommandsNames.find(name => name.startsWith(this.currentParameterValue));
          case '<modes>':
            if (this.selectedCommand === 'transfer') {
              return ['All', 'Background', 'Booster', 'Card', 'Emoticon', 'Foil', 'Gems', 'Unknown']
                .find(name => name.toLowerCase().startsWith(this.currentParameterValue.toLowerCase()));
            }

            if (this.selectedCommand === 'redeem^') {
              return ['FD', 'FF', 'FKMD', 'SD', 'SF', 'SI', 'SKMG', 'V']
                .find(name => name.toLowerCase().startsWith(this.currentParameterValue.toLowerCase()));
            }

            return;
          case '<type>':
            if (this.selectedCommand !== 'input') return;

            return ['DeviceID', 'Login', 'Password', 'SteamGuard', 'SteamParentalCode', 'TwoFactorAuthentication']
              .find(name => name.toLowerCase().startsWith(this.currentParameterValue.toLowerCase()));
          case '<settings>':
            if (this.selectedCommand !== 'privacy') return;

            return ['Private', 'FriendsOnly', 'Public']
              .find(name => name.toLowerCase().startsWith(this.currentParameterValue.toLowerCase()));
          // no default
        }
      },
      selectedCommand() {
        if (!this.command) return;
        return this.allCommandsNames.find(command => command === this.command.split(' ')[0]);
      },
    },
    watch: {
      log() {
        this.$nextTick(() => {
          this.$refs.terminal.scrollTop = Math.max(0, this.$refs.terminal.scrollHeight - this.$refs.terminal.clientHeight);
        });
      },
    },
    async created() {
      this.commandHistory.load();
      this.asfCommands = await this.loadCommands();
    },
    mounted() {
      this.log.push({ type: 'in', time: this.getTimestamp(), message: this.$t('commands-default') });
      this.$refs['terminal-input'].focus();
    },
    methods: {
      async sendCommand() {
        const commandToExecute = this.command.trim();
        this.command = '';

        if (!commandToExecute) return;

        this.commandHistoryIndex = -1;
        this.commandHistory.add(commandToExecute);

        const response = { type: 'in', time: '...', message: '...' };

        this.log.push({ type: 'out', time: this.getTimestamp(), message: commandToExecute });
        this.log.push(response);

        try {
          const result = await this.executeCommand(commandToExecute);
          response.message = result.trim();
        } catch (err) {
          response.message = `Error: ${err.message}`;
        } finally {
          response.time = this.getTimestamp();
        }
      },
      async executeCommand(commandToExecute) {
        switch (commandToExecute.split(' ')[0]) {
          case 'commands':
            return this.$t('terminal-available-commands', { commands: this.commandsNames.join(', '), uiCommands: this.uiCommandsNames.join(', ') });
          case 'help':
            if (commandToExecute.split(' ')[1]) return this.commandHelp(commandToExecute.split(' ')[1]);
            return this.$t('terminal-help-text');
          case 'clear':
            // eslint-disable-next-line no-return-assign
            return this.log = [];
          case 'clearhistory':
            this.commandHistory.clear();
            return this.$t('terminal-command-cleared');
        }

        return this.$http.command(commandToExecute);
      },
      getTimestamp() {
        return new Date().toLocaleTimeString();
      },
      commandHelp(command) {
        const foundCommand = this.allCommands.find(allCommand => allCommand.command.split(' ')[0] === command);
        if (foundCommand) return foundCommand.description;
        return this.$t('terminal-no-help', { command });
      },
      focusInput() {
        const selectedText = getSelectedText();
        if (selectedText) return;
        this.$refs['terminal-input'].focus();
      },
      autocomplete() {
        if (!this.selectedCommand) this.command = this.suggestedCommand || this.command;

        if (this.selectedCommand && this.suggestedParameterValue) {
          const splitCommand = this.command.split(' ');
          const splitCurrentParameter = splitCommand[splitCommand.length - 1].split(',');

          this.command = [...splitCommand.slice(0, -1), [...splitCurrentParameter.slice(0, -1), this.suggestedParameterValue].join(',')].join(' ');
        } else if (this.command === 'oa') {
          this.command = 'owns ASF';
        } else if (this.command === 'sa') {
          this.command = 'status ASF';
        } else if (this.command === 'r') {
          this.command = 'redeem';
        } else if (this.command === 'r^') {
          this.command = 'redeem^';
        } else if (this.command === '') {
          const tabPressTime = Date.now();
          if (tabPressTime - this.lastTabPressTime <= 500) this.command = 'commands';
          this.lastTabPressTime = tabPressTime;
        }
      },
      historyPrevious() {
        if (this.commandHistoryIndex + 1 < this.commandHistory.length) {
          this.commandHistoryIndex++;
          this.command = this.commandHistory.get(this.commandHistoryIndex);
        }

        this.moveCursorToEnd();
      },
      historyNext() {
        if (this.commandHistoryIndex > 0) {
          this.commandHistoryIndex--;
          this.command = this.commandHistory.get(this.commandHistoryIndex);
        } else if (this.commandHistoryIndex === 0) {
          this.commandHistoryIndex = -1;
          this.command = '';
        }
      },
      clearTerminal() {
        this.log = [];
      },
      jumpToStart() {
        const el = this.$refs['terminal-input'];

        if (el.setSelectionRange) setTimeout(() => el.setSelectionRange(0, 0), 0);
      },
      removeAfterCursor() {
        const pos = this.$refs['terminal-input'].selectionStart;
        this.command = this.command.substr(0, pos);
      },
      removeBeforeCursor() {
        const pos = this.$refs['terminal-input'].selectionStart;
        const inputLength = this.$refs['terminal-input'].value.length;
        this.command = this.command.substr(pos, inputLength);
      },
      moveCursorToEnd() {
        const el = this.$refs['terminal-input'];
        const len = this.command.length;

        if (el.setSelectionRange) setTimeout(() => el.setSelectionRange(len, len), 0);
      },
      deleteBeforeCursorAndJumpToStart() {
        this.removeBeforeCursor();
        this.jumpToStart();
      },
      parseCommandsHTML(commandsWikiRaw) {
        const virtualDOM = createVirtualDOM(commandsWikiRaw);
        const commandsTableHTML = virtualDOM.querySelector('.markdown-heading > h2').parentElement.nextElementSibling;

        return Array.from(commandsTableHTML.querySelectorAll('tbody tr'))
          .map(tableRow => tableRow.textContent.trim().split('\n'))
          .map(([command, access, description]) => ({ command, access, description }));
      },
      async fetchCommands() {
        const { locale } = this.$i18n;
        const wiki = await fetchWiki('Commands', this.version, locale);
        const commands = this.parseCommandsHTML(wiki);

        storage.set(`cache:asf-commands:${locale}`, { timestamp: Date.now(), commands });

        return commands;
      },
      async loadCommands() {
        const { locale } = this.$i18n;
        const commandsCache = storage.get(`cache:asf-commands:${locale}`);

        if (commandsCache) {
          const { timestamp, commands } = commandsCache;
          if (timestamp > Date.now() - 6 * 60 * 60 * 1000) return commands;
        }

        return this.fetchCommands();
      },
    },
  };
</script>

<style lang="scss">
  .commands {
    display: grid;
    grid-template-rows: 1fr;

    > div {
      min-height: 0;
    }
  }

  .sign-input {
    cursor: pointer;
  }

  .timestamp {
    margin-right: 0.5em;
  }
</style>
