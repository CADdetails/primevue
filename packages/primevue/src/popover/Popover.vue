<template>
    <Portal :appendTo="appendTo">
        <transition name="p-popover" @enter="onEnter" @leave="onLeave" @after-leave="onAfterLeave" v-bind="ptm('transition')">
            <div v-if="visible" :ref="containerRef" v-focustrap role="dialog" :aria-modal="visible" @click="onOverlayClick" :class="cx('root')" v-bind="ptmi('root')">
                <slot v-if="$slots.container" name="container" :closeCallback="hide" :keydownCallback="(event) => onButtonKeydown(event)"></slot>
                <template v-else>
                    <div :class="cx('content')" @click.capture="onContentClick" @mousedown.capture="onContentClick" @keydown="onContentKeydown" v-bind="ptm('content')">
                        <slot></slot>
                    </div>
                </template>
            </div>
        </transition>
    </Portal>
</template>

<script>
import { $dt } from '@primeuix/styled';
import { addStyle, focus, isClient, isTouchDevice, setAttribute } from '@primeuix/utils/dom';
import { ZIndex } from '@primeuix/utils/zindex';
import { ConnectedOverlayScrollHandler } from '@primevue/core/utils';
import FocusTrap from 'primevue/focustrap';
import OverlayEventBus from 'primevue/overlayeventbus';
import Portal from 'primevue/portal';
import Ripple from 'primevue/ripple';
import BasePopover from './BasePopover.vue';

export default {
    name: 'Popover',
    extends: BasePopover,
    inheritAttrs: false,
    emits: ['show', 'hide'],
    props: {
        isModal: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            visible: false
        };
    },
    watch: {
        dismissable: {
            immediate: true,
            handler(newValue) {
                if (newValue) {
                    this.bindOutsideClickListener();
                } else {
                    this.unbindOutsideClickListener();
                }
            }
        }
    },
    selfClick: false,
    target: null,
    eventTarget: null,
    outsideClickListener: null,
    scrollHandler: null,
    resizeListener: null,
    container: null,
    styleElement: null,
    overlayEventListener: null,
    documentKeydownListener: null,
    beforeUnmount() {
        if (this.dismissable) {
            this.unbindOutsideClickListener();
        }

        if (this.scrollHandler) {
            this.scrollHandler.destroy();
            this.scrollHandler = null;
        }

        this.destroyStyle();
        this.unbindResizeListener();
        this.target = null;

        if (this.container && this.autoZIndex) {
            ZIndex.clear(this.container);
        }

        if (this.overlayEventListener) {
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
        }

        this.container = null;
    },
    mounted() {
        if (this.breakpoints) {
            this.createStyle();
        }
    },
    methods: {
        toggle(event, target) {
            if (this.visible) this.hide();
            else this.show(event, target);
        },
        show(event, target) {
            this.visible = true;
            this.eventTarget = event.currentTarget;
            this.target = target || event.currentTarget;
        },
        hide() {
            this.visible = false;
        },
        onContentClick() {
            this.selfClick = true;
        },
        onEnter(el) {
            addStyle(el, { position: 'absolute', top: '0' });
            this.alignOverlay();

            if (this.dismissable) {
                this.bindOutsideClickListener();
            }

            this.bindScrollListener();
            this.bindResizeListener();

            if (this.autoZIndex) {
                ZIndex.set('overlay', el, this.baseZIndex + this.$primevue.config.zIndex.overlay);
            }

            this.overlayEventListener = (e) => {
                if (this.container.contains(e.target) || e.target.closest('.p-popover') === this.container) {
                    this.selfClick = true;
                }
            };

            this.focus();
            OverlayEventBus.on('overlay-click', this.overlayEventListener);
            this.$emit('show');

            if (this.closeOnEscape) {
                this.bindDocumentKeyDownListener();
            }
        },
        onLeave() {
            this.unbindOutsideClickListener();
            this.unbindScrollListener();
            this.unbindResizeListener();
            this.unbindDocumentKeyDownListener();
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
            this.$emit('hide');
        },
        onAfterLeave(el) {
            if (this.autoZIndex) {
                ZIndex.clear(el);
            }
        },
        alignOverlay() {
            if (!this.container || !this.target) return;

            const containerEl = this.container;
            const targetEl = this.target;
            console.log('▶ Popover alignOverlay triggered');
            console.log('Target Element:', this.target);
            console.log('Container Element:', this.container);
            // Detect if target is inside a dialog
            const dialogContent = targetEl.closest('.p-dialog-content');
            const isInDialog = !!dialogContent;

            if (isInDialog) {
                // Position relative to dialog content box
                const offsetParent = containerEl.offsetParent;
                if (!offsetParent) return;

                const top = targetEl.offsetTop + targetEl.offsetHeight + 8;
                const left = targetEl.offsetLeft;

                containerEl.style.position = 'absolute';
                containerEl.style.top = `${top}px`;
                containerEl.style.left = `${left}px`;

                const arrowLeft = targetEl.offsetWidth / 2;
                containerEl.style.setProperty($dt('popover.arrow.left').name, `${arrowLeft}px`);
            } else {
                if (!this.container || !this.target) return;

                const containerEl = this.container;
                const targetEl = this.target;
                const spacing = 8;
                const minBuffer = 100;

                const targetRect = targetEl.getBoundingClientRect();
                const containerRect = containerEl.getBoundingClientRect();
                const popoverHeight = containerRect.height || 200; // fallback in case not rendered yet

                const spaceBelow = window.innerHeight - targetRect.bottom - spacing;
                const spaceAbove = targetRect.top - spacing;

                const shouldFlip = spaceBelow < popoverHeight + spacing && spaceAbove > popoverHeight + spacing + minBuffer;
                // Calculate position
                const top = shouldFlip ? targetRect.top - popoverHeight - spacing : targetRect.bottom + spacing;

                const left = targetRect.left;
                const arrowLeft = targetRect.width / 2;

                containerEl.style.position = 'fixed';
                containerEl.style.top = `${top}px`;
                containerEl.style.left = `${left}px`;
                containerEl.style.setProperty($dt('popover.arrow.left').name, `${arrowLeft}px`);

                // Handle flip classes/attributes
                if (shouldFlip) {
                    containerEl.setAttribute('data-p-popover-flipped', 'true');
                    if (!this.isUnstyled) containerEl.classList.add('p-popover-flipped');
                } else {
                    containerEl.removeAttribute('data-p-popover-flipped');
                    if (!this.isUnstyled) containerEl.classList.remove('p-popover-flipped');
                }

                // Debug logs
                console.log('[Popover alignOverlay]', {
                    shouldFlip,
                    spaceBelow,
                    spaceAbove,
                    popoverHeight,
                    targetRect,
                    top,
                    left
                });
            }
        },
        onContentKeydown(event) {
            if (event.code === 'Escape' && this.closeOnEscape) {
                this.hide();
                focus(this.target);
            }
        },
        onButtonKeydown(event) {
            switch (event.code) {
                case 'ArrowDown':
                case 'ArrowUp':
                case 'ArrowLeft':
                case 'ArrowRight':
                    event.preventDefault();

                default:
                    break;
            }
        },
        focus() {
            let focusTarget = this.container.querySelector('[autofocus]');

            if (focusTarget) {
                focusTarget.focus();
            }
        },
        onKeyDown(event) {
            if (event.code === 'Escape' && this.closeOnEscape) {
                this.visible = false;
            }
        },
        bindDocumentKeyDownListener() {
            if (!this.documentKeydownListener) {
                this.documentKeydownListener = this.onKeyDown.bind(this);
                window.document.addEventListener('keydown', this.documentKeydownListener);
            }
        },
        unbindDocumentKeyDownListener() {
            if (this.documentKeydownListener) {
                window.document.removeEventListener('keydown', this.documentKeydownListener);
                this.documentKeydownListener = null;
            }
        },
        bindOutsideClickListener() {
            if (!this.outsideClickListener && isClient()) {
                this.outsideClickListener = (event) => {
                    if (event.composedPath().includes(this.container)) this.selfClick = true;

                    if (this.visible && !this.selfClick && !this.isTargetClicked(event)) {
                        this.visible = false;
                    }
                    this.selfClick = false;
                };

                document.addEventListener('click', this.outsideClickListener);
            }
        },
        unbindOutsideClickListener() {
            if (this.outsideClickListener) {
                document.removeEventListener('click', this.outsideClickListener);
                this.outsideClickListener = null;
                this.selfClick = false;
            }
        },
        bindScrollListener() {
            if (!this.scrollHandler) {
                this.scrollHandler = new ConnectedOverlayScrollHandler(this.target, () => {
                    if (this.visible) {
                        this.visible = false;
                    }
                });
            }

            this.scrollHandler.bindScrollListener();
        },
        unbindScrollListener() {
            if (this.scrollHandler) {
                this.scrollHandler.unbindScrollListener();
            }
        },
        bindResizeListener() {
            if (!this.resizeListener) {
                this.resizeListener = () => {
                    if (this.visible && !isTouchDevice()) {
                        this.visible = false;
                    }
                };

                window.addEventListener('resize', this.resizeListener);
            }
        },
        unbindResizeListener() {
            if (this.resizeListener) {
                window.removeEventListener('resize', this.resizeListener);
                this.resizeListener = null;
            }
        },
        isTargetClicked(event) {
            return this.eventTarget && (this.eventTarget === event.target || this.eventTarget.contains(event.target));
        },
        containerRef(el) {
            this.container = el;
        },
        createStyle() {
            if (!this.styleElement && !this.isUnstyled) {
                this.styleElement = document.createElement('style');
                this.styleElement.type = 'text/css';
                setAttribute(this.styleElement, 'nonce', this.$primevue?.config?.csp?.nonce);
                document.head.appendChild(this.styleElement);

                let innerHTML = '';

                for (let breakpoint in this.breakpoints) {
                    innerHTML += `
                        @media screen and (max-width: ${breakpoint}) {
                            .p-popover[${this.$attrSelector}] {
                                width: ${this.breakpoints[breakpoint]} !important;
                            }
                        }
                    `;
                }

                this.styleElement.innerHTML = innerHTML;
            }
        },
        destroyStyle() {
            if (this.styleElement) {
                document.head.removeChild(this.styleElement);
                this.styleElement = null;
            }
        },
        onOverlayClick(event) {
            OverlayEventBus.emit('overlay-click', {
                originalEvent: event,
                target: this.target
            });
        }
    },
    directives: {
        focustrap: FocusTrap,
        ripple: Ripple
    },
    components: {
        Portal
    }
};
</script>
