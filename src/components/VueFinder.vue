<template>
  <div class="vuefinder" ref="root" tabindex="0">
    <div :class="app.theme.actualValue">
      <div
        :class="
          app.fullScreen
            ? 'vuefinder__main__fixed'
            : 'vuefinder__main__relative'
        "
        :style="
          !app.fullScreen
            ? 'max-height: ' + maxHeight + ';height: ' + maxHeight
            : ''
        "
        class="vuefinder__main__container"
        @mousedown="app.emitter.emit('vf-contextmenu-hide')"
        @touchstart="app.emitter.emit('vf-contextmenu-hide')"
      >
        <Toolbar />
        <Breadcrumb />
        <div class="vuefinder__main__content">
          <TreeView />
          <Explorer />
        </div>
        <Statusbar />
      </div>

      <Transition name="fade">
        <Component v-if="app.modal.visible" :is="app.modal.type" />
      </Transition>

      <ContextMenu />
    </div>
  </div>
</template>

<script setup>
import { inject, onMounted, provide, ref, watch } from "vue";
import ServiceContainer from "../ServiceContainer.js";
import { useHotkeyActions } from "../composables/useHotkeyActions.js";

import Toolbar from "../components/Toolbar.vue";
import Breadcrumb from "../components/Breadcrumb.vue";
import Explorer from "../components/Explorer.vue";
import ContextMenu from "../components/ContextMenu.vue";
import Statusbar from "../components/Statusbar.vue";
import TreeView from "../components/TreeView.vue";
import { menuItems as contextMenuItems } from "../utils/contextmenu.js";

const emit = defineEmits(["select", "update:path"]);

const props = defineProps({
  id: {
    type: String,
    default: "vf",
  },
  request: {
    type: [String, Object],
    required: true,
  },
  persist: {
    type: Boolean,
    default: false,
  },
  path: {
    type: String,
    default: "",
  },
  features: {
    type: [Array, Boolean],
    default: true,
  },
  debug: {
    type: Boolean,
    default: false,
  },
  theme: {
    type: String,
    default: "system",
  },
  locale: {
    type: String,
    default: null,
  },
  maxHeight: {
    type: String,
    default: "600px",
  },
  maxFileSize: {
    type: String,
    default: "10mb",
  },
  fullScreen: {
    type: Boolean,
    default: false,
  },
  showTreeView: {
    type: Boolean,
    default: false,
  },
  pinnedFolders: {
    type: Array,
    default: [],
  },
  showThumbnails: {
    type: Boolean,
    default: true,
  },
  selectButton: {
    type: Object,
    default(rawProps) {
      return {
        active: false,
        multiple: false,
        click: (items) => {
          // items is an array of selected items
          //
        },
        ...rawProps,
      };
    },
  },
  onError: {
    type: Function,
    default: null,
  },
  loadingIndicator: {
    type: String,
    default: "circular",
  },
  contextMenuItems: {
    type: Array,
    default: () => contextMenuItems,
  },
});

// the object is passed to all components as props
const app = ServiceContainer(props, inject("VueFinderOptions"));
provide("ServiceContainer", app);
const { setStore } = app.storage;

//  Define root element
const root = ref(null);
app.root = root;

// Define dragSelect object
const ds = app.dragSelect;

useHotkeyActions(app);

const updateItems = (data) => {
  Object.assign(app.fs.data, data);
  ds.clearSelection();
  ds.refreshSelection();
};

/** @type {AbortController} */
let controller;
app.emitter.on("vf-fetch-abort", () => {
  controller.abort();
  app.fs.loading = false;
});

// Fetch data
app.emitter.on(
  "vf-fetch",
  async ({
    params,
    body = null,
    onSuccess = null,
    onError = null,
    noCloseModal = false,
  }) => {
    if (["index", "search"].includes(params.q)) {
      if (controller) {
        controller.abort();
      }
      app.fs.loading = true;
    }

    controller = new AbortController();
    const signal = controller.signal;

    const token = props.request.getToken
      ? await props.request.getToken()
      : null;

    if (params.q === "download") {
      const baseUrl = props.request.baseUrl;
      const method = params.m || "POST";
      const queryString = new URLSearchParams(params).toString();
      const url = `${baseUrl}?${queryString}`;

      fetch(url, {
        method,
        headers: {
          ...props.request.headers,
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
        body: body
          ? body instanceof FormData
            ? body
            : JSON.stringify(body)
          : null,
        abortSignal: signal,
      })
        .then((response) => {
          const disposition = response.headers.get("Content-Disposition");
          let filename = "folder.zip"; // значение по умолчанию

          if (disposition && disposition.includes("filename=")) {
            const match = disposition.match(/filename="?([^"]+)"?/);
            if (match && match[1]) {
              filename = match[1];
            }
          }

          return response.blob().then((blob) => ({ blob, filename }));
        })
        .then(({ blob, filename }) => {
          const url = window.URL.createObjectURL(blob);
          const a = document.createElement("a");
          a.href = url;
          a.download = filename;
          document.body.appendChild(a); // Firefox fix
          a.click();
          a.remove();
          window.URL.revokeObjectURL(url);
        })
        .catch((err) => {
          console.error("Download error", err);
        });

      return;
    }

    app.requester
      .send({
        url: "",
        method: params.m || "get",
        params,
        body,
        abortSignal: signal,
        headers: {
          ...(props.request.headers || {}),
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
      })
      .then((data) => {
        app.fs.adapter = data.adapter;
        if (app.persist) {
          app.fs.path = data.dirname;
          setStore("path", app.fs.path);
        }

        if (!noCloseModal) {
          app.modal.close();
        }
        updateItems(data);
        if (onSuccess) {
          onSuccess(data);
        }
      })
      .catch((e) => {
        console.error(e);
        if (onError) {
          onError(e);
        }
      })
      .finally(() => {
        if (["index", "search"].includes(params.q)) {
          app.fs.loading = false;
        }
      });
  }
);

app.emitter.on("vf-download", ({ item }) => {
  app.emitter.emit("vf-fetch", {
    params: {
      q: "download",
      m: "post",
      adapter: app.fs.adapter,
      path: app.fs.data.dirname,
    },
    body: {
      path: item.path,
    },
    onSuccess: () => {
      app.emitter.emit("vf-toast-push", { label: t("The folder downloaded.") });
    },
    onError: (e) => {
      onError(e);
    },
  });
});

/**
 * fetchPath fetches the items of the given path
 * if no path is given, the backend should return the root of the current adapter
 * @param path {string | undefined} example: 'media://public'
 */
function fetchPath(path) {
  let pathExists = {};

  if (path && path.includes("://")) {
    pathExists = {
      adapter: path.split("://")[0],
      path: path,
    };
  }

  app.emitter.emit("vf-fetch", {
    params: { q: "index", adapter: app.fs.adapter, ...pathExists },
    onError:
      props.onError ??
      ((e) => {
        if (e.message) {
          app.emitter.emit("vf-toast-push", {
            label: e.message,
            type: "error",
          });
        }
      }),
  });
}

// fetch initial data
onMounted(() => {
  // app.fs.adapter can be null at first, until we get the adapter list it will be the first one from response
  // later we can set default adapter from a prop value

  // if there is a path coming from the prop, we should use it.
  fetchPath(app.fs.path);

  // We re-fetch the data if the path prop is updated
  watch(
    () => props.path,
    (path) => {
      fetchPath(path);
    }
  );

  // Emit select event
  ds.onSelect((items) => {
    emit("select", items);
  });

  // Emit update:path event
  watch(
    () => app.fs.data.dirname,
    (path) => {
      emit("update:path", path);
    }
  );
});
</script>
