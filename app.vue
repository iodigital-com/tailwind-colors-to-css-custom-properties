<template>
  <div class="wrapper">
    <div class="group">
      <label for="inputTextArea">Input:</label>
      <textarea
        v-model="input"
        name="input"
        id="inputTextArea"
        :class="{
          valid: state === 'valid',
          invalid: state === 'invalid',
        }"
      ></textarea>
    </div>
    <div class="group">
      <label for="inputTextArea">Output color format:</label>
      <select v-model="conversion" name="conversion" id="">
        <option value="toHex">Hex</option>
        <option value="toRgbString">Rgb</option>
        <option value="toHslString">Hsl</option>
        <option value="toCmykString">Cmyk</option>
        <option value="toHwbString">Hwb</option>
        <option value="toLchString">Lch</option>
      </select>
      <label for="outputTextArea">Output:</label>
      <textarea
        v-model="output"
        name="output"
        id="outputTextArea"
        readonly
      ></textarea>
    </div>
  </div>
</template>

<script setup>
import { colord, extend } from 'colord';
import cmykPlugin from 'colord/plugins/cmyk';
import hwbPlugin from 'colord/plugins/hwb';
import lchPlugin from 'colord/plugins/lch';

extend([cmykPlugin, hwbPlugin, lchPlugin]);

const input = ref('');
const conversion = ref('toHex');
const state = computed(() => {
  if (!input.value.length) return 'clean';
  try {
    JSON.parse(input.value);
    return 'valid';
  } catch {
    return 'invalid';
  }
});

const convertToCss = (input) => `
:root {
  ${Object.entries(input)
    .map(([key, value]) => `--${key}: ${value};`)
    .join('\n  ')}
}
`;

const convert = (input) =>
  Object.entries(input).reduce((acc, [key, value]) => {
    const converted = colord(value)[conversion.value]();
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

const output = computed(() => {
  if (!input.value.length) return '';
  try {
    const inputAsObj = JSON.parse(input.value);
    const normalized = normalize(inputAsObj);
    const converted = convert(normalized);
    const css = convertToCss(converted);
    return css;
  } catch (e) {
    console.error(e);
    return 'Invalid JSON';
  }
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
