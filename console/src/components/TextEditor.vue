<script lang="ts" setup>
import type { useTagQueryFetchProps } from "@/composables/use-tag";
import { TagsExtension } from "@/extensions/tags";
import { consoleApiClient, ucApiClient } from "@halo-dev/api-client";
import { VLoading } from "@halo-dev/components";
import {
  ExtensionsKit,
  RichTextEditor,
  VueEditor,
  type Extensions,
} from "@halo-dev/richtext-editor";
import { utils, type PluginModule } from "@halo-dev/ui-shared";
import type { UseQueryReturnType } from "@tanstack/vue-query";
import type { AxiosRequestConfig } from "axios";
import { onMounted, ref, shallowRef, watch } from "vue";

const props = withDefaults(
  defineProps<{
    html: string;
    raw: string;
    isEmpty: boolean;
    tagQueryFetch: (props: useTagQueryFetchProps) => UseQueryReturnType<unknown, unknown>;
  }>(),
  {
    html: "",
    raw: "",
    isEmpty: true,
  }
);

const emit = defineEmits<{
  (event: "update:raw", value: string): void;
  (event: "update:html", value: string): void;
  (event: "update", value: string): void;
  (event: "update:isEmpty", value: boolean | undefined): void;
}>();

const editor = shallowRef<VueEditor>();

const supportedPluginNames = ["editor-hyperlink-card", "hybrid-edit-block", "shiki"];

const customExtensions = [
  TagsExtension.configure({
    tagQueryFetch: props.tagQueryFetch,
  }),
];

const isInitialized = ref(false);

onMounted(async () => {
  const enabledPlugins = window.enabledPlugins.filter((plugin) =>
    supportedPluginNames.includes(plugin.name)
  );
  const enabledPluginNames = enabledPlugins.map((plugin) => plugin.name);
  const enabledPluginModules: PluginModule[] = enabledPluginNames
    .map((name) => {
      if (window[name as keyof Window]) {
        return window[name as keyof Window];
      }
    })
    .filter(Boolean);

  const extensionsFromPlugins: Extensions = [];

  for (const pluginModule of enabledPluginModules) {
    const callbackFunction = pluginModule?.extensionPoints?.["default:editor:extension:create"];

    if (typeof callbackFunction !== "function") {
      continue;
    }

    const extensions = await callbackFunction();

    extensionsFromPlugins.push(...extensions);
  }

  editor.value = new VueEditor({
    content: props.raw,
    extensions: [
      ExtensionsKit.configure({
        placeholder: {
          placeholder: "有什么想说的吗...",
        },
        image: {
          uploadImage: handleUpload,
        },
        video: {
          uploadVideo: handleUpload,
        },
        audio: {
          uploadAudio: handleUpload,
        },
        gallery: {
          uploadImage: handleUpload,
        },
        customExtensions: [...customExtensions, ...extensionsFromPlugins],
      }),
    ],
    autofocus: "end",
    onUpdate: () => {
      emit("update:raw", editor.value?.getHTML() + "");
      emit("update:html", editor.value?.getHTML() + "");
      emit("update:isEmpty", editor.value?.isEmpty);
      emit("update", editor.value?.getHTML() + "");
    },
    onCreate: () => {
      isInitialized.value = true;
    },
  });
});

async function handleUpload(file: File, options?: AxiosRequestConfig) {
  if (utils.permission.has(["system:attachments:manage"])) {
    const { data } = await consoleApiClient.storage.attachment.uploadAttachmentForConsole(
      {
        file,
      },
      options
    );
    return data;
  } else if (utils.permission.has(["uc:attachments:manage"])) {
    const { data } = await ucApiClient.storage.attachment.uploadAttachmentForUc(
      {
        file,
      },
      options
    );
    return data;
  } else {
    throw new Error("Permission denied");
  }
}

watch(
  () => props.raw,
  () => {
    if (props.raw !== editor.value?.getHTML()) {
      editor.value?.commands.setContent(props.raw);
    }
  }
);
</script>
<template>
  <div class=":uno: halo-moment-editor relative">
    <VLoading v-if="!isInitialized" />
    <RichTextEditor v-else-if="editor" :editor="editor" locale="zh-CN"> </RichTextEditor>
  </div>
</template>

<style lang="scss">
.halo-moment-editor {
  border: 1px solid #e6ebf2;
  border-radius: 24px;
  background:
    radial-gradient(circle at top left, rgba(255, 255, 255, 0.96), rgba(247, 250, 253, 0.96)),
    #fff;
  box-shadow:
    inset 0 1px 0 rgba(255, 255, 255, 0.8),
    0 18px 40px rgba(15, 23, 42, 0.04);
  overflow: hidden;

  .editor-content {
    background: transparent;
  }

  .ProseMirror {
    min-height: 12rem;
    padding: 1.4rem 1.5rem 2rem !important;
    font-size: 1rem;
    line-height: 1.9;
  }
}
</style>
