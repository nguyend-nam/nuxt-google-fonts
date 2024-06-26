<template>
  <div>
    <Head>
      <Style
        type="text/css"
        :children="
          fontFaces
            .filter(Boolean)
            .map((f) => f.fontFace)
            .join(' ')
        "
      />
    </Head>
    <div v-if="isLoading">
      <div class="px-6 md:px-14 pb-4">
        <div class="w-full rounded-md bg-gray-100 p-4 animate-pulse h-[60px] mt-8" />
        <div class="w-full rounded-md bg-gray-100 p-4 animate-pulse h-[84px] mt-8" />
        <div class="flex flex-col w-full gap-[1px] mt-12">
          <div
            v-for="(_, index) in new Array(5).fill(true)"
            :key="index"
            class="w-full rounded-md bg-gray-100 p-4 animate-pulse h-[132px]"
          />
        </div>
      </div>
    </div>
    <div v-else-if="!!currentFont">
      <div class="px-6 md:px-14 pb-4">
        <div
          class="mt-8 flex flex-col md:flex-row items-start md:items-center gap-4 justify-between"
        >
          <h2 class="text-6xl">
            {{ currentFont.family }}
          </h2>
          <Button
            v-if="selectedStore.isStored(currentFont.family)"
            @click="() => selectedStore.removeSelectedStyle(currentFont?.family || '')"
            >Remove font family</Button
          >
          <Button v-else type="primary" @click="addCurrentFont">Add font family</Button>
        </div>
        <FontDetailStyles
          :preview="fontDetailPreview"
          :font-size="fontSize"
          @input="onPreviewInput"
          @set-font-size="setFontSize"
        />
        <div class="mt-12 border-b border-gray-200">
          <div
            v-for="(variant, index) in variantStyles"
            :key="variant.fontFamily + index"
            class="p-4 border-t w-full border-gray-200"
          >
            <p class="text-sm text-gray-700 mb-2">
              {{
                [
                  getFontWeight(variant.fontWeight),
                  variant.fontWeight,
                  variant.fontStyle === 'italic'
                    ? variant.fontStyle[0].toUpperCase() + variant.fontStyle.slice(1)
                    : '',
                ].join(' ')
              }}
            </p>
            <FontVariantContentEditable
              :preview="fontDetailPreview"
              :style="{ fontSize: `${fontSize}px`, ...variant }"
              @input="onPreviewInput"
            />
          </div>
        </div>
      </div>
    </div>
    <div v-else>
      <div class="px-6 md:px-14 pb-4">
        <h2> Font not found </h2>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { FontItem, FontWeight } from '~/types/fonts.type';
import { convertParamToFamily } from '~/utils/string';
import { FONT_DETAIL_PREVIEW_SENTENCE } from '~/constants/preview';
import { Button } from 'ant-design-vue';
import { useSelected } from '~/stores/selected';

definePageMeta({
  layout: 'font-detail',
});
const { fonts, isLoading } = await useFetchAllFontsV2();
const route = useRoute();
const currentFont = ref<FontItem | null>(null);

watch(
  () => [fonts, route.params],
  () => {
    const currentFontData = fonts.value.find(
      (font) => font.family === convertParamToFamily(route.params.family as string),
    );

    if (currentFontData) {
      currentFont.value = currentFontData;
    }
  },
  { immediate: true, deep: true },
);

const fontFaces = ref<{ fontFace: string }[]>([]);

watch(
  () => [currentFont, fonts],
  () => {
    if (!!currentFont.value && currentFont.value?.variants.length > 0) {
      currentFont.value?.variants.forEach((variant) => {
        const weight = variant === 'regular' ? '400' : variant;
        const surl = currentFont.value?.files[variant];

        if (weight) {
          fontFaces.value.push({
            fontFace: `
@font-face {
  font-family: '${currentFont.value?.family} script=${
    currentFont.value?.subsets.includes('latin') ? 'latin' : currentFont.value?.subsets[0]
  } rev=1';
  font-style: ${weight.includes('italic') ? 'italic' : 'normal'};
  font-weight: ${weight.includes('italic') ? weight.slice(0, 3) : weight};
  font-display: block;
  src: url(${surl}) format('woff2');
  }
`,
          });
        }
      });
    }
  },
  { immediate: true, deep: true },
);

const variantStyles: {
  fontFamily: string;
  fontWeight: FontWeight;
  fontStyle: string;
  fontStretch: string;
  lineHeight: string;
  fontCategory: string;
}[] = [];

watch(
  () => [currentFont, fonts],
  () => {
    if (currentFont.value) {
      const { subsets, family } = currentFont.value;
      for (let i = 0; i < currentFont.value.variants.length; i++) {
        const fontDetails = currentFont.value;

        if (fontDetails.variants[i].includes('italic')) {
          const props = {
            fontFamily: `"${family} script=${subsets.includes('latin') ? 'latin' : subsets[0]} rev=1"`,
            fontWeight: `${
              fontDetails.variants[i].slice(0, 3) === 'ita'
                ? '400'
                : fontDetails.variants[i].slice(0, 3)
            }` as FontWeight,
            fontStyle: `italic`,
            fontStretch: `normal`,
            lineHeight: `initial`,
            fontCategory: fontDetails.category,
          };
          variantStyles.push(props);
        } else {
          const props = {
            fontFamily: `"${family} script=${subsets.includes('latin') ? 'latin' : subsets[0]} rev=1"`,
            fontWeight: `${
              fontDetails.variants[i] === 'regular' ? '400' : fontDetails.variants[i]
            }` as FontWeight,
            fontStyle: `regular`,
            fontStretch: `normal`,
            lineHeight: `initial`,
            fontCategory: fontDetails.category,
          };
          variantStyles.push(props);
        }
      }
    }
  },
  { immediate: true, deep: true },
);

const fontDetailPreview = ref<string>(FONT_DETAIL_PREVIEW_SENTENCE);

const onPreviewInput = (value: string) => {
  fontDetailPreview.value = value;
};

const fontSize = ref<number>(48);

const setFontSize = (value?: number) => {
  if (value) {
    fontSize.value = value;
  }
};

const selectedStore = useSelected();

const addCurrentFont = () => {
  if (currentFont.value) {
    selectedStore.addSelectedStyle(
      currentFont.value.family,
      convertFontStyles(currentFont.value.variants || []),
    );
  }
};

onUnmounted(() => {
  fontFaces.value = [];
});
</script>

<style scoped></style>
