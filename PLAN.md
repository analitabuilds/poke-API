# PokéAPI Search App — Implementation Plan

A simple landing page where a user can search for a Pokémon by name or number and see its info and artwork, pulled live from [pokeapi.co](https://pokeapi.co/).

Team: 2 people, splitting work by **feature (vertical slice)**, not by frontend/backend layer — both people write Ruby, views, and CSS across the project.

---

## Search by name or number?

You don't have to choose — PokéAPI's single endpoint accepts either:

```
GET https://pokeapi.co/api/v2/pokemon/{name}
GET https://pokeapi.co/api/v2/pokemon/{id}
```

e.g. `.../pokemon/pikachu` and `.../pokemon/25` return the same Pokémon. So the UI is **one search box** — normalize whatever the user types (strip whitespace, downcase; the API expects lowercase names) and send it straight to the endpoint. A 404 means "not found."

Other notes:
- No API key, no rate limit — but PokéAPI's fair-use policy asks you to avoid hammering it (cache if you can, don't request on every keystroke).
- Images live in the response under `sprites` — for a nice landing-page image use `sprites.other["official-artwork"].front_default`.
- Useful display fields: `name`, `id`, `height`, `weight`, `types` (array of `{type: {name:}}`), `abilities`, `stats`.

---

## Architecture

No database needed for the MVP — this is a live pass-through to an external API.

```
config/routes.rb                         → root "pokemon#index"
app/controllers/pokemon_controller.rb    → handles the search
app/services/pokeapi_client.rb           → talks to PokéAPI, returns plain data
app/views/pokemon/index.html.erb         → search form + result display
app/views/pokemon/_summary.html.erb      → image, name, number, types
app/views/pokemon/_details.html.erb      → height, weight, abilities, stats
```

`PokeapiClient` is a plain Ruby class (no gem needed) using `Net::HTTP` + `JSON.parse` — no need to add Faraday/HTTParty for something this small. It returns **nil on 404** and a plain hash/Struct on success, so the controller/view never touch HTTP details directly.

### Data contract

Agree on this shape before splitting work — it's what lets both people build in parallel without blocking each other:

```ruby
# PokeapiClient.find("pikachu") returns on success:
{
  id: 25,
  name: "pikachu",
  image_url: "https://raw.githubusercontent.com/.../25.png",
  types: ["electric"],
  height: 4,
  weight: 60,
  abilities: ["static", "lightning-rod"]
}
# returns nil if not found
```

---

## Phase 0 — Together (45–60 min)

- Pair-program `PokeapiClient#find(query)` fully: the `Net::HTTP` call, JSON parsing, success case *and* the 404 → `nil` case. This is the shared foundation everything else builds on, so it's worth finishing together rather than splitting.
- Wire up `routes.rb` and a bare `PokemonController#index` skeleton (calls the client, no view logic yet).
- Set up feature branches for Phase 1.
- Log a `learnings.md` entry together on what `Net::HTTP` / JSON parsing taught you.

---

## Phase 1 — Split into two full-stack features (parallel branches)

Each person owns a feature top-to-bottom: controller logic, view/partial, styling, and its own test. No layer boundaries — everyone touches Ruby and markup.

### Partner A — "Search & Summary" feature
- Controller: happy-path branch — `@pokemon` gets set when the client returns data.
- View: the main result partial (`_summary.html.erb`) — image, name, dex number, types — plus its styling.
- Test: searching `"pikachu"` and `"25"` both render the correct summary (proves name/number search both work).

### Partner B — "Empty States & Extended Details" feature
- Controller: edge-case branches — blank query, not-found (client returned `nil`).
- View: the "search a Pokémon to get started" empty state, the "not found" message, and the extended details partial (`_details.html.erb`) shown alongside Partner A's summary.
- Test: blank search shows the prompt; garbage input shows "not found"; a valid search renders extended details correctly.

> Both partials render inside `index.html.erb` side by side — keep each in its own partial file so the two branches don't conflict on the same lines.

---

## Phase 2 — Together

- Merge both branches, integrate the partials into one view, manually test both flows end-to-end.
- Cross-review: read each other's PR so you both understand the whole app, not just your slice.

---

## Phase 3 — Split again, swap territory

Rotate so neither person is stuck in the same corner of the app twice.

### Partner A
- "Random Pokémon" button — new route + controller action + view + test (touches routing you didn't own last time).

### Partner B
- Input normalization & resilience — trimming whitespace, case-insensitivity, numeric-string handling, network-timeout handling (touches the client you didn't own last time).

### Both
- Responsive/mobile CSS pass on your own feature's markup from Phase 1.

---

## Phase 4 — Together

- README with setup instructions + screenshot.
- Final `learnings.md` pass.
- Stretch goals if time allows: caching identical lookups with `Rails.cache`, deploy via the existing Kamal config.

---

## Git workflow

- `main` stays deployable.
- Feature branches per slice (e.g. `feature/search-summary`, `feature/empty-states`).
- Small PRs, review each other's code before merging.
