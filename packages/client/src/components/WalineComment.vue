<template>
  <div data-waline>
    <Reaction />

    <CommentBox v-if="!reply" @submit="onSubmit" />

    <div class="wl-meta-head">
      <div class="wl-count">
        <span v-if="count" class="wl-num" v-text="count" />
        {{ i18n.comment }}
      </div>

      <ul class="wl-sort">
        <li
          v-for="item in sortingMethods"
          :key="item"
          :class="[item === commentSorting ? 'active' : '']"
          @click="() => onSortByChange(item)"
        >
          {{ i18n[item] }}
        </li>
      </ul>
    </div>

    <div class="wl-cards">
      <CommentCard
        v-for="comment in data"
        :key="comment.objectId"
        :root-id="comment.objectId"
        :comment="comment"
        :reply="reply"
        :edit="edit"
        @reply="onReply"
        @edit="onEdit"
        @submit="onSubmit"
        @status="onStatusChange"
        @delete="onDelete"
        @sticky="onSticky"
        @like="onLike"
      />
    </div>

    <div v-if="status === 'error'" class="wl-operation">
      <button
        type="button"
        class="wl-btn"
        @click="refresh"
        v-text="i18n.refresh"
      />
    </div>

    <template v-else>
      <div v-if="status === 'loading'" class="wl-loading">
        <LoadingIcon :size="30" />
      </div>

      <div v-else-if="!data.length" class="wl-empty" v-text="i18n.sofa" />

      <!-- Load more button -->
      <div v-else-if="page < totalPages" class="wl-operation">
        <button
          type="button"
          class="wl-btn"
          @click="loadMore"
          v-text="i18n.more"
        />
      </div>
    </template>

    <!-- Copyright Information -->
    <div v-if="config.copyright" class="wl-power">
      Powered by
      <a
        href="https://github.com/walinejs/waline"
        target="_blank"
        rel="noopener noreferrer"
      >
        Waline
      </a>
      v{{ version }}
    </div>
  </div>
</template>

<script setup lang="ts">
/* eslint-disable vue/no-unused-properties */
import { useStyleTag } from '@vueuse/core';
import { computed, onMounted, onUnmounted, provide, ref, watch } from 'vue';
import Reaction from './ArticleReaction.vue';
import CommentBox from './CommentBox.vue';
import CommentCard from './CommentCard.vue';
import { LoadingIcon } from './Icons.js';
import { deleteComment, getComment, updateComment } from '../api/index.js';
import { useUserInfo, useLikeStorage } from '../composables/index.js';
import { getConfig, getDarkStyle } from '../utils/index.js';
import { version } from '../version.js';

import type {
  WalineComment,
  WalineCommentStatus,
  WalineLocale,
} from '../typings/index.js';

type SortKey = 'insertedAt_desc' | 'insertedAt_asc' | 'like_desc';

// TODO: Currently complex types and type imports from other files are not supported. It is possible to support type imports in the future.
// @see https://vuejs.org/api/sfc-script-setup.html#typescript-only-features
// this issue may be addressed in Vue3.3
// the following line is the same as WalineProps

type WalineCommentSorting = 'latest' | 'oldest' | 'hottest';

type WalineEmojiPresets =
  | `//${string}`
  | `http://${string}`
  | `https://${string}`;

interface WalineEmojiInfo {
  /**
   * ??????????????? Emoji ??????
   *
   * Emoji name show on tab
   */
  name: string;
  /**
   * ?????????????????????
   *
   * Current folder link
   */
  folder?: string;
  /**
   * Emoji ??????????????????
   *
   * Common prefix of Emoji icons
   */
  prefix?: string;
  /**
   * Emoji ????????????????????????????????????????????????
   *
   * Type of Emoji icons, will be regarded as file extension
   */
  type?: string;
  /**
   * ?????????????????? Emoji ??????
   *
   * Emoji icon show on tab
   */
  icon: string;
  /**
   * Emoji ????????????
   *
   * Emoji image list
   */
  items: string[];
}

type WalineLoginStatus = 'enable' | 'disable' | 'force';

interface WalineSearchImageData extends Record<string, unknown> {
  /**
   * ????????????
   *
   * Image link
   */
  src: string;

  /**
   * ????????????
   *
   * @description ??????????????? alt ??????
   *
   * Image title
   *
   * @description Used for alt attribute of image
   */
  title?: string;

  /**
   * ???????????????
   *
   * @description ???????????????????????????????????????????????????????????????????????????
   *
   * Image preview link
   *
   * @description For better loading performance, we will use this thumbnail first in the list
   *
   * @default src
   */
  preview?: string;
}

type WalineSearchResult = WalineSearchImageData[];

interface WalineSearchOptions {
  /**
   * ????????????
   *
   * Search action
   */
  search: (word: string) => Promise<WalineSearchResult>;

  /**
   * ????????????????????????????????????
   *
   * Default result when opening list
   *
   * @default () => search('')
   */
  default?: () => Promise<WalineSearchResult>;

  /**
   * ?????????????????????
   *
   * @description ???????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
   *
   * Fetch more action
   *
   * @description It will be triggered when the list scrolls to the bottom. If your search service supports paging, you should set this to achieve infinite scrolling
   *
   * @default (word) => search(word)
   */
  more?: (word: string, currentCount: number) => Promise<WalineSearchResult>;
}

type WalineMeta = 'nick' | 'mail' | 'link';

type WalineImageUploader = (image: File) => Promise<string>;

type WalineHighlighter = (code: string, lang: string) => string;

type WalineTexRenderer = (blockMode: boolean, tex: string) => string;

const props = defineProps<{
  /**
   * Waline ??????????????????
   *
   * Waline server address url
   */
  serverURL: string;

  /**
   * ?????? _?????????_ ?????????????????????????????? _?????????_ ??????????????????????????? _?????????_ ??????????????????
   *
   * ???????????????????????? `window.location.pathname`
   *
   * Article path id. Used to distinguish different _article pages_ to ensure loading the correct comment list under the _article page_.
   *
   * You can set it to `window.location.pathname`
   */
  path: string;

  /**
   * ?????????????????????
   *
   * `Meta` ?????????: `'nick'`, `'mail'`, `'link'`
   *
   * Reviewer attributes.
   *
   * Optional values for `Meta`: `'nick'`, `'mail'`, `'link'`
   *
   * @default ['nick', 'mail', 'link']
   */
  meta?: WalineMeta[];

  /**
   * ??????**?????????**????????????????????????
   *
   * Set required fields, default anonymous with nickname
   *
   * @default []
   */
  requiredMeta?: WalineMeta[];

  /**
   * ???????????????????????????????????????????????????????????????
   *
   * @more ????????? `0` ????????????
   *
   * Comment word s limit. When a single number is filled in, it 's the maximum number of comment words.
   *
   * @more No limit when set to `0`.
   *
   * @default 0
   */
  wordLimit?: number | [number, number];

  /**
   * ?????????????????????????????????
   *
   * number of pages per page
   *
   * @default 10
   */
  pageSize?: number;

  /**
   * Waline ????????????
   *
   * ?????????:
   *
   * - `'zh'`
   * - `'zh-cn'`
   * - `'zh-CN'`
   * - `'zh-tw'`
   * - `'zh-TW'`
   * - `'en'`
   * - `'en-US'`
   * - `'en-us'`
   * - `'jp'`
   * - `'jp-jp'`
   * - `'jp-JP'`
   * - `'pt-br'`
   * - `'pt-BR'`
   * - `'ru'`
   * - `'ru-ru'`
   * - `'ru-RU'`
   *
   * Display language for waline
   *
   * Optional value:
   *
   * - `'zh'`
   * - `'zh-cn'`
   * - `'zh-CN'`
   * - `'zh-tw'`
   * - `'zh-TW'`
   * - `'en'`
   * - `'en-US'`
   * - `'en-us'`
   * - `'jp'`
   * - `'jp-jp'`
   * - `'jp-JP'`
   * - `'pt-br'`
   * - `'pt-BR'`
   * - `'ru'`
   * - `'ru-ru'`
   * - `'ru-RU'`
   *
   * @default 'zh-CN'
   */
  lang?: string;

  /**
   * ????????? waline ????????????
   *
   * @see [???????????????](https://waline.js.org/client/i18n.html)
   *
   * Custom display language in waline
   *
   * @see [I18n](https://waline.js.org/en/client/i18n.html)
   */
  locale?: Partial<WalineLocale>;

  /**
   * ????????????????????????
   *
   * Sorting method for comment list
   *
   * @default 'latest'
   */
  commentSorting?: WalineCommentSorting;

  /**
   * ??????????????????????????????
   *
   * @more ?????? `'auto'` ????????????????????????????????????????????? CSS ????????????????????????????????????????????????????????????
   *
   * Whether to enable darkmode support
   *
   * @more Setting `'auto'` will display darkmode due to device settings. Filling in CSS selector will enable darkmode only when the selector match waline ancestor nodes.
   */
  dark?: string | boolean;

  /**
   * ???????????????
   *
   * Set Emojis
   *
   * @default ['//unpkg.com/@waline/emojis@1.1.0/weibo']
   */
  emoji?: (WalineEmojiInfo | WalineEmojiPresets)[] | boolean;

  /**
   * ??????????????????
   *
   * Customize Search feature
   *
   * @default true
   */
  search?: WalineSearchOptions | boolean;

  /**
   * ????????????
   *
   * Code highlighting
   *
   * @default true
   */

  highlighter?: WalineHighlighter | boolean;

  /**
   * ?????????????????????????????????????????????????????????
   *
   * ??????????????????????????????????????????
   *
   * Custom image upload callback to manage picture by yourself.
   *
   * We will pass a picture file object when execute it.
   *
   * @default true
   */

  imageUploader?: WalineImageUploader | boolean;

  /**
   * ???????????????????????????????????????????????????
   *
   * Custom math formula parse callback for preview.
   *
   * @default true
   */
  texRenderer?: WalineTexRenderer | boolean;

  /**
   *
   * ??????????????????????????????:
   *
   * - `'enable'`: ???????????? (??????)
   * - `'disable'`: ?????????????????????????????????????????????
   * - `'force'`: ????????????????????????????????????????????????????????????
   *
   * Login mode status, optional values:
   *
   * - `'enable'`: enable login (default)
   * - `'disable'`: Login is disabled, users should fill in information to comment
   * - `'force'`: Forced login, users must login to comment
   *
   * @default 'enable'
   */
  login?: WalineLoginStatus;

  /**
   * ?????????????????????????????????
   *
   * ???????????? Waline?????????????????????????????????
   *
   * Whether show copyright in footer
   *
   * We strongly recommended you to keep it on to support waline
   *
   * @default true
   */
  copyright?: boolean;

  /**
   * recaptcha v3 client key
   */
  recaptchaV3Key?: string;

  /**
   * reaction
   */
  reaction?: string[] | boolean;
}>();

const sortKeyMap: Record<WalineCommentSorting, SortKey> = {
  latest: 'insertedAt_desc',
  oldest: 'insertedAt_asc',
  hottest: 'like_desc',
};
const sortingMethods = Object.keys(sortKeyMap) as WalineCommentSorting[];

const userInfo = useUserInfo();
const likeStorage = useLikeStorage();

const status = ref<'loading' | 'success' | 'error'>('loading');

const count = ref(0);
const page = ref(1);
const totalPages = ref(0);

const config = computed(() => getConfig(props));

// eslint-disable-next-line vue/no-ref-object-destructure
const commentSorting = ref(config.value.commentSorting);

const data = ref<WalineComment[]>([]);
const reply = ref<WalineComment | null>(null);
const edit = ref<WalineComment | null>(null);

const darkmodeStyle = computed(() => getDarkStyle(config.value.dark));

const i18n = computed(() => config.value.locale);

useStyleTag(darkmodeStyle);

let abort: () => void;

const getCommentData = (pageNumber: number): void => {
  const { serverURL, path, pageSize } = config.value;
  const controller = new AbortController();

  status.value = 'loading';

  abort?.();

  getComment({
    serverURL,
    lang: config.value.lang,
    path,
    pageSize,
    sortBy: sortKeyMap[commentSorting.value],
    page: pageNumber,
    signal: controller.signal,
    token: userInfo.value?.token,
  })
    .then((resp) => {
      status.value = 'success';
      count.value = resp.count;
      data.value.push(...resp.data);
      page.value = pageNumber;
      totalPages.value = resp.totalPages;
    })
    .catch((err) => {
      if (err.name !== 'AbortError') {
        console.error(err.message);
        status.value = 'error';
      }
    });

  abort = controller.abort.bind(controller);
};

const loadMore = (): void => getCommentData(page.value + 1);

const refresh = (): void => {
  count.value = 0;
  data.value = [];
  getCommentData(1);
};

const onSortByChange = (item: WalineCommentSorting): void => {
  if (commentSorting.value !== item) {
    commentSorting.value = item;
    refresh();
  }
};

const onReply = (comment: WalineComment | null): void => {
  reply.value = comment;
};

const onEdit = (comment: WalineComment | null): void => {
  edit.value = comment;
};

const onSubmit = (comment: WalineComment): void => {
  if (edit.value) {
    edit.value.comment = comment.comment;
    edit.value.orig = comment.orig;
  } else if (comment.rid) {
    const repliedComment = data.value.find(
      ({ objectId }) => objectId === comment.rid
    );

    if (!repliedComment) return;

    if (!Array.isArray(repliedComment.children)) repliedComment.children = [];

    repliedComment.children.push(comment);
  } else data.value.unshift(comment);
};

const onStatusChange = async ({
  comment,
  status,
}: {
  comment: WalineComment;
  status: WalineCommentStatus;
}): Promise<void> => {
  if (comment.status === status) return;

  const { serverURL, lang } = config.value;

  await updateComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId: comment.objectId,
    status,
  });

  comment.status = status;
};

const onSticky = async (comment: WalineComment): Promise<void> => {
  if (comment.rid) return;

  const { serverURL, lang } = config.value;

  await updateComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId: comment.objectId,
    sticky: comment.sticky ? 0 : 1,
  });

  comment.sticky = !comment.sticky;
};

const onDelete = async ({ objectId }: WalineComment): Promise<void> => {
  if (!confirm('Are you sure you want to delete this comment?')) return;

  const { serverURL, lang } = config.value;

  await deleteComment({
    serverURL,
    lang,
    token: userInfo.value?.token,
    objectId: objectId,
  });

  // delete comment from data
  data.value.some((item, index) => {
    if (item.objectId === objectId) {
      data.value = data.value.filter((_item, i) => i !== index);

      return true;
    }

    return item.children.some((child, childIndex) => {
      if (child.objectId === objectId) {
        data.value[index].children = item.children.filter(
          (_item, i) => i !== childIndex
        );

        return true;
      }

      return false;
    });
  });
};

const onLike = async (comment: WalineComment): Promise<void> => {
  const { serverURL, lang } = config.value;
  const { objectId } = comment;
  const hasLiked = likeStorage.value.includes(objectId);

  await updateComment({
    serverURL,
    lang,
    objectId,
    token: userInfo.value?.token,
    like: !hasLiked,
  });

  if (hasLiked)
    likeStorage.value = likeStorage.value.filter((id) => id !== objectId);
  else {
    likeStorage.value = [...likeStorage.value, objectId];

    if (likeStorage.value.length > 50)
      likeStorage.value = likeStorage.value.slice(-50);
  }

  comment.like = (comment.like || 0) + (hasLiked ? -1 : 1);
};

provide('config', config);

onMounted(() => {
  watch(
    () => [props.serverURL, props.path],
    () => refresh(),
    { immediate: true }
  );
});
onUnmounted(() => abort?.());
</script>
