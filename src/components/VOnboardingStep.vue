<template>
  <div v-show="show">
    <svg
      style="
        width: 100%;
        height: 100%;
        position: fixed;
        top: 0;
        left: 0;
        opacity: 0.5;
        z-index: var(--v-onboarding-overlay-z, 10);
        pointer-events: none;
      "
    >
      <path :d="path" />
    </svg>
    <div
      ref="stepElement"
      style="position: relative; z-index: var(--v-onboarding-step-z, 20)"
    >
      <slot v-if="step">
        <div class="v-onboarding-item">
          <div class="v-onboarding-item__header">
            <span
              v-if="step.content.title"
              class="v-onboarding-item__header-title"
              >{{ step.content.title }}</span
            >
          </div>
          <p
            v-if="step.content.description"
            class="v-onboarding-item__description"
            v-html="step.content.description"
          ></p>
          <div class="v-onboarding-item__actions">
            <button
              v-if="!isFirst"
              type="button"
              @click="onPrevious"
              class="v-onboarding-btn-secondary"
            >
              Previous
            </button>
            <button
              @click="onNext"
              type="button"
              class="v-onboarding-btn-primary"
            >
              {{ isLast ? "Accept" : "Next" }}
            </button>
          </div>
        </div>
      </slot>
      <div data-popper-arrow />
    </div>
  </div>
</template>
<script lang="ts">
import useGetElement from "../composables/useGetElement";
import useSvgOverlay from "../composables/useSvgOverlay";
import { StepEntity } from "../types/StepEntity";
import { VOnboardingWrapperOptions } from "../types/VOnboardingWrapper";
import { createPopper } from "@popperjs/core";
import merge from "lodash.merge";
import {
  computed,
  ComputedRef,
  defineComponent,
  inject,
  onMounted,
  ref,
  watch
} from "vue";
export default defineComponent({
  name: "VOnboardingStep",
  setup(_, { slots }) {
    const show = ref(false);
    const nextStep = inject("next-step", () => {});
    const previousStep = inject("previous-step", () => {});
    const exit = inject("exit", () => {});
    const options = inject<ComputedRef<VOnboardingWrapperOptions>>("options");
    const mergedOptions = computed(() =>
      merge({}, options?.value, step.value.options)
    );
    const isFirst = inject<ComputedRef<boolean>>("is-first-step");
    const isLast = inject<ComputedRef<boolean>>("is-last-step");
    const step = inject<ComputedRef<StepEntity>>(
      "step",
      {} as ComputedRef<StepEntity>
    );

    const onNext = () => {
      beforeStepEnd();
      nextStep();
    };
    const onPrevious = () => {
      beforeStepEnd();
      previousStep();
    };
    watch(
      () => step.value,
      (_, oldStep) => {
        if (!slots.default?.()) return;
        beforeStepEnd(oldStep);
      }
    );
    const stepElement = ref<HTMLElement | null>(null);
    const { updatePath, path } = useSvgOverlay();

    const attachElement = () => {
      const element = useGetElement(step?.value?.attachTo?.element);
      if (element && stepElement.value) {
        show.value = true;
        if (mergedOptions.value?.scrollToStep?.enabled) {
          element.scrollIntoView(mergedOptions.value?.scrollToStep?.options);
        }
        createPopper(element, stepElement.value, mergedOptions.value.popper);
        if (mergedOptions.value?.overlay?.enabled) {
          updatePath(element, {
            padding: mergedOptions.value?.overlay?.padding,
            borderRadius: mergedOptions.value?.overlay?.borderRadius
          });
        }
        setTargetElementClassName(element);
      }
    };
    const beforeStepStart = async () => {
      await step?.value?.on?.beforeStep?.();
      attachElement();
    };
    const beforeStepEnd = (stepObj = step.value) => {
      stepObj?.on?.afterStep?.();
      unsetTargetElementClassName(
        useGetElement(stepObj?.attachTo?.element),
        stepObj.attachTo?.classList
      );
    };

    const setTargetElementClassName = (
      element = useGetElement(step.value.attachTo.element)
    ) => {
      const classList = step.value.attachTo.classList;
      if (!classList || !element) return;
      element.classList.add(...classList);
    };
    const unsetTargetElementClassName = (
      element = useGetElement(step.value.attachTo.element),
      classList?: string[]
    ) => {
      if (!classList || !element) return;
      element.classList.remove(...classList);
    };
    onMounted(beforeStepStart);
    return {
      stepElement,
      onNext,
      onPrevious,
      path,
      show,
      step,
      isFirst,
      isLast,
      exit
    };
  }
});
</script>
