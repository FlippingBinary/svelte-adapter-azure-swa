# svelte-adapter-azure-swa

> :warning: WARNING: this project is considered to be in BETA until SvelteKit is available for general use and the Adapter API is stable. Please report any issues you encounter.

Adapter for Svelte apps that creates an Azure Static Web App, using an Azure function for dynamic server rendering. If your app is purely static, you may be able to use [adapter-static](https://www.npmjs.com/package/@sveltejs/adapter-static) instead.

## Usage

See the [demo repo](https://github.com/geoffrich/sveltekit-azure-swa-demo) for an example integration with the SvelteKit demo app.

Run `npm install -D svelte-adapter-azure-swa`.

Then in your `svelte.config.js`:

```js
import azure from 'svelte-adapter-azure-swa';

export default {
	kit: {
		...
		adapter: azure()
	}
};
```

You will need to create an `api/` folder in your project root containing a [`host.json`](https://docs.microsoft.com/en-us/azure/azure-functions/functions-host-json) and a `package.json` (see samples below). The adapter will output the `render` Azure function for SSR in that folder. The `api` folder needs to be in your repo so that Azure can recognize the API at build time. However, you can add `api/render` to your .gitignore so that the generated function is not in source control.

### Sample `host.json`

```json
{
	"version": "2.0",
	"extensionBundle": {
		"id": "Microsoft.Azure.Functions.ExtensionBundle",
		"version": "[2.*, 3.0.0)"
	}
}
```

### Sample `package.json`

It's okay for this to be empty. Not including it causes the Azure Function build to fail.

```json
{}
```

## Azure configuration

When deploying to Azure, you will need to properly [configure your build](https://docs.microsoft.com/en-us/azure/static-web-apps/build-configuration?tabs=github-actions) so that both the static files and API are deployed.

| property          | value          |
| ----------------- | -------------- |
| `app_location`    | `./`           |
| `api_location`    | `api`          |
| `output_location` | `build/static` |

## Running locally with the Azure SWA CLI

You can debug using the [Azure Static Web Apps CLI](https://github.com/Azure/static-web-apps-cli). Note that the CLI is currently in preview and you may encounter issues.

To run the CLI, install `@azure/static-web-apps-cli` and the [Azure Functions Core Tools](https://github.com/Azure/static-web-apps-cli#serve-both-the-static-app-and-api) and add a `swa-cli.config.json` to your project (see sample below). Run `npm run build` to build your project and `swa start` to start the emulator. See the [CLI docs](https://github.com/Azure/static-web-apps-cli) for more information on usage.

### Sample `swa-cli.config.json`

```json
{
	"configurations": {
		"app": {
			"outputLocation": "./build/static",
			"apiLocation": "./api"
		}
	}
}
```

## Options

### customStaticWebAppConfig

An object containing additional Azure SWA [configuration options](https://docs.microsoft.com/en-us/azure/static-web-apps/configuration). This will be merged with the `staticwebapp.config.json` generated by the adapter.

It can be useful to understand what's happening 'under the hood'. All requests for dynamic responses must be rewritten for the server-side rendering (SSR) function to render the page. If a request is not rewritten properly, bad things can happen. If the method requires a dynamic response (such as a `POST` request), SWA will respond with a `405` error. If the request method is `GET` but the file was not pre-rendered, SWA will use the `navigationFallback` rule to rewrite the request, which will fail if not set correctly.

This adapter will ensure several routes are defined, along with `navigationFallback`, so dynamic rendering will occur correctly. If you define a catch-all wildcard route (`route: '/*'` or `route: '*'`), those settings will be used by every route this adapter creates. This will allow you to set a default `allowedRoles`, among other things. If you override the `rewrite` key of that route, you may force or prevent SSR from taking place.

If you want to force a route to deliver only dynamically-generated responses, set `rewrite: 'ssr'` inside the route. Files that are not rendered by SvelteKit, such as static assets, may be inaccessible if you do this. Conversely, set `rewrite: undefined` to disable SSR in a route.

If you want to prevent dynamic responses for requests in a route that has no dynamic content (such as an `/images` folder), you can set `exclude` in the `navigationFallback` rule to an array of routes that should not be dynamically handled. Read [Microsoft's documentation](https://learn.microsoft.com/en-us/azure/static-web-apps/configuration#fallback-routes) for a description of how `exclude` works.

**Note:** customizing this config (especially `routes`) has the potential to break how SvelteKit handles the request. Make sure to test any modifications thoroughly.

```js
import azure from 'svelte-adapter-azure-swa';

export default {
	kit: {
		...
		adapter: azure({
			customStaticWebAppConfig: {
				routes: [
					{
						route: '/login',
						allowedRoles: ['admin']
					}
				],
				globalHeaders: {
					'X-Content-Type-Options': 'nosniff',
					'X-Frame-Options': 'DENY',
					'Content-Security-Policy': "default-src https: 'unsafe-eval' 'unsafe-inline'; object-src 'none'",
				},
				mimeTypes: {
					'.json': 'text/json'
				},
				responseOverrides: {
					'401': {
						'redirect': '/login',
						'statusCode': 302
					}
				}
			}
		})
	}
};
```

### esbuildOptions

An object containing additional [esbuild options](https://esbuild.github.io/api/#build-api). Currently only supports [external](https://esbuild.github.io/api/#external). If you require additional options to be exposed, plese [open an issue](https://github.com/geoffrich/svelte-adapter-azure-swa/issues).

```js
import azure from 'svelte-adapter-azure-swa';

export default {
	kit: {
		...
		adapter: azure({
			esbuildOptions: {
				external: ['fsevents']
			}
		})
	}
};
```
