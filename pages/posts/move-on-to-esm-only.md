---
title: Move on to ESM-only
date: 2025-02-05T00:00:00Z
lang: en
duration: 15min
description: Let's move on to ESM-only
---

[[toc]]

Three years ago, I wrote a post about [shipping ESM & CJS in a single package](/posts/publish-esm-and-cjs), advocating for dual CJS/ESM formats to ease user migration and trying to make the best of both worlds. Back then, I didn't fully agree with [aggressively shipping ESM-only](https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c), as I considered the ecosystem wasn't ready, especially since the push was mostly from low-level libraries. Over time, as tools and the ecosystem have evolved, my perspective has gradually shifted towards more and more on adopting ESM-only.

As of 2025, a decade has passed since ESM was first introduced in 2015. Modern tools and libraries have increasingly adopted ESM as the primary module format. According to {@wooorm}'s [script](https://github.com/wooorm/npm-esm-vs-cjs), the packages that ships ESM on npm in 2021 was **7.8%**, and by the end of 2024, it had reached [**25.8%**](https://github.com/wooorm/npm-esm-vs-cjs). Although a significant portion of packages still use CJS, the trend clearly shows a good shift towards ESM.

<figure>
  <img src="/images/npm-esm-vs-cjs-2024.svg" dark:filter-invert />
  <figcaption text-center>ESM adoption over time, generated by the <code>npm-esm-vs-cjs</code> script. Last updated at 2024-11-27</figcaption>
</figure>

Here in this post, I'd like to share my thoughts on the current state of the ecosystem and why I believe it's time to move on to ESM-only.

## The Toolings are Ready

### Modern Tools

With the rise of [Vite](https://vite.dev) as a popular modern frontend build tool, many meta-frameworks like [Nuxt](https://nuxtjs.org), [SvelteKit](https://kit.svelte.dev), [Astro](https://astro.build), [SolidStart](https://solidstart.dev), [Remix](https://remix.run), [Storybook](https://storybook.js.org), [Redwood](https://redwoodjs.com), and many others are all built on top of Vite nowadays, that **treating ESM as a first-class citizen**.

As a complement, we have also testing library [Vitest](https://vitest.dev), which was designed for ESM from the day one with powerful module mocking capability and efficient fine-grain caching support.

CLI tools like [`tsx`](https://github.com/privatenumber/tsx) and [`jiti`](https://github.com/unjs/jiti) offer a seamless experience for running TypeScript and ESM code without requiring additional configuration. This simplifies the development process and reduces the overhead associated with setting up a project to use ESM.

Other tools, for example, [ESLint](https://eslint.org), in the recent v9.0, introduced a new flat config system that enables native ESM support with `eslint.config.mjs`, even in CJS projects.

### Top-Down & Bottom-Up

Back in 2021, when {@sindresorhus} first started migrating all his packages to ESM-only, for example, `find-up` and `execa`, it was a bold move. I consider this move as a **bottom-up** approach, as the packages that rather low-level and many their dependents are not ready for ESM yet. I was worried that this would force those dependents to stay on the old version of the packages, which might result in the ecosystem being fragmented. (As of today, I actually appreciate that move bringing us quite a lot of high-quality ESM packages, regardless that the process wasn't super smooth).

It's way easier for an ESM or Dual formats package to depend on CJS packages, but not the other way around. In terms of smooth adoption, I believe the **top-down** approach is more effective in pushing the ecosystem forward. With the support of high-level frameworks and tools from top-down, it's no longer a significant obstacle to use ESM-only packages. The remaining challenges in terms of ESM adoption primarily lie with package authors needing to migrate and ship their code in ESM format.

### Requiring ESM in Node.js

The [capability to `require()` ESM modules](https://joyeecheung.github.io/blog/2024/03/18/require-esm-in-node-js/) in Node.js, [initiated](https://github.com/nodejs/node/pull/51977) by {@joyeecheung}, marks an **incredible milestone**. This feature allows packages to be published as ESM-only while still being consumable by CJS codebases with minimal modifications. It helps avoid the [async infection](/posts/async-sync-in-between) (also known as [Red Functions](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)) introduced by dynamic `import()` ESM, which can be pretty hard, if not impossible in some cases, to migrate and adapt.

This feature was recently [unflagged](https://github.com/nodejs/node/pull/55085) and [backported to Node.js v22](https://github.com/nodejs/node/pull/55217) ([and soon v20](https://github.com/nodejs/node/pull/56927)), which means it should be available to many developers already. Consider the [top-down or bottom-up](#top-down--bottom-up) metaphor, this feature actually makes it possible to start ESM migration also from **middle-out**, as it allows import chains like `ESM → CJS → ESM → CJS` to work seamlessly.

To solve the interop issue between CJS and ESM in this case, [Node.js also introduced](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require) a new `export { Foo as 'module.exports' }` syntax in ESM to export CJS-compatible exports (by [this PR](https://github.com/nodejs/node/pull/54563)). This allows package authors to publish ESM-only packages while still supporting CJS consumers, without even introducing breaking changes (expcet for changing the required Node.js version).

For more details on the progress and discussions around this feature, keep track on [this issue](https://github.com/nodejs/node/issues/52697).

## The Troubles with Dual Formats

While dual CJS/ESM packages have been a quite helpful transition mechanism, they come with their own set of challenges. Maintaining two separate formats can be cumbersome and error-prone, especially when dealing with complex codebases. Here are some of the issues that arise when maintaining dual formats:

### Interop Issues

Fundamentally, CJS and ESM are different module systems with distinct design philosophies. Although Node.js has made it possible to import CJS modules in ESM, dynamically import ESM in CJS, and even `require()` ESM modules, there are still many tricky cases that can lead to interop issues.

One key difference is that CJS typically uses a single `module.exports` object, while ESM supports both default and named exports. When authoring code in ESM and transpiling to CJS, handling exports can be particularly challenging, especially when the exported value is a non-object, such as a function or a class. Additionally, to make the types correct, we also need to introduce further complications with `.d.mts` and `.d.cts` declaration files. And so on...

As I am trying to explain this problem deeper, I found that I actually wish you didn't even need to be bothered with this problem at all. It's frankly too complicated and frustrating. If you are just a user of packages, let alone the package authors to worry about that. This is one of the reasons I advocate for the entire ecosystem to transition to ESM, to leave these problems behind and spare everyone from this unnecessary hassle.

### Dependency Resolution

When a package has both CJS and ESM formats, the resolution of dependencies can become convoluted. For example, if a package depends on another package that only ships ESM, the consumer must ensure that the ESM version is used. This can lead to version conflicts and dependency resolution issues, especially when dealing with transitive dependencies.

Also for packages that are designed to used with singleton pattern, this might introduce multiple copies of the same package and cause unexpected behaviors.

### Package Size

Shipping dual formats essentially doubles the package size, as both CJS and ESM bundles need to be included. While a few extra kilobytes might not seem significant for a single package, the overhead can quickly add up in projects with hundreds of dependencies, leading to the infamous node_modules bloat. Therefore, package authors should keep an eye on their package size. Moving to ESM-only is a way to optimize it, especially if the package doesn't have strong requirements on CJS.

## When Should We Move to ESM-only?

This post does not intend to diminish the value of dual-format publishing. Instead, I want to encourage evaluating the current state of the ecosystem and the potential benefits of transitioning to ESM-only.

There are several factors to consider when deciding whether to move to ESM-only:

### New Packages

I strongly recommend that **all new packages** be released as ESM-only, as there are no legacy dependencies to consider. New adopters are likely already using a modern, ESM-ready stack, there being ESM-only should not affect the adoption. Additionally, maintaining a single module system simplifies development, reduces maintenance overhead, and ensures that your package benefits from future ecosystem advancements.

### Browser-targeted Packages

If a package is primarily targeted for the browser, it makes total sense to ship ESM-only. In most cases, browser packages go through a bundler, where ESM provides significant advantages in static analysis and tree-shaking. This leads to smaller and more optimized bundles, which would also improve loading performance and reduce bandwidth consumption for end users.

### Standalone CLI

For a standalone CLI tool, it's no difference to end users whether it's ESM or CJS. However, using ESM would enable your dependencies to also be ESM, facilitating the ecosystem's transition to ESM from a [top-down approach](#top-down--bottom-up).

### Node.js Support

If a package is targeting the evergreen Node.js versions, it's a good time to consider ESM-only, especially with the recent [`require(ESM)` support](#requiring-esm-in-nodejs).

### Know Your Consumers

If a package already has certain users, it's essential to understand the dependents' status and requirements. For example, for an ESLint plugin/utils that requires ESLint v9, while ESLint v9's new config system supports ESM natively even in CJS projects, there is no blocker for it to be ESM-only.

Definitely, there are different factors to consider for different projects. But in general, I believe the ecosystem is ready for more packages to move to ESM-only, and it's a good time to evaluate the benefits and potential challenges of transitioning.

## How Far We Are?

The transition to ESM is a gradual process that requires collaboration and effort from the entire ecosystem. Which I believe we are on a good track moving forward.

To improve the transparency and visibility of the ESM adoption, I recently built a visualized tool called [Node Modules Inspector](https://github.com/antfu/node-modules-inspector) for analyzing your packages's dependencies. It provides insights into the ESM adoption status of your dependencies and helps identify potential issues when migrating to ESM.

Here are some screenshots of the tool to give you a quick impression:

<figure>
  <img src="/images/node-modules-inspector-1.png" scale-110 />
  <figcaption text-center>Node Modules Inspector - Overview</figcaption>
</figure>

<figure>
  <img src="/images/node-modules-inspector-2.png" scale-110 />
  <figcaption text-center>Node Modules Inspector - Dependency Graph</figcaption>
</figure>

<figure>
  <img src="/images/node-modules-inspector-3.png" scale-110 />
  <figcaption text-center>Node Modules Inspector - Reports like ESM Adoptions and Duplicated Packages</figcaption>
</figure>

This tool is still in its early stages, but I hope it will be a valuable resource for package authors and maintainers to track the ESM adoption progress of their dependencies and make informed decisions about transitioning to ESM-only.

To learn more about how to use it and inspect your projects, check the repository <GitHubLink repo="antfu/node-modules-inspector" />.

## Moving Forward

I am planning to gradually transition the packages I maintain to ESM-only and take a closer look at the dependencies we rely on. We also have plenty of exciting ideas for the Node Modules Inspector, aiming to provide more useful insights and help find the best path forward.

I look forward to a more portable, resilient, and optimized JavaScript/TypeScript ecosystem.

I hope this post has shed some light on the benefits of moving to ESM-only and the current state of the ecosystem. If you have any thoughts or questions, feel free to reach out using the links below. Thank you for reading!
