---
title: "Configure Vue.js with environment variables"
date: 2022-08-18T15:22:54+01:00
draft: false
summary: "Inject environment specific settings into your Vue.js build, allowing you to change key configuration such as API endpoints"
---

Vue.js comes with a built in way to inject different variables depending on what type of build you perform.

By default when you run `npm run serve`, `NODE_ENV` is set to `development`. When you run `npm run build`, `NODE_ENV` is to set to `production`.

If you reference an environment variable beginning with the prefix `VUE_APP_`, (eg. `process.env.VUE_APP_MY_VARIABLE`), this will be replaced with the actual value at compile time by `webpack`.

# Setting Environment Variables

You can obviously set environment variables on your machine directly, but this is hard to maintain. Instead you can utilise `dotenv` which is built into `webpack`.

- Environment variables set into `.env.development` will be loaded when you run `npm run serve`.
- Environment variables set into `.env.production` will be loaded when you run `npm run build`.

There are also other ways of setting environment variables, such as in GitHub workflows, removing the need for the `.env.production` file.

# Example

Let's imagine that our single page application has to consume an API. When we're running locally this will reference a localhost address, but then a different address once deployed.

**.env.development**

```env
VUE_APP_API_BASE_URL=http://localhost:3000
```

**.env.production**

```env
VUE_APP_API_BASE_URL=https://my-api.com
```

Then in our code, we can reference the following, at it will be replaced at compile time:

```ts
console.log(process.env.VUE_APP_API_BASE_URL);
```

# References

- https://cli.vuejs.org/guide/mode-and-env.html#environment-variables
