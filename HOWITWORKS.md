# How Moon Rider Works

## Overview

Moon Rider is a WebXR music game (Beat Saber-style) built on A-Frame (Three.js WebXR wrapper). Players hit notes synchronized to music loaded from the BeatSaver API.

---

## Playlist Management

### State (`src/state/index.js:145-148`)

```js
playlist: '',           // active playlist ID (empty = none)
playlists: require('../constants/playlists'),
playlistMenuOpen: false,
playlistTitle: ''
```

### Playlist Definitions (`src/constants/playlists.js`)

Playlists are statically defined as an array of `{ name, author, title }` objects:
- `favorites` — special local playlist (stored in localStorage)
- Curated BeastSaber playlists (Flow, Weeks, Picks, Intro to Dance, Mom's Maps)

### Interaction Flow

1. User clicks "PLAYLISTS" → `playlistmenuopen` → `playlistMenuOpen: true`
2. `playlistMenu.html` renders all playlist cards via `bind-for` over `playlists` state
3. User clicks a card → `menu-playlist.js` emits `playlistselect` with ID + title
4. State handler (`state/index.js:647-654`): sets `playlist` + `playlistTitle`, clears genre/search
5. `search.js` detects the change, fetches songs:
   - **Favorites**: reads from `localStorage['favorites-v2']`
   - **Others**: calls `https://api.beatsaver.com/playlists/id/{id}/...`
6. Results flow into `state.search.results` and display in the menu

### Favorites (special case)

Stored in `localStorage` as `'favorites-v2'` (JSON array of challenge objects). Toggled via `favoritetoggle` handler (`state/index.js:346-369`). Loaded directly from localStorage instead of the API.

### Key Files

| File | Role |
|---|---|
| `src/constants/playlists.js` | Static playlist definitions |
| `src/state/index.js:642-662` | `playlistselect`, `playlistclear`, `playlistmenuopen` handlers |
| `src/components/search.js:78-149` | Fetches songs for selected playlist |
| `src/components/menu-playlist.js` | Click handler on playlist cards |
| `src/templates/playlistMenu.html` | Playlist grid UI |

> **Note:** No queue system — playlists just populate the search results; the user still picks individual songs to play.

---

## Genre / Category Selection

### State (`src/state/index.js:94-96`)

```js
genre: '',            // active genre name (empty = none)
genres: require('../constants/genres'),
genreMenuOpen: false
```

### Genre Definitions (`src/constants/genres.js`)

18 hardcoded genres in a 3×6 grid layout, each with `{ name, row, column }`:

| Row 1 | Row 2 | Row 3 |
|---|---|---|
| Pop | Electronic | Alternative |
| R&B | Hip Hop | Anime |
| Rap | House | Comedy |
| Rock | J-Pop | Dubstep |
| Soundtrack | K-Pop | Dance |
| Video Games | Meme | |

`row` and `column` are used to UV-map the genre icon from a 6×3 sprite sheet (`assets/img/genres.png`).

### Interaction Flow

1. User clicks "GENRES" → `genremenuopen` → `genreMenuOpen: true`
2. `genreMenu.html` renders all 18 genres via `bind-for` over `genres` state
3. User clicks a genre card → `menu-genre.js` emits `genreselect` with genre name
4. State handler (`state/index.js:409-415`): sets `genre`, clears search + playlist
5. `scene.html:13` binds genre to `search` component automatically:
   ```html
   bind__search="genre: genre; playlist: playlist; query: search.query; ..."
   ```
6. `search.js` maps genre name → BeatSaver API tag and fetches results:

   | Moon Rider Name | BeatSaver Tag |
   |---|---|
   | Video Games | `video-game-soundtrack` |
   | Soundtrack | `tv-movie-soundtrack` |
   | R&B | `rb` |
   | Rap / Hip Hop | `hip-hop-rap` |
   | Meme / Comedy | `comedy-meme` |
   | others | lowercase name |

### Key Files

| File | Role |
|---|---|
| `src/constants/genres.js` | 18 genre definitions with grid positions |
| `src/state/index.js:404-423` | `genreselect`, `genreclear`, `genremenuopen` handlers |
| `src/components/menu-genre.js` | Click handler — emits `genreselect` |
| `src/components/search.js:112-133` | Genre → BeatSaver tag mapping + URL building |
| `src/templates/genreMenu.html` | Genre grid UI with sprite atlas |
| `src/templates/menu.html:315-344` | GENRES / CLEAR GENRE buttons + selected label |

---

## Filter Mutual Exclusivity

Genre, playlist, and search query are **mutually exclusive**:
- Selecting a genre clears playlist and search query
- Selecting a playlist clears genre and search query
- Typing a search query clears genre and playlist

---

## Search & Song Loading (`src/components/search.js`)

The `search` component is the central hub for fetching songs. It reacts to changes in `genre`, `playlist`, `query`, and `difficultyFilter` (bound via `scene.html`).

### URL Construction Logic

```
No filter (popular):  https://api.beatsaver.com/maps/hot/0
Genre selected:       https://beatsaver.com/api/search/text/{page}?sortOrder=Rating&automapper=true&tags={tag}
Playlist selected:    https://api.beatsaver.com/playlists/id/{id}/{page}
Favorites:            localStorage['favorites-v2']
Text search:          https://api.beatsaver.com/search/text/{page}?q={query}
```

Results are normalized via `src/lib/convert-beatmap.js` (handles BeatSaver v2/v3 format differences) then emitted as `searchresults` event into state.

---

## State Management

All state lives in `src/state/index.js` via `aframe-state-component` (`AFRAME.registerState`). Components bind to state via `bind__` attributes in `src/scene.html`. Mutations happen via named event handlers emitted on the scene element.

```
el.sceneEl.emit('genreselect', { name: 'Pop' })
  → state handler fires
  → state updates
  → bound components re-render
```
