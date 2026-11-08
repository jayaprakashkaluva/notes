# React in production

**Follow one URL from source code to pixels, then choose an architecture for it.**

Imagine opening `/products/42` in a new tab. Who puts “Trail Mug” on the screen? In a client-rendered app, a host can send an HTML file with an empty mount point. JavaScript running in your browser fills it in. In a server-rendered app, the response can already contain `<h1>Trail Mug</h1>`. The browser can display that heading while the JavaScript for an interactive quantity button is still downloading. The finished pages can look identical. Their first HTTP responses tell different stories. [React client APIs](https://react.dev/reference/react-dom/client) · [MDN on how browsers work](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work)

**Does a React app require Express? No.** Express is an optional framework for handling HTTP requests in Node.js. A static host can serve a React frontend, an existing Java service can own its API, and users can run the UI in their browsers. In that design Node.js can finish its job when the build finishes. [Vite static deployment](https://vite.dev/guide/static-deploy) · [Express routing](https://expressjs.com/en/starter/basic-routing/)

| If you want to… | Start here |
| --- | --- |
| Follow the code into production | [What gets deployed](#what-gets-deployed) |
| Understand what the browser does | [One page through two request paths](#one-page-through-two-request-paths) |
| Decide whether to deploy a Node service | [Node and Express](#node-and-express) |
| Choose a design for a real application | [Choosing per route](#choosing-per-route) |
| See the difference yourself | [Lab with the same component on both routes](#lab-with-the-same-component-on-both-routes) |

## What gets deployed

A **build** transforms source into artifacts. A **deployment** makes a particular release available to visitors. A **request** is what happens later when someone visits a URL. One machine can perform all three jobs in a small project. They can also happen at different times, on different machines owned by different teams.

```text
 Developer commits components, styles, images, package.json and lockfile
                                |
                                v
 Build machine (often CI) uses Node.js to run the build tool
                                |
             +------------------+--------------------+
             |                                       |
             v                                       v
 Browser files                             Server code, when needed
 HTML, JS, CSS, images                     request handler + its dependencies
             |                                       |
             v                                       v
 Static host / content delivery            Application service or function
 network (CDN) serves file bytes           runs code when a request arrives
             |                                       |
             +--------------- HTTP ------------------+
                                |
                                v
 Browser parses HTML, runs client JS, lays out the page and paints
```

A CDN caches responses near users. Its **origin** is the upstream system it contacts when it needs a response. An origin could be object storage containing built files or a service executing application code. A host can route different URL paths to different origins behind the same domain.

### A static release with Vite

Consider a project with `src/Product.jsx`, `index.html`, `package.json` and `package-lock.json`. During development a Vite dev server transforms modules as you work. In this example the production build script runs `vite build`. If the project uses npm and has a matching lockfile, a build machine can run the following from the project directory. [Vite production builds](https://vite.dev/guide/build) · [npm clean install](https://docs.npmjs.com/cli/v11/commands/npm-ci/)

```sh
npm ci
npm run build
```

1. The build tool follows imports. With React's JSX tooling, a source expression such as `<Product />` becomes JavaScript that describes a React element. The browser entry and its dependencies are bundled for the browser. Producing JavaScript for `Product` and rendering the product heading into HTML are separate operations. A plain Vite client build does the former. [Vite production builds](https://vite.dev/guide/build) · [React on creating a root](https://react.dev/reference/react-dom/client/createRoot)
2. Vite's default output directory is `dist/`. A representative build contains `index.html`, `assets/index-a1b2.js` and an optional CSS file. Names vary. The built HTML references the emitted asset URLs. Public assets can also be copied with their original names. [Vite static deployment](https://vite.dev/guide/static-deploy) · [Vite assets](https://vite.dev/guide/assets)
3. Publish the **contents** of that output directory at the site's intended base path. Point the domain at the host, configure HTTPS, and configure its route rules. If the site lives at `/shop/`, align the Vite base, host rules and client router with that prefix. A browser visiting the page asks the host for files by URL. Nobody installs npm on the visitor's machine. [Vite public base path](https://vite.dev/guide/build#public-base-path)
4. Smoke-test the deployed URL directly, including a nested route, a real asset and a missing asset. Vite's `preview` command is for local inspection. In this static design the production host only has to return file bytes and appropriate HTTP headers. [Vite static deployment](https://vite.dev/guide/static-deploy)

### What changes for a request-rendered release?

For request-time server-side rendering (SSR), deploy browser assets **and** executable server output. For example, a Vite SSR integration can build a browser entry for hydration and a separate entry for rendering on the server. A Node service or compatible hosting adapter loads the server entry, matches a URL, gets data and returns HTML. A managed function packages that request handler for the provider. A long-running service usually starts a process that listens on a port. Uploading browser assets alone leaves the request handler undeployed. [Vite SSR guide](https://vite.dev/guide/ssr) · [Next.js deployment options](https://nextjs.org/docs/app/getting-started/deploying)

A deployment review should answer three concrete questions. Which public file does the HTML reference? Which executable handles a page that needs fresh HTML? Which service owns the data? The three answers can name three different systems.

## One page through two request paths

Assume a first visit, empty browser caches, no service worker and a product API. The diagrams trace dependencies. Browsers can overlap downloads, parsing and painting.

### Client-side rendering (CSR)

```text
Browser                         Static host / CDN                  Product API
   | GET /products/42 ----------------->|                               |
   |<------------- index.html shell ----|                               |
   | GET /assets/app-a1b2.js ---------->|                               |
   |<------------------ JavaScript -----|                               |
   | run route code, optionally show loading state                      |
   | GET /api/products/42 --------------------------------------------->|
   |<---------------------- JSON with name, description and price ------|
   | React commits the product heading into the DOM                     |
   | browser lays out and paints the heading                            |
```

The initial document in this design has a mount point such as `<div id="root"></div>` and a script reference. The site's host maps the product URL to that document. Once the script executes, the router reads the URL, the app obtains product data and React can display it. A site might preload data or embed an initial snapshot to shorten this path. A loading screen can appear before the product. [React createRoot](https://react.dev/reference/react-dom/client/createRoot) · [React Router SPA deployment](https://reactrouter.com/how-to/spa)

**Inside the browser.** A component is commonly a function. Its props are inputs from its parent. Its state stores values React should remember between renders. Calling a state setter requests an update. During **render**, React calls components to compute their output. During **commit**, React DOM changes the relevant browser nodes where needed. The DOM is the browser's in-memory representation of the document. The browser applies styles, computes layout, paints and composites the result. A React render can produce the same output and require no DOM change. [React render and commit](https://react.dev/learn/render-and-commit) · [React state](https://react.dev/reference/react/useState) · [MDN browser rendering](https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work)

**What happens on a click?** A quantity button that calls `setQuantity(n => n + 1)` can change “Quantity: 1” to “Quantity: 2” entirely in the open tab. Reloading starts that in-memory state again. An “Add to cart” action that should survive a reload or appear on another device needs a persistence design. For an account-backed cart, send a request to a service that authorizes the user and saves the change. React alone does neither that network request nor that storage. [React state](https://react.dev/reference/react/useState) · [OWASP authorization guidance](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

**What happens on navigation?** A client router can intercept an in-app link, update browser history, fetch code or data and render the next view. Pasting the same URL into a new tab makes a document request to the host. Configure a single-page app (SPA) host to serve the entry document for recognized page paths. Keep `/api/*` and asset paths out of the fallback. Otherwise a missing script or API route can receive HTML with a misleading success status. [React Router SPA deployment](https://reactrouter.com/how-to/spa)

### Request-time server-side rendering (SSR)

```text
Browser                       Page renderer                       Product service
   | GET /products/42 ------------>|                                      |
   |                               | read product 42 -------------------->|
   |                               |<------------------- product data ----|
   |                               | render components using that data    |
   |<-------- HTML containing <h1>Trail Mug</h1> and script reference ----|
   | parse HTML and display available content                             |
   | load client JS from the asset host                                   |
   | hydrate interactive components with matching initial data            |
   | user clicks quantity button -> local state update                    |
```

On an uncached request, the page renderer can read from a service or an authorized data store before sending the relevant HTML. The browser can display text and use ordinary links from that HTML. A React `onClick` handler is executable behavior supplied by client JavaScript. **Hydration** connects client logic to the React-generated DOM already in the document. Afterwards, local state updates and client-router navigations can work much like CSR. A fresh document request starts the server path again, subject to caching. [React hydration](https://react.dev/reference/react-dom/client/hydrateRoot) · [React Router rendering strategies](https://reactrouter.com/start/framework/rendering)

**Why hydration sometimes fails.** The first client render must agree with the server-rendered output. If the server prints a price from version A and the browser hydrates with version B, the initial text can differ. A date formatted in two time zones can do the same. Give the client the same initial snapshot via the framework's safe serialization mechanism, and refresh it after hydration when necessary. Keep credentials out of everything sent to the browser. Browser-specific work such as reading local storage can run in a client effect. React effects run only on the client, so a fetch started solely in `useEffect` cannot populate the initial server HTML. [React hydration pitfalls](https://react.dev/reference/react-dom/client/hydrateRoot) · [React useEffect](https://react.dev/reference/react/useEffect)

**Streaming.** A renderer can send the page shell and a placeholder while a slower section is still being prepared. For example, the product name can appear while reviews load behind a Suspense boundary. React provides `renderToPipeableStream` for Node streams and `renderToReadableStream` for Web Streams environments. Plan which failures need a 404 or redirect before starting the HTTP response, because its status is committed once headers have been sent. Verify through the deployed proxy or CDN that chunks reach the browser progressively. [React streaming and status codes](https://react.dev/reference/react-dom/server/renderToPipeableStream)

### Comparing the first visit

| Design question | CSR with an HTML shell | Request-time SSR with hydration | Static site generation (SSG) |
| --- | --- | --- | --- |
| When is the initial route content produced? | In the browser after client code runs | At request time on a render, possibly reused from a cache | Ahead of visits, commonly during a build |
| What can the first HTML contain? | Mount point, scripts, optional shell | Route-specific markup or a streamed shell followed by content | Previously generated route-specific markup |
| What must run for the page request? | File-serving infrastructure | A compatible renderer on a cache miss | File-serving infrastructure for prebuilt routes |
| When can a React button respond? | After the client renders its handler | Once the relevant interactive region hydrates | Once hydrated if the page includes interactive React |
| Where does freshness come from? | Client data fetching and its cache policies | Request data and any HTML or data caches | Rebuild, supported regeneration, or a later client update |
| Typical cost to investigate | JS startup followed by a data-fetch waterfall | Server/data latency, render CPU and client hydration | Build duration, invalidation and stale content |

SSG can produce the same visible initial HTML as request-time SSR. Seeing a heading in the HTTP response proves the heading arrived as HTML. Identifying *when* it was produced requires the build configuration, cache behavior or renderer logs. A single site can combine all three strategies. [React Router rendering strategies](https://reactrouter.com/start/framework/rendering) · [Rendering on the web](https://web.dev/articles/rendering-on-the-web)

**Where React Server Components fit.** In a supporting framework, a Server Component can run during a build or on a server for a request. Its implementation stays on the server side of the module boundary. Its rendered result can include Client Components for state and browser events. SSR describes how initial HTML is produced. Server Components describe where particular component implementations execute and what crosses to the client. Frameworks can combine the two, and a Client Component can contribute to initial server-generated HTML before hydrating in the browser. [React Server Components](https://react.dev/reference/rsc/server-components) · [Next.js Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)

## Node and Express

**Node and Node.js are names for the same JavaScript runtime.** A runtime executes JavaScript and supplies APIs to the program. The browser supplies the DOM, `window` and events. Node supplies APIs such as filesystem access and HTTP listeners. Both environments can execute a pure component function. Code that reaches for `window.localStorage` while rendering assumes a browser. Code that reads a database password from the process environment belongs in a trusted server module. [Node.js introduction](https://nodejs.org/learn/getting-started/introduction-to-nodejs) · [React hydration](https://react.dev/reference/react-dom/client/hydrateRoot)

| Place | What Node might run | What happens after a deployment? |
| --- | --- | --- |
| Developer laptop | Vite dev server, tests, local tools | Visitors use the deployed URL independently of the developer machine. |
| Build worker | npm scripts, compiler, bundler, static page generation | The job can exit after it publishes the artifacts. |
| Production API | Request handlers for reads, writes and authentication | A process or function runs when the API receives traffic. |
| Production page renderer | Framework server or SSR handler | It executes for requests that need rendering and are not satisfied by a cache. |
| Visitor's browser | Client JavaScript executes in the browser's own runtime | React DOM and the app handle UI updates there. |

Node can use asynchronous I/O so other requests can make progress while one waits for a database or network response. Long synchronous calculations still occupy the JavaScript event loop. A burst of expensive SSR work can therefore slow other handlers in the same process. Measure render CPU, queueing and downstream time separately before choosing caching, more instances or separate workers. [Node.js on the event loop and worker pool](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)

### When would you choose Express?

Express runs inside Node. It matches HTTP methods and paths to handlers and lets you compose **middleware**, functions that inspect or modify a request, send a response, or pass control onward. For example, an API might run session verification, then an order-ownership check, then the handler that returns order data. Express can also serve files or call a React server renderer. Installing it alone does none of the application-specific work. [Express routing](https://expressjs.com/en/starter/basic-routing/) · [Express middleware](https://expressjs.com/en/guide/using-middleware/)

| Situation | A plausible production design | Who needs Express? |
| --- | --- | --- |
| Public site generated during a build | Static host serves prebuilt pages and assets | Nobody in the page-serving path |
| React admin UI with an existing Java API | CDN serves UI files. A reverse proxy routes `/api/*` to Java. | Nobody, unless a new Node service has a specific job |
| Frontend needs a single response assembled from several internal APIs | A backend for frontend (BFF) owns the aggregation, authorization and response shape | Express is an option if the team wants to own a Node service |
| Framework already supplies SSR, routes and request handlers | Deploy its supported server or adapter | An additional Express process is optional |
| Small custom HTTP service | Node's built-in `node:http` can listen and respond directly | Express earns its place when its routing and middleware simplify the service |

In the Java example, the browser can request `/api/orders/42` from the same origin as the UI even though a different process handles it. A reverse proxy forwards that path internally. If the browser calls a different origin directly, the API may need CORS headers so the browser permits the frontend to read the response. CORS controls browser access to cross-origin responses. The API must separately authenticate and authorize protected operations for every caller. [MDN on CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) · [OWASP authorization guidance](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

## Choosing per route

Start with a requirement you can test. **Who needs to see which content, how soon, how fresh, and shared with whom?** Then assign responsibility for HTML, data, mutations and caching. Here is a worked design for a shop. Its assumptions are illustrative.

| Route and requirement | Design choice | What would change the decision? |
| --- | --- | --- |
| `/help/shipping` is public, identical for everyone and changes when editors publish | Generate HTML during the build. Serve it from a CDN. Rebuild on publication. | If publishing all pages exceeds the editorial deadline, consider generating or revalidating only affected pages. |
| `/products/42` needs searchable product text on the first visit. Descriptions tolerate five minutes of staleness. | Serve public product HTML with a bounded shared-cache lifetime. Generate on a cache miss or prebuild popular pages. Fetch a signed-in shopper's offers separately. | If public prices vary by region, include region in the cache design or move the price out of shared HTML. |
| `/account/orders` is private, highly interactive and already backed by a Java order service | Serve a static CSR shell. Load orders through a same-origin authenticated API. Design loading, empty, error and retry states. | If measurements show an unacceptable delay before the first order appears, test authenticated SSR with private caching rules. |
| `POST /api/checkout` changes durable state | The trusted service verifies identity, ownership, current price and stock and records the transaction. Give retries a way to identify the same checkout attempt. | Rendering strategy alone cannot deliver correctness for concurrent purchases or retries. Those are service and storage decisions. |

For public search traffic, crawlers differ. Google documents a JavaScript rendering stage. Putting essential text, links and metadata in the returned HTML can make them available to clients with limited JavaScript support. Verify status codes for removed products and inspect the HTML that crawlers actually receive. For a private dashboard, measure time to usable account data instead. [Google Search Central on JavaScript](https://developers.google.com/search/docs/crawling-indexing/javascript/javascript-seo-basics)

### Release and cache rules for this design

| Response | Example policy | Reason |
| --- | --- | --- |
| Content-hashed public JavaScript or CSS | `Cache-Control: public, max-age=31536000, immutable` | New bytes get a new URL, so the previous version can stay cached. |
| A static entry document that must pick up releases quickly | `Cache-Control: no-cache` and a validator such as an ETag | A cache can store the document but must validate it before reuse. |
| Public product HTML with a five-minute shared-cache budget | `Cache-Control: public, max-age=0, s-maxage=300` on a correctly configured shared cache | Browsers revalidate. The shared cache can reuse the response during the chosen window. Include every relevant variant in the cache key. |
| Sensitive order data or personalized HTML | `Cache-Control: no-store`, and configure the CDN to bypass caching for the route | Keep this response out of browser and shared HTTP caches. Apply the rule to both HTML and API responses containing sensitive data. |

The time values above are example product decisions. Public responses must be safe to share with every requester who matches the cache key. An authenticated user's name, negotiated price or order number in shared HTML is a privacy bug. Test with two separate accounts. Never inject service secrets into client bundles. Vite exposes `VITE_*` variables to client code by default and substitutes their values during a build. Changing a server environment variable later cannot change bytes already deployed in a browser asset. [MDN HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching) · [Vite environment variables](https://vite.dev/guide/env-and-mode.html)

**Rolling out and rolling back.** Publish the new hashed assets before switching traffic to HTML that references them, or use a host that activates the whole release atomically. Keep earlier assets available long enough for existing tabs to load deferred code. An old tab may still call the new API, so plan compatibility across releases. For an SSR service, verify a deep link, a missing product and the health of the new service before sending all traffic to it. A rollback should restore a matching HTML/server and asset release. [Vite on load errors after deployments](https://vite.dev/guide/build#load-error-handling) · [MDN HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching)

### What to measure before deciding

- **When does useful content appear?** Compare the document response time (TTFB, time to first byte) with Largest Contentful Paint (LCP). Fast shell delivery can still precede a long wait for product data. SSR can move the wait into the document request. Inspect the request waterfall and the actual LCP element.
- **When does interaction feel responsive?** Measure Interaction to Next Paint (INP) in the field. Also exercise the very first button click on a slow device. A visible button can precede hydration. Track Cumulative Layout Shift (CLS) when a placeholder gives way to real content. [Core Web Vitals](https://web.dev/articles/defining-core-web-vitals-thresholds) · [Rendering on the web](https://web.dev/articles/rendering-on-the-web)
- **Where does the system spend time and money?** Compare cold visits, repeat visits and in-app navigation. Record client JS bytes, cache hit rate, API latency, SSR time, errors and compute per visit. Segment by device, region and signed-in state. Test what appears when the reviews service is slow, when a product is missing and when a deploy occurs while an old tab is open.

A short architecture decision record for this shop might read: “We chose a static CSR shell for orders because every result is private, an authorized Java API already exists, and most sessions involve repeated filtering. The downside is the first-load data request after startup. We will compare time to first order row on slower devices against our agreed budget, then trial SSR if the waterfall shows a material delay we can remove.” This records a reason, a cost and a way to revise the decision.

## Lab with the same component on both routes

This small local experiment serves one production-built browser bundle on two URLs. The server returns an empty root at `/csr` and renders the identical product component into the response at `/ssr`. The data is fixed to isolate rendering. Express handles the two HTTP routes for convenience. A static host could serve the CSR document without Express.

### Build and run

Use a fresh folder, Node 22.12 or later in the 22.x line (or another Node version supported by Vite 7), and npm. Run the commands from that folder. On Windows, `npm.cmd` and `npx.cmd` work in PowerShell if script execution is restricted. The lab uses Vite 7, React 19 and Express 5 as a reproducible example. [Vite 7 Node support](https://vite.dev/blog/announcing-vite7)

```sh
npm init -y
npm install react@19 react-dom@19 express@5
npm install -D vite@7
```

Create the following four files in the same folder. `Product.mjs` uses `createElement` so that Node can execute it directly. The heading is equivalent to the JSX expression `<h1>Trail Mug</h1>`.

**Product.mjs**

```js
import { createElement as h, useState } from "react";

export default function Product() {
  const [quantity, setQuantity] = useState(1);
  return h("main", null,
    h("h1", null, "Trail Mug"),
    h("button", { onClick: () => setQuantity(n => n + 1) },
      "Quantity: " + quantity)
  );
}
```

**index.html**

```html
<!doctype html>
<html lang="en">
  <head><meta charset="utf-8"><title>Product lab</title></head>
  <body>
    <div id="root"></div>
    <script type="module" src="/client.js"></script>
  </body>
</html>
```

**client.js**

```js
import { createElement } from "react";
import { createRoot, hydrateRoot } from "react-dom/client";
import Product from "./Product.mjs";

const root = document.getElementById("root");
const app = createElement(Product);
if (root.hasChildNodes()) {
  hydrateRoot(root, app);
} else {
  createRoot(root).render(app);
}
```

**server.mjs**

```js
import express from "express";
import { readFile } from "node:fs/promises";
import { createElement } from "react";
import { renderToString } from "react-dom/server";
import Product from "./Product.mjs";

const app = express();
const shell = await readFile("dist/index.html", "utf8");
app.use("/assets", express.static("dist/assets"));

app.get("/csr", (req, res) => res.type("html").send(shell));
app.get("/ssr", (req, res) => {
  const markup = renderToString(createElement(Product));
  res.type("html").send(shell.replace(
    '<div id="root"></div>', '<div id="root">' + markup + '</div>'
  ));
});

app.listen(4174, "127.0.0.1", () => {
  console.log("Open http://127.0.0.1:4174/csr or /ssr");
});
```

Run the production build, then start the local HTTP server.

```sh
npx vite build
node server.mjs
```

Open [the CSR page](http://127.0.0.1:4174/csr) and [the SSR page](http://127.0.0.1:4174/ssr). This example uses `renderToString` to make the whole response easy to inspect. It does synchronous rendering and has limited Suspense support. A production streaming design can use the framework's streaming renderer. The lab's fixed HTML placeholder and two known routes keep the example small. [React renderToString](https://react.dev/reference/react-dom/server/renderToString) · [Express static file serving](https://expressjs.com/en/starter/static-files/)

### Inspect the evidence

| Experiment | Expected on `/csr` | Expected on `/ssr` |
| --- | --- | --- |
| Open DevTools → Network, reload, select the **document**, then read **Response**. Find `id="root"`. | Inside the root there is no heading. | The root contains the heading and button. |
| Open **Elements** after scripts run. | The live DOM contains “Trail Mug”. | The live DOM also contains “Trail Mug”. |
| Disable JavaScript in DevTools and reload. | The root stays empty. | The heading and button appear. Clicking does not increment the count. |
| Re-enable JavaScript, reload and click the button. | Quantity increments, with no API call. | Quantity increments after hydration, with no API call. |
| Reload after incrementing. | Quantity starts at 1 again. | Quantity starts at 1 again. |

In a second terminal on Windows you can read the document bodies directly. The important place to look is **inside** the root element. A document title, preload instruction or embedded data block can also contain product words.

```powershell
curl.exe -s http://127.0.0.1:4174/csr
curl.exe -s http://127.0.0.1:4174/ssr
```

**One more experiment.** After loading both routes, stop the server with Ctrl+C. Click the quantity button in an already loaded tab. Its local update can still work because the necessary code is in the browser. A fresh navigation with caching disabled now fails because the browser needs a document from the host. This is the difference between executing the UI and delivering its files.

## Design review exercises

1. **The page works when you click a link, but refreshing `/orders/42` returns 404. Where do you look first?** Inspect the network response for the *document*. The client router handled the link while the host handled the refresh. For a CSR design, check the host's page fallback, the router base and the exclusions for assets and APIs.
2. **A user briefly sees somebody else's name on a product page. What is the first design question?** Ask whether shared HTML or embedded hydration data contained identity-specific content. Compare responses from separate authenticated sessions, cache keys and cache-hit headers. Bypass and invalidate affected shared content as needed. Fix the boundary before trying to hide the name in client code.
3. **A product page needs SEO, but stock changes every few seconds. Must every request render the whole page afresh?** A plausible design sends public, indexable product text as prebuilt or cached HTML, fetches current availability separately and verifies inventory and price on the purchase operation. If the first response must include current stock, put that requirement explicitly in the renderer and cache design.
4. **A team proposes adding Express because the frontend uses npm. What evidence would justify the service?** Ask which request it would handle, who handles it today, what trust or aggregation requirement it satisfies, and who will operate it. If the only artifacts are HTML, JS and CSS and the existing API already serves the data, a static host can deliver the frontend.
5. **A product page has great TTFB but still looks blank for two seconds. What would you inspect?** Compare the document body, script download and execution, the start of the data request, and the LCP element. If the product request only starts after a large script runs, evaluate loading data earlier, reducing client code, prebuilding the route or rendering the product HTML on the server. Repeat under the same device and network conditions.

For your next application, pick one public route, one private route and one write operation. Draw the HTTP exchanges for a cold visit and a repeat visit. Mark where code executes, which response can be shared, what can fail, who enforces permissions and which measurement would cause you to change the design. If two routes need different answers, record the decisions separately.
