# UTRGV Map

SvelteKit + MapLibre campus map.

The original single-file prototype is kept at `legacy/utrgv-maplibre-v4(7).html`.

## Developing

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```sh
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```sh
npm run build
```

You can preview the production build with `npm run preview`.

## GitHub Pages

This project uses `@sveltejs/adapter-static` and writes output to `build/`.

If your repo is published at `https://<user>.github.io/<repo>/`, set `BASE_PATH` to `/<repo>` during the build so asset URLs resolve correctly.

> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.
