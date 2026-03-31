<script lang="ts" setup>
import { momentsConsoleApiClient, momentsCoreApiClient } from "@/api";
import type { Moment, MomentMedia, MomentMediaTypeEnum } from "@/api/generated";
import MediaCard from "@/components/MediaCard.vue";
import { useConsoleTagQueryFetch } from "@/composables/use-tag";
import { IconEye, IconEyeOff, Toast, VButton, VLoading } from "@halo-dev/components";
import type { AttachmentLike } from "@halo-dev/ui-shared";
import { useQueryClient } from "@tanstack/vue-query";
import { cloneDeep } from "es-toolkit";
import { computed, defineAsyncComponent, onMounted, ref, toRaw } from "vue";
import SendMoment from "~icons/ic/sharp-send";
import TablerPhoto from "~icons/tabler/photo";

const TextEditor = defineAsyncComponent({
  loader: () => import("@/components/TextEditor.vue"),
  loadingComponent: VLoading,
});

const props = withDefaults(
  defineProps<{
    moment?: Moment;
  }>(),
  {
    moment: undefined,
  }
);

const emit = defineEmits<{
  (event: "update"): void;
  (event: "cancel"): void;
}>();

const queryClient = useQueryClient();

const initMoment: Moment = {
  spec: {
    content: {
      raw: "",
      html: "",
      medium: [],
    },
    releaseTime: new Date().toISOString(),
    owner: "",
    // @unocss-skip-start
    visible: "PUBLIC",
    // @unocss-skip-end
    tags: [],
    approved: true,
  },
  metadata: {
    generateName: "moment-",
    name: "",
  },
  kind: "Moment",
  apiVersion: "moment.halo.run/v1alpha1",
};

onMounted(() => {
  if (props.moment) {
    formState.value = cloneDeep(props.moment);
  }
});

const formState = ref<Moment>(cloneDeep(initMoment));
const saving = ref<boolean>(false);
const attachmentSelectorModal = ref(false);
const isUpdateMode = computed(() => !!formState.value.metadata.creationTimestamp);
const isEditorEmpty = ref<boolean>(true);
const handlerCreateOrUpdateMoment = async () => {
  if (saveDisable.value) {
    return;
  }
  try {
    saving.value = true;
    queryEditorTags();
    if (isUpdateMode.value) {
      handleUpdate();
    } else {
      handleSave(formState.value);
      handleReset();
    }
  } catch (error) {
    console.error(error);
  } finally {
    saving.value = false;
  }
};

const handleSave = async (moment: Moment) => {
  moment.spec.releaseTime = new Date().toISOString();
  moment.spec.approved = true;

  await momentsConsoleApiClient.moment.createMoment({
    moment: moment,
  });

  queryClient.invalidateQueries({ queryKey: ["plugin:moments:list"] });

  Toast.success("发布成功");
};

const handleUpdate = async () => {
  await momentsCoreApiClient.moment.patchMoment({
    name: formState.value.metadata.name,
    jsonPatchInner: [
      {
        op: "add",
        path: "/spec/tags",
        value: formState.value.spec.tags || [],
      },
      {
        op: "add",
        path: "/spec/content",
        value: formState.value.spec.content,
      },
      {
        op: "add",
        path: "/spec/visible",
        value: formState.value.spec.visible || false,
      },
    ],
  });

  emit("update");

  queryClient.invalidateQueries({ queryKey: ["plugin:moments:list"] });

  Toast.success("发布成功");
};

const parse = new DOMParser();
const queryEditorTags = function () {
  let tags: Set<string> = new Set();
  let document: Document = parse.parseFromString(formState.value.spec.content.raw!, "text/html");
  let nodeList: NodeList = document.querySelectorAll("a.tag");
  if (nodeList) {
    for (let tagNode of nodeList) {
      if (tagNode.textContent) {
        tags.add(tagNode.textContent);
      }
    }
  }
  formState.value.spec.tags = Array.from(tags);
};

const handleReset = () => {
  formState.value = toRaw(cloneDeep(initMoment));
  isEditorEmpty.value = true;
};

const supportImageTypes: string[] = [
  "image/apng",
  "image/avif",
  "image/bmp",
  "image/gif",
  "image/x-icon",
  "image/jpg",
  "image/jpeg",
  "image/png",
  "image/svg+xml",
  "image/tiff",
  "image/webp",
];

const supportVideoTypes: string[] = ["video/*"];

const supportAudioTypes: string[] = ["audio/*"];

const accepts = [...supportImageTypes, ...supportVideoTypes, ...supportAudioTypes];

const mediumWhitelist: Map<string, MomentMediaTypeEnum> = new Map([
  ["image", "PHOTO"],
  ["video", "VIDEO"],
  ["audio", "AUDIO"],
]);

const onAttachmentsSelect = async (attachments: AttachmentLike[]) => {
  const medias: {
    url: string;
    cover?: string;
    displayName?: string;
    type?: string;
  }[] = attachments
    .map((attachment) => {
      if (typeof attachment === "string") {
        return {
          url: attachment,
          cover: attachment,
        };
      }
      if ("url" in attachment) {
        return {
          url: attachment.url,
          cover: attachment.url,
        };
      }
      if ("spec" in attachment) {
        return {
          url: attachment.status?.permalink,
          cover: attachment.status?.permalink,
          displayName: attachment.spec.displayName,
          type: attachment.spec.mediaType,
        };
      }
    })
    .filter(Boolean) as {
    url: string;
    cover?: string;
    displayName?: string;
    type?: string;
  }[];
  if (!formState.value.spec.content.medium) {
    formState.value.spec.content.medium = [];
  }
  medias.forEach((media) => {
    if (!addMediumVerify(media)) {
      return false;
    }
    if (!media.type) {
      return false;
    }
    let fileType = media.type.split("/")[0];
    formState.value.spec.content.medium?.push({
      type: mediumWhitelist.get(fileType),
      url: media.url,
      originType: media.type,
    } as MomentMedia);
  });
};

const saveDisable = computed(() => {
  let medium = formState.value.spec.content.medium;
  if (medium !== undefined && medium.length > 0 && medium.length <= 9) {
    return false;
  }
  if (!isEditorEmpty.value) {
    return false;
  }

  if (isUpdateMode.value) {
    let oldVisible = props.moment?.spec.visible;
    if (oldVisible != formState.value.spec.visible) {
      return false;
    }
  }

  return true;
});

const removeMedium = (media: MomentMedia) => {
  let formMedium = formState.value.spec.content.medium;
  if (!formMedium) {
    return;
  }
  let index: number = formMedium.indexOf(media);
  if (index > -1) {
    formMedium.splice(index, 1);
  }
};

const handlerCancel = () => {
  emit("cancel");
};

const uploadMediumNum = 9;

const addMediumVerify = (media?: {
  url: string;
  cover?: string;
  displayName?: string;
  type?: string;
}) => {
  let formMedium = formState.value.spec.content.medium;
  if (!formMedium || formMedium.length == 0) {
    return true;
  }

  if (formMedium.length >= uploadMediumNum) {
    Toast.warning("最多允许添加 " + uploadMediumNum + " 个附件");
    return false;
  }

  if (media) {
    if (formState.value.spec.content.medium?.filter((item) => item.url == media.url).length != 0) {
      Toast.warning("已过滤重复添加的附件");
      return false;
    }
  }

  return true;
};

function handleToggleVisible() {
  // @unocss-skip-start
  const { visible: currentVisible } = formState.value.spec;
  // @unocss-skip-end
  formState.value.spec.visible = currentVisible === "PUBLIC" ? "PRIVATE" : "PUBLIC";
}

function handleKeydown(event: KeyboardEvent) {
  if (event.ctrlKey && event.key === "Enter") {
    handlerCreateOrUpdateMoment();
    return false;
  }
}
</script>

<template>
  <div class=":uno: moment-composer card shrink overflow-hidden">
    <AttachmentSelectorModal
      v-model:visible="attachmentSelectorModal"
      v-permission="['system:attachments:view']"
      :min="1"
      :max="9"
      :accepts="accepts"
      @select="onAttachmentsSelect"
    />
    <div class=":uno: moment-composer-shell">
      <div class=":uno: moment-composer-header">
        <div class=":uno: moment-composer-copy">
          <span class=":uno: moment-composer-kicker">
            {{ isUpdateMode ? "EDIT MOMENT" : "NEW MOMENT" }}
          </span>
          <strong>{{ isUpdateMode ? "润色一下这条瞬间" : "像朋友圈一样，记录此刻的小事" }}</strong>
          <p>{{ formState.spec.content.medium?.length || 0 }}/9 个附件 · Ctrl + Enter 快速发布</p>
        </div>
        <button
          v-tooltip="{
            content: formState.spec.visible === 'PRIVATE' ? '仅自己可见' : '所有访客可见',
          }"
          type="button"
          class=":uno: moment-visibility-toggle"
          :class="formState.spec.visible === 'PRIVATE' ? ':uno: is-private' : ':uno: is-public'"
          @click="handleToggleVisible()"
        >
          <IconEyeOff v-if="formState.spec.visible === 'PRIVATE'" class=":uno: text-base" />
          <IconEye v-else class=":uno: text-base" />
          <span>{{ formState.spec.visible === "PRIVATE" ? "仅自己可见" : "公开可见" }}</span>
        </button>
      </div>
      <div class=":uno: moment-composer-editor-wrap">
    <TextEditor
      v-model:raw="formState.spec.content.raw"
      v-model:html="formState.spec.content.html"
      v-model:isEmpty="isEditorEmpty"
      :tag-query-fetch="useConsoleTagQueryFetch"
      class=":uno: moment-composer-editor min-h-[11rem]"
      tabindex="-1"
      @keydown="handleKeydown"
    />
      </div>
    <div v-if="formState.spec.content.medium?.length" class=":uno: moment-composer-media img-box flex">
      <ul class=":uno: grid grid-cols-3 w-full gap-2 sm:w-[25rem]" role="list">
        <li
          v-for="(media, index) in formState.spec.content.medium"
          :key="index"
          class=":uno: moment-composer-media-item inline-block overflow-hidden"
        >
          <MediaCard :media="media" @remove="removeMedium"></MediaCard>
        </li>
      </ul>
    </div>
    <div class=":uno: moment-composer-footer">
      <button
        type="button"
        class=":uno: moment-attach-button"
        @click="addMediumVerify() && (attachmentSelectorModal = true)"
      >
        <TablerPhoto class=":uno: moment-attach-icon" />
        <span>添加图片 / 视频 / 音频</span>
      </button>

      <div class=":uno: moment-composer-actions">
        <div
          v-tooltip="{
            content: formState.spec.visible === 'PRIVATE' ? `私有访问` : '公开访问',
          }"
          class=":uno: group flex cursor-pointer items-center justify-center rounded-full p-2"
          :class="
            formState.spec.visible === 'PRIVATE'
              ? ':uno: hover:bg-red-600/10'
              : ':uno: hover:bg-green-600/10'
          "
          @click="handleToggleVisible()"
        >
          <IconEyeOff
            v-if="formState.spec.visible === 'PRIVATE'"
            class=":uno: size-full text-base text-gray-600 group-hover:text-red-600"
          />
          <IconEye
            v-else
            class=":uno: size-full text-base text-gray-600 group-hover:text-green-600"
          />
        </div>

        <button
          v-if="isUpdateMode"
          type="button"
          class=":uno: moment-secondary-action"
          @click="handlerCancel"
        >
          <span class=":uno: text-xs"> 取消 </span>
        </button>

        <div
          v-permission="['plugin:moments:manage', 'uc:plugin:moments:publish']"
          class=":uno: h-fit"
        >
          <VButton
            v-model:disabled="saveDisable"
            :loading="saving"
            size="sm"
            type="primary"
            class=":uno: moment-primary-action"
            @click="handlerCreateOrUpdateMoment"
          >
            <template #icon>
              <SendMoment class=":uno: size-full scale-[1.1]" />
            </template>
            {{ isUpdateMode ? "保存瞬间" : "发布瞬间" }}
          </VButton>
        </div>
      </div>
    </div>
    </div>
  </div>
</template>
