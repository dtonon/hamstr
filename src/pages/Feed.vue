<template>
  <q-page>
    <PageHeader logo class="page-header">
      <template #addon>
        <q-btn-toggle
          v-if="availableFeeds.length > 1"
          v-model="activeFeed"
          rounded
          toggle-color="primary"
          size="sm"
          class="feed-selector"
          :options="[
            {value: 'following', icon: 'group'},
            {value: 'global', icon: 'public'},
          ]"
        />
      </template>
    </PageHeader>

    <div class="post-editor" v-if="app.isSignedIn">
      <PostEditor />
    </div>

    <q-tab-panels v-model="activeFeed" keep-alive animated>
      <q-tab-panel v-for="feed in availableFeeds" :key="feed" :name="feed">
        <Feed :feed="feedDef(feed)" :ref="feed" @load="onFeedLoaded" />
      </q-tab-panel>
    </q-tab-panels>
  </q-page>
</template>

<script>
import {defineComponent} from 'vue'
import PageHeader from 'components/PageHeader.vue'
import PostEditor from 'components/CreatePost/PostEditor.vue'
import Feed from 'components/Feed/Feed.vue'
import {useAppStore} from 'stores/App'
import {useNostrStore} from 'src/nostr/NostrStore'
import {EventKind} from 'src/nostr/model/Event'
import {NoteOrder, useNoteStore} from 'src/nostr/store/NoteStore'

const ZERO_PUBKEY = '0000000000000000000000000000000000000000000000000000000000000000'

const myContacts = () => {
  const app = useAppStore()
  const nostr = useNostrStore()
  const contacts = nostr.getContacts(app.myPubkey)
  return contacts?.map(contact => contact.pubkey).concat(app.myPubkey)
}

const Feeds = {
  global: {
    name: 'global',
    filters: {
      kinds: [EventKind.NOTE], // TODO Deletions
      limit: 20,
    },
    hideBots: true,
  },
  following: {
    name: 'following',
    filters: () => {
      let authors = myContacts()
      if (!authors || !authors.length) authors = [ZERO_PUBKEY]
      return {
        kinds: [EventKind.NOTE],
        authors,
        limit: 50,
      }
    },
    data: () => {
      let authors = myContacts()
      if (!authors || !authors.length) return []
      let notes = []
      const store = useNoteStore()
      for (const author of authors) {
        notes = notes.concat(store.postsByAuthor(author, NoteOrder.CREATION_DATE_DESC))
      }
      return notes
    },
    hideBots: false,
  },
}

export default defineComponent({
  name: 'FeedPage',
  components: {
    PageHeader,
    PostEditor,
    Feed,
  },
  setup() {
    return {
      app: useAppStore(),
      nostr: useNostrStore(),
    }
  },
  data() {
    return {
      feeds: {},
      activeFeed: this.app.isSignedIn ? 'following' : 'global',
      initialized: false,
    }
  },
  computed: {
    availableFeeds() {
      const feeds = []
      if (this.app.isSignedIn) feeds.push('following')
      feeds.push('global')
      return feeds
    },
    contacts() {
      if (!this.app.isSignedIn) return
      return this.nostr.getContacts(this.app.myPubkey)
    },
  },
  methods: {
    feedDef(feed) {
      return Feeds[feed]
    },
    initFeed() {
      if (this.initialized) return
      if (!this.contacts) return
      this.initialized = true
      this.activeFeed = this.contacts?.length
        ? 'following'
        : 'global'
    },
    onFeedLoaded(feed) {
      if (this.activeFeed === 'following'
        && feed?.name === this.activeFeed
        && !this.contacts?.length
        && !this.initialized
      ) {
        this.activeFeed = 'global'
        this.initialized = true
      }
    },
  },
  watch: {
    contacts() {
      this.initFeed()
      this.$refs.following?.[0]?.reload()
    },
  },
  mounted() {
    this.initFeed()
  }
})
</script>

<style lang="scss" scoped>
@import "assets/theme/colors.scss";
@import "assets/variables.scss";

.feed-selector {
  background-color: rgba($color: $color-dark-gray, $alpha: 0.2);
  //box-shadow: none;
}

@media screen and (max-width: $phone) {
  .post-editor {
    display: none;
  }
}
</style>
<style lang="scss">
@import "assets/theme/colors.scss";

.q-tab-panels {
  background: unset;
}
.q-tab-panel {
  padding: 0;
}
.q-panel.scroll {
  overflow: inherit;
}

.feed-selector {
  button:first-child {
    padding: 4px 10px 4px 12px;
  }
  button:last-child {
    padding: 4px 12px 4px 10px;
  }
}
</style>
