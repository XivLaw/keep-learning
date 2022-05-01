# 1 plugins

## 1.1 docs 插件
`路径：plugin/docs`

该插件用于将 `components/*` 下的 `demo` 文件夹内的文件转化为对应的样例，并将 `<docs/>` 部分与其他 vue 文件的标签进行分离，最后将文件转化为 markdown，样例如下

`components/affix/demo/basic.vue`

```html

<docs>
---
order: 0
title:
  zh-CN: 基本
  en-US: Basic
---

## zh-CN

最简单的用法。

## en-US

The simplest usage.

</docs>

<template>
  <a-affix :offset-top="top">
    <a-button type="primary" @click="top += 10">Affix top</a-button>
  </a-affix>
  <br />
  <a-affix :offset-bottom="bottom">
    <a-button type="primary" @click="bottom += 10">Affix bottom</a-button>
  </a-affix>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue';
export default defineComponent({
  setup() {
    const top = ref<number>(10);
    const bottom = ref<number>(10);
    return {
      top,
      bottom,
    };
  },
});
</script>

```

### 源码解析
`路径：plugin/docs/vueToMarkdown.ts`

```typescript
import path from 'path';
import LRUCache from 'lru-cache';
import slash from 'slash';
// 引入 fetchCode 工具类，用于拆分对应标签的代码
import fetchCode from '../md/utils/fetchCode';

// eslint-disable-next-line @typescript-eslint/no-var-requires
const debug = require('debug')('vitepress:md');
const cache = new LRUCache<string, MarkdownCompileResult>({ max: 1024 });

interface MarkdownCompileResult {
  vueSrc: string;
}

// 将组件库内的 demo 文件转化为 markdown 文件
export function createVueToMarkdownRenderFn(root: string = process.cwd()): any {
  return (src: string, file: string): MarkdownCompileResult => {
    const relativePath = slash(path.relative(root, file));

    const cached = cache.get(src);
    if (cached) {
      debug(`[cache hit] ${relativePath}`);
      return cached;
    }
    // 截取 docs 标签，并与其他标签分离
    const start = Date.now();
    const docs = fetchCode(src, 'docs')?.trim();
    const template = fetchCode(src, 'template');
    const script = fetchCode(src, 'script');
    const style = fetchCode(src, 'style');
    // 形成新的 markdown 代码
    const newContent = `${docs}
\`\`\`vue
${template}
${script}
${style}
\`\`\`
`;
    debug(`[render] ${file} in ${Date.now() - start}ms.`);
    const result = {
      vueSrc: newContent?.trim(),
      ignore: !docs,
    };
    cache.set(src, result);
    return result;
  };
}

```

## 1.2 md 插件
`路径：plugin/md`

该插件主要作用是通过扩展 `markdown-it` 插件，实现将 `markdown` 源码转换为 `vue` 代码，从而实现将 `markdown` 作为demo样例，展示到 docs 项目当中。

```typescript
import { createMarkdownToVueRenderFn } from './markdownToVue';
import type { MarkdownOptions } from './markdown/markdown';
import type { Plugin } from 'vite';

interface Options {
  root?: string;
  markdown?: MarkdownOptions;
}

export default (options: Options = {}): Plugin => {
  const { root, markdown } = options;
  const markdownToVue = createMarkdownToVueRenderFn(root, markdown);
  return {
    name: 'mdToVue',
    async transform(code, id) {
      if (id.endsWith('.md')) {
        // transform .md files into vueSrc so plugin-vue can handle it
        return { code: (await markdownToVue(code, id)).vueSrc, map: null };
      }
    },
  };
};

```