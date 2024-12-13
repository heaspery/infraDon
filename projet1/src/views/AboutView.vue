<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

// Déclaration de l'interface Post
interface Post {
  _id?: string;
  _rev?: string; // Pour gérer les révisions
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
  };
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      isFetched: false, // Indique si les données ont été récupérées
    };
  },

  mounted() {
    this.initDatabase(); // Initialisation de la base de données à la montée du composant
  },

  methods: {
    addDocument() {
      const newDocument: Post = {
        post_name: 'Nouveau post',
        post_content: 'Contenu du nouveau post',
        attributes: {
          creation_date: new Date().toISOString(),
        },
      };
      this.postDocument(newDocument);
    },

    postDocument(document: Post) {
      if (!this.storage) {
        console.error("Base de données non initialisée.");
        return;
      }

      this.storage
        .post(document)
        .then(() => {
          console.log('Document ajouté avec succès');
          if (this.isFetched) {
            this.fetchData(); // Met à jour les données uniquement si déjà récupérées
          }
        })
        .catch((error) => {
          console.error('Erreur lors de l\'ajout du document:', error);
        });
    },

    fetchDocument(id: string | null) {
      if (!this.storage || !id) {
        console.warn('Base de données non initialisée ou ID invalide.');
        return;
      }

      this.storage
        .get(id)
        .then((doc) => {
          this.document = doc as Post;
          console.log('Document récupéré:', doc);
        })
        .catch((error) => {
          console.error('Erreur lors de la récupération du document:', error);
        });
    },

    fetchData() {
      if (!this.storage) {
        console.error('Base de données non initialisée.');
        return;
      }

      this.storage
        .allDocs({
          include_docs: true,
          attachments: false, // Désactivation des pièces jointes pour de meilleures performances si non nécessaires
        })
        .then((result: any) => {
          console.log('Récupération des données réussie', result);
          this.postsData = result.rows.map((row: any) => row.doc as Post);
          this.isFetched = true;
        })
        .catch((error: any) => {
          console.error('Erreur lors de la récupération des données:', error);
        });
    },

    initDatabase() {
      console.log('Connexion à la base de données distante');
      this.storage = new PouchDB('http://admin:M3w24@localhost:5984/post');

      this.storage
        .info()
        .then((info) => {
          console.log('Connecté à la base de données distante:', info);
        })
        .catch((error) => {
          console.warn('Échec de la connexion à la base de données distante:', error);
        });
    },

    deleteDocument(id: string) {
      if (!this.storage) {
        console.error('Base de données non initialisée.');
        return;
      }

      this.storage
        .get(id)
        .then((doc) => this.storage!.remove(doc))
        .then(() => {
          console.log('Document supprimé avec succès');
          if (this.isFetched) {
            this.fetchData(); // Met à jour les données uniquement si déjà récupérées
          }
        })
        .catch((error) => {
          console.error('Erreur lors de la suppression du document:', error);
        });
    },
  },
};
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

  <button @click="fetchData">Récupérer les données de la base distante</button>
  <button @click="addDocument">Ajouter un document</button>
</template>
