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
      post: null as Post | null,
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
        localDatabase.sync(remoteDatabaseUrl, {
          live: true,
          retry: true
        })
        //   .on('change', this.fetchPost(this.$route.params.id as string))

        this.storage = localDatabase
        this.fetchPost(this.$route.params.id as string)
      })
    },

    fetchPost(post_id: string) {
      this.storage
        ?.get(post_id)
        .then((result: any) => {
          console.log('Récupération des données réussie', result)
          this.post = result

          if (this.post) {
            this.post_name = this.post.post_name
            this.post_content = this.post.post_content
          }
        })
        .catch((error: any) => {
          console.error('Erreur lors de la récupération des données:', error)
        })
    },

    updatePost() {
      this.storage
        ?.put({
          ...this.post,
          post_name: this.post_name,
          post_content: this.post_content
        })
        .then(() => {
          console.log('Post modifié avec succès')
        })
        .catch((error) => {
          console.error('Erreur lors de la modification du document:', error)
        })
    },
    deletePost() {
      if (this.post?._id) {
        this.storage
          ?.get(this.post?._id)
          .then((doc) => this.storage?.remove(doc))
          .then(() => this.$router.push('/posts'))
          .catch((error) => {
            console.error('Erreur lors de la suppression du document:', error)
          })
      }
    }
  }
}
</script>

<template>
  <div>
    <router-link to="/posts">Retour</router-link>
    <h3>Posts : {{ $route.params.id }}</h3>
    <h2>Titre du post : {{ post_name }}</h2>
    <h2>Contenu du post : {{ post_content }}</h2>
    <br />
    <form @submit.prevent>
      <label for="post_name">Nom du post</label>
      <input v-model="post_name" id="post_name" type="text" required />
      <label for="post_content">Contenu du post</label>
      <input v-model="post_content" id="post_content" type="text" required />
      <button @click="updatePost">Modifier</button>
      <button @click="deletePost">Supprimer</button>
    </form>
  </div>
</template>
