<template>
  <transition name="msgbox-fade">
    <div class="kzjr-message-box__wrapper" tabindex="-1" v-show="visible" @click.self="handleWrapperClick" role="dialog" aria-modal="true" :aria-label="title || 'dialog'">
      <div class="kzjr-message-box" :class="[customClass, center && 'kzjr-message-box--center']">
        <div class="kzjr-message-box__header" v-if="title !== null">
          <div class="kzjr-message-box__title">
            <div :class="['kzjr-message-box__status', typeClass]" v-if="typeClass && center">
            </div>
            <span>{{ title }}</span>
          </div>
          <div class="kzjr-message-box__headerbtn" aria-label="Close" v-if="showClose" @click="handleAction('cancel')" @keydown.enter="handleAction('cancel')">
            <img src="../../../img/new_close2.png" alt="" class="border">
            <img src="../../../img/new_close1.png" alt="" class="close">            
          </div>
        </div>
        <div class="kzjr-message-box__content">
          <div :class="['kzjr-message-box__status', typeClass]" v-if="typeClass && !center && message !== ''">
          </div>
          <div class="kzjr-message-box__message" v-if="message !== ''">
            <slot>
              <p v-if="!dangerouslyUseHTMLString">{{ message }}</p>
              <p v-else v-html="message"></p>
            </slot>
          </div>
          <div class="kzjr-message-box__input" v-show="showInput">
            <el-input v-model="inputValue" :type="inputType" @keydown.enter.native="handleInputEnter" :placeholder="inputPlaceholder" ref="input"></el-input>
            <div class="kzjr-message-box__errormsg" :style="{ visibility: !!editorErrorMessage ? 'visible' : 'hidden' }">{{ editorErrorMessage }}</div>
          </div>
        </div>
        <div class="kzjr-message-box__btns">
          <el-button :loading="cancelButtonLoading" :class="[ cancelButtonClasses ]" v-if="showCancelButton" :round="roundButton" size="medium" @click.native="handleAction('cancel')" @keydown.enter="handleAction('cancel')">
            {{ cancelButtonText || '取消' }}
          </el-button>
          <el-button :loading="confirmButtonLoading" ref="confirm" :class="[ confirmButtonClasses ]" v-show="showConfirmButton" :round="roundButton" size="medium" @click.native="handleAction('confirm')" @keydown.enter="handleAction('confirm')">
            {{ confirmButtonText || '确定'}}
          </el-button>
        </div>
      </div>
    </div>
  </transition>
</template>

<script type="text/babel">
  import Popup from 'kzjr_ui/src/utils/popup';
  import ElInput from 'kzjr_ui/packages/input';
  import ElButton from 'kzjr_ui/packages/button';
  import { addClass, removeClass } from 'kzjr_ui/src/utils/dom';
  import Dialog from 'kzjr_ui/src/utils/aria-dialog';
  let messageBox;
  let typeMap = {
    success: 'success',
    info: 'info',
    warning: 'warning',
    error: 'error'
  };
  export default {
    mixins: [Popup],
    props: {
      modal: {
        default: true
      },
      lockScroll: {
        default: true
      },
      showClose: {
        type: Boolean,
        default: true
      },
      closeOnClickModal: {
        default: true
      },
      closeOnPressEscape: {
        default: true
      },
      closeOnHashChange: {
        default: true
      },
      center: {
        default: true,
        type: Boolean
      },
      roundButton: {
        default: false,
        type: Boolean
      }
    },
    components: {
      ElInput,
      ElButton
    },
    computed: {
      typeClass () {
        return this.type && typeMap[this.type] ? `kzjr-icon-${typeMap[this.type]}` : '';
      },
      confirmButtonClasses () {
        return `kzjr-button--primary ${this.confirmButtonClass}`;
      },
      cancelButtonClasses () {
        return `${this.cancelButtonClass}`;
      }
    },
    methods: {
      getSafeClose () {
        const currentId = this.uid;
        return () => {
          this.$nextTick(() => {
            if (currentId === this.uid) this.doClose();
          });
        };
      },
      doClose () {
        if (!this.visible) return;
        this.visible = false;
        this._closing = true;
        this.onClose && this.onClose();
        messageBox.closeDialog(); // 解绑
        if (this.lockScroll) {
          setTimeout(() => {
            if (this.modal && this.bodyOverflow !== 'hidden') {
              document.body.style.overflow = this.bodyOverflow;
              document.body.style.paddingRight = this.bodyPaddingRight;
            }
            this.bodyOverflow = null;
            this.bodyPaddingRight = null;
          }, 200);
        }
        this.opened = false;
        if (!this.transition) {
          this.doAfterClose();
        }
        setTimeout(() => {
          if (this.action) this.callback(this.action, this);
        });
      },
      handleWrapperClick () {
        if (this.closeOnClickModal) {
          this.handleAction('cancel');
        }
      },
      handleInputEnter () {
        if (this.inputType !== 'textarea') {
          return this.handleAction('confirm');
        }
      },
      handleAction (action) {
        if (this.$type === 'prompt' && action === 'confirm' && !this.validate()) {
          return;
        }
        this.action = action;
        if (typeof this.beforeClose === 'function') {
          this.close = this.getSafeClose();
          this.beforeClose(action, this, this.close);
        } else {
          this.doClose();
        }
      },
      validate () {
        if (this.$type === 'prompt') {
          const inputPattern = this.inputPattern;
          if (inputPattern && !inputPattern.test(this.inputValue || '')) {
            this.editorErrorMessage = this.inputErrorMessage;
            addClass(this.getInputElement(), 'invalid');
            return false;
          }
          const inputValidator = this.inputValidator;
          if (typeof inputValidator === 'function') {
            const validateResult = inputValidator(this.inputValue);
            if (validateResult === false) {
              this.editorErrorMessage = this.inputErrorMessage;
              addClass(this.getInputElement(), 'invalid');
              return false;
            }
            if (typeof validateResult === 'string') {
              this.editorErrorMessage = validateResult;
              addClass(this.getInputElement(), 'invalid');
              return false;
            }
          }
        }
        this.editorErrorMessage = '';
        removeClass(this.getInputElement(), 'invalid');
        return true;
      },
      getFistFocus () {
        const $btns = this.$el.querySelector('.kzjr-message-box__btns .kzjr-button');
        const $title = this.$el.querySelector('.kzjr-message-box__btns .kzjr-message-box__title');
        return $btns && $btns[0] || $title;
      },
      getInputElement () {
        const inputRefs = this.$refs.input.$refs;
        return inputRefs.input || inputRefs.textarea;
      }
    },
    watch: {
      inputValue: {
        immediate: true,
        handler (val) {
          this.$nextTick(_ => {
            if (this.$type === 'prompt' && val !== null) {
              this.validate();
            }
          });
        }
      },
      visible (val) {
        if (val) {
          this.uid++;
          if (this.$type === 'alert' || this.$type === 'confirm') {
            this.$nextTick(() => {
              this.$refs.confirm.$el.focus();
            });
          }
          this.focusAfterClosed = document.activeElement;
          messageBox = new Dialog(this.$el, this.focusAfterClosed, this.getFistFocus());
        }
        // prompt
        if (this.$type !== 'prompt') return;
        if (val) {
          setTimeout(() => {
            if (this.$refs.input && this.$refs.input.$el) {
              this.getInputElement().focus();
            }
          }, 500);
        } else {
          this.editorErrorMessage = '';
          removeClass(this.getInputElement(), 'invalid');
        }
      }
    },
    mounted () {
      if (this.closeOnHashChange) {
        window.addEventListener('hashchange', this.close);
      }
    },
    beforeDestroy () {
      if (this.closeOnHashChange) {
        window.removeEventListener('hashchange', this.close);
      }
      setTimeout(() => {
        messageBox.closeDialog();
      });
    },
    data () {
      return {
        uid: 1,
        title: undefined,
        message: '',
        type: '',
        customClass: '',
        showInput: false,
        inputValue: null,
        inputPlaceholder: '',
        inputType: 'text',
        inputPattern: null,
        inputValidator: null,
        inputErrorMessage: '',
        showConfirmButton: true,
        showCancelButton: false,
        action: '',
        confirmButtonText: '',
        cancelButtonText: '',
        confirmButtonLoading: false,
        cancelButtonLoading: false,
        confirmButtonClass: '',
        confirmButtonDisabled: false,
        cancelButtonClass: '',
        editorErrorMessage: null,
        callback: null,
        dangerouslyUseHTMLString: false,
        focusAfterClosed: null,
        isOnComposition: false
      };
    }
  };
</script>
