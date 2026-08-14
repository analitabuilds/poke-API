# Implementation Plan — Poke-API App

A simple Rails landing page where a user enters a Pokémon name or number and
the app pulls the relevant details from [PokeAPI](https://pokeapi.co/).

Team: 2 people, both beginners with Rails.
Stack: Rails 8.0.2, Ruby 3.4.4 (already scaffolded in this repo).

## Name vs. number: support both

PokeAPI's endpoint accepts either the numeric ID or the lowercase name in the
same path slot:

```
GET https://pokeapi.co/api/v2/pokemon/{id or name}/
```

Don't make the user choose — one search box, normalize the input, let the API
handle it.

Normalization rules to apply before the request:
- `.strip.downcase`
- replace spaces with hyphens (multi-word Pokémon like "Mr. Mime" are
  `mr-mime`, "Type: Null" is `type-null`)
- if the input is purely numeric, pass it straight through as the ID

PokeAPI's fair-use policy asks that responses be **cached locally** rather
than re-fetched on every request. This repo already has `solid_cache` in the
Gemfile, which is a natural fit (see Phase 3).

## Architecture

- Plain Rails app, no JS framework needed — one controller, a couple of
  views, a small service object that talks to PokeAPI.
- `app/services/poke_api_client.rb` — wraps the HTTP call, returns a plain
  hash or a small PORO.
- `app/controllers/pokemon_controller.rb` — one `index` action: renders the
  search form, and re-renders with results when a query param is present.
- `app/views/pokemon/index.html.erb` — search form + results partial.
- Gem to add: `faraday` (simple, well-documented HTTP client — friendlier
  than raw `Net::HTTP` for beginners).

## Phases

### Phase 1 — Setup & exploration (together, ~1 session)

- Both poke at the PokeAPI in a browser / `curl` for a couple of Pokémon
  (one by name, one by number, one that doesn't exist) so you both see the
  real JSON shape before writing code.
- Add `gem "faraday"` to the Gemfile, `bundle install`.
- Add the route: `get "pokemon", to: "pokemon#index"` and set it as root
  (`root "pokemon#index"`).
- Agree on the contract: what hash/object shape the service object returns
  to the controller, e.g.:

  ```ruby
  { id:, name:, sprite_url:, types:, stats:, height:, weight: }
  ```

  This is the seam the two branches below will meet at.

### Phase 2 — Split work (parallel, each on own branch)

**Partner A — API integration (backend)**
- `PokeApiClient` service: takes raw user input, normalizes it, calls
  Faraday, parses JSON.
- Handle the "not found" case (404 from PokeAPI) and network errors — return
  `nil` or raise a custom error the controller can rescue.
- Controller action: calls the service, sets `@pokemon` or `@error`.
- Basic request/controller tests using `webmock`/`VCR` (or a stubbed
  service) so tests don't hit the real network.

**Partner B — Frontend (views/UI)**
- Search form (single text input, works for name or number).
- Results view: sprite image, name, ID, height/weight, types, base stats —
  build this against a sample JSON fixture so you're not blocked waiting on
  Partner A's service.
- "Pokémon not found" / empty-state UI.
- Basic styling in `app/assets/stylesheets/application.css` (no CSS
  framework installed, plain CSS is fine for v1).

The seam is small and well-defined (one hash shape), so this split works
even for beginners — you're mostly not touching the same files.

### Phase 3 — Integrate (together)

- Merge both branches, wire the real service output into the real view.
- Add caching:

  ```ruby
  Rails.cache.fetch("pokemon/#{normalized_query}", expires_in: 1.day) { ... }
  ```

  around the Faraday call — satisfies PokeAPI's fair-use ask and makes
  repeat searches instant.
- Test the golden path (valid name, valid number) and edge cases (garbage
  input, misspelled name, uppercase/spaces) manually in the browser.

### Phase 4 — Polish

- Loading/error states, minor styling pass.
- README update with real setup instructions (currently the Rails default
  stub).
- Optional stretch goals: recently-searched list, "random Pokémon" button.

## Notes

- Learnings from working through this plan belong in the separate learnings
  markdown file (added by Partner B on their own branch), not in this file.
  This document should stay a plan of record; update it only when the plan
  itself changes.
