<script setup>
import { ref } from "vue";
import SpotifyPlayer from "../components/SpotifyPlayer.vue";
import { searchSpotify } from "../services/spotify";

const types = {
  track: "Chansons",
  artist: "Artistes",
  album: "Albums",
};

const query = ref("");
const searchType = ref("track");

const activeQuery = ref("");
const activeType = ref("track");

const items = ref([]);
const selectedTrack = ref(null);
const searched = ref(false);
const loading = ref(false);
const error = ref("");
const offset = ref(0);
const hasNext = ref(false);

async function loadResults(text, type, pageOffset) {
  if (loading.value || !text.trim()) return;

  loading.value = true;
  error.value = "";

  try {
    const result = await searchSpotify(text, type, pageOffset);

    items.value = result.items;
    activeQuery.value = text;
    activeType.value = type;
    offset.value = pageOffset;
    hasNext.value = result.hasNext && pageOffset < 1000;
    searched.value = true;
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
}

function search() {
  return loadResults(query.value.trim(), searchType.value, 0);
}

function changePage(direction) {
  return loadResults(
    activeQuery.value,
    activeType.value,
    offset.value + direction * 10
  );
}

function imageUrl(item) {
  const images =
    activeType.value === "track" ? item.album?.images : item.images;

  return images?.[0]?.url || "";
}

function artistNames(item) {
  return (item.artists || []).map((artist) => artist.name).join(", ");
}

function duration(milliseconds) {
  const seconds = Math.floor((milliseconds || 0) / 1000);

  return `${Math.floor(seconds / 60)}:${String(seconds % 60).padStart(2, "0")}`;
}
</script>

<template>
  <div class="music-app">
    <header class="topbar">
      <a href="/" class="brand">Vue Music</a>
      <span class="subtitle">Recherche musicale avec Spotify</span>
    </header>

    <main class="content">
      <section class="intro">
        <span class="eyebrow">EXPLORER AVEC SPOTIFY</span>
        <h1>Explorez votre musique</h1>
        <p>Recherchez des chansons, des artistes et des albums.</p>
      </section>

      <form class="search-form" @submit.prevent="search">
        <div class="field query-field">
          <label for="query">Votre recherche</label>

          <input
            id="query"
            v-model="query"
            type="search"
            placeholder="Exemple : Toto, Adele..."
            maxlength="200"
            required
          />
        </div>

        <div class="field">
          <label for="type">Type de recherche</label>

          <select id="type" v-model="searchType">
            <option
              v-for="(label, value) in types"
              :key="value"
              :value="value"
            >
              {{ label }}
            </option>
          </select>
        </div>

        <button :disabled="loading || !query.trim()">
          {{ loading ? "Recherche..." : "Rechercher" }}
        </button>
      </form>

      <p v-if="error" class="error" role="alert">
        {{ error }}
      </p>

      <div v-if="selectedTrack" class="player-container">
        <SpotifyPlayer :track-id="selectedTrack.id" />

        <button class="outline" @click="selectedTrack = null">
          Fermer le lecteur
        </button>
      </div>

      <section class="results" :aria-busy="loading">
        <h2 v-if="searched">
          {{ types[activeType] }} — « {{ activeQuery }} »
        </h2>

        <p v-if="loading" class="muted" role="status">
          Chargement des résultats...
        </p>

        <p v-else-if="!searched" class="muted">
          Choisissez un type et lancez une recherche.
        </p>

        <p v-else-if="items.length === 0" class="muted">
          Aucun résultat trouvé.
        </p>

        <div
          class="result-list"
          :class="{ 'card-grid': activeType !== 'track' }"
        >
          <article
            v-for="(item, index) in items"
            :key="`${activeType}-${item.id || index}`"
            class="result"
            :class="{
              'track-row': activeType === 'track',
              'entity-card': activeType !== 'track',
              selected:
                activeType === 'track' &&
                selectedTrack?.id === item.id,
            }"
          >
            <img
              v-if="imageUrl(item)"
              :src="imageUrl(item)"
              :alt="item.name"
              class="artwork"
              loading="lazy"
            />

            <div v-else class="artwork placeholder">Sans image</div>

            <div class="details">
              <h3>{{ item.name }}</h3>

              <template v-if="activeType === 'track'">
                <p>{{ artistNames(item) }}</p>
                <p class="small">
                  {{ item.album?.name }} · {{ duration(item.duration_ms) }}
                </p>
              </template>

              <template v-else-if="activeType === 'album'">
                <p>{{ artistNames(item) }}</p>
                <p class="small">
                  {{ item.release_date || "Date non renseignée" }}
                  <span v-if="item.total_tracks != null">
                    · {{ item.total_tracks }} titres
                  </span>
                </p>
              </template>

              <p v-else>Artiste</p>

              <a
                v-if="item.external_urls?.spotify"
                :href="item.external_urls.spotify"
                target="_blank"
                rel="noopener noreferrer"
              >
                Voir sur Spotify
              </a>
            </div>

            <button
              v-if="activeType === 'track' && item.id"
              class="player-button"
              :aria-label="`Afficher le lecteur de ${item.name}`"
              :aria-pressed="selectedTrack?.id === item.id"
              @click="selectedTrack = item"
            >
              {{ selectedTrack?.id === item.id ? "Sélectionné" : "Lecteur" }}
            </button>
          </article>
        </div>

        <nav
          v-if="searched && (offset > 0 || hasNext)"
          class="pagination"
          aria-label="Pages de résultats"
        >
          <button
            :disabled="loading || offset === 0"
            @click="changePage(-1)"
          >
            Précédent
          </button>

          <span>Page {{ offset / 10 + 1 }}</span>

          <button
            :disabled="loading || !hasNext"
            @click="changePage(1)"
          >
            Suivant
          </button>
        </nav>
      </section>

      <footer>
        Données provenant de Spotify. Lecture via Spotify Embed.
      </footer>
    </main>
  </div>
</template>

<style scoped>
.music-app {
  min-height: 100vh;
  background: #121212;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 16px;
  padding: 22px 5%;
  background: #191919;
  border-bottom: 1px solid #333;
}

.brand {
  color: #1ed760;
  font-size: 28px;
  font-weight: bold;
  text-decoration: none;
}

.subtitle,
.muted,
.intro p {
  color: #b3b3b3;
  line-height: 1.6;
}

.content {
  max-width: 1200px;
  margin: auto;
  padding: 40px 24px;
}

.intro {
  margin-bottom: 30px;
}

.eyebrow {
  color: #1ed760;
  font-size: 12px;
  font-weight: bold;
  letter-spacing: 2px;
}

.intro h1 {
  font-size: clamp(28px, 5vw, 44px);
}

.search-form {
  display: flex;
  align-items: flex-end;
  flex-wrap: wrap;
  gap: 14px;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.query-field {
  flex: 1;
  min-width: 200px;
}

label {
  font-size: 14px;
  font-weight: bold;
}

input,
select {
  min-height: 50px;
  padding: 12px 18px;
  border: 1px solid #666;
  border-radius: 12px;
  background: #242424;
  color: white;
  font: inherit;
}

input:focus,
select:focus {
  outline: 2px solid #1ed760;
  outline-offset: 2px;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.outline {
  background: transparent;
  color: white;
  border: 1px solid #777;
  padding: 10px 18px;
}

.player-container {
  max-width: 750px;
  margin: 24px auto;
}

.results {
  margin-top: 32px;
}

.result-list {
  display: grid;
  gap: 14px;
}

.result {
  padding: 16px;
  border: 1px solid transparent;
  border-radius: 12px;
  background: #1c1c1c;
}

.result.selected {
  border-color: #1ed760;
}

.track-row {
  display: grid;
  grid-template-columns: 64px minmax(0, 1fr) auto;
  align-items: center;
  gap: 16px;
}

.artwork {
  display: block;
  object-fit: contain;
}

.track-row .artwork {
  width: 64px;
  height: 64px;
}

.placeholder {
  display: grid;
  place-items: center;
  background: #333;
  color: #ccc;
  font-size: 11px;
}

.details {
  min-width: 0;
}

.details h3 {
  margin: 0 0 8px;
  font-size: 17px;
  overflow-wrap: anywhere;
}

.details p {
  margin: 5px 0;
  color: #b3b3b3;
  overflow-wrap: anywhere;
}

.details .small,
.details a {
  font-size: 12px;
}

.details a {
  display: inline-block;
  margin-top: 8px;
  color: #1ed760;
}

.player-button {
  padding: 10px 14px;
  font-size: 12px;
}

.card-grid {
  grid-template-columns: repeat(auto-fill, minmax(210px, 1fr));
}

.entity-card .artwork {
  width: 100%;
  aspect-ratio: 1;
  margin-bottom: 16px;
}

.pagination {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 14px;
  margin-top: 28px;
}

footer {
  margin-top: 35px;
  color: #b3b3b3;
  font-size: 12px;
}

@media (max-width: 600px) {
  .search-form {
    align-items: stretch;
    flex-direction: column;
  }

  .query-field {
    min-width: 0;
  }

  .track-row {
    grid-template-columns: 48px minmax(0, 1fr);
  }

  .track-row .artwork {
    width: 48px;
    height: 48px;
  }

  .player-button {
    grid-column: 2;
    justify-self: start;
  }

  .pagination {
    gap: 8px;
    font-size: 12px;
  }

  .pagination button {
    padding: 10px 12px;
  }
}
</style>