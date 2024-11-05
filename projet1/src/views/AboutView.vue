<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id?: string  // 
  doc: {
    post_name: string
    post_content: string
    attributes: {
      creation_date: string
    }
  }
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      newPostName: '',
      newPostContent: ''
    }
  },

  mounted() {
    this.initLocalDatabase()
    this.fetchData()
  },

  methods: {
    addLocalDocument() {
      const db = ref(this.storage).value
      if (db) {
        const newDocument = {
          doc: {
            post_name: this.newPostName,
            post_content: this.newPostContent,
            attributes: {
              creation_date: new Date().toISOString()
            }
          }
        }

        db.post(newDocument)
          .then(() => {
            console.log('Document ajouté dans la base locale')
            this.newPostName = ''
            this.newPostContent = ''
            this.fetchData()
          })
          .catch((error) => {
            console.log('Erreur lors de l\'ajout', error)
          })
      }
    },

    // Récupère les données depuis la base locale
    fetchData() {
      const storage = ref(this.storage)
      const self = this
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then((result: any) => {
            console.log('fetchData success', result)
            self.postsData = result.rows.map((row: any) => row.doc)
          })
          .catch((error: any) => {
            console.log('fetchData error', error)
          })
      }
    },

    // Initialisation de la base de données locale
    initLocalDatabase() {
      const localDb = new PouchDB('localDb')
      if (localDb) {
        console.log("Base de données locale 'localDb' créée")
      } else {
        console.warn('Échec de la création de la base de données locale')
      }
      this.storage = localDb
    },

    initDatabase() {
      const db = new PouchDB('http://admin:M3w24@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },

    replicateDb() {
    const db = ref(this.storage).value
    if (db) {
      db.replicate.from('http://admin:M3w24@localhost:5984/post')
        .then(() => {
          console.log('Replication from remote to local successful')
          this.fetchData()  // Actualisez les données locales
        })
        .catch((error) => {
         console.error('Replication error', error)
        })
  }
}

  }
}
</script>

<template>
  <h1>Nombre de posts: {{ postsData.length }}</h1>

  <!-- Formulaire pour ajouter un nouveau post -->
  <form @submit.prevent="addLocalDocument">
    <label for="postName">Nom du post:</label>
    <input v-model="newPostName" id="postName" placeholder="Nom du post" />

    <label for="postContent">Contenu du post:</label>
    <textarea v-model="newPostContent" id="postContent" placeholder="Contenu du post"></textarea>

    <button type="submit">Ajouter le post</button>
  </form>

  <!-- Liste des posts -->
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.doc.post_name }}
        <em style="font-size: x-small" v-if="post.doc.attributes?.creation_date">
          - {{ post.doc.attributes.creation_date }}
        </em>
        <p>{{ post.doc.post_content }}</p>
      </div>
    </li>
  </ul>
</template>
