<script lang="ts">
import PouchDB from 'pouchdb'

// Déclaration de l'interface Post
interface Post {
  _id?: string
  _rev?: string // Pour gérer les révisions
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
  }
}

export default {
  data() {
    return {
      storage: null as PouchDB.Database | null,
      posts: [] as Post[],
      document: null as Post | null,
      post_name: '',
      post_content: ''
    }
  },

  mounted() {
    this.initDatabase() // Initialisation de la base de données à la montée du composant
  },

  methods: {
    //Initialisation et mise en place de la synchronisation bilatérale des données
    initDatabase() {
      console.log('Connexion à la base de données distante')
      const localDatabaseUrl = 'posts'
      const remoteDatabaseUrl = 'http://admin:M3w24@localhost:5984/posts'

      const localDatabase = new PouchDB(localDatabaseUrl)

      localDatabase.replicate.from(remoteDatabaseUrl).on('complete', () => {
        localDatabase
          .sync(remoteDatabaseUrl, {
            live: true,
            retry: true
          })
          .on('change', this.fetchPosts)

        this.storage = localDatabase
        this.fetchPosts()
      })
    },

    fetchPosts() {
      if (!this.storage) {
        console.error('Base de données non initialisée.')
        return
      }

      this.storage
        .allDocs({
          include_docs: true,
          attachments: false // Désactivation des pièces jointes pour de meilleures performances si non nécessaires
        })
        .then((result: any) => {
          console.log('Récupération des données réussie', result)
          this.posts = result.rows.map((row: any) => row.doc as Post)
        })
        .catch((error: any) => {
          console.error('Erreur lors de la récupération des données:', error)
        })
    },

    //Création d'un nouveau post et ajout dans la base de données
    createPost() {
      if (!this.storage) {
        console.error('Base de données non initialisée.')
        return
      }

      //On ne veut pas créer un post vide
      if (this.post_content !== '' && this.post_name !== '') {
        const newPost: Post = {
          post_name: this.post_name,
          post_content: this.post_content,
          attributes: {
            creation_date: new Date().toISOString()
          }
        }

        this.storage
          .post(newPost)
          .then(() => {
            console.log('Document ajouté avec succès')
            this.post_name = ""
            this.post_content = ""
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du document:", error)
          })
      } else console.log('Ne peut pas ajouter un post vide')
    },

    viewPost(post_id?: string) {
      this.$router.push(`/posts/${post_id}`)
    }
  }
}
</script>

<template>
  <!--Formulaire pour entrer de nouveaux posts-->
  <form @submit.prevent>
    <label for="post_name">Nom du post</label>
    <input v-model="post_name" id="post_name" type="text" required />
    <label for="post_content">Contenu du post</label>
    <input v-model="post_content" id="post_content" type="text" required />
    <button @click="createPost">Ajouter</button>
  </form>

  <!--Liste des nouveaux posts-->
  <h1>Nombre de posts: {{ posts.length }}</h1>
  <ul v-if="posts.length">
    <li v-for="post in posts" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name || 'Nom du post indisponible' }}
        <em style="font-size: x-small" v-if="post.attributes?.creation_date">
          - {{ post.attributes.creation_date }}
        </em>
      </div>
      <div class="ucfirst">
        {{ post.post_content || 'Contenu du post indisponible' }}
      </div>
      <button @click="viewPost(post._id)">Voir post</button>
    </li>
  </ul>
  <p v-else>Aucun post disponible.</p>
</template>
