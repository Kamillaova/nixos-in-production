# The Nix store

Through this book we've only interacted with the Nix ecosystem through two interfaces:

- the Nix expression language

  Think: `.nix` files.  This is the Nix source code that we edit to define our flakes and NixOS configurations that we build and deploy.


- the Nix command-line interface

  In other words, the `nix` command-line tool and its subcommands.

However, we've glossed over an important implementation detail of how all of the Nix tooling works, which is the Nix store.  The only clue so far that the Nix store exists is various log messages or error messages that display long filepaths whose names begin with `/nix/store/…`, like this:

```
copying path '/nix/store/ayaymfnf7mrwv9v9a7jkajggy2lw13w8-libavif-0.11.1' from 'https://cache.nixos.org'...
copying path '/nix/store/igzabwg6mivj9j4cbkd570mws0haj9n7-gd-2.3.3' from 'https://cache.nixos.org'...
building '/nix/store/0nq1q1dfq97r7zp6mm9zmcn89v79aqaf-ensure-all-wrappers-paths-exist.drv'...
copying path '/nix/store/9iiqv6c3q8zlsqfp75vd2r7f1grwbxh7-nginx-1.24.0' from 'https://cache.nixos.org'...
building '/nix/store/9vn8j8gw99zp2fiqnbnfvy7c8jbicma3-etc-os-release.drv'...
building '/nix/store/pa40bsjxs7wjhvhp5r6w4549gc3qnrrl-firewall-start.drv'...
```

You don't really need to understand how the Nix store works on the happy path (if nothing goes wrong), but if things break or you need to do something weird or fancy then you will probably need a better understanding of the Nix store.  This chapter will dive into the Nix store and cover the most important concepts and commands you need to effectively work with the store.

## Introduction

So what *is* the Nix store?

The most important part of the Nix store is the `/nix/store` directory[^1], which stores two types of things:

- Derivations

  Derivations are files that end with a `.drv` extension that are stored inside of the `/nix/store` directory.  For example, the name of a derivation might be something like `/nix/store/mlvp9pl1k8a014iqvmfihcpi3xgas01c-hello-2.12.1.drv`.

  Derivations represent language-agnostic instructions for how to create build products.


- Build products

  All of the other paths inside of the `/nix/store` (the ones that don't end with `.drv`) are build products.

  These paths can be individual files (like `/nix/store/z37qsb6fnzyicmq26342lfqdssjjl5aa-fix-diff-D.patch`) or directories (like `/nix/store/7b0rz3bnx7msw2wawkv1hhn5lqf1b0wi-python3-3.11.6/`).

You might wonder: what is the deal with derivations?  What purpose do they serve?  Why does Nix store derivations inside the Nix store alongside build products?

## Derivations

You can think of derivations as an intermediate representation in Nix's build pipeline, which has three stages:

- the Nix expression language

  This is the Nix source code stored in `.nix` files that users are expected to edit and interact with.

  You can think of Nix expressions as the "frontend" for the Nix programming language.


- derivation files

  Nix source code is converted to `.drv` files stored inside the `/nix/store` directory.  These derivation files do **NOT** store Nix source code, but rather they store plain/inert data.  In case you are curious, the file format derivations use is ATerm, but this is an even more obscure implementation detail.  The Nix command-line interface converts derivation files to JSON when displaying them to end users so you can safely ignore the fact that they are stored as ATerm values.

  You can think of derivations as the "backend" for the Nix programming language but also the "frontend" for the Nix store.

- Nix build products

  These are the executables, directories, and files we build with Nix.

  You can think of build products as the "backend" for the Nix store.

The process of converting a Nix expression to a derivation is called "instantiation" and the process of converting a derivation to a build product is called "realization".  This means that a complete Nix build has two steps: instantiation + realization:

![](./pipeline.png)

You can think of the "Nix store" as more of a service rather than a store.  Most Nix experts who use the phrase "Nix store" are referring not just to the repository of derivations and build products but all of the supporting infrastructure (like the Nix daemon and Nix command-line interface) that converts derivations to build products.

The Nix programming language and the Nix store work together to provide the complete build pipeline and derivations are the interface between the two.

{blurb, class:information}
The reason for this separation of responsibilities is so that you can (in principle) swap out the front-end language and replace the Nix programming language with a different language if you don't like Nix while still getting all the nice feature of the Nix store like caching, remote builds, and closure management.  In fact, if you're a company with an existing build tool that is showing its age you might consider swapping out your tool's backend with the Nix store while keeping your existing frontend the same.  This is outside the scope of this book, though.

In fact, this is not a hypothetical scenario: [Guix](https://guix.gnu.org/) works this way.  Guix is sort of based on the Nix store[^2], but replaces the frontend language with Guile Scheme.  Any language that can generate derivation files can interoperate with the Nix store.
{/blurb}

[^1]: Technically the location of this directory is configurable using [the `store` `nix.conf` option](https://nixos.org/manual/nix/stable/command-ref/conf-file.html#conf-store) but the vast majority of Nix installations will use the default.  Moreover, you probably don't want to change the store path because then you can't use the public Nix cache or share build products with Nix stores that use a different store path.

[^2]: Guix technically uses a fork of the original Nix store, so unfortunately it won't interoperate with Nix's store.  This is partly because at the time Nix's store supported some features specific to the Nix programming language and Guix also wanted its store to support features specific to Guile Scheme.  In other words, both tools were guilty of "layer violations" although Nix has been steadily fixing its own layer violations.

TODO:

- Closures
  - Runtime
  - Buildtime
- GC roots
- `nix` commands
- `nix.conf` options (e.g. `auto-optimise-store` or `keep-{outputs,derivations}` or `min-free` / `max-free`)