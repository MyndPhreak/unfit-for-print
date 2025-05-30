<template>
  <div class="card-container w-40 sm:w-48 md:w-56 lg:w-64 xl:w-72 aspect-[3/4]">
    <div
      ref="card"
      class="card cursor-pointer"
      @mousemove="handleMouseMove"
      @mouseleave="resetTransform"
      :class="{ 'card--flipped': flipped, 'card--winner': isWinner }"
    >
      <div class="card__inner">
        <!-- Front Side -->
        <div class="card__face card__front">
          <slot name="front">
            <div class="card-content p-4 xl:text-4xl md:text-3xl sm:text-2xl leading-none">
              <p>{{ cardText }}</p>
              <div class="absolute bottom-0 left-0 m-3 text-xl opacity-10 hover:opacity-50 transition-opacity duration-500">
                <UTooltip :text="`Card ID ` + cardId">
                  <Icon name="mdi:cards"  class="align-middle text-slate-900"/>
                  <span class="text-sm align-middle ml-1 text-slate-900">{{ cardPack }}</span>
                </UTooltip>
              </div>
            </div>
          </slot>
        </div>

        <!-- Back Side -->
        <div class="card__face card__back">
          <slot name="back">
            <div class="card-content">
              <img
                :src="backLogoUrl"
                alt="Card Back Logo"
                class="w-3/4 max-w-[10rem] object-contain opacity-75"
                draggable="false"
              />
            </div>
          </slot>
          <div v-if="shine" class="card__shine" :style="shineStyle"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useAppwrite } from "~/composables/useAppwrite";
const {playSfx, getRandomInRange} = useSfx();
const { vibrate, stop, isSupported } = useVibrate({ pattern: [getRandomInRange([1,3]), 2, getRandomInRange([1,3])] })
import { isMobile } from '@basitcodeenv/vue3-device-detect'
import { useSpeech } from '~/composables/useSpeech'
const {speak} = useSpeech('1SM7GgM6IMuvQlz2BwM3')
function playRandomFlip() {
	vibrate()
	playSfx([
		'/sounds/sfx/flip1.wav',
		'/sounds/sfx/flip2.wav',
		'/sounds/sfx/flip3.wav',
	], { volume: 0.75, pitch: [0.95, 1.05] })
}
const props = defineProps<{
  cardId?: string
  text?: string
  cardPack?: string
  backLogoUrl?: string
  flipped?: boolean
  threeDeffect?: boolean
  shine?: boolean
  maskUrl?: string
  isWinner?: boolean
}>();

const fallbackText = ref('');
const cardText = computed(() => props.text || fallbackText.value);
const cardPack = ref(props.cardPack || null);

const card = ref<HTMLElement | null>(null);
const rotation = ref({ x: 0, y: 0 });
const shineOffset = ref({ x: 0, y: 0 });

function animateShine() {
  const ease = 0.05;
  shineOffset.value.x += (rotation.value.x - shineOffset.value.x) * ease;
  shineOffset.value.y += (rotation.value.y - shineOffset.value.y) * ease;
  requestAnimationFrame(animateShine);
}

const shineStyle = computed(() => {
  const angle = (-shineOffset.value.y + shineOffset.value.x) * 2 + 45;
  const offsetX = -shineOffset.value.y + 50;
  const offsetY = -shineOffset.value.x + 50;
  return {
    background: `
      linear-gradient(
        ${angle}deg,
        transparent,
        red,
        transparent,
        orange,
        transparent,
        yellow,
        transparent,
        green,
        transparent,
        cyan,
        transparent,
        blue,
        transparent,
        violet,
        transparent,
        red
      )
    `,
    backgroundPosition: `${offsetX}% ${offsetY}%`,
    backgroundSize: "500% 500%",
    mixBlendMode: "screen" as "screen",
    WebkitMaskImage: `url(${props.maskUrl})`,
    maskImage: `url(${props.maskUrl})`,
    WebkitMaskRepeat: "no-repeat",
    maskRepeat: "no-repeat",
    WebkitMaskSize: "cover",
    maskSize: "cover",
    WebkitMaskPosition: "center",
    maskPosition: "center",
    opacity: 0.25,
    transition: "background-position 250ms linear, background 250ms linear",
  };
});

function handleMouseMove(e: MouseEvent) {
  if (!card.value) return;
	if (isMobile) return;

  const cardRect = card.value.getBoundingClientRect();
  const x = e.clientX - cardRect.left;
  const y = e.clientY - cardRect.top;
  const centerX = cardRect.width / 2;
  const centerY = cardRect.height / 2;

  const rotateX = ((y - centerY) / centerY) * 15;
  const rotateY = ((centerX - x) / centerX) * 15;

  rotation.value = { x: rotateX, y: rotateY };

  applyTransform(rotateX, rotateY);
}

function applyTransform(rotateX = 0, rotateY = 0) {
  if (!card.value) return;

  const intensity = props.threeDeffect ? 1 : 0.3;
  const flipTransform = props.flipped ? "rotateY(180deg)" : "";
  const tiltTransform = `
    rotateX(${rotateX * intensity}deg)
    rotateY(${rotateY * intensity}deg)
  `;

  card.value.style.transform = `${flipTransform} ${tiltTransform}`;
}

function resetTransform() {
  if (card.value) {
    rotation.value = { x: 0, y: 0 };
    applyTransform(0, 0);
  }
}

watch(
  () => props.flipped,
  (newValue, oldValue) => {
    if (newValue !== oldValue) {
      // Only play when the value actually changes
      playRandomFlip();
    }
  }
);

watch(
  () => props.flipped,
  () => {
    applyTransform(rotation.value.x, rotation.value.y);
  },
  { immediate: true }
);

onMounted(async () => {
  if (!props.text) {
    const { databases } = useAppwrite();
    if (!databases) return;
    const config = useRuntimeConfig();
    const doc = await databases.getDocument(config.public.appwriteDatabaseId, config.public.appwriteWhiteCardCollectionId, props.cardId);
    fallbackText.value = doc.text;
    cardPack.value = doc.pack || null;
  }
  resetTransform();
  animateShine();
});
</script>

<style scoped>
.card-container {
  perspective: 1500px;
  display: flex;
  justify-content: center;
  align-items: center;
  -webkit-touch-callout: none; /* iOS Safari */
  -webkit-user-select: none; /* Safari */
  -moz-user-select: none; /* Old versions of Firefox */
  -ms-user-select: none; /* Internet Explorer/Edge */
  user-select: none;
}
.card-container:hover {
  z-index: 100 !important;
}
.card {
  width: 100%;
  height: 100%;
  background: transparent;
  border-radius: 12px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  transform-style: preserve-3d;
  position: relative;
  transition: transform 0.6s cubic-bezier(0.175, 0.885, 0.32, 1.275),
    box-shadow 0.3s ease;
  will-change: transform;
}

.card:hover {
  box-shadow: 0 16px 32px rgba(0, 0, 0, 0.2);
}

.card--flipped {
  transform: rotateY(180deg);
}

.card__inner {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
}
.card__face {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: "Bebas Neue", sans-serif;
  font-size: 1.25rem;
  text-align: center;
  border-radius: 12px;
  overflow: hidden;
  transform-style: preserve-3d;
  /* Ensure content is positioned correctly */
  z-index: 1;
  outline: 1px solid transparent;
}
.card__face::before {
  content: "";
  position: absolute;
  inset: 0;
  border: 6px solid rgba(0, 0, 0, 0.25);
  pointer-events: none;
  z-index: 10;
  border-radius: 12px;
  -webkit-box-shadow: inset 0 0 100px 0 rgba(0, 0, 0, 0.25);
  -moz-box-shadow: inset 0 0 100px 0 rgba(0, 0, 0, 0.25);
  box-shadow: inset 0 0 100px 0 rgba(0, 0, 0, 0.25);
}

.card__front,
.card__back {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  transform-style: preserve-3d;
  border-radius: 12px;
}
.card__front {
  background-color: #e7e1de;
}
.card__back {
  background-color: #e7e1de;
  transform: rotateY(180deg);
}

/* Ensure shine effect is properly positioned on both sides */
.card__front .card__shine,
.card__back .card__shine {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 12px;
}

/* We'll use backface-visibility instead of display:none to allow 3D effects */
.card__front {
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
}

.card__back {
  backface-visibility: hidden;
  -webkit-backface-visibility: hidden;
  transform: rotateY(180deg);
}
.card__shine {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 100;
  transition: background-position 250ms linear;
  border-radius: 12px;
}

.card-content {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1;
  color: black;
  border-radius: 12px;
}

/* Winner animation styles */
.card--winner {
  animation: winner-pulse 2s ease-in-out;
  box-shadow: 0 0 15px 5px rgba(34, 197, 94, 0.6);
}

@keyframes winner-pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(34, 197, 94, 0.7);
    outline: 0 solid rgba(34, 197, 94, 0);
  }
  50% {
    box-shadow: 0 0 20px 10px rgba(34, 197, 94, 0.8);
    outline: 4px solid rgba(34, 197, 94, 0.8);
  }
  100% {
    box-shadow: 0 0 15px 5px rgba(34, 197, 94, 0.6);
    outline: 2px solid rgba(34, 197, 94, 0.6);
  }
}
</style>
