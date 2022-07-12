<template>
  <div class="wrapper">
    <div class="group">
      <label for="inputTextArea">Input:</label>
      <textarea
        v-model="inputConfig"
        name="inputConfig"
        id="inputTextArea"
        :class="{
          valid: state === 'valid',
          invalid: state === 'invalid',
        }"
      ></textarea>
    </div>
    <div class="group">
      <label for="selectOutputFormat">Output color format:</label>
      <select v-model="conversion" name="conversion" id="selectOutputFormat">
        <option value="toHex">Hex</option>
        <option value="toRgbString">Rgb</option>
        <option value="toHslString">Hsl</option>
        <option value="toCmykString">Cmyk</option>
        <option value="toHwbString">Hwb</option>
        <option value="toLchString">Lch</option>
      </select>
      <label for="outputTextArea">Output CSS:</label>
      <textarea
        v-model="outputCSS"
        name="outputCSS"
        id="outputTextArea"
        readonly
      ></textarea>
      <label for="outputTextArea">Output config:</label>
      <textarea
        v-model="outputConfig"
        name="output"
        id="outputTextArea"
        readonly
      ></textarea>
    </div>
  </div>
</template>

<script setup>
import { colord, extend, getFormat } from 'colord';
import cmykPlugin from 'colord/plugins/cmyk';
import hwbPlugin from 'colord/plugins/hwb';
import lchPlugin from 'colord/plugins/lch';

extend([cmykPlugin, hwbPlugin, lchPlugin]);

const inputConfig = ref('');
const outputCSS = ref('');
const outputConfig = ref('');
const conversion = ref('toHex');
const state = computed(() => {
  if (!inputConfig.value.length) return 'clean';
  try {
    JSON.parse(inputConfig.value);
    return 'valid';
  } catch {
    return 'invalid';
  }
});

const convertToCss = (input) => `:root {
  ${Object.entries(input)
    .map(([key, value]) => `--${key}: ${value};`)
    .join('\n  ')}
}`;

const getVariableKey = (str, prefix) => {
  if (prefix) {
    return `var(--${[prefix, str].join('-')})`;
  } else {
    return `var(--${str})`;
  }
};

const convertConfig = (input, prefix) =>
  Object.entries(input).reduce((acc, [groupKey, groupValue]) => {
    if (typeof groupValue === 'string') {
      return { ...acc, [groupKey]: getVariableKey(groupKey, prefix) };
    }

    return {
      ...acc,
      [groupKey]: convertConfig(groupValue, groupKey),
    };
  }, {});

const convert = (input) =>
  Object.entries(input).reduce((acc, [key, value]) => {
    const converted = getFormat(value)
      ? colord(value)[conversion.value]()
      : value;
    return { ...acc, [key]: converted };
  }, {});

const normalize = (input, prefix) =>
  Object.entries(input).reduce((acc, [groupKey, groupValue]) => {
    if (typeof groupValue === 'string') {
      if (prefix) {
        return { ...acc, [[prefix, groupKey].join('-')]: groupValue };
      }
      return { ...acc, [groupKey]: groupValue };
    }

    return { ...acc, ...normalize(groupValue, groupKey) };
  }, {});

const run = (input) => {
  if (!input.length) return;
  try {
    const inputAsObj = JSON.parse(input);
    const normalized = normalize(inputAsObj);
    const converted = convert(normalized);
    const css = convertToCss(converted);
    const config = JSON.stringify(convertConfig(inputAsObj), null, 2);

    console.log({ css, config });

    outputCSS.value = css;
    outputConfig.value = config;

    return css;
  } catch (e) {
    console.error(e);
    return 'Invalid JSON';
  }
};

watch(inputConfig, (value) => {
  run(value);
});

watch(conversion, (value) => {
  run(inputConfig.value);
});

// watch();
</script>

<style>
html {
  font-family: sans-serif;
}

body {
  margin: 0;
  padding: 0;
}

.wrapper {
  padding: 1.5rem;
  min-block-size: calc(100vh - 3rem);
  display: flex;
  gap: 1.5rem;
}

.group {
  display: flex;
  flex-grow: 1;
  flex-direction: column;
  gap: 0.75rem;
}

textarea {
  font-family: monospace;
  resize: none;
  flex-grow: 1;
}

textarea.invalid {
  outline: 2px dashed red;
  border: 0;
}

textarea.valid {
  outline: 2px solid lime;
  border: 0;
}
</style>
