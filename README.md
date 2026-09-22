# phoenix-inertia-react

Agent skill for **Phoenix + Inertia.js + React + TypeScript** using the official
Hex package [`inertia`](https://hex.pm/packages/inertia)
([inertiajs/inertia-phoenix](https://github.com/inertiajs/inertia-phoenix)).

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Install

```bash
npx skills add pkayokay/phoenix-inertia-react
```

Run inside a Phoenix app to install for that project. Use `-g` for all projects on your machine.

## Usage

The skill loads when the agent works on Phoenix / Inertia / React / TypeScript tasks.

**Examples:**

```
Set up Inertia with React and TypeScript in this Phoenix app
```

```
Add a controller and TSX page for users index with deferred stats
```

```
Wire up form validation errors from an Ecto changeset
```

## Stack

- Phoenix + Hex `inertia` (~> 2.6)
- React + TypeScript (`@inertiajs/react`, `.tsx` pages)
- esbuild (default); Vite if already in use

## License

MIT
