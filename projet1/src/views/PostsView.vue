<script lang="ts">
import PouchDB from 'pouchdb'
import findPlugin from 'pouchdb-find'
PouchDB.plugin(findPlugin)

// Déclaration de l'interface Post
interface Post {
  _id?: string
  _rev?: string // Pour gérer les révisions
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
  }
  _attachments?: {

  }
}

export default {
  data() {
    return {
      storage: null as PouchDB.Database | null,
      posts: [] as Post[],
      document: null as Post | null,
      post_name: '',
      post_content: '',
      filterValue: '',
      attachment: null as File | null,
      filteredPosts: <Post[]>[],
      filterApplied: false,
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

        if (this.storage) {
          this.storage.createIndex({
            index: {
              fields: ['post_name']
            }
          }).then(() => {
            console.log('Index sur "post_name" créé avec succès');
          }).catch(error => {
            console.error('Erreur lors de la création de l\'index :', error);
          });
        }

      });
    },

    filterPosts(userInput: string) {
      if (!this.storage) {
        console.error('Base de données non initialisée');
        return;
      }

      this.storage.find({
        selector: {
          post_name: { $regex: `${userInput}` } // Recherche insensible à la casse
        },
        limit: 10,
        sort: ['post_name'] // Tri des résultats
      })
        .then(result => {
          console.log('Données filtrées :', result.docs);
          this.filteredPosts = result.docs as Post[]; // Met à jour les posts filtrés à afficher
        })
        .catch(error => {
          console.error('Erreur lors du filtrage des posts :', error);
        });

    },

    fetchPosts() {
      if (!this.storage) {
        console.error('Base de données non initialisée.')
        return
      }

      this.storage
        .allDocs({
          include_docs: true,
          attachments: true,
        })
        .then((result: any) => {
          console.log('Récupération des données réussie', result)
          this.posts = result.rows.map((row: any) => row.doc as Post) // Récupérer les documents
          .filter((post: any) => !post._id.startsWith('_design/')); // Exclure ceux qui commencent par '_design/'
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
          },
          _attachments: {
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
    },

    handleFileChange(event: Event) {
      const target = event.target as HTMLInputElement; // Cast pour avoir accès à files[]
      this.attachment = target.files ? target.files[0] : null;
      console.log(this.attachment)
    },

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
    <label for="asset">Fichier du post</label>
    <input ref="asset" id="asset" type="file" @change="handleFileChange" required />    <button @click="createPost">Ajouter</button>
  </form>

  <!--Formulaire de recherche -->
  <div class="topnav" style="margin-top: 50px; margin-bottom: 20px">
    <label for="filtre">Rechercher un post par nom</label>
    <input v-model="filterValue" name="filtre" type="text" placeholder="Rechercher..">
    <button @click="filterPosts(filterValue)">Appliquer</button>
  </div>

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

  <!-- Résultats -->
  <div style="margin-top :60 px" v-if="filteredPosts.length > 0">
    <h2>Résultats filtrés :</h2>
    <ul>
      <li v-for="post in filteredPosts" :key="post._id">
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
  </div>

</template>
