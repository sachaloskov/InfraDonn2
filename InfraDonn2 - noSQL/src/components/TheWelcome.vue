<template>
  <div class="container">
    <h1>Gestion des Posts</h1>

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
        <button @click="toggleOffline">{{ isOffline ? 'En ligne' : 'Offline' }}</button>
        <button @click="syncBidirectional" :disabled="isSyncing || isOffline">Synchroniser</button>
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
    <div v-if="filteredPosts.length === 0" class="empty">Aucun post</div>

    <article v-for="post in filteredPosts" :key="post._id" class="post">
      <h3>{{ post.title }}</h3>
      <p class="content">{{ post.content }}</p>
      <p class="meta">
        <strong>{{ post.author }}</strong> - {{ post.date }} • <em>{{ post.likes || 0 }} Likes</em>
      </p>
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
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import PouchDB from 'pouchdb'

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

const localDB = ref<any | null>(null)
const remoteDB = ref<any | null>(null)
const remoteUrl = 'http://admin:admin@localhost:5984/test_infradonn2'

const posts = ref<Post[]>([])
const comments = ref<CommentDoc[]>([])
const filteredPosts = ref<Post[]>([])
const syncStatus = ref<string>('Non synchronisé')
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

const initDB = async () => {
  localDB.value = new PouchDB('posts_local')
  remoteDB.value = new PouchDB(remoteUrl)
  watchLocalChanges()
  await fetchAllLocal()
}

const watchLocalChanges = () => {
  if (!localDB.value) return
  if (changesFeed && changesFeed.cancel) changesFeed.cancel()
  changesFeed = localDB.value.changes({ since: 'now', live: true, include_docs: true })
    .on('change', async () => await fetchAllLocal())
    .on('error', (err: any) => console.error('changes feed err', err))
}

const startLiveSync = () => {
  if (!localDB.value || !remoteDB.value || isOffline.value) return
  stopLiveSync()
  isSyncing.value = true
  syncStatus.value = 'Démarrage réplication live...'
  liveSync = localDB.value.sync(remoteDB.value, { live: true, retry: true })
    .on('change', async () => {
      syncStatus.value = 'Synchronisé ✓'
      await fetchAllLocal()
    })
    .on('paused', (err: any) => {
      if (err) syncStatus.value = 'Pause (erreur) ✗'
      else syncStatus.value = 'Pause (à jour)'
      isSyncing.value = false
    })
    .on('active', () => {
      syncStatus.value = 'Synchronisation active'
      isSyncing.value = true
    })
    .on('denied', (err: any) => {
      syncStatus.value = 'Refusé ✗'
      console.error('sync denied', err)
    })
    .on('complete', () => {
      syncStatus.value = 'Sync complete'
      isSyncing.value = false
    })
    .on('error', (err: any) => {
      syncStatus.value = 'Erreur ✗'
      isSyncing.value = false
      console.error('sync error', err)
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
  stopLiveSync()
  if (isOffline.value) {
    syncStatus.value = 'Offline'
  } else {
    syncStatus.value = "En ligne (live sync désactivé)"
  }
}

const fetchAllLocal = async () => {
  if (!localDB.value) return
  try {
    const res = await localDB.value.allDocs({ include_docs: true })
    const docs = res.rows.map((r: any) => r.doc).filter(Boolean)
    posts.value = docs.filter((d: any) => d.title && !d._deleted) as Post[]
    comments.value = docs.filter((d: any) => d.type === 'comment' && !d._deleted) as CommentDoc[]
    applySearchSort()
  } catch (e) {
    console.error('fetchAllLocal error', e)
  }
}

const syncBidirectional = async () => {
  if (!localDB.value || !remoteDB.value || isOffline.value) return
  syncStatus.value = 'Synchronisation...'
  isSyncing.value = true
  try {
    await localDB.value.sync(remoteDB.value)
    syncStatus.value = 'Synchronisé ✓'
    await fetchAllLocal()
  } catch (e) {
    syncStatus.value = 'Erreur ✗'
    console.error(e)
  } finally {
    isSyncing.value = false
  }
}

const activateLiveSync = () => {
  if (!isOffline.value && localDB.value && remoteDB.value) {
    startLiveSync()
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
    date: newPost.value.date,
    likes: 0,
  }
  await localDB.value.put(doc)
  newPost.value = { title: '', author: '', content: '', date: new Date().toLocaleString(), likes: 0 }
  await fetchAllLocal()
  syncStatus.value = 'Modifications locales'
}

const updatePost = async (post: Post) => {
  const title = prompt('', post.title)
  if (title === null) return
  const content = prompt('', post.content)
  if (content === null) return
  try {
    const doc = await localDB.value.get(post._id)
    doc.title = title
    doc.content = content
    await localDB.value.put(doc)
    await fetchAllLocal()
    syncStatus.value = 'Modifications locales'
  } catch (e) {
    console.error('updatePost', e)
  }
}

const deletePost = async (post: Post) => {
  try {
    const doc = await localDB.value.get(post._id)
    await localDB.value.remove(doc)
    await fetchAllLocal()
    syncStatus.value = 'Modifications locales'
  } catch (e) {
    console.error('deletePost', e)
  }
}

const likePost = async (post: Post) => {
  try {
    const doc = await localDB.value.get(post._id)
    doc.likes = (doc.likes || 0) + 1
    await localDB.value.put(doc)
    await fetchAllLocal()
    syncStatus.value = 'Modifications locales'
  } catch (e) {
    console.error('like error', e)
  }
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
  await localDB.value.put(doc)
  newComment.value = { author: '', content: '' }
  await fetchAllLocal()
  syncStatus.value = 'Modifications locales'
}

const updateComment = async (c: CommentDoc) => {
  const content = prompt('', c.content)
  if (content === null) return
  try {
    const doc = await localDB.value.get(c._id)
    doc.content = content
    await localDB.value.put(doc)
    await fetchAllLocal()
    syncStatus.value = 'Modifications locales'
  } catch (e) {
    console.error('updateComment', e)
  }
}

const deleteComment = async (c: CommentDoc) => {
  try {
    const doc = await localDB.value.get(c._id)
    await localDB.value.remove(doc)
    await fetchAllLocal()
    syncStatus.value = 'Modifications locales'
  } catch (e) {
    console.error('deleteComment', e)
  }
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
  await localDB.value.bulkDocs(docs)
  await fetchAllLocal()
  syncStatus.value = 'Modifications locales'
}

const applySearchSort = async () => {
  let list = [...posts.value]
  const q = searchQuery.value.trim().toLowerCase()
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
  }
  filteredPosts.value = list
}

const clearSearch = async () => {
  searchQuery.value = ''
  sortBy.value = 'none'
  await fetchAllLocal()
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
/* Barre info des posts */
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
/* Commentaires démarqués */
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
/* Nouveau post form - meilleure mise en page */
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
@media (max-width: 600px) {
  .container { padding:1rem 2vw; font-size:0.98rem }
  .factory-controls, .search-panel, .buttons { flex-direction: column; gap:0.4rem }
  .form-row { flex-direction: column; gap: 0.4rem; }
}
</style>
