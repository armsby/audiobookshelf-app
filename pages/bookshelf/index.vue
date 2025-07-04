<template>
  <div class="w-full h-full overflow-y-auto relative">
    <div v-if="attemptingConnection" class="w-full pt-4 flex items-center justify-center">
      <widgets-loading-spinner />
      <p class="pl-4">{{ $strings.MessageAttemptingServerConnection }}</p>
    </div>
    <div v-if="shelves.length && isLoading" class="w-full pt-4 flex items-center justify-center">
      <widgets-loading-spinner />
      <p class="pl-4">{{ $strings.MessageLoadingServerData }}</p>
    </div>

    <div class="w-full pb-4" :class="{ 'py-6': altViewEnabled }">
      <template v-for="(shelf, index) in shelves">
        <bookshelf-shelf :key="shelf.id" :label="getShelfLabel(shelf)" :entities="shelf.entities" :type="shelf.type" :style="{ zIndex: shelves.length - index }" />
      </template>
    </div>

    <div v-if="!shelves.length && !isLoading" class="absolute top-0 left-0 w-full h-full flex items-center justify-center">
      <div>
        <p class="mb-4 text-center text-xl">
          {{ $strings.MessageBookshelfEmpty }}
        </p>
        <div class="w-full" v-if="!user">
          <div class="flex justify-center items-center mb-3">
            <span class="material-symbols text-error text-lg">cloud_off</span>
            <p class="pl-2 text-error text-sm">{{ $strings.MessageAudiobookshelfServerNotConnected }}</p>
          </div>
        </div>
        <div class="flex justify-center">
          <ui-btn v-if="!user" small @click="$router.push('/connect')" class="w-32">{{ $strings.ButtonConnect }}</ui-btn>
        </div>
      </div>
    </div>
    <div v-else-if="!shelves.length && isLoading && !attemptingConnection" class="absolute top-0 left-0 z-50 w-full h-full flex items-center justify-center">
      <ui-loading-indicator :text="$strings.MessageLoading" />
    </div>
  </div>
</template>

<script>
export default {
  props: {},
  data() {
    return {
      shelves: [],
      isFirstNetworkConnection: true,
      lastServerFetch: 0,
      lastServerFetchLibraryId: null,
      lastLocalFetch: 0,
      localLibraryItems: [],
      isLoading: false,
      syncProgress: null,
      syncInterval: null,
      librarySize: 0
    }
  },
  watch: {
    networkConnected(newVal) {
      // Update shelves when network connect status changes
      console.log(`[categories] Network changed to ${newVal} - fetch categories. ${this.lastServerFetch}/${this.lastLocalFetch}`)

      if (newVal) {
        // Fetch right away the first time network connects
        if (this.isFirstNetworkConnection) {
          this.isFirstNetworkConnection = false
          console.log(`[categories] networkConnected true first network connection. lastServerFetch=${this.lastServerFetch}`)
          this.fetchCategories()
          return
        }

        setTimeout(() => {
          // Using timeout because making this fetch as soon as network gets connected will often fail on Android
          console.log(`[categories] networkConnected true so fetching categories. lastServerFetch=${this.lastServerFetch}`)
          this.fetchCategories()
        }, 4000)
      } else {
        console.log(`[categories] networkConnected false so fetching categories`)
        this.fetchCategories()
      }
    }
  },
  computed: {
    user() {
      return this.$store.state.user.user
    },
    networkConnected() {
      return this.$store.state.networkConnected
    },
    isIos() {
      return this.$platform === 'ios'
    },
    currentLibraryName() {
      return this.$store.getters['libraries/getCurrentLibraryName']
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    currentLibraryMediaType() {
      return this.$store.getters['libraries/getCurrentLibraryMediaType']
    },
    currentLibraryIsPodcast() {
      return this.currentLibraryMediaType === 'podcast'
    },
    altViewEnabled() {
      return this.$store.getters['getAltViewEnabled']
    },
    localMediaProgress() {
      return this.$store.state.globals.localMediaProgress
    },
    attemptingConnection() {
      return this.$store.state.attemptingConnection
    }
  },
  methods: {
    getShelfLabel(shelf) {
      if (shelf.labelStringKey && this.$strings[shelf.labelStringKey]) return this.$strings[shelf.labelStringKey]
      return shelf.label
    },
    getLocalMediaItemCategories() {
      const localMedia = this.localLibraryItems
      if (!localMedia?.length) return []

      const categories = []
      const books = []
      const podcasts = []
      const booksContinueListening = []
      const podcastEpisodesContinueListening = []
      localMedia.forEach((item) => {
        if (item.mediaType == 'book') {
          item.progress = this.$store.getters['globals/getLocalMediaProgressById'](item.id)
          if (item.progress && !item.progress.isFinished && item.progress.progress > 0) booksContinueListening.push(item)
          books.push(item)
        } else if (item.mediaType == 'podcast') {
          const podcastEpisodeItemCloner = { ...item }
          item.media.episodes = item.media.episodes.map((ep) => {
            ep.progress = this.$store.getters['globals/getLocalMediaProgressById'](item.id, ep.id)
            if (ep.progress && !ep.progress.isFinished && ep.progress.progress > 0) {
              podcastEpisodesContinueListening.push({
                ...podcastEpisodeItemCloner,
                recentEpisode: ep
              })
            }
            return ep
          })
          podcasts.push(item)
        }
      })

      // Local continue listening shelves, only shown offline
      if (booksContinueListening.length) {
        categories.push({
          id: 'local-books-continue',
          label: this.$strings.LabelContinueBooks,
          type: 'book',
          localOnly: true,
          entities: booksContinueListening.sort((a, b) => {
            if (a.progress && b.progress) {
              return b.progress.lastUpdate > a.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }
      if (podcastEpisodesContinueListening.length) {
        categories.push({
          id: 'local-episodes-continue',
          label: this.$strings.LabelContinueEpisodes,
          type: 'episode',
          localOnly: true,
          entities: podcastEpisodesContinueListening.sort((a, b) => {
            if (a.recentEpisode.progress && b.recentEpisode.progress) {
              return b.recentEpisode.progress.lastUpdate > a.recentEpisode.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }

      // Local books and local podcast shelves
      if (books.length) {
        categories.push({
          id: 'local-books',
          label: this.$strings.LabelLocalBooks,
          type: 'book',
          entities: books.sort((a, b) => {
            if (a.progress && a.progress.isFinished) return 1
            else if (b.progress && b.progress.isFinished) return -1
            else if (a.progress && b.progress) {
              return b.progress.lastUpdate > a.progress.lastUpdate ? 1 : -1
            }
            return 0
          })
        })
      }
      if (podcasts.length) {
        categories.push({
          id: 'local-podcasts',
          label: this.$strings.LabelLocalPodcasts,
          type: 'podcast',
          entities: podcasts
        })
      }

      return categories
    },
    async fetchCategories() {
      console.log(`[categories] fetchCategories networkConnected=${this.networkConnected}, lastServerFetch=${this.lastServerFetch}, lastLocalFetch=${this.lastLocalFetch}`)

      // TODO: Find a better way to keep the shelf up-to-date with local vs server library because this is a disaster
      const isConnectedToServerWithInternet = this.user && this.currentLibraryId && this.networkConnected
      if (isConnectedToServerWithInternet) {
        if (this.lastServerFetch && Date.now() - this.lastServerFetch < 5000 && this.lastServerFetchLibraryId == this.currentLibraryId) {
          console.log(`[categories] fetchCategories server fetch was ${Date.now() - this.lastServerFetch}ms ago so not doing it.`)
          return
        } else {
          console.log(`[categories] fetchCategories fetching from server. Last was ${this.lastServerFetch ? Date.now() - this.lastServerFetch + 'ms' : 'Never'} ago. lastServerFetchLibraryId=${this.lastServerFetchLibraryId} and currentLibraryId=${this.currentLibraryId}`)
          this.lastServerFetchLibraryId = this.currentLibraryId
          this.lastServerFetch = Date.now()
          this.lastLocalFetch = 0
        }
      } else {
        if (this.lastLocalFetch && Date.now() - this.lastLocalFetch < 5000) {
          console.log(`[categories] fetchCategories local fetch was ${Date.now() - this.lastLocalFetch}ms ago so not doing it.`)
          return
        } else {
          console.log(`[categories] fetchCategories fetching from local. Last was ${this.lastLocalFetch ? Date.now() - this.lastLocalFetch + 'ms' : 'Never'} ago`)
          this.lastServerFetchLibraryId = null
          this.lastServerFetch = 0
          this.lastLocalFetch = Date.now()
        }
      }

      this.isLoading = true

      // Set local library items first
      this.localLibraryItems = await this.$db.getLocalLibraryItems()
      const localCategories = this.getLocalMediaItemCategories()
      this.shelves = localCategories
      console.log('[categories] Local shelves set', this.shelves.length, this.lastLocalFetch)

      if (isConnectedToServerWithInternet) {
        // Check library size first
        await this.checkLibrarySize()
        
        // For large libraries, start metadata sync
        if (this.librarySize > 5000) {
          await this.startMetadataSync()
        }
        
        const categories = await this.$nativeHttp.get(`/api/libraries/${this.currentLibraryId}/personalized?minified=1&include=rssfeed,numEpisodesIncomplete&limit=100`, { connectTimeout: 10000 }).catch((error) => {
          console.error('[categories] Failed to fetch categories', error)
          return []
        })
        console.log('[categories] Server returned categories:', categories.map(c => ({ id: c.id, label: c.label, type: c.type, entityCount: c.entities?.length || 0 })))
        
        // Check user's media progress
        const userMediaProgress = this.$store.state.user.mediaProgress || []
        const inProgressItems = userMediaProgress.filter(mp => mp.progress > 0 && mp.progress < 1)
        console.log('[categories] User has', inProgressItems.length, 'items in progress:', inProgressItems.map(mp => ({ libraryItemId: mp.libraryItemId, progress: mp.progress })))
        if (!categories.length) {
          // Failed to load categories so use local shelves
          console.warn(`[categories] Failed to get server categories so using local categories`)
          this.lastServerFetch = 0
          this.lastLocalFetch = Date.now()
          this.isLoading = false
          console.log('[categories] Local shelves set from failure', this.shelves.length, this.lastLocalFetch)
          return
        }

        this.shelves = categories.map((cat) => {
          if (cat.type == 'book' || cat.type == 'podcast' || cat.type == 'episode') {
            // Map localLibraryItem to entities
            cat.entities = cat.entities.map((entity) => {
              const localLibraryItem = this.localLibraryItems.find((lli) => {
                return lli.libraryItemId == entity.id
              })
              if (localLibraryItem) {
                entity.localLibraryItem = localLibraryItem
              }
              return entity
            })
          }
          return cat
        })

        // Only add the local shelf with the same media type
        const localShelves = localCategories.filter((cat) => cat.type === this.currentLibraryMediaType && !cat.localOnly)
        this.shelves.push(...localShelves)
        console.log('[categories] Server shelves set', this.shelves.length, this.lastServerFetch)
      }

      this.isLoading = false
    },
    libraryChanged() {
      if (this.currentLibraryId) {
        console.log(`[categories] libraryChanged so fetching categories`)
        this.fetchCategories()
      }
    },
    audiobookAdded(audiobook) {
      // TODO: Check if audiobook would be on this shelf
      if (!this.search) {
        this.fetchCategories()
      }
    },
    audiobookUpdated(audiobook) {
      this.shelves.forEach((shelf) => {
        if (shelf.type === 'books') {
          shelf.entities = shelf.entities.map((ent) => {
            if (ent.id === audiobook.id) {
              return audiobook
            }
            return ent
          })
        } else if (shelf.type === 'series') {
          shelf.entities.forEach((ent) => {
            ent.books = ent.books.map((book) => {
              if (book.id === audiobook.id) return audiobook
              return book
            })
          })
        }
      })
    },
    removeBookFromShelf(audiobook) {
      this.shelves.forEach((shelf) => {
        if (shelf.type === 'books') {
          shelf.entities = shelf.entities.filter((ent) => {
            return ent.id !== audiobook.id
          })
        } else if (shelf.type === 'series') {
          shelf.entities.forEach((ent) => {
            ent.books = ent.books.filter((book) => {
              return book.id !== audiobook.id
            })
          })
        }
      })
    },
    initListeners() {
      this.$eventBus.$on('library-changed', this.libraryChanged)
      this.$eventBus.$on('local-media-progress-updated', this.localMediaProgressUpdated)
      this.$eventBus.$on('user_media_progress_updated', this.streamingMediaProgressUpdated)
    },
    removeListeners() {
      this.$eventBus.$off('library-changed', this.libraryChanged)
      this.$eventBus.$off('local-media-progress-updated', this.localMediaProgressUpdated)
      this.$eventBus.$off('user_media_progress_updated', this.streamingMediaProgressUpdated)
    },
    localMediaProgressUpdated() {
      console.log('[categories] Local media progress updated, refreshing categories')
      // Refresh categories to update Continue Listening shelves
      this.fetchCategories()
    },
    streamingMediaProgressUpdated(payload) {
      console.log('[categories] Streaming media progress updated', payload)
      // Refresh categories to update Continue Listening shelves
      this.fetchCategories()
    },
    async checkLibrarySize() {
      try {
        // First check sync progress
        const syncProgress = await this.$db.getLibrarySyncProgress(this.currentLibraryId)
        if (syncProgress && syncProgress.total > 0) {
          this.librarySize = syncProgress.total
          console.log(`[categories] Library size from sync: ${syncProgress.total}`)
          return
        }

        // Fallback to stats endpoint
        const response = await this.$nativeHttp.get(
          `/api/libraries/${this.currentLibraryId}/stats`,
          { connectTimeout: 5000 }
        ).catch(() => null)

        if (response && response.totalItems) {
          this.librarySize = response.totalItems
          console.log(`[categories] Library size: ${response.totalItems}`)
        }
      } catch (error) {
        console.error('[categories] Failed to check library size:', error)
      }
    },
    async startMetadataSync() {
      try {
        console.log('[categories] Starting metadata sync for large library')
        
        // Start sync in background
        await this.$db.syncLibraryMetadata(this.currentLibraryId, false)
        
        // Show sync progress in console (could add UI indicator if needed)
        this.syncInterval = setInterval(async () => {
          const progress = await this.$db.getLibrarySyncProgress(this.currentLibraryId)
          this.syncProgress = {
            synced: progress.synced,
            total: progress.total,
            percentage: progress.percentage,
            isComplete: progress.synced >= progress.total
          }
          
          console.log(`[categories] Sync progress: ${progress.synced}/${progress.total} (${progress.percentage.toFixed(1)}%)`)

          if (this.syncProgress.isComplete) {
            clearInterval(this.syncInterval)
            this.syncInterval = null
            console.log('[categories] Metadata sync complete')
          }
        }, 5000) // Check every 5 seconds
      } catch (error) {
        console.error('[categories] Failed to start metadata sync:', error)
      }
    }
  },
  async mounted() {
    if (this.$route.query.error) {
      this.$toast.error(this.$route.query.error)
    }

    this.initListeners()
    await this.$store.dispatch('globals/loadLocalMediaProgress')
    console.log(`[categories] mounted so fetching categories`)
    this.fetchCategories()
  },
  beforeDestroy() {
    this.removeListeners()
    if (this.syncInterval) {
      clearInterval(this.syncInterval)
    }
  }
}
</script>
