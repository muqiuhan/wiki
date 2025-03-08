#ocaml #fp #database #postgresql

> On dealing with dependencies in your Dune-powered OCaml app and interfacing with the most popular DBMS in town.

_This article is part of Hands-on OCaml, a series of articles that I’m working on that is focusing on doing web app development with OCaml._

The project we will be building throughout the series is a To-Do List app, which connects to a PostgreSQL database as its datastore. In the previous article, we have covered initializing and bootstrapping our project with Dune; if you haven’t seen it, check out the link below:

This tutorial will build upon the foundation we have laid out in the previous article in the series. While you can of course follow along without actually doing the tutorial, it is recommended to give the article a read first to be sure that we have the required knowledge in place.

In this second article of Hands-on OCaml series, we will explore how to manage dependencies in OCaml project; in particular, we will bring in and use a third-party library to deal with DB operations.

# Requirements

Assuming you followed the previous article, you should have both `opam` and `jbuilder` installed. Verify their installation as follows:

```
$ opam --version  
1.2.2
$ jbuilder --version  
1.0+beta20
```
We would also need to have a local PostgreSQL instance up and running. I’m assuming readers have had prior experience with PostgreSQL, so I’m not going to expand about it here, but you should be able to install it via your OS package manager (e.g. `apt` on Ubuntu or `brew` on MacOS) or via a Docker container. Verify that it is running and and you can connect to it via `psql`. At the time of this writing, I am using locally installed PostgreSQL 10.4.

# Preparing our project

In this section we’ll see how we will prepare our project. Again, this tutorial assumes that you have done the initial [Dune setup tutorial](https://medium.com/@bobbypriambodo/starting-an-ocaml-app-project-using-dune-d4f74e291de8). If you haven’t, do check it out. You might also want to remove the previously created `.mli` and `.ml` files from `bin` and `lib` since we won’t need them anymore.

## Our first opam file

First off, we will fetch some dependencies! Unlike the previous tutorial where we install packages directly, we are going to use a different and cleaner method of installing dependencies: through opam files.

Make sure you’re in the `todolist` directory, create a file named `todolist.opam` with the following contents:

```
opam-version: "1.2"
name: "todolist"
version: "1.0.0"
maintainer: "Your Name <email@example.com>"

depends: [
  "jbuilder" {build}
  "lwt"
  "lwt_ppx"
  "caqti"
  "caqti-lwt"
  "caqti-driver-postgresql"
]
```