<template>
  <div class="container">
    <h1>Gestion des posts</h1>

    <!-- panneau pour gérer la synchro online/offline -->
    <div class="sync-panel">
      <h3>Synchronisation</h3>
      <p>
        Statut:
        <strong v-if="isOffline" class="error">
          Offline
        </strong>
        <strong
          v-else
          :class="{
            success: syncStatus.includes('✓') || syncStatus === 'Live sync',
            error: syncStatus.includes('✗'),
            pending: !syncStatus.includes('✓') && !syncStatus.includes('✗') && syncStatus !== 'Live sync',
          }"
        >
          {{ syncStatus }}
        </strong>
      </p>
      <div class="buttons">
        <button @click="toggleOffline">{{ isOffline ? 'Passer en ligne' : 'Passer Offline' }}</button>
      </div>
    </div>

    <!-- génération automatique de plusieurs posts -->
    <section class="factory">
      <h3>Factory</h3>
      <div class="factory-controls">
        <input v-model.number="factoryCount" type="number" min="1" placeholder="Nombre à générer" />
        <button @click="generateFactory">Générer</button>
      </div>
    </section>

    <!-- barre de recherche et tri des posts -->
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

    <!-- bouton pour limiter l’affichage à 10 posts -->
    <div v-if="filteredPosts.length > 10 || showAllPosts" class="buttons" style="margin-bottom: 0.5rem;">
      <button @click="toggleShowAll">
        {{ showAllPosts ? 'Afficher uniquement le Top 10' : 'Afficher tous les posts' }}
      </button>
    </div>

    <!-- message quand il n'y a aucun post -->
    <div v-if="displayedPostsWithMedia.length === 0" class="empty">Aucun post</div>

    <!-- affichage de chaque post -->
    <article v-for="post in displayedPostsWithMedia" :key="post._id" class="post">
      <h3>{{ post.title }}</h3>
      <p class="content">{{ post.content }}</p>
      <p class="meta">
        <strong>{{ post.author }}</strong> - {{ post.date }} • <em>{{ post.likes || 0 }} Likes</em>
      </p>

      <!-- affichage du média lié au post -->
      <div v-if="postMedia(post._id)" class="post-media-display">
        <h4>Média associé</h4>
        <img v-if="isImage(postMedia(post._id)?.type || '')" :src="postMedia(post._id)?.url" :alt="postMedia(post._id)?.name || ''" class="responsive-media-img">
        <video v-else-if="isVideo(postMedia(post._id)?.type || '')" :src="postMedia(post._id)?.url" controls class="responsive-media-video" :title="postMedia(post._id)?.name || ''"></video>
        <p v-else class="media-info">Fichier: {{ postMedia(post._id)?.name || 'Nom inconnu' }} (Type: {{ postMedia(post._id)?.type || 'Inconnu' }})</p>
      </div>

      <!-- aperçu du premier commentaire du post -->
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

      <!-- actions principales sur le post -->
      <div class="post-actions buttons">
        <button @click="updatePost(post)">Modifier</button>
        <button @click="deletePost(post)">Supprimer</button>
        <button @click="likePost(post)">Like</button>
        <button @click="openComments(post)">Commentaires ({{ countComments(post._id) }})</button>

        <!-- boutons de gestion du média -->
        <label class="media-upload-button">
          Ajouter/Remplacer Média
          <input type="file" @change="handleFileSelect($event, post._id)" style="display: none;" />
        </label>
        <button v-if="postMedia(post._id)" @click="deleteMediaAttachment(post)">Supprimer Média</button>
      </div>

      <!-- bloc de gestion des commentaires -->
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

        <!-- formulaire d’ajout d’un commentaire -->
        <div class="add-comment">
          <input v-model="newComment.author" placeholder="Auteur" />
          <textarea v-model="newComment.content" rows="2" placeholder="Commentaire"></textarea>
          <button @click="addComment(post)">Ajouter</button>
        </div>
      </section>
    </article>

    <hr />

    <!-- formulaire pour créer un nouveau post -->
    <h2>Nouveau post</h2>
    <form class="form" @submit.prevent="createPost">
      <div class="form-row">
        <input v-model="newPost.title" placeholder="Titre" />
        <input v-model="newPost.author" placeholder="Auteur" />
      </div>
      <textarea v-model="newPost.content" rows="4" placeholder="Contenu"></textarea>
      <button type="submit">Publier</button>
    </form>

    <!-- fenêtre modale quand il y a un conflit de synchro -->
    <div v-if="conflictToResolve" class="conflict-modal-overlay">
      <div class="conflict-modal">
        <h2>Conflit détecté</h2>
        <p class="conflict-info">
          Un conflit a été détecté !
          <br>Veuillez choisir la version à conserver :
        </p>

        <div class="conflict-versions">
          <!-- version locale (ou choisie comme locale) -->
          <div class="version-card local-version">
            <h3>Version 1</h3>
            <div class="version-details">
              <p v-if="conflictToResolve.local.title"><strong>Titre :</strong> {{ conflictToResolve.local.title }}</p>
              <p v-if="conflictToResolve.local.content"><strong>Contenu :</strong> {{ conflictToResolve.local.content }}</p>
              <p v-if="conflictToResolve.local.author"><strong>Auteur :</strong> {{ conflictToResolve.local.author }}</p>
              <p v-if="conflictToResolve.local.date"><strong>Date :</strong> {{ conflictToResolve.local.date }}</p>
              <p v-if="conflictToResolve.local.likes !== undefined"><strong>Likes :</strong> {{ conflictToResolve.local.likes }}</p>
              <p class="version-rev"><strong>Révision :</strong> {{ conflictToResolve.local._rev }}</p>
            </div>
            <button class="btn-keep-local" @click="keepLocalVersion">
              ✓ Garder cette version
            </button>
          </div>

          <!-- version distante (ou considérée comme distante) -->
          <div class="version-card remote-version">
            <h3>Version 2</h3>
            <div class="version-details">
              <p v-if="conflictToResolve.remote.title"><strong>Titre :</strong> {{ conflictToResolve.remote.title }}</p>
              <p v-if="conflictToResolve.remote.content"><strong>Contenu :</strong> {{ conflictToResolve.remote.content }}</p>
              <p v-if="conflictToResolve.remote.author"><strong>Auteur :</strong> {{ conflictToResolve.remote.author }}</p>
              <p v-if="conflictToResolve.remote.date"><strong>Date :</strong> {{ conflictToResolve.remote.date }}</p>
              <p v-if="conflictToResolve.remote.likes !== undefined"><strong>Likes :</strong> {{ conflictToResolve.remote.likes }}</p>
              <p class="version-rev"><strong>Révision :</strong> {{ conflictToResolve.remote._rev }}</p>
            </div>
            <button class="btn-keep-remote" @click="keepRemoteVersion">
              ✓ Garder cette version
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// imports Vue + PouchDB
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

// interface d'un post
interface Post {
  _id?: string
  _rev?: string
  type?: string
  title: string
  author: string
  content: string
  date: string
  likes?: number
  _attachments?: PouchDB.Core.Attachments
}

// interface d'un commentaire
interface CommentDoc {
  _id?: string
  _rev?: string
  type: 'comment'
  postId: string
  author: string
  content: string
  date: string
}

// structure pour gérer un conflit
interface ConflictResolution {
  docId: string
  local: any
  remote: any
  allRevs: string[]
}

// refs vers les bases locale et distante
const localDB = ref<any | null>(null)
const remoteDB = ref<any | null>(null)
const remoteUrl = 'http://admin:admin@localhost:5984/test_infradonn2'

// listes de posts et commentaires
const posts = ref<Post[]>([])
const comments = ref<CommentDoc[]>([])
const filteredPosts = ref<Post[]>([])
const syncStatus = ref<string>('Initialisation...')
let liveSync: any = null
let changesFeed: any = null

// mode offline activé ou non
const isOffline = ref(false)

// données du nouveau post / commentaire
const newPost = ref<Post>({ title: '', author: '', content: '', date: new Date().toLocaleString(), likes: 0 })
const newComment = ref<any>({ author: '', content: '' })
const activeCommentsPostId = ref<string | null>(null)
const factoryCount = ref<number>(50)
const searchQuery = ref('')
const sortBy = ref('none')

// contrôle de l’affichage (top 10 ou tout)
const showAllPosts = ref(false)

// gestion des conflits de docs
const conflictToResolve = ref<ConflictResolution | null>(null)
const conflictQueue = ref<ConflictResolution[]>([])

// suivi des révisions modifiées en local
const locallyModifiedRevs = ref(new Map<string, string>())

// map idPost -> données de média
const attachmentsData = ref(new Map<string, { url: string, name: string, type: string }>())

// ensemble des conflits déjà traités
const processedConflicts = ref(new Set<string>())

// récupère le média associé à un post
const postMedia = (postId?: string) => {
  if (!postId) return null
  return attachmentsData.value.get(postId)
}

// helpers pour tester le type de fichier
const isImage = (type: string) => type.startsWith('image/')
const isVideo = (type: string) => type.startsWith('video/')

// charge l'attachement d'un post depuis PouchDB
const loadAttachmentForPost = async (post: Post) => {
  if (!localDB.value || !post._id || !post._attachments) return
  if (attachmentsData.value.has(post._id)) return

  const attachmentNames = Object.keys(post._attachments)
  if (attachmentNames.length === 0) return

  const attachmentName = attachmentNames[0]

  try {
    const blob = await localDB.value.getAttachment(post._id, attachmentName)
    if (blob) {
      const url = URL.createObjectURL(blob)
      attachmentsData.value.set(post._id, {
        url,
        name: attachmentName!,
        type: blob.type!
      })
    }
  } catch (e) {
    console.error("Erreur lors du chargement de l'attachement pour le post", post._id, ":", e)
  }
}

// posts affichés (top 10 ou tout) + chargement des médias
const displayedPostsWithMedia = computed(() => {
  const postsToDisplay = showAllPosts.value ? filteredPosts.value : filteredPosts.value.slice(0, 10)
  postsToDisplay.forEach(post => {
    if (post._attachments && Object.keys(post._attachments).length > 0) {
      loadAttachmentForPost(post)
    }
  })
  return postsToDisplay
})

// création des index PouchDB (recherche/tri)
const initIndexes = async () => {
  if (!localDB.value) return
  
  await localDB.value.createIndex({
    index: { fields: ['type', 'likes'] }
  })
  
  await localDB.value.createIndex({
    index: { fields: ['type', 'title', 'author'] }
  })
  
  await localDB.value.createIndex({
    index: { fields: ['type', 'postId'] }
  })
}

// init de la base locale + sync initiale
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

// récupère tous les posts + commentaires de la base locale
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
    
    attachmentsData.value.forEach(data => URL.revokeObjectURL(data.url))
    attachmentsData.value.clear()

    await applySearchSort()
  } catch (e) {
    console.error("Erreur lors de la récupération de toutes les données locales:", e)
  }
}

// écoute les changements locaux (live) pour MAJ l'affichage
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

// quand un conflit est détecté, on prépare les 2 versions
const handleConflictDetected = async (doc: any) => {
  if (!localDB.value || !doc._conflicts || doc._conflicts.length === 0) return

  const docId = doc._id
  
  const conflictKey = `${docId}:${doc._rev}:${doc._conflicts.join(',')}`
  
  if (processedConflicts.value.has(conflictKey)) {
    return
  }
  processedConflicts.value.add(conflictKey)

  try {
    const winnerDoc = doc
    
    const conflictRev = doc._conflicts[0]
    const loserDoc = await localDB.value.get(docId, { rev: conflictRev })

    const localRev = locallyModifiedRevs.value.get(docId)
    
    let localDoc: any
    let remoteDoc: any
    
    const getRevNum = (rev: string | undefined): number => {
      if (!rev) return 0
      const parts = rev.split('-')
      return parts[0] ? parseInt(parts[0], 10) : 0
    }
    
    if (localRev && typeof localRev === 'string') {
      const winnerRevNum = getRevNum(winnerDoc._rev)
      const loserRevNum = getRevNum(loserDoc._rev)
      const localRevNum = getRevNum(localRev)
      
      if (Math.abs(winnerRevNum - localRevNum) <= Math.abs(loserRevNum - localRevNum)) {
        localDoc = winnerDoc
        remoteDoc = loserDoc
      } else {
        localDoc = loserDoc
        remoteDoc = winnerDoc
      }
    } else {
      const winnerDate = new Date(winnerDoc.date || 0).getTime()
      const loserDate = new Date(loserDoc.date || 0).getTime()
      
      if (winnerDate >= loserDate) {
        localDoc = winnerDoc
        remoteDoc = loserDoc
      } else {
        localDoc = loserDoc
        remoteDoc = winnerDoc
      }
    }

    const allRevs = [winnerDoc._rev, ...doc._conflicts]

    const conflict: ConflictResolution = {
      docId,
      local: localDoc,
      remote: remoteDoc,
      allRevs
    }

    conflictQueue.value.push(conflict)
    
    if (!isOffline.value && !conflictToResolve.value) {
      showNextConflict()
    }
  } catch (e) {
    console.error("Erreur lors de la détection du conflit:", e)
    processedConflicts.value.delete(conflictKey)
  }
}

// affiche le prochain conflit de la file
const showNextConflict = () => {
  if (conflictQueue.value.length > 0) {
    conflictToResolve.value = conflictQueue.value.shift() || null
  } else {
    conflictToResolve.value = null
  }
}

// l'utilisateur choisit de garder la version locale
const keepLocalVersion = async () => {
  if (!conflictToResolve.value || !localDB.value) return

  try {
    const { docId, local } = conflictToResolve.value
    
    const docsToUpdate: any[] = []
    
    const versionToKeep = { ...local }
    delete versionToKeep._conflicts
    
    const currentDoc = await localDB.value.get(docId, { conflicts: true })
    versionToKeep._rev = currentDoc._rev
    
    docsToUpdate.push(versionToKeep)
    
    if (currentDoc._conflicts) {
      for (const rev of currentDoc._conflicts) {
        docsToUpdate.push({
          _id: docId,
          _rev: rev,
          _deleted: true
        })
      }
    }
    
    await localDB.value.bulkDocs(docsToUpdate)
    
    const updatedDoc = await localDB.value.get(docId)
    locallyModifiedRevs.value.set(docId, updatedDoc._rev)

    syncStatus.value = 'Conflit résolu (version locale conservée) ✓'
    showNextConflict()
    await fetchAllLocalFromDB()
  } catch (e) {
    console.error("Erreur lors de la conservation de la version locale:", e)
    syncStatus.value = 'Erreur résolution conflit ✗'
  }
}

// l'utilisateur choisit de garder la version distante
const keepRemoteVersion = async () => {
  if (!conflictToResolve.value || !localDB.value) return

  try {
    const { docId, remote } = conflictToResolve.value
    
    const currentDoc = await localDB.value.get(docId, { conflicts: true })
    
    const docsToUpdate: any[] = []
    
    const versionToKeep = { ...remote }
    delete versionToKeep._conflicts
    versionToKeep._rev = currentDoc._rev
    
    docsToUpdate.push(versionToKeep)
    
    if (currentDoc._conflicts) {
      for (const rev of currentDoc._conflicts) {
        docsToUpdate.push({
          _id: docId,
          _rev: rev,
          _deleted: true
        })
      }
    }
    
    await localDB.value.bulkDocs(docsToUpdate)
    
    const updatedDoc = await localDB.value.get(docId)
    locallyModifiedRevs.value.set(docId, updatedDoc._rev)

    syncStatus.value = 'Conflit résolu (version distante conservée) ✓'
    showNextConflict()
    await fetchAllLocalFromDB()
  } catch (e) {
    console.error("Erreur lors de la conservation de la version distante:", e)
    syncStatus.value = 'Erreur résolution conflit ✗'
  }
}

// démarre la synchro live avec la base distante
const startLiveSync = () => {
  if (!localDB.value || !remoteDB.value || isOffline.value) return
  stopLiveSync()
  
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
        syncStatus.value = 'Live sync'
      }
    })
    .on('active', () => {
      syncStatus.value = 'Synchronisation en cours...'
    })
    .on('denied', (err: any) => {
      console.error("Sync denied:", err)
      syncStatus.value = 'Accès à la BDD distante refusé ✗'
    })
    .on('complete', () => {
      syncStatus.value = 'Sync complete'
    })
    .on('error', (err: any) => {
      console.error("Sync error:", err)
      syncStatus.value = `Erreur sync ✗: ${err.name || err.message}`
    })
}

// arrête la synchro live (utilisé pour le mode offline)
const stopLiveSync = () => {
  if (liveSync && liveSync.cancel) {
    liveSync.cancel()
    liveSync = null
  }
}

// switch online/offline
const toggleOffline = async () => {
  isOffline.value = !isOffline.value

  if (isOffline.value) {
    stopLiveSync()
    syncStatus.value = 'Offline'
  } else {
    syncStatus.value = 'Retour en ligne...'

    if (!conflictToResolve.value && conflictQueue.value.length > 0) {
      showNextConflict()
    }

    startLiveSync()
  }
}

// enregistre qu'un doc a été modifié en local
const trackLocalModification = (docId: string, rev: string) => {
  locallyModifiedRevs.value.set(docId, rev)
}

// création d’un nouveau post
const createPost = async () => {
  if (!localDB.value) return
  
  const doc: Post = {
    _id: `post:${Date.now()}`,
    type: 'post',
    title: newPost.value.title,
    author: newPost.value.author,
    content: newPost.value.content,
    date: new Date().toLocaleString(),
    likes: 0,
  }
  
  try {
    const result = await localDB.value.put(doc)
    trackLocalModification(doc._id!, result.rev)
    
    newPost.value = { title: '', author: '', content: '', date: new Date().toLocaleString(), likes: 0 }
    syncStatus.value = 'Modification locale (sync auto)'
  } catch (e) {
    console.error("Erreur lors de la création du post:", e)
  }
}

// modification d’un post existant (via prompt)
const updatePost = async (post: Post) => {
  const title = prompt('Nouveau titre:', post.title)
  if (title === null) return
  const content = prompt('Nouveau contenu:', post.content)
  if (content === null) return
  
  try {
    const doc = await localDB.value.get(post._id)
    doc.title = title
    doc.content = content
    const result = await localDB.value.put(doc)
    trackLocalModification(post._id!, result.rev)
    
    syncStatus.value = 'Modification locale (sync auto)'
  } catch (e) {
    console.error("Erreur lors de la mise à jour du post:", e)
  }
}

// suppression d’un post
const deletePost = async (post: Post) => {
  if (!confirm('Supprimer ce post ?')) return
  
  try {
    const doc = await localDB.value.get(post._id)
    await localDB.value.remove(doc)
    locallyModifiedRevs.value.delete(post._id!)
    
    syncStatus.value = 'Suppression locale (sync auto)'
  } catch (e) {
    console.error("Erreur lors de la suppression du post:", e)
  }
}

// gestion du like d’un post
const likePost = async (post: Post) => {
  try {
    const doc = await localDB.value.get(post._id)
    doc.likes = (doc.likes || 0) + 1
    const result = await localDB.value.put(doc)
    trackLocalModification(post._id!, result.rev)
    
    syncStatus.value = 'Like ajouté (sync auto)'
  } catch (e) {
    console.error("Erreur lors du like du post:", e)
  }
}

// ouvre / ferme la section commentaires d’un post
const openComments = (post: Post) => {
  activeCommentsPostId.value = activeCommentsPostId.value === post._id ? null : post._id || null
}

// retourne les commentaires liés à un post
const commentsByPost = (postId: string | undefined) => {
  if (!postId) return []
  return comments.value.filter(c => c.postId === postId)
}

// nombre de commentaires d’un post
const countComments = (postId: string | undefined) => commentsByPost(postId).length

// ajout d’un commentaire pour un post
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
    const result = await localDB.value.put(doc)
    trackLocalModification(doc._id!, result.rev)
    
    newComment.value = { author: '', content: '' }
    syncStatus.value = 'Commentaire ajouté (sync auto)'
  } catch (e) {
    console.error("Erreur lors de l'ajout du commentaire:", e)
  }
}

// modification d’un commentaire
const updateComment = async (c: CommentDoc) => {
  const content = prompt('Nouveau commentaire:', c.content)
  if (content === null) return
  
  try {
    const doc = await localDB.value.get(c._id)
    doc.content = content
    const result = await localDB.value.put(doc)
    trackLocalModification(c._id!, result.rev)
    
    syncStatus.value = 'Commentaire modifié (sync auto)'
  } catch (e) {
    console.error("Erreur lors de la mise à jour du commentaire:", e)
  }
}

// suppression d’un commentaire
const deleteComment = async (c: CommentDoc) => {
  if (!confirm('Supprimer ce commentaire ?')) return
  
  try {
    const doc = await localDB.value.get(c._id)
    await localDB.value.remove(doc)
    locallyModifiedRevs.value.delete(c._id!)
    
    syncStatus.value = 'Commentaire supprimé (sync auto)'
  } catch (e) {
    console.error("Erreur lors de la suppression du commentaire:", e)
  }
}

// ajout / remplacement d’un fichier attaché à un post
const handleFileSelect = async (event: Event, postId?: string) => {
  if (!postId || !localDB.value) return

  const input = event.target as HTMLInputElement
  if (!input.files || input.files.length === 0) {
    console.warn("Aucun fichier sélectionné.")
    return
  }

  const file = input.files[0]
  if (!file) {
    console.error("Le fichier sélectionné est invalide.")
    return
  }

  try {
    const doc = await localDB.value.get(postId)
    const result = await localDB.value.putAttachment(doc._id, file.name, doc._rev, file, file.type)
    trackLocalModification(postId, result.rev)
    
    syncStatus.value = 'Média ajouté/remplacé (sync auto)'

    if (attachmentsData.value.has(postId)) {
      URL.revokeObjectURL(attachmentsData.value.get(postId)!.url)
      attachmentsData.value.delete(postId)
    }
    await fetchAllLocalFromDB()
  } catch (e) {
    console.error("Erreur lors de la sauvegarde de l'attachement:", e)
    syncStatus.value = 'Erreur ajout/remplacement média ✗'
  } finally {
    input.value = ''
  }
}

// suppression du média lié à un post
const deleteMediaAttachment = async (post: Post) => {
  if (!post._id || !localDB.value || !post._attachments) return
  if (!confirm('Supprimer le média associé à ce post ?')) return

  const attachmentNames = Object.keys(post._attachments)
  if (attachmentNames.length === 0) {
    console.warn("Aucun attachement à supprimer pour ce post.")
    return
  }

  const attachmentName = attachmentNames[0]

  try {
    const doc = await localDB.value.get(post._id)
    const result = await localDB.value.removeAttachment(doc._id, attachmentName, doc._rev)
    trackLocalModification(post._id, result.rev)
    
    syncStatus.value = 'Média supprimé (sync auto)'

    if (attachmentsData.value.has(post._id)) {
      URL.revokeObjectURL(attachmentsData.value.get(post._id)!.url)
      attachmentsData.value.delete(post._id)
    }
    await fetchAllLocalFromDB()
  } catch (e) {
    console.error("Erreur lors de la suppression de l'attachement:", e)
    syncStatus.value = 'Erreur suppression média ✗'
  }
}

// génération en masse de posts "factory"
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
    const results = await localDB.value.bulkDocs(docs)
    results.forEach((result: any, index: number) => {
      if (result.ok) {
        trackLocalModification(docs[index]._id, result.rev)
      }
    })
    
    syncStatus.value = `${n} posts générés (sync auto)`
  } catch (e) {
    console.error("Erreur lors de la génération de posts:", e)
  }
}

// applique la recherche + tri sur les posts (via PouchDB find)
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
    console.warn("Erreur PouchDB find, fallback sur le tri/filtre manuel:", e)
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

// réinitialise recherche et tri
const clearSearch = async () => {
  searchQuery.value = ''
  sortBy.value = 'none'
  await applySearchSort()
  showAllPosts.value = false
}

// bascule top 10 / tous les posts
const toggleShowAll = () => {
  showAllPosts.value = !showAllPosts.value
}

// on lance l’init à l’arrivée sur le composant
onMounted(() => { initDB() })

// nettoyage des listeners et des URLs d’objets
onBeforeUnmount(() => {
  stopLiveSync()
  if (changesFeed && changesFeed.cancel) changesFeed.cancel()
  attachmentsData.value.forEach(data => URL.revokeObjectURL(data.url))
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
}

.version-card.local-version {
  border-color: #4caf50;
  background: #f1f8e9;
}

.version-card.local-version:hover {
  border-color: #2e7d32;
}

.version-card.remote-version {
  border-color: #2196f3;
  background: #e3f2fd;
}

.version-card.remote-version:hover {
  border-color: #1565c0;
}

.version-card h3 {
  color: #13407b;
  margin-bottom: 1rem;
  font-size: 1.1rem;
  text-align: center;
  padding-bottom: 0.8rem;
  border-bottom: 2px solid #e0e0e0;
}

.local-version h3 {
  color: #2e7d32;
  border-color: #a5d6a7;
}

.remote-version h3 {
  color: #1565c0;
  border-color: #90caf9;
}

.version-details {
  background: white;
  padding: 1rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  min-height: 180px;
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

.media-upload-button {
  padding: 0.5rem 1.1rem;
  font-size: 1rem;
  font-weight: 500;
  background: #28a745;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background 0.15s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.media-upload-button:hover {
  background: #218838;
}

.post-media-display {
  margin-top: 1rem;
  padding: 1rem;
  background: #f0f7f0;
  border: 1px solid #c3e6cb;
  border-radius: 7px;
  text-align: center;
}

.post-media-display h4 {
  color: #28a745;
  margin-bottom: 0.8rem;
}

.responsive-media-img,
.responsive-media-video {
  max-width: 100%;
  height: auto;
  border-radius: 5px;
  display: block;
  margin: 0 auto;
}

.media-info {
  font-size: 0.95rem;
  color: #495057;
  margin-top: 0.5rem;
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