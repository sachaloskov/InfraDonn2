<template>
  <div class="container">
    <h1>Gestion des posts</h1>

    <div class="sync-panel">
      <h3>Synchronisation</h3>
      <p>
        Statut:
        <strong
          :class="{
            success: syncStatus.includes('✓'),
            error: syncStatus.includes('✗'),
            pending: !syncStatus.includes('✓') && !syncStatus.includes('✗'),
          }"
        >
          {{ syncStatus }}
        </strong>
      </p>
      <div class="buttons">
        <button @click="toggleOffline">{{ isOffline ? 'Passer en ligne' : 'Passer Offline' }}</button>
        <button @click="syncBidirectional" :disabled="isSyncing || isOffline">Sync manuelle</button>
      </div>
    </div>

    <section class="factory">
      <h3>Factory</h3>
      <div class="factory-controls">
        <input v-model.number="factoryCount" type="number" min="1" placeholder="Nombre à générer" />
        <button @click="generateFactory">Générer</button>
      </div>
    </section>

    <section class="search-panel">
      <input v-model="searchQuery" placeholder="Recherche titre ou auteur..." @keyup.enter="applySearchSort" />
      <select v-model="sortBy">
        <option value="none">Trier: aucun</option>
        <option value="likes_desc">Likes ↓</option>
        <option value="likes_asc">Likes ↑</option>
      </select>
      <button @click="applySearchSort">Appliquer</button>
      <button @click="clearSearch">Effacer</button>
    </section>

    <h2>Posts ({{ filteredPosts.length }})</h2>

    <div v-if="filteredPosts.length > 10 || showAllPosts" class="buttons" style="margin-bottom: 0.5rem;">
      <button @click="toggleShowAll">
        {{ showAllPosts ? 'Afficher uniquement le Top 10' : 'Afficher tous les posts' }}
      </button>
    </div>

    <div v-if="displayedPosts.length === 0" class="empty">Aucun post</div>

    <article v-for="post in displayedPosts" :key="post._id" class="post">
      <h3>{{ post.title }}</h3>
      <p class="content">{{ post.content }}</p>
      <p class="meta">
        <strong>{{ post.author }}</strong> - {{ post.date }} • <em>{{ post.likes || 0 }} Likes</em>
      </p>

      <div
        v-if="commentsByPost(post._id)?.length > 0"
        class="first-comment"
      >
        <p><strong>Premier commentaire :</strong></p>
        <p>{{ commentsByPost(post._id)?.[0]?.content }}</p>
        <p class="first-comment-footer">
          <strong>{{ commentsByPost(post._id)?.[0]?.author }}</strong> -
          {{ commentsByPost(post._id)?.[0]?.date }}
        </p>
      </div>

      <div class="post-actions buttons">
        <button @click="updatePost(post)">Modifier</button>
        <button @click="deletePost(post)">Supprimer</button>
        <button @click="likePost(post)">Like</button>
        <button @click="openComments(post)">Commentaires ({{ countComments(post._id) }})</button>
      </div>

      <section v-if="activeCommentsPostId === post._id" class="comments">
        <h4>Commentaires</h4>

        <div v-for="c in commentsByPost(post._id)" :key="c._id" class="comment">
          <div class="comment-cartouche">
            <p>{{ c.content }}</p>
            <p class="meta"><strong>{{ c.author }}</strong> - {{ c.date }}</p>
            <div class="buttons">
              <button @click="updateComment(c)">Modifier</button>
              <button @click="deleteComment(c)">Supprimer</button>
            </div>
          </div>
        </div>

        <div class="add-comment">
          <input v-model="newComment.author" placeholder="Auteur" />
          <textarea v-model="newComment.content" rows="2" placeholder="Commentaire"></textarea>
          <button @click="addComment(post)">Ajouter</button>
        </div>
      </section>
    </article>

    <hr />

    <h2>Nouveau post</h2>
    <form class="form" @submit.prevent="createPost">
      <div class="form-row">
        <input v-model="newPost.title" placeholder="Titre" />
        <input v-model="newPost.author" placeholder="Auteur" />
      </div>
      <textarea v-model="newPost.content" rows="4" placeholder="Contenu"></textarea>
      <button type="submit">Publier</button>
    </form>

    <div v-if="conflictToResolve" class="conflict-modal-overlay" @click.self="closeConflictModal">
      <div class="conflict-modal">
        <h2>Conflit détecté</h2>
        <p class="conflict-info">
          Un conflit a été détecté pour le document <strong>{{ getDocumentType(conflictToResolve.current) }}</strong>.
          <br>Veuillez choisir la version à conserver :
        </p>

        <div class="conflict-versions">
          <div class="version-card">
            <h3>Version locale (Application)</h3>
            <div class="version-details">
              <p v-if="conflictToResolve.current.title"><strong>Titre :</strong> {{ conflictToResolve.current.title }}</p>
              <p v-if="conflictToResolve.current.content"><strong>Contenu :</strong> {{ conflictToResolve.current.content }}</p>
              <p v-if="conflictToResolve.current.author"><strong>Auteur :</strong> {{ conflictToResolve.current.author }}</p>
              <p v-if="conflictToResolve.current.date"><strong>Date :</strong> {{ conflictToResolve.current.date }}</p>
              <p v-if="conflictToResolve.current.likes !== undefined"><strong>Likes :</strong> {{ conflictToResolve.current.likes }}</p>
              <p class="version-rev"><strong>Révision :</strong> {{ conflictToResolve.current._rev }}</p>
            </div>
            <button class="btn-keep-local" @click="keepLocalVersion">
              Garder cette version (Locale)
            </button>
          </div>

          <div class="version-card">
            <h3>Version distante (Base de données)</h3>
            <div class="version-details">
              <p v-if="conflictToResolve.conflicting.title"><strong>Titre :</strong> {{ conflictToResolve.conflicting.title }}</p>
              <p v-if="conflictToResolve.conflicting.content"><strong>Contenu :</strong> {{ conflictToResolve.conflicting.content }}</p>
              <p v-if="conflictToResolve.conflicting.author"><strong>Auteur :</strong> {{ conflictToResolve.conflicting.author }}</p>
              <p v-if="conflictToResolve.conflicting.date"><strong>Date :</strong> {{ conflictToResolve.conflicting.date }}</p>
              <p v-if="conflictToResolve.conflicting.likes !== undefined"><strong>Likes :</strong> {{ conflictToResolve.conflicting.likes }}</p>
              <p class="version-rev"><strong>Révision :</strong> {{ conflictToResolve.conflicting._rev }}</p>
            </div>
            <button class="btn-keep-remote" @click="keepRemoteVersion">
              Garder cette version (Distante)
            </button>
          </div>
        </div>

        <div class="conflict-actions">
          <button class="btn-cancel" @click="closeConflictModal">Annuler (garder locale par défaut)</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

interface Post {
  _id?: string
  _rev?: string
  type?: string
  title: string
  author: string
  content: string
  date: string
  likes?: number
}

interface CommentDoc {
  _id?: string
  _rev?: string
  type: 'comment'
  postId: string
  author: string
  content: string
  date: string
}

interface ConflictResolution {
  current: any
  conflicting: any
  conflictRev: string
}

const localDB = ref<any | null>(null)
const remoteDB = ref<any | null>(null)
const remoteUrl = 'http://admin:admin@localhost:5984/test_infradonn2'

const posts = ref<Post[]>([])
const comments = ref<CommentDoc[]>([])
const filteredPosts = ref<Post[]>([])
const syncStatus = ref<string>('Initialisation...')
const isSyncing = ref<boolean>(false)
let liveSync: any = null
let changesFeed: any = null

const isOffline = ref(false)

const newPost = ref<Post>({ title: '', author: '', content: '', date: new Date().toLocaleString(), likes: 0 })
const newComment = ref<any>({ author: '', content: '' })
const activeCommentsPostId = ref<string | null>(null)
const factoryCount = ref<number>(50)
const searchQuery = ref('')
const sortBy = ref('none')

const showAllPosts = ref(false)

const conflictToResolve = ref<ConflictResolution | null>(null)
const conflictQueue = ref<ConflictResolution[]>([])

const displayedPosts = computed(() => {
  return showAllPosts.value ? filteredPosts.value : filteredPosts.value.slice(0, 10)
})

const getDocumentType = (doc: any) => {
  if (doc.type === 'post') return `Post "${doc.title || 'Sans titre'}"`
  if (doc.type === 'comment') return `Commentaire`
  return 'Document'
}

const initIndexes = async () => {
  if (!localDB.value) return
  
  await localDB.value.createIndex({
    index: {
      fields: ['type', 'likes']
    }
  })
  
  await localDB.value.createIndex({
    index: {
      fields: ['type', 'title', 'author']
    }
  })
  
  await localDB.value.createIndex({
    index: {
      fields: ['type', 'postId']
    }
  })
}

const initDB = async () => {
  localDB.value = new PouchDB('posts_local')
  remoteDB.value = new PouchDB(remoteUrl)
  
  await initIndexes()
  
  syncStatus.value = 'Synchronisation initiale...'
  try {
    await localDB.value.sync(remoteDB.value)
    syncStatus.value = 'Sync initiale OK ✓'
  } catch (e) {
    syncStatus.value = 'Sync initiale échouée ✗'
  }
  
  await fetchAllLocalFromDB()
  
  startLiveSync()
  
  watchLocalChanges()
}

const fetchAllLocalFromDB = async () => {
  if (!localDB.value) return
  try {
    const postsResult = await localDB.value.find({
      selector: { 
        type: 'post',
        _id: { $gt: null }
      },
      limit: 9999
    })
    posts.value = postsResult.docs as Post[]
    
    const commentsResult = await localDB.value.find({
      selector: { 
        type: 'comment',
        _id: { $gt: null }
      },
      limit: 9999
    })
    comments.value = commentsResult.docs as CommentDoc[]
    
    await applySearchSort()
  } catch (e) {}
}

const watchLocalChanges = () => {
  if (!localDB.value) return
  if (changesFeed && changesFeed.cancel) changesFeed.cancel()
  
  changesFeed = localDB.value.changes({ 
    since: 'now', 
    live: true, 
    include_docs: true,
    conflicts: true
  })
    .on('change', async (change: any) => {
      if (change.doc && change.doc._conflicts && change.doc._conflicts.length > 0) {
        await handleConflictDetected(change.doc)
      }
      await fetchAllLocalFromDB()
    })
}

const handleConflictDetected = async (doc: any) => {
  if (!localDB.value || !doc._conflicts || doc._conflicts.length === 0) return

  try {
    const conflictRev = doc._conflicts[0]
    const conflictingDoc = await localDB.value.get(doc._id, { rev: conflictRev })

    const conflict: ConflictResolution = {
      current: doc,
      conflicting: conflictingDoc,
      conflictRev
    }

    conflictQueue.value.push(conflict)

    if (!isOffline.value && !conflictToResolve.value) {
      showNextConflict()
    }
  } catch (e) {}
}

const showNextConflict = () => {
  if (conflictQueue.value.length > 0) {
    conflictToResolve.value = conflictQueue.value.shift() || null
  } else {
    conflictToResolve.value = null
  }
}

const keepLocalVersion = async () => {
  if (!conflictToResolve.value || !localDB.value) return
  
  try {
    await localDB.value.remove(
      conflictToResolve.value.current._id,
      conflictToResolve.value.conflictRev
    )
    
    syncStatus.value = 'Conflit résolu (version locale conservée) ✓'
    
    showNextConflict()
    
    await fetchAllLocalFromDB()
  } catch (e) {
    syncStatus.value = 'Erreur résolution conflit ✗'
  }
}

const keepRemoteVersion = async () => {
  if (!conflictToResolve.value || !localDB.value) return
  
  try {
    const remoteDoc = conflictToResolve.value.conflicting
    const currentDoc = conflictToResolve.value.current
    
    await localDB.value.remove(
      currentDoc._id,
      conflictToResolve.value.conflictRev
    )
    
    remoteDoc._rev = currentDoc._rev
    await localDB.value.put(remoteDoc)
    
    syncStatus.value = 'Conflit résolu (version distante conservée) ✓'
    
    showNextConflict()
    
    await fetchAllLocalFromDB()
  } catch (e) {
    syncStatus.value = 'Erreur résolution conflit ✗'
  }
}

const closeConflictModal = async () => {
  await keepLocalVersion()
}

const startLiveSync = () => {
  if (!localDB.value || !remoteDB.value || isOffline.value) return
  stopLiveSync()
  
  isSyncing.value = true
  syncStatus.value = 'Démarrage live sync...'
  
  liveSync = localDB.value.sync(remoteDB.value, { 
    live: true, 
    retry: true,
    batch_size: 100,
    batches_limit: 10
  })
    .on('change', async (info: any) => {
      syncStatus.value = 'Synchronisé ✓'
      
      if (info.change && info.change.docs) {
        for (const doc of info.change.docs) {
          if (doc._conflicts && doc._conflicts.length > 0) {
            await handleConflictDetected(doc)
          }
        }
      }
      
      await fetchAllLocalFromDB()
    })
    .on('paused', (err: any) => {
      if (err) {
        syncStatus.value = 'Pause (erreur) ✗'
      } else {
        syncStatus.value = 'Live sync actif (à jour)'
      }
      isSyncing.value = false
    })
    .on('active', () => {
      syncStatus.value = 'Synchronisation en cours...'
      isSyncing.value = true
    })
    .on('denied', () => {
      syncStatus.value = 'Refusé ✗'
    })
    .on('complete', () => {
      syncStatus.value = 'Sync complete'
      isSyncing.value = false
    })
    .on('error', () => {
      syncStatus.value = 'Erreur sync ✗'
      isSyncing.value = false
    })
}

const stopLiveSync = () => {
  if (liveSync && liveSync.cancel) {
    liveSync.cancel()
    liveSync = null
  }
}

const toggleOffline = () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    stopLiveSync()
    syncStatus.value = 'Mode Offline'
  } else {
    syncStatus.value = 'Retour en ligne...'

    if (!conflictToResolve.value && conflictQueue.value.length > 0) {
      showNextConflict()
    }

    syncBidirectional().then(() => {
      startLiveSync()
    })
  }
}

const syncBidirectional = async () => {
  if (!localDB.value || !remoteDB.value || isOffline.value) return
  
  syncStatus.value = 'Synchronisation bidirectionnelle...'
  isSyncing.value = true
  
  try {
    const result = await localDB.value.sync(remoteDB.value)
    
    if (result.pull && result.pull.docs) {
      for (const doc of result.pull.docs) {
        if (doc._conflicts && doc._conflicts.length > 0) {
          await handleConflictDetected(doc)
        }
      }
    }
    
    syncStatus.value = 'Synchronisé ✓'
    await fetchAllLocalFromDB()
  } catch (e) {
    syncStatus.value = 'Erreur sync ✗'
  } finally {
    isSyncing.value = false
  }
}

const createPost = async () => {
  if (!localDB.value) return
  
  const doc: any = {
    _id: `post:${Date.now()}`,
    type: 'post',
    title: newPost.value.title,
    author: newPost.value.author,
    content: newPost.value.content,
    date: new Date().toLocaleString(),
    likes: 0,
  }
  
  try {
    await localDB.value.put(doc)
    newPost.value = { title: '', author: '', content: '', date: new Date().toLocaleString(), likes: 0 }
    syncStatus.value = 'Modification locale (sync auto)'
  } catch (e) {}
}

const updatePost = async (post: Post) => {
  const title = prompt('Nouveau titre:', post.title)
  if (title === null) return
  const content = prompt('Nouveau contenu:', post.content)
  if (content === null) return
  
  try {
    const doc = await localDB.value.get(post._id)
    doc.title = title
    doc.content = content
    await localDB.value.put(doc)
    syncStatus.value = 'Modification locale (sync auto)'
  } catch (e) {}
}

const deletePost = async (post: Post) => {
  if (!confirm('Supprimer ce post ?')) return
  
  try {
    const doc = await localDB.value.get(post._id)
    await localDB.value.remove(doc)
    syncStatus.value = 'Suppression locale (sync auto)'
  } catch (e) {}
}

const likePost = async (post: Post) => {
  try {
    const doc = await localDB.value.get(post._id)
    doc.likes = (doc.likes || 0) + 1
    await localDB.value.put(doc)
    syncStatus.value = 'Like ajouté (sync auto)'
  } catch (e) {}
}

const openComments = (post: Post) => {
  activeCommentsPostId.value = activeCommentsPostId.value === post._id ? null : post._id || null
}

const commentsByPost = (postId: string | undefined) => {
  if (!postId) return []
  return comments.value.filter(c => c.postId === postId)
}

const countComments = (postId: string | undefined) => commentsByPost(postId).length

const addComment = async (post: Post) => {
  if (!post._id) return
  
  const doc: CommentDoc = {
    _id: `comment:${Date.now()}`,
    type: 'comment',
    postId: post._id,
    author: newComment.value.author || 'Anonyme',
    content: newComment.value.content || '',
    date: new Date().toLocaleString(),
  }
  
  try {
    await localDB.value.put(doc)
    newComment.value = { author: '', content: '' }
    syncStatus.value = 'Commentaire ajouté (sync auto)'
  } catch (e) {}
}

const updateComment = async (c: CommentDoc) => {
  const content = prompt('Nouveau commentaire:', c.content)
  if (content === null) return
  
  try {
    const doc = await localDB.value.get(c._id)
    doc.content = content
    await localDB.value.put(doc)
    syncStatus.value = 'Commentaire modifié (sync auto)'
  } catch (e) {}
}

const deleteComment = async (c: CommentDoc) => {
  if (!confirm('Supprimer ce commentaire ?')) return
  
  try {
    const doc = await localDB.value.get(c._id)
    await localDB.value.remove(doc)
    syncStatus.value = 'Commentaire supprimé (sync auto)'
  } catch (e) {}
}

const generateFactory = async () => {
  const n = factoryCount.value || 50
  const docs: any[] = []
  
  for (let i = 0; i < n; i++) {
    docs.push({
      _id: `post:factory:${Date.now()}_${i}`,
      type: 'post',
      title: `Titre factory ${Math.random().toString(36).slice(2, 9)} #${i}`,
      author: `Auteur ${Math.random().toString(36).slice(2, 6)}`,
      content: `Contenu généré ${i}`,
      date: new Date().toLocaleString(),
      likes: Math.floor(Math.random() * 100),
    })
  }
  
  try {
    await localDB.value.bulkDocs(docs)
    syncStatus.value = `${n} posts générés (sync auto)`
  } catch (e) {}
}

const applySearchSort = async () => {
  if (!localDB.value) return

  const q = searchQuery.value.trim().toLowerCase()

  const selector: any = {
    type: 'post'
  }

  if (q) {
    selector.$or = [
      { title: { $regex: `(?i)${q}` } },
      { author: { $regex: `(?i)${q}` } }
    ]
  }

  const sort: any[] = []
  if (sortBy.value === 'likes_desc') {
    sort.push({ likes: 'desc' })
  } else if (sortBy.value === 'likes_asc') {
    sort.push({ likes: 'asc' })
  } else {
    sort.push({ likes: 'desc' })
  }

  try {
    const result = await localDB.value.find({
      selector,
      sort,
      limit: 9999
    })
    filteredPosts.value = result.docs as Post[]
    showAllPosts.value = false
  } catch (e) {
    let list = [...posts.value]
    if (q) {
      list = list.filter(
        p =>
          (p.title && p.title.toLowerCase().includes(q)) ||
          (p.author && p.author.toLowerCase().includes(q))
      )
    }
    if (sortBy.value === 'likes_desc') {
      list.sort((a, b) => (b.likes || 0) - (a.likes || 0))
    } else if (sortBy.value === 'likes_asc') {
      list.sort((a, b) => (a.likes || 0) - (b.likes || 0))
    } else {
      list.sort((a, b) => (b.likes || 0) - (a.likes || 0))
    }
    filteredPosts.value = list
    showAllPosts.value = false
  }
}

const clearSearch = async () => {
  searchQuery.value = ''
  sortBy.value = 'none'
  await applySearchSort()
  showAllPosts.value = false
}

const toggleShowAll = () => {
  showAllPosts.value = !showAllPosts.value
}

onMounted(() => { initDB() })
onBeforeUnmount(() => {
  stopLiveSync()
  if (changesFeed && changesFeed.cancel) changesFeed.cancel()
})
</script>

<style scoped>
.container {
  padding: 2rem 1rem;
  max-width: 700px;
  margin: 0 auto;
  background: #fff;
  color: #222;
  font-family: 'Segoe UI', Arial, sans-serif;
  font-size: 1.07rem;
}
h1, h2, h3, h4 {
  color: #13407b;
  font-weight: 700;
}
.sync-panel {
  background: #e5f0fa;
  padding: 1.25rem;
  border-radius: 10px;
  margin-bottom: 1.25rem;
  box-shadow: 0 1px 6px #a5bedf33;
}
.success { color: #188038; }
.error { color: #e53935; }
.pending { color: #ff9800; }
.buttons {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
  margin-top: 1rem;
}
button {
  padding: 0.5rem 1.1rem;
  font-size: 1rem;
  font-weight: 500;
  background: #13407b;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
}
button:hover:not(:disabled) {
  background: #012859;
}
button:disabled {
  opacity: .65;
  cursor: not-allowed;
}
input, textarea, select {
  padding: 0.45rem;
  font-size: 1rem;
  border: 1.3px solid #bccbe7;
  border-radius: 5px;
  outline: none;
  margin-bottom: 0.4rem;
  background: #f8fafc;
  color: #222;
}
.factory-controls, .search-panel {
  display:flex;
  gap:.6rem;
  align-items:center;
  margin:.7rem 0;
}
.post {
  border: 1.5px solid #d5e2f5;
  padding: 1.25rem;
  margin: 1rem 0;
  border-radius: 7px;
  background: #fcfdff;
  transition: box-shadow 0.2s;
}
.post:hover {
  box-shadow: 0 2px 9px #acc8ff22;
}
.empty {
  padding: 1.2rem;
  background: #f3f6fa;
  border-radius: 6px;
  text-align: center;
  color: #888;
}
.meta {
  background: #f6f8fb;
  font-size: 0.97rem;
  color: #3a4e71;
  font-style: italic;
  border-top: 1.2px solid #dbe8f9;
  padding: 0.51rem 0.85rem 0.27rem 0.85rem;
  margin: 0.5rem -1rem 0.2rem -1rem;
  border-radius: 0 0 7px 7px;
  letter-spacing: .04em;
}
.first-comment {
  margin: 0.8rem 0;
  padding: 0.8rem 1rem 0.6rem 1rem;
  background: #f0f6ff;
  border-left: 4px solid #1e74ff;
  border-radius: 7px;
}
.first-comment-footer {
  margin-top: 0.4rem;
  font-size: 0.95rem;
  color: #3a4e71;
  font-style: italic;
}
.comments {
  background: #eaf3fa;
  padding: 1rem 0.7rem 0.7rem 0.7rem;
  border-radius: 7px;
  margin-top: 0.9rem;
  border: 2px solid #91bddf;
}
.comment {
  margin-bottom: 1.1rem;
}
.comment-cartouche {
  background: #ffffff;
  border-left: 6px solid #229cff;
  border-radius: 7px;
  padding: 0.8rem 1rem;
  box-shadow: 0 2px 10px #0460cf12;
}
.comment:last-child {
  margin-bottom: 0;
}
.add-comment input, .add-comment textarea {
  width: 94%;
}
.add-comment button {
  margin-top: 0.3rem;
}
.form {
  background: #eff5fa;
  padding: 1.1rem .7rem .9rem .7rem;
  border-radius: 7px;
  box-shadow: 0 2px 10px #7eaecd17;
  margin-bottom: 2rem;
}
.form-row {
  display: flex;
  gap: 0.8rem;
  margin-bottom: 0.7rem;
}
.form input {
  width: 100%;
  margin-bottom: 0;
}
.form textarea {
  width: 100%;
  margin-bottom: 0.8rem;
  min-height: 60px;
}
.form button {
  width: 100%;
  margin-top: 0.3rem;
}
.conflict-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.75);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  padding: 1rem;
}

.conflict-modal {
  background: #ffffff;
  border-radius: 12px;
  padding: 2rem;
  max-width: 900px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
}

.conflict-modal h2 {
  color: #d32f2f;
  margin-bottom: 1rem;
  font-size: 1.8rem;
  text-align: center;
}

.conflict-info {
  background: #fff3cd;
  border-left: 4px solid #ffc107;
  padding: 1rem;
  margin-bottom: 1.5rem;
  border-radius: 6px;
  color: #856404;
  text-align: center;
}

.conflict-versions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
  margin-bottom: 1.5rem;
}

.version-card {
  border: 2px solid #e0e0e0;
  border-radius: 10px;
  padding: 1.5rem;
  background: #fafafa;
  transition: all 0.3s;
}

.version-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  border-color: #13407b;
}

.version-card h3 {
  color: #13407b;
  margin-bottom: 1rem;
  font-size: 1.2rem;
  text-align: center;
  padding-bottom: 0.8rem;
  border-bottom: 2px solid #e0e0e0;
}

.version-details {
  background: white;
  padding: 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  min-height: 200px;
}

.version-details p {
  margin: 0.6rem 0;
  line-height: 1.5;
  word-wrap: break-word;
}

.version-details strong {
  color: #13407b;
}

.version-rev {
  font-size: 0.85rem;
  color: #666;
  font-style: italic;
  margin-top: 1rem;
  padding-top: 0.8rem;
  border-top: 1px solid #e0e0e0;
}

.btn-keep-local {
  width: 100%;
  background: #2e7d32;
  padding: 0.8rem;
  font-size: 1.05rem;
  font-weight: 600;
}

.btn-keep-local:hover {
  background: #1b5e20;
}

.btn-keep-remote {
  width: 100%;
  background: #1976d2;
  padding: 0.8rem;
  font-size: 1.05rem;
  font-weight: 600;
}

.btn-keep-remote:hover {
  background: #0d47a1;
}

.conflict-actions {
  text-align: center;
  padding-top: 1rem;
  border-top: 2px solid #e0e0e0;
}

.btn-cancel {
  background: #757575;
  padding: 0.7rem 2rem;
  font-size: 1rem;
}

.btn-cancel:hover {
  background: #424242;
}

@media (max-width: 768px) {
  .conflict-versions {
    grid-template-columns: 1fr;
    gap: 1rem;
  }
  
  .conflict-modal {
    padding: 1.5rem;
  }
  
  .version-details {
    min-height: auto;
  }
}

@media (max-width: 600px) {
  .container { padding:1rem 2vw; font-size:0.98rem }
  .factory-controls, .search-panel, .buttons { flex-direction: column; gap:0.4rem }
  .form-row { flex-direction: column; gap: 0.4rem; }
}
</style>