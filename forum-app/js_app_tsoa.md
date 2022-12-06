TypeScript Application using tsoa
=================================

This guide shows how to build a TypeScript application using Node.js,
npm, and tsoa. It is recommended to read through the
`build_apps/js_app_ts:TypeScript Application`{.interpreted-text
role="ref"} page first.

Using [tsoa](https://github.com/lukeautry/tsoa) as framework provides
the following advantages over not using a framework:

-   TypeScript types and JSDoc annotations are used to generate OpenAPI
    definitions and validate request data.
-   App metadata (`app.json`) is auto-generated as much as possible.

The source code for the example app can be found in the
[samples/apps/forum](https://github.com/microsoft/CCF/tree/main/samples/apps/forum)
folder of the CCF git repository.

::: {.note}
::: {.title}
Note
:::

tsoa currently focuses on JSON as content type. Using other content
types is possible but requires to
`manually specify the OpenAPI definition <build_apps/js_app_tsoa>`{.interpreted-text
role="ref"}.
:::

Prerequisites
-------------

The following tools are assumed to be installed on the development
machine:

-   Node.js
-   npm

Folder Layout
-------------

The sample app has the following folder layout:

``` {.bash}
$ tree --dirsfirst forum
forum
├── src
│   ├── controllers
│   │   ├── csv.ts
│   │   ├── poll.ts
│   │   └── site.ts
│   ├── models
│   │   └── poll.ts
│   ├── services
│   │   ├── csv.ts
│   │   └── poll.ts
│   ├── authentication.ts
│   ├── constants.ts
│   └── error_handler.ts
├── tsoa-support
│   ├── postprocess.js
│   └── routes.ts.tmpl
├── app.tmpl.json
├── package.json
├── rollup.config.js
├── tsconfig.json
└── tsoa.json
```

It contains these files:

-   `src/controllers/*.ts`:
    `build_apps/js_app_tsoa:Controllers`{.interpreted-text role="ref"}.
-   `src/models/*.ts`: Data models shared between different app
    components.
-   `src/services/*.ts`: Business logic, called by controllers.
-   `src/authentication.ts`: [authentication
    module](https://tsoa-community.github.io/docs/authentication.html).
    See also
    `build_apps/auth/jwt_ms_example:JWT Authentication example using Microsoft Identity Platform`{.interpreted-text
    role="ref"}.
-   `src/constants.ts`: app-wide constants.
-   `src/error_handler.ts`: global error handler.
-   `tsoa-support/*`: Supporting scripts used during
    `build_apps/js_app_tsoa:Conversion to an app bundle`{.interpreted-text
    role="ref"}.
-   `app.tmpl.json`:
    `App metadata <build_apps/js_app_tsoa:Metadata>`{.interpreted-text
    role="ref"}.
-   `package.json`: Dependencies and build command.
-   `rollup.config.js`: Rollup configuration, see
    `build_apps/js_app_tsoa:Conversion to an app bundle`{.interpreted-text
    role="ref"} for more details.
-   `tsconfig.json`: TypeScript compiler configuration.
-   `tsoa.json`: tsoa configuration.


```Note
Rollup requires exactly one entry-point module. The `auto-generated <build_apps/js_app_tsoa:Conversion to an app bundle>`{.interpreted-text role="ref"} `build/endpoints.ts` module serves that purpose and re-exports all endpoint handlers from the other files in the same folder. Keeping endpoint handlers in separate modules and referencing those directly in `app.tmpl.json` allows for fine-grained control over which other modules are loaded, per endpoint. This in turn may improve load time and/or memory consumption, for example if not all endpoints share the same npm package dependencies.
```

Dependencies
------------

The sample uses several runtime and development packages (see
`package.json`). One of them is the :typedoc`ccf-app`{.interpreted-text
role="package"} package. This package is referenced locally using
`file:`. You should replace this with a reference to a published version
(adjust the version number accordingly):

``` {.json}
"@microsoft/ccf-app": "~0.19.4",
```

Now you can continue with installing all dependencies:

``` {.bash}
$ npm install
```

Controllers
-----------

In tsoa, a controller represents a URL path, or route, together with
handlers for each supported HTTP method. Typically, each controller is
defined in its own module. tsoa discovers controllers through a list of
search locations specified in `tsoa.json`:

``` {.json}
{
    "controllerPathGlobs": [
        "src/controllers/*.ts"
    ]
}
```

As an example, the `/polls` route of the sample app is implemented as in
[src/controllers/poll.ts](https://github.com/microsoft/CCF/tree/main/samples/apps/forum/src/controllers/poll.ts).

For more information on how to write controllers, see the [tsoa
documentation](https://tsoa-community.github.io/docs/getting-started.html#defining-a-simple-controller).

::: {.note}
::: {.title}
Note
:::

`Endpoint handler functions <build_apps/js_app_bundle:Endpoint handlers>`{.interpreted-text
role="ref"}, as required by CCF\'s JavaScript app bundles, are
auto-generated from controllers during the
`conversion to an app bundle <build_apps/js_app_tsoa:Conversion to an app bundle>`{.interpreted-text
role="ref"}.
:::

::: {.tip}
::: {.title}
Tip
:::

See the :typedoc`ccf-app`{.interpreted-text role="package"} package API
documentation for how to access the Key-Value Store and other CCF
functionality. Although not recommended, instead of using the
:typedoc`ccf-app`{.interpreted-text role="package"} package, all native
CCF functionality can also be directly accessed through the
:typedoc`ccf <ccf-app/global/CCF>`{.interpreted-text role="interface"}
global variable.
:::

Request / Response objects
--------------------------

Using CCF\'s
:typedoc`Response <ccf-app/endpoints/Response>`{.interpreted-text
role="interface"} object is not needed when using tsoa because the
return value always has to be the body itself. Headers and the status
code can be set using [Controller
methods](https://tsoa-community.github.io/reference/classes/_tsoa_runtime.controller-1.html).

Sometimes though it is necessary to access CCF\'s
:typedoc`Request <ccf-app/endpoints/Request>`{.interpreted-text
role="interface"} object, for example when the request body is not JSON.
In this case, instead of using `@Body() body: MyType` as function
argument, `@Request() request: ccfapp.Request` can be used. See
[src/controllers/csv.ts](https://github.com/microsoft/CCF/tree/main/samples/apps/forum/src/controllers/csv.ts)
for a concrete example.

::: {.warning}
::: {.title}
Warning
:::

Requesting CCF\'s
:typedoc`Request <ccf-app/endpoints/Request>`{.interpreted-text
role="interface"} object via `@Request()` instead of using `@Body()`
disables automatic schema validation.
:::

Metadata
--------

App metadata is stored in an `app.tmpl.json` file in the root of the app
project. The file follows the
`metadata format <build_apps/js_app_bundle:Metadata>`{.interpreted-text
role="ref"} used by app bundles, except that the `"openapi"` field is
optional.

During
`conversion to an app bundle <build_apps/js_app_tsoa:Conversion to an app bundle>`{.interpreted-text
role="ref"} the following happens:

1.  `app.tmpl.json` is created (if it doesn\'t exist yet) and from then
    on kept up-to-date. URL paths or HTTP methods that don\'t exist
    anymore are removed, new ones are added with default metadata.
2.  The final `dist/app.json` file is generated by auto-populating
    `"openapi"` fields, if missing.

Conversion to an App Bundle
---------------------------

Preparing the app for deployment means converting it to CCF\'s native
JavaScript application format, an
`app bundle <build_apps/js_app_bundle:JavaScript Application Bundle>`{.interpreted-text
role="ref"}. This involves the following steps:

-   transform TypeScript into JavaScript,
-   transform bare imports (`lodash`) into relative imports
    (`./node_modules/lodash/lodash.js`),
-   transform old-style CommonJS modules into native JavaScript modules,
-   transform request/response TypeScript types into OpenAPI
    definitions,
-   generate a module with CCF endpoint handlers for each tsoa
    controller (`build/*Proxy.ts`),
-   generate a single entry-point module for Rollup
    (`build/endpoints.ts`),
-   generate the final `app.json` metadata file with OpenAPI definitions
    (`dist/app.json`),
-   store all files according to the app bundle folder structure
    (`dist/`).

For this, the sample app relies on the [TypeScript
compiler](https://www.npmjs.com/package/typescript),
[rollup](https://rollupjs.org),
[tsoa-cli](https://www.npmjs.com/package/@tsoa/cli), and custom scripts.
See `package.json`, `rollup.config.js`, `tsoa.json`, and `tsoa-support/`
for details.

The conversion command is invoked with

``` {.bash}
$ npm run build
```

The app bundle can now be found in the `dist/` folder and is ready to be
deployed.

Deployment
----------

After the app was converted to an app bundle, it can be wrapped into a
proposal and deployed. See the
`Deployment section of the app bundle page <build_apps/js_app_bundle:Deployment>`{.interpreted-text
role="ref"} for further details.
