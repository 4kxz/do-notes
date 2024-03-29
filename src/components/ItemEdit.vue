<script lang="ts">
import { Component, Vue, Watch } from 'vue-property-decorator';
import { dateToString, splitText, joinText } from '@/utils';
import { FontAwesomeIcon as FaIcon } from '@fortawesome/vue-fontawesome';
import { items } from '@/database';
import { Route } from 'vue-router';
import { Throttle } from 'lodash-decorators';
import Item from '@/models/Item';
import ItemCard from '@/components/ItemCard.vue';

@Component({ components: { ItemCard, FaIcon } })
export default class ItemEdit extends Vue {
  id: string | null = null;
  item: Item | null = null;
  backup: Item | null = null;
  synced = false;

  @Watch('$route', { immediate: true })
  onRouteChanged(route: Route) {
    items
      .doc(route.params.id)
      .get()
      .then(snapshot => {
        this.id = snapshot.id;
        this.item = snapshot.data() as Item;
        this.backup = { ...this.item } as Item;
        this.synced = true;
      });
  }

  get text() {
    if (this.item) {
      return joinText(this.item.title, this.item.content);
    }
    return '';
  }

  set text(value: string) {
    if (this.item) {
      this.synced = false;
      const { title, content } = splitText(value);
      this.item.title = title;
      this.item.content = content;
      this.item.format = 'markdown';
      this.upload();
    }
  }

  @Throttle(2500)
  upload() {
    if (this.id && this.item) {
      this.item.updated = new Date();
      items
        .doc(this.id)
        .update(this.item)
        .then(() => {
          this.synced = true;
        });
    }
  }

  updated() {
    // Update textarea size to match content
    const editor = document.getElementById('text-editor');
    if (editor !== null && editor.scrollHeight > editor.clientHeight) {
      editor.style.height = editor.scrollHeight + 'px';
    }
  }

  uploadImage() {
    const file = document.getElementById('file');
    const item = this.item;
    if (item !== null && file !== null) {
      const files = (file as HTMLInputElement).files;
      if (files?.length) {
        const reader = new FileReader();
        reader.readAsDataURL(files[0]);
        reader.onload = () => {
          if (reader.result) {
            item.content = reader.result.toString();
            item.format = 'image';
            this.upload();
          }
        };
      }
    }
  }

  get lastUpdated() {
    return dateToString(this.item?.updated || this.item?.created);
  }

  undo() {
    this.item = { ...this.backup } as Item;
  }
}
</script>

<template>
  <div v-if="item" class="wrapper" v-title="`ItemEdit – ${item.title}`">
    <div class="editor">
      <header>
        <button v-on:click="undo"><fa-icon icon="undo" /> Undo</button>
        <button v-on:click="uploadImage">
          <fa-icon icon="upload" />Upload
        </button>
        <input id="file" type="file" />
      </header>
      <textarea id="text-editor" v-model="text"></textarea>
    </div>
    <div class="viewer">
      <header>
        <span v-if="lastUpdated">Last updated on {{ lastUpdated }}</span>
      </header>
      <item-card
        v-bind:item="item"
        v-bind:root="item.parent == null"
        v-bind:class="[synced ? 'synced' : 'outdated']"
      />
    </div>
  </div>
</template>

<style lang="scss" scoped>
div.wrapper {
  @apply flex flex-1;
  > * {
    @apply flex-1 flex flex-col;
  }
  header {
    @apply bg-highlightbg text-highlightfg;
    > * {
      @apply p-2 inline-block;
    }
  }
}
div.editor {
  @apply bg-highlightbg text-highlightfg;
  button {
    &:hover {
      @apply bg-keybg text-keyfg;
    }
  }
  textarea {
    @apply p-2 font-mono bg-overlaybg text-overlayfg overflow-hidden;
  }
}
div.viewer {
  div.item {
    @apply flex-1;
  }
  div.item.outdated {
    @apply opacity-75;
  }
}
</style>
