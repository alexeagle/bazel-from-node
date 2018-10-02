# From Node

This repo demonstrates how to bootstrap a Bazel development environment assuming
you have Node.js and yarn already installed on your machine.

You don't need to install any other dependencies like Java or Bazel.

This illustrates a typical workflow for a frontend developer, who already knows
how to use `node`, `npm`, or `yarn`.
Such an engineer might work at a company where the corporate IT department
manages the image for developer machines and doesn't give developers
administrator rights on their machine, so installing other software is hard.

## Prerequisites

`node` version 8 or 10 installed, and optionally `yarn`.

## Bootstrapping a bazel build

We'll start from a `package.json` file, which we can create in a new directory
with `yarn`.

```sh
$ yarn init -y
```

Now we'll install a toolchain for building TypeScript code:

```sh
$ yarn add -D @bazel/bazel @bazel/typescript typescript
```

## Bazel setup

Bazel always operates inside a Workspace, defined as a directory containing a
`WORKSPACE` file.
This is how Bazel finds its plugins, and we need the TypeScript plugin,
which is called rules_typescript. Create a file called
`WORKSPACE` and put this in it:

```python
http_archive(
    name = "build_bazel_rules_typescript",
    url = "https://github.com/bazelbuild/rules_typescript/archive/0.19.1.zip",
    strip_prefix = "rules_typescript-0.19.1",
)

# Fetch our Bazel dependencies that aren't distributed on npm
load("@build_bazel_rules_typescript//:package.bzl", "rules_typescript_dependencies")
rules_typescript_dependencies()

# Setup the NodeJS toolchain
load("@build_bazel_rules_nodejs//:defs.bzl", "node_repositories", "yarn_install")
node_repositories()

# Setup TypeScript toolchain
load("@build_bazel_rules_typescript//:defs.bzl", "ts_setup_workspace")
ts_setup_workspace()

yarn_install(
    name = "npm",
    package_json = "//:package.json",
    yarn_lock = "//:yarn.lock",
)
```

Note how we referenced our `package.json` and `yarn.lock` file so Bazel will
know how to install its own copy of our dependencies.

To let Bazel find these two files, we also need a file named `BUILD.bazel` next
to the `WORKSPACE` file. This is where we will put our build system
configuration, for now it can be empty.

## TypeScript setup

We already installed the TypeScript package. Now we can use it to initialize a
new TypeScript project in this directory:

```sh
$ yarn run tsc --init
```

## Building TypeScript code with Bazel

Now we can create a simple TypeScript app, let's call it `app.ts`:

```typescript
const el: HTMLDivElement = document.createElement('div');
el.innerText = 'Hello, TypeScript';
el.className = 'ts1';
document.body.appendChild(el);
```

To configure Bazel, we don't tell it what to do. We just describe the layout of
our sources. In this case, we need to add this to `BUILD.bazel`:

```python
load("@build_bazel_rules_typescript//:defs.bzl", "ts_library", "ts_devserver")
ts_library(name = "app", srcs = [":app.ts"])
```

> The load statement is just like an `import` or `require` statement to get the
> `ts_library` symbol, which is declaring that we'd like to compile these files
> together into a library.

Now we can ask Bazel to build this:

```sh
$ yarn run bazel build :app
Target //:app up-to-date:
  bazel-bin/app.d.ts
```

> Of course we could set up scripts in the `package.json` file to make a
> shortcut for running our build or test commands.

## Serving the app to a browser

> 😢 This step is currently broken on Windows 😢

One reason to consider using Bazel is that it can build code in many different
languages as part of a single build. We will illustrate this by using a
development server written in Go. (We wrote it in Go to support massive web
apps at Google which have thousands of files).

To be able to compile Go code, add this to the end of `WORKSPACE`:

```python
load("@io_bazel_rules_go//go:def.bzl", "go_rules_dependencies", "go_register_toolchains")
go_rules_dependencies()
go_register_toolchains()
```

Then add this to your `BUILD.bazel` file:

```python
ts_devserver(name = "devserver", deps = [":app"])
```

You can now run the server with

```sh
$ yarn run bazel run :devserver
```

Click the link that's printed there, and you should see "Hello, TypeScript"
appear in the browser.
