# `@inngest/eslint-plugin`

> [!WARNING]
> This package is currently in alpha and is undocumented. Use with caution.

An ESLint plugin and config for [`inngest`](/packages/inngest/).

## Getting started

Install the package using whichever package manager you'd prefer as a [dev dependency](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#devdependencies).

```sh
npm install -D @inngest/eslint-plugin
```

Add the plugin to your ESLint configuration file with the recommended config.

```json
{
  "plugins": ["@inngest"],
  "extends": ["plugin:@inngest/recommended"]
}
```

You can also manually configure each rule instead of using the `plugin:@inngest/recommend` config.

```json
{
  "plugins": ["@inngest"],
  "rules": {
    "@inngest/await-inngest-send": "warn"
  }
}
```

See below for a list of all rules available to configure.

## Rules

- [@inngest/await-inngest-send](#inngestawait-inngest-send)
- [@inngest/no-nested-steps](#inngestno-nested-steps)
- [@inngest/no-variable-mutation-in-step](#inngestno-variable-mutation-in-step)

### @inngest/await-inngest-send

You should use `await` or `return` before `inngest.send().

```json
"@inngest/await-inngest-send": "warn" // recommended
```

In serverless environments, it's common that runtimes are forcibly killed once a request handler has resolved, meaning any pending promises that are not performed before that handler ends may be cancelled.

```ts
// ❌ Bad
inngest.send({ name: "some.event" });
```
```ts
// ✅ Good
await inngest.send({ name: "some.event" });
```

#### When not to use it

There are cases where you have deeper control of the runtime or when you'll safely `await` the send at a later time, in which case it's okay to turn this rule off.

### @inngest/no-nested-steps

Use of `step.*` within a `step.run()` function is not allowed.

```json
"@inngest/no-nested-steps": "error" // recommended
```

Nesting `step.run()` calls is not supported and will result in an error at runtime. If your steps are nested, they're probably reliant on each other in some way. If this is the case, extract them into a separate function that runs them in sequence instead.

```ts
// ❌ Bad
await step.run("a", async () => {
  const someValue = "...";
  await step.run("b", () => {
    return use(someValue);
  });
});
```
```ts
// ✅ Good
const aThenB = async () => {
  const someValue = await step.run("a", async () => {
    return "...";
  });

  return step.run("b", async () => {
    return use(someValue);
  });
};

await aThenB();
```

### @inngest/no-variable-mutation-in-step

Do not mutate variables inside `step.run()`, return the result instead.

```json
"@inngest/no-variable-mutation-in-step": "error" // recommended
```

Inngest executes your function multiple times over the course of a single run, memoizing state as it goes. This means that code within calls to `step.run()` is not called on every execution.

This can be confusing if you're using steps to update variables within the function's closure, like so:

```ts
// ❌ Bad
// THIS IS WRONG!  step.run only runs once and is skipped for future
// steps, so userID will not be defined.
let userId;

// Do NOT do this!  Instead, return data from step.run.
await step.run("get-user", async () => {
  userId = await getRandomUserId();
});

console.log(userId); // undefined
```

Instead, make sure that any variables needed for the overall function are _returned_ from calls to `step.run()`.

```ts
// ✅ Good
// This is the right way to set variables within step.run :)
const userId = await step.run("get-user", () => getRandomUserId());

console.log(userId); // 123
```
