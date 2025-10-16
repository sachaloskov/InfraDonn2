<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

interface Comment {
  id: string
  author: string
  content: string
  date: string
}

interface Post {
  id: number
  title: string
  author: string
  content: string
  date: string
  comments?: Comment[]
}

interface DBRecord {
  _id: string
  _rev?: string
  post: Post
}

const storage = ref<any>(null)
const postsData = ref<DBRecord[]>([])

const newPost = ref<Post>({
  id: Date.now(),
  title: '',
  author: '',
  content: '',
  date: new Date().toISOString(),
  comments: []
})

const newComment = ref<Record<number, Partial<Comment>>>({})

const getCommentInput = (postId: number) => {
  if (!newComment.value[postId]) {
    newComment.value[postId] = { author: '', content: '' }
  }
  return newComment.value[postId]
}

const initDatabase = async () => {
  const db = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
  try {
    const info = await db.info()
    console.log('✅ Connecté à la base :', info.db_name)
    storage.value = db
  } catch (err) {
    console.error('Erreur de connexion à la base :', err)
  }
}

const fetchData = async () => {
  if (!storage.value) return
  const result = await storage.value.allDocs({ include_docs: true })
  postsData.value = result.rows
    .filter((row: any) => row.doc && row.doc.post)
    .map((row: any) => row.doc as DBRecord)
}

const addPost = async () => {
  if (!storage.value) return
  if (!newPost.value.title || !newPost.value.content) {
    alert('Veuillez remplir tous les champs.')
    return
  }

  const doc: DBRecord = {
    _id: new Date().toISOString(),
    post: { ...newPost.value, id: Date.now() }
  }

  await storage.value.put(doc)
  await fetchData()

  newPost.value = {
    id: Date.now(),
    title: '',
    author: '',
    content: '',
    date: new Date().toISOString(),
    comments: []
  }
}

const updatePost = async (record: DBRecord) => {
  if (!storage.value) return
  try {
    const existing = await storage.value.get(record._id)
    await storage.value.put({ ...existing, post: record.post })
    await fetchData()
  } catch (err) {
    console.error('Erreur de mise à jour du post :', err)
  }
}

const deletePost = async (record: DBRecord) => {
  if (!storage.value) return
  if (!confirm('Voulez-vous vraiment supprimer ce post ?')) return

  const doc = await storage.value.get(record._id)
  await storage.value.remove(doc)
  await fetchData()
}

const addComment = async (record: DBRecord) => {
  const post = record.post
  const c = getCommentInput(post.id)

  if (!c.author || !c.content) {
    alert('Veuillez remplir tous les champs du commentaire.')
    return
  }

  const comment: Comment = {
    id: Date.now().toString(),
    author: c.author,
    content: c.content,
    date: new Date().toISOString()
  }

  post.comments = post.comments || []
  post.comments.push(comment)

  await updatePost(record)
  newComment.value[post.id] = { author: '', content: '' }
}

const deleteComment = async (record: DBRecord, commentId: string) => {
  record.post.comments = record.post.comments?.filter(c => c.id !== commentId)
  await updatePost(record)
}

onMounted(async () => {
  await initDatabase()
  await fetchData()
})
</script>

<template>
  <div class="page">
    <h1>📰 Gestion des Posts</h1>

    <form @submit.prevent="addPost" class="form-post">
      <h2>✍️ Ajouter un nouveau post</h2>
      <div class="form-grid">
        <input v-model="newPost.title" placeholder="Titre" />
        <input v-model="newPost.author" placeholder="Auteur" />
      </div>
      <textarea v-model="newPost.content" placeholder="Contenu"></textarea>
      <button type="submit" class="btn btn-primary">➕ Ajouter</button>
    </form>

    <article v-for="record in postsData" :key="record._id" class="card">
      <header class="card-header">
        <div>
          <h2>{{ record.post.title }}</h2>
          <p class="meta">
            ✍️ {{ record.post.author }} — 
            <span>{{ new Date(record.post.date).toLocaleString('fr-CH') }}</span>
          </p>
        </div>
        <div class="actions">
          <button class="btn btn-edit" @click="updatePost(record)">✏️ Modifier</button>
          <button class="btn btn-delete" @click="deletePost(record)">🗑️ Supprimer</button>
        </div>
      </header>

      <p class="content">{{ record.post.content }}</p>

      <section v-if="record.post.comments?.length" class="comments">
        <h3>💬 Commentaires :</h3>
        <ul>
          <li v-for="comment in record.post.comments" :key="comment.id" class="comment">
            <div>
              <p>{{ comment.content }}</p>
              <p class="comment-meta">
                — {{ comment.author }}, {{ new Date(comment.date).toLocaleString('fr-CH') }}
              </p>
            </div>
            <button class="btn btn-delete small" @click="deleteComment(record, comment.id)">🗑️</button>
          </li>
        </ul>
      </section>

      <form @submit.prevent="addComment(record)" class="form-comment">
        <h4>Ajouter un commentaire :</h4>
        <div class="form-grid">
          <input v-model="getCommentInput(record.post.id).author" placeholder="Votre nom" />
          <input v-model="getCommentInput(record.post.id).content" placeholder="Votre commentaire" />
        </div>
        <button type="submit" class="btn btn-secondary">💬 Ajouter commentaire</button>
      </form>

      <details class="json">
        <summary>Voir le JSON brut</summary>
        <pre>{{ JSON.stringify(record.post, null, 2) }}</pre>
      </details>
    </article>

    <p v-if="!postsData.length" class="empty">Aucun post pour le moment.</p>
  </div>
</template>

<style scoped>
/* --- Mise en page générale --- */
.page {
  max-width: 950px;
  margin: 40px auto;
  padding: 20px;
  font-family: "Segoe UI", Roboto, sans-serif;
  color: #333;
  background: #f9fbfd;
}

/* --- Titres --- */
h1 {
  text-align: center;
  color: #1e3a8a;
  margin-bottom: 40px;
}
h2 {
  color: #222;
  margin-bottom: 8px;
}
h3, h4 {
  color: #444;
}

/* --- Formulaire ajout post --- */
.form-post {
  background: #fff;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.08);
  margin-bottom: 40px;
}
.form-post .form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-bottom: 12px;
}
.form-post input,
.form-post textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 14px;
  resize: vertical;
}

/* --- Boutons --- */
.btn {
  cursor: pointer;
  border: none;
  border-radius: 6px;
  padding: 8px 14px;
  font-size: 14px;
  transition: all 0.2s;
}
.btn-primary {
  background: #2563eb;
  color: white;
}
.btn-primary:hover {
  background: #1e40af;
}
.btn-secondary {
  background: #10b981;
  color: white;
}
.btn-secondary:hover {
  background: #059669;
}
.btn-edit {
  background: #f59e0b;
  color: white;
}
.btn-edit:hover {
  background: #d97706;
}
.btn-delete {
  background: #ef4444;
  color: white;
}
.btn-delete:hover {
  background: #b91c1c;
}
.btn.small {
  padding: 4px 8px;
  font-size: 12px;
}

/* --- Cartes de post --- */
.card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  margin-bottom: 30px;
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}
.meta {
  color: #777;
  font-size: 13px;
}
.content {
  margin-top: 15px;
  line-height: 1.6;
}

/* --- Commentaires --- */
.comments {
  background: #f5f7fb;
  border-radius: 8px;
  padding: 12px 16px;
  margin-top: 20px;
}
.comments h3 {
  margin-bottom: 10px;
}
.comment {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: white;
  border-left: 4px solid #3b82f6;
  border-radius: 6px;
  padding: 10px;
  margin-bottom: 10px;
}
.comment-meta {
  font-size: 12px;
  color: #777;
  margin-top: 4px;
}

/* --- Formulaire commentaire --- */
.form-comment {
  margin-top: 15px;
  background: #f9fafb;
  padding: 10px;
  border-radius: 8px;
}
.form-comment .form-grid {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 10px;
  margin-bottom: 10px;
}
.form-comment input {
  padding: 8px;
  border-radius: 6px;
  border: 1px solid #ccc;
}

/* --- JSON brut --- */
.json summary {
  cursor: pointer;
  color: #2563eb;
  margin-top: 12px;
}
.json pre {
  background: #f1f5f9;
  padding: 10px;
  border-radius: 6px;
  font-size: 13px;
  overflow-x: auto;
}

/* --- Message vide --- */
.empty {
  text-align: center;
  color: #888;
  margin-top: 40px;
}
</style>