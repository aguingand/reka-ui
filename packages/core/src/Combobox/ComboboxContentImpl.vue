<script lang="ts">
import type { Ref } from 'vue'
import { createContext, useForwardExpose, useForwardProps, useHideOthers } from '@/shared'
import { useBodyScrollLock } from '@/shared/useBodyScrollLock'

import type { DismissableLayerEmits, DismissableLayerProps } from '@/DismissableLayer'
import type { PopperContentProps } from '@/Popper'

export type ComboboxContentImplEmits = DismissableLayerEmits

export interface ComboboxContentImplProps extends PopperContentProps, DismissableLayerProps {
  /**
   * The positioning mode to use, <br>
   * `inline` is the default and you can control the position using CSS. <br>
   * `popper` positions content in the same way as our other primitives, for example `Popover` or `DropdownMenu`.
   */
  position?: 'inline' | 'popper'
  /** The document.body will be lock, and scrolling will be disabled. */
  bodyLock?: boolean
  /**
   * Allow component to be dismissableLayer.
   * @deprecated (Will be removed in version 2.0, use `Listbox` instead)
   */
  dismissable?: boolean
}

export const [injectComboboxContentContext, provideComboboxContentContext]
  = createContext<{
    position: Ref<'inline' | 'popper'>
  }>('ComboboxContent')
</script>

<script setup lang="ts">
import { computed, toRefs } from 'vue'
import { injectComboboxRootContext } from './ComboboxRoot.vue'
import { DismissableLayer } from '@/DismissableLayer'
import { PopperContent } from '@/Popper'
import { Primitive } from '@/Primitive'
import { ListboxContent } from '@/Listbox'

const props = withDefaults(defineProps<ComboboxContentImplProps>(), {
  position: 'inline',
  dismissable: true,
})
const emits = defineEmits<ComboboxContentImplEmits>()

const { position } = toRefs(props)
const rootContext = injectComboboxRootContext()

const { forwardRef, currentElement } = useForwardExpose()
useBodyScrollLock(props.bodyLock)
useHideOthers(rootContext.parentElement)

const pickedProps = computed(() => {
  if (props.position === 'popper')
    return props
  else return {}
})

const forwardedProps = useForwardProps(pickedProps.value)

const popperStyle = {
  // Ensure border-box for floating-ui calculations
  'boxSizing': 'border-box',
  '--reka-combobox-content-transform-origin':
        'var(--reka-popper-transform-origin)',
  '--reka-combobox-content-available-width':
        'var(--reka-popper-available-width)',
  '--reka-combobox-content-available-height':
        'var(--reka-popper-available-height)',
  '--reka-combobox-trigger-width': 'var(--reka-popper-anchor-width)',
  '--reka-combobox-trigger-height': 'var(--reka-popper-anchor-height)',
}

provideComboboxContentContext({ position })
</script>

<template>
  <ListboxContent as-child>
    <DismissableLayer
      as-child
      :disable-outside-pointer-events="disableOutsidePointerEvents"
      @dismiss="rootContext.onOpenChange(false)"
      @focus-outside="(ev) => {
        // if clicking inside the combobox, prevent dismiss
        if (rootContext.parentElement.value?.contains(ev.target as Node)) ev.preventDefault()
        emits('focusOutside', ev)
      }"
      @interact-outside="emits('interactOutside', $event)"
      @escape-key-down="emits('escapeKeyDown', $event)"
      @pointer-down-outside="(ev) => {
        // if clicking inside the combobox, prevent dismiss
        if (rootContext.parentElement.value?.contains(ev.target as Node)) ev.preventDefault()
        emits('pointerDownOutside', ev)
      }"
    >
      <component
        :is="position === 'popper' ? PopperContent : Primitive"
        v-bind="{ ...$attrs, ...forwardedProps }"
        :id="rootContext.contentId"
        :ref="forwardRef"
        :data-state="rootContext.open.value ? 'open' : 'closed'"
        :style="{
          // flex layout so we can place the scroll buttons properly
          display: 'flex',
          flexDirection: 'column',
          // reset the outline by default as the content MAY get focused
          outline: 'none',
          ...(position === 'popper' ? popperStyle : {}),
        }"
      >
        <slot />
      </component>
    </DismissableLayer>
  </ListboxContent>
</template>
