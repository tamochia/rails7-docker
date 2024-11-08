# Rails 7 application development environment on Docker containers

Setting Up a Docker Development Environment for Rails Applications.

When setting up a Docker-based environment to develop Rails applications, there are two common patterns:

- Using a pre-built container image that includes a basic Rails application structure
- Preparing a container image with Ruby and Bundler installed, then running rails new inside the container

This guide explains how to build with the latter pattern.

In this Rails application, we will not use Webpack or Node.js (yarn) and instead adopt the following stack:

- **Propshaft** - Handles the asset pipeline (digest hashes for filenames, public directory deployment)
- **Importmap** - Manages JavaScript and CSS dependencies without bundling
- **DartSass(dartsass-rails)** - For SCSS (Sass) transpilation
- **Tailwind CSS** - As the CSS framework

To run this application, we define two services (containers) in Docker Compose:

- **web** - a container for the Rails application
- **db** - a container for the database service 

## Setting environment variables

In the `.env` file, set the Rails project directory path, MySQL root password, container execution user name and UID.

**Example**

```bash
MYSQL_ROOT_PASSWORD=hogehoge
RAILS_APP_PATH=/sample_app
RUN_USER=woody
RUN_UID=1000
```

## Steps

### 1. Building Docker images.

Generate an empty file `Gemfile.lock` and then build the Docker image.

```bash
$ touch web/Gemfile.lock
$ docker compose build
```

### 2. Run the `rails new`.

When the `rails new` command is executed, a Dockerfile is automatically generated, so save it in advance.

```bash
$ cp -p web/Dockerfile web/Dockerfile.bak
```

Run the `rails new` command to generate a Rails project.

```bash
$ docker compose run --rm web bundle exec rails new . -a propshaft -d mysql --css tailwind --force --no-deps --skip-bundle --skip-git --skip-decrypted-diffs
```

- `-a propshaft` - Asset pipeline is Propshaft
- `-d mysql` - DBMS is MySQL
- `--css tailwind` - CSS framework is TailWind CSS

These are specified as options.

### 3. Modify the `Gemfile`. 

Modify the Gemfile to use DartSass (and dependent foreman).

```rb
source "https://rubygems.org"

# Bundle edge Rails instead: gem "rails", github: "rails/rails", branch: "main"
gem "rails", "~> 7.2.2"
:
  gem "selenium-webdriver"
end

gem "dartsass-rails"
gem "foreman"
```

### 4. Rebuild Docker images.

First, restore the saved Dockerfile.

```bash
$ mv web/Dockerfile web/Dockerfile.prod
$ mv web/Dockerfile.bak web/Dockerfile
```

Build the Docker image again because the Gemfile has been changed.

```bash
$ docker compose build
```

### 5. Run the `bundle install`.

Run bundle install as root user. Gems will be installed in volume (/usr/local/bundle/gems).

```bash
$ docker compose run --user root --rm web bundle install
```

### 6. Setting up DartSass, Importmap, and Tailwind CSS.

DartSass, Importmap, and Tailwind CSS are each set up within a Rails project.

```bash
$ docker compose run --rm web bin/rails dartsass:install importmap:install tailwindcss:install
```

On the way, an error (foreman installation failure) is output in the DartSass setup, but this is not a problem since foreman is already installed.

### 7. Modify `web/Procfile.dev`.

```diff
$ diff -U 1 web/Procfile.dev.ORG web/Procfile.dev
--- web/Procfile.dev.ORG        2024-11-06 14:53:17.343053768 +0900
+++ web/Procfile.dev    2024-11-06 14:54:23.117596740 +0900
@@ -1,2 +1,2 @@
-web: bin/rails server -p 3000
+web: bin/rails server -p 3000 -b 0.0.0.0
 css: bin/rails dartsass:watch
```

### 8. Modify `web/config/database.yml`.


```diff
$ diff -U 1 web/config/database.yml.ORG web/config/database.yml
--- web/config/database.yml.ORG 2024-11-06 14:49:28.235741129 +0900
+++ web/config/database.yml     2024-11-06 14:55:54.944611733 +0900
@@ -15,5 +15,5 @@
   pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
-  username: root
-  password:
-  host: <%= ENV.fetch("DB_HOST") { "localhost" } %>
+  username: dbadm
+  password: hogehoge
+  host: db
````
You may specify root instead of dbadm.

### 9. Start the container and create a DB.

```bash
$ docker compose up -d
$ docker compose exec web bin/rake db:create
```

### 10. Access http://localhost:3000/ in your host's browser.

