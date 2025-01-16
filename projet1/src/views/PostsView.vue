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
    [key: string]: {
      content_type: string; // Type MIME du fichier
      data: File; // Le fichier lui-même
    }
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
      if (this.post_content !== '' && this.post_name !== '' && this.attachment !== null) {
        const newPost: Post = {
          post_name: this.post_name,
          post_content: this.post_content,
          attributes: {
            creation_date: new Date().toISOString()
          },
          _attachments: {
          }
        }

        // Vérifiez si un fichier est sélectionné
        if (this.attachment) {
          const file = this.attachment;
          // Ajout de l'attachement
          newPost._attachments![file.name] = {
            content_type: file.type, // Type MIME du fichier
            data: file // Le fichier lui-même
          };
        }

        this.storage
          .post(newPost)
          .then(() => {
            console.log('Document ajouté avec succès')
            this.post_name = ""
            this.post_content = ""
            this.attachment = null;
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
      const target = event.target as HTMLInputElement;
      if (target.files && target.files.length > 0) {
        this.attachment = target.files[0]; // Récupérer le premier fichier
        console.log(this.attachment); // Afficher l'objet File pour vérifier
      }
    }

  }
}
</script>

<template>

  <h1>Ajouter un post</h1>
  <form @submit.prevent id="newPost">
    <label for="post_name">Nom du post</label>
    <input v-model="post_name" id="post_name" type="text" required />
    <label for="post_content">Contenu du post</label>
    <input v-model="post_content" id="post_content" type="text" required />
    <label for="asset">Fichier du post</label>
    <input ref="asset" id="asset" type="file" @change="handleFileChange" required /> <button
      @click="createPost">Ajouter</button>
  </form>

  <div id="posts">
    <div id="allposts">

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
          <div v-if="post._attachments">
            <p>Fichier attaché:
              <a :href="'data:' + post._attachments[Object.keys(post._attachments)[0]].content_type + ';base64,' + post._attachments[Object.keys(post._attachments)[0]].data"
                download>
                {{ Object.keys(post._attachments)[0] }}
              </a>
            </p>
          </div>
          <button @click="viewPost(post._id)">Voir post</button>
        </li>
      </ul>
    </div>

    <div id="filterPost">

      <div class="topnav" style="margin-top: 50px; margin-bottom: 20px">
        <label for="filtre">Rechercher un post par nom</label>
        <input v-model="filterValue" name="filtre" type="text" placeholder="Rechercher..">
        <button @click="filterPosts(filterValue)">Appliquer</button>
      </div>
      
      <div style="margin-top :60 px" v-if="filteredPosts.length > 0">
        <h2>Résultats filtrés :</h2>
        <ul>
          <li class="post" v-for="post in filteredPosts" :key="post._id">
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

      <div v-else>
        <p>Aucun post ne correspond à votre recherche.</p>
      </div>
    </div>
  </div>
</template>


<style scoped>
.topnav,
form {
  display: flex;
  flex-direction: column;
}


#posts {
  display: flex;
  justify-content: space-between;
  gap: 20px; /* Espacement entre les deux colonnes */
}

#allposts,
#filterPost {
  flex: 1; /* Chaque section prend la même largeur */
}
</style>
