<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

// Déclaration de l'interface Post
declare interface Post {
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
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      isFetched: false // Indique si les données ont été récupérées
    }
  },

  mounted() {
    this.initDatabase() // Initialisation de la base de données à la montée du composant
  },

  methods: {
    // Méthode pour ajouter ou mettre à jour un document
    putDocument(document: Post) {
      if (this.storage) {
        this.storage
          .put(document)
          .then(() => {
            console.log('Document ajouté ou mis à jour avec succès')
            if (this.isFetched) {
              this.fetchData() // Met à jour les données uniquement si déjà récupérées
            }
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout ou de la mise à jour du document:", error)
          })
      }
    },

    addDocument() {
      const newDocument: Post = {
        post_name: 'Nouveau post',
        post_content: 'Contenu du nouveau post',
        attributes: {
          creation_date: new Date().toISOString()
        }
      }
      this.postDocument(newDocument)
    },

    postDocument(document: Post) {
      if (this.storage) {
        this.storage
          .post(document)
          .then(() => {
            console.log('Document ajouté avec succès')
            if (this.isFetched) {
              this.fetchData() // Met à jour les données uniquement si déjà récupérées
            }
          })
          .catch((error) => {
            console.error("Erreur lors de l'ajout du document:", error)
          })
      }
    },

    // Méthode pour récupérer un document par son ID
    fetchDocument(id: string | null) {
      if (this.storage && id) {
        this.storage
          .get(id)
          .then((doc) => {
            this.document = doc as Post
            console.log('Document récupéré:', doc)
          })
          .catch((error) => {
            console.error('Erreur lors de la récupération du document:', error)
          })
      }
    },

    // Méthode pour récupérer tous les documents
    fetchData() {
      if (this.storage) {
        this.storage
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then((result: any) => {
            console.log('Récupération des données réussie', result)
            this.postsData = result.rows.map((row: any) => row.doc)
            this.isFetched = true // Indique que les données ont été récupérées
          })
          .catch((error: any) => {
            console.error('Erreur lors de la récupération des données:', error)
          })
      }
    },

    // Méthode pour initialiser la connexion à la base de données distante
    initDatabase() {
      console.log('Connexion à la base de données distante')
      this.storage = new PouchDB('http://admin:M3w24@localhost:5984/post')
      this.storage
        .info()
        .then((info) => {
          console.log('Connecté à la base de données distante:', info)
        })
        .catch((error) => {
          console.warn('Échec de la connexion à la base de données distante', error)
        })
    },

    deleteDocument(id: string) {
      if (this.storage) {
        const self = this
        this.storage
          .get(id)
          .then((doc) => {
            return self?.storage?.remove(doc)
          })
          .then(() => {
            console.log('Document supprimé avec succès')
            if (this.isFetched) {
              this.fetchData() // Met à jour les données uniquement si déjà récupérées
            }
          })
          .catch((error) => {
            console.error('Erreur lors de la suppression du document:', error)
          })
      }
    }
  },

    updateDistantDatabase() {
    const remoteDB = 'http://admin:M3w24@localhost:5984/post'; // Base de données distante
    if (this.storage) {
      this.storage
        .replicate.to(remoteDB) // Synchronisation locale -> distante
        .on('complete', () => {
          console.log('Synchronisation vers la base distante terminée avec succès');
        })
        .on('error', (error) => {
          console.error('Erreur lors de la synchronisation vers la base distante:', error);
        });
    } else {
      console.warn('La base de données locale n’est pas initialisée');
    }
  },

/*       watchRemoteDatabase() {
      const remoteDB = 'http://admin:M3w24@localhost:5984/post'; // Base de données distante
      if (this.storage) {
        const syncHandler = this.storage.sync(remoteDB, {
          live: true, // Observation en temps réel
          retry: true // Réessaie en cas d’échec
        });

        syncHandler
          .on('change', (info) => {
            console.log('Modification détectée:', info);
            this.fetchData(); // Met à jour les données locales après un changement
          })
          .on('paused', (info) => {
            console.log('Synchronisation en pause:', info);
          })
          .on('active', () => {
            console.log('Synchronisation active');
          })
          .on('denied', (err) => {
            console.error('Accès refusé lors de la synchronisation:', err);
          })
          .on('error', (err) => {
            console.error('Erreur lors de la synchronisation en temps réel:', err);
          });
      } else {
        console.warn('La base de données locale n’est pas initialisée');
      }
    } */
}

</script>

<template>
  <h1>Nombre de posts: {{ postsData.length }}</h1>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name || 'Nom du post indisponible' }}
        <em style="font-size: x-small" v-if="post.attributes?.creation_date">
          - {{ post.attributes.creation_date }}
        </em>
      </div>
      <button @click="deleteDocument(post._id!)">Supprimer</button>
    </li>
  </ul>
  <p v-if="!postsData.length">Aucun post disponible.</p>

  <button @click="fetchData">Récupérer les données de la base locale</button>
  <button @click="updateDistantDatabase">Synchroniser les données vers la base distante</button>
  <button @click="addDocument">Ajouter un document</button>
</template>