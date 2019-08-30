<script>
const ACTION = 'action';
const WAITING = 'waiting';
const SUCCESS = 'success';
const ERROR = 'error';

const LABEL = {
  create: {
    action:  'Create',
    waiting: 'Creating&hellip;',
    success: 'Created',
    error:   'Error',
  },
  apply: {
    action:  'Apply',
    waiting: 'Applying&hellip;',
    success: 'Applied',
    error:   'Error',
  },
  save: {
    action:  'Save',
    waiting: 'Saving&hellip;',
    success: 'Saved',
    error:   'Error',
  },
  delete: {
    action:  'Delete',
    waiting: 'Deleting&hellip;',
    success: 'Deleted',
    error:   'Error',
  }
};

export default {
  props: {
    mode: {
      type:    String,
      default: 'save',
    },
    delay: {
      type:    Number,
      default: 5000,
    },

    name: {
      type:    String,
      default: null,
    },
    disabled: {
      type:    Boolean,
      default: false,
    },
    type: {
      type:    String,
      default: 'button'
    },
    tabIndex: {
      type:    Number,
      default: null,
    },

    actionColor: {
      type:    String,
      default: 'bg-primary',
    },
    waitingColor: {
      type:    String,
      default: 'bg-primary',
    },
    successColor: {
      type:    String,
      default: 'bg-success',
    },
    errorColor: {
      type:    String,
      default: 'bg-error',
    },

    actionLabel: {
      type:    String,
      default: null,
    },
    waitingLabel: {
      type:    String,
      default: null,
    },
    successLabel: {
      type:    String,
      default: null,
    },
    errorLabel: {
      type:    String,
      default: null,
    },
  },

  data() {
    return {
      phase: ACTION,
      timer: null,
    };
  },

  computed: {
    classes() {
      const key = `${ this.phase }Color`;
      const color = this[key];

      const out = {
        btn:     true,
        [color]: true,
      };

      return out;
    },

    label() {
      const override = this[`${ this.phase }Label`];

      if ( override ) {
        return override;
      }

      return LABEL[this.mode][this.phase];
    },

    isSpinning() {
      return this.phase === WAITING;
    },

    isDisabled() {
      return this.disabled || this.phase === WAITING;
    }
  },

  methods: {
    clicked() {
      if ( this.isDisabled ) {
        return;
      }

      clearTimeout(this.timer);

      this.phase = WAITING;

      this.$emit('click', (success) => {
        this.done(success);
      });
    },

    done(success) {
      this.phase = (success ? SUCCESS : ERROR );
      this.timer = setTimeout(() => {
        this.timerDone();
      }, this.delay );
    },

    timerDone() {
      if ( this.phase === SUCCESS || this.phase === ERROR ) {
        this.phase = ACTION;
      }
    }
  }
};
</script>

<template>
  <button
    :class="classes"
    :name="name"
    :type="type"
    :disabled="isDisabled"
    :tab-index="tabIndex"
    @click="clicked"
  >
    <i v-if="isSpinning" class="icon icon-spinner icon-spin mr-5" />
    <span v-html="label" />
  </button>
</template>