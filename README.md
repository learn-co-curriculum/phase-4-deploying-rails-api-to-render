# Deploying a Rails API to Render

## Learning Goals

- Set up your local environment for deploying with Render
- Deploy a basic Rails application to Render

## Introduction

In this lesson, we'll be deploying a basic, standalone Rails API application to
Render. We'll give instructions to generate the application from scratch and
talk through the steps to get the code running on a Render server.

In coming lessons, we'll learn how to add more complexity to the application
with a React frontend. Since the setup for a Rails-React application is a bit
trickier, it'll be beneficial to see the setup for Rails alone first. Let's get
started!

## Environment Setup

To make sure you're able to deploy your application, you'll need to do the
following:

### Sign Up for a Render Account

You can sign up at for a free account at
[https://dashboard.render.com/register][Render signup]. Sign up using whichever
method you prefer then, from the Render dashboard, go ahead and connect Render
to your GitHub account. Once you've done that, you should see all your repos
listed in the "Connect a repository" window.

### Install the Latest Ruby Version

Verify which version of Ruby you're running by entering this in the terminal:

```console
$ ruby -v
```

We recommend version 2.7.4. If you need to upgrade you can install it using rvm:

```console
$ rvm install 2.7.4 --default
```

You should also install the latest versions of `bundler` and `rails`:

```console
$ gem install bundler
$ gem install rails
```

### Install PostgreSQL

Render requires that you use PostgreSQL for your database instead of SQLite.
PostgreSQL (or just Postgres for short) is an advanced database management
system with more features than SQLite. If you don't already have it installed,
you'll need to set it up.

#### PostgreSQL Installation for WSL

To install Postgres for WSL, run the following commands from your Ubuntu
terminal:

```console
$ sudo apt update
$ sudo apt install postgresql postgresql-contrib libpq-dev
```

Then confirm that Postgres was installed successfully:

```console
$ psql --version
```

Run this command to start the Postgres service:

```console
$ sudo service postgresql start
```

Finally, you'll also need to create a database user so that you are able to
connect to the database from Rails. First, check what your operating system
username is:

```console
$ whoami
```

If your username is "ian", for example, you'd need to create a Postgres user
with that same name. To do so, run this command to open the Postgres CLI:

```console
$ sudo -u postgres -i
```

From the Postgres CLI, run this command (replacing "ian" with your username):

```console
$ createuser -sr ian
```

Then enter `control + d` or type `logout` to exit.

[This guide][postgresql wsl] has more info on setting up Postgres on WSL if you
get stuck.

#### PostgreSQL Installation for OSX

To install Postgres for OSX, you can use Homebrew:

```console
$ brew install postgresql
```

Once Postgres has been installed, run this command to start the Postgres
service:

```console
$ brew services start postgresql
```

Phew! With that out of the way, let's get started on building our Rails
application and deploying it to Render.

## Creating a Rails App to Deploy

We'll be following the steps in the [Getting Started with Ruby on Rails on
Render][getting started with rails] guide, so if you get stuck and are looking
for more assistance, check that guide first.

The first thing we'll need to do is create our new Rails application. Make
sure you're in a non-lab directory, then run:

```console
$ rails new bird-app --api --minimal --database=postgresql
```

This will set up our app to run in API mode, with the minimum dependencies
needed, and with PostgreSQL as the database.

Next, we'll need to configure our `Gemfile.lock` file to support the same OS as
Render, which runs Ubuntu. This way, regardless of what OS you're using in
development, `bundler` will be able to install the same gems on Render using any
Ubuntu-specific gem dependencies.

`cd` into the app, and run this command:

```console
$ bundle lock --add-platform x86_64-linux
```

This will add additional platforms to your `Gemfile.lock` file that will allow
the necessary dependencies to be installed after you deploy your app.

## Building the Demo App

Next, let's set up up a migration, model, route, and controller so that
we have some data to display in our application:

```console
$ rails g resource Bird name species
```

Add this data to the `db/seeds.rb` file:

```rb
Bird.create!(name: 'Black-Capped Chickadee', species: 'Poecile Atricapillus')
Bird.create!(name: 'Grackle', species: 'Quiscalus Quiscula')
Bird.create!(name: 'Common Starling', species: 'Sturnus Vulgaris')
Bird.create!(name: 'Mourning Dove', species: 'Zenaida Macroura')
```

Then run this command to generate the database and run the migrations and seed
file:

```console
$ rails db:create db:migrate db:seed
```

> `rails db:create` creates a new PostgreSQL database to be associated with your
> application based on the configuration in the `config/database.yml` file.
> Unlike with SQLite, the actual database file isn't created in the `db` folder;
> it lives elsewhere in your file system, depending on your PostgreSQL
> configuration. If you have problems with this step, see the
> **Troubleshooting** section below.

Next, edit the `app/birds_controller.rb` file and add an `index` action:

```rb
  # GET /birds
  def index
    birds = Bird.all
    render json: birds
  end
```

Finally, open `config/routes.rb`, un-comment out the root path definition and
update it to:

```rb
root "birds#index"
```

To make sure the app works locally before deploying, run `rails s`. If you visit
either [http://localhost:3000](http://localhost:3000) or
[http://localhost:3000/birds](http://localhost:3000/birds), you should see the
JSON for the list of birds.

## Preparing your App for Deployment

Before we can deploy our app, we need to make a few modifications.

First, open the `config/database.yml` file, scroll down to the `production`
section, and update the code to the following:

```yml
production:
  <<: *default
  url: <%= ENV['DATABASE_URL'] %>
```

Next, open `config/puma.rb` and find the section shown below. Here, you will
un-comment out two lines of code and make one small edit:

```rb
# Specifies the number of `workers` to boot in clustered mode.
# Workers are forked web server processes. If using threads and workers together
# the concurrency of the application would be max `threads` * `workers`.
# Workers do not work on JRuby or Windows (both of which do not support
# processes).
#
workers ENV.fetch("WEB_CONCURRENCY") { 4 } ### CHANGE: Un-comment out this line; update the value to 4

# Use the `preload_app!` method when specifying a `workers` number.
# This directive tells Puma to first boot the application and load code
# before forking the application. This takes advantage of Copy On Write
# process behavior so workers use less memory.
#
preload_app! ### CHANGE: Un-comment out this line
```

Next, open the `config/environments/production.rb` file and find the following
line:

```rb
config.public_file_server.enabled = ENV["RAILS_SERVE_STATIC_FILES"].present?
```

Update it to the following:

```rb
config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present? || ENV['RENDER'].present?
```

Finally, inside the `bin` folder create a `birds-build.sh` script and copy the
following into it:

```sh
#!/usr/bin/env bash
# exit on error
set -o errexit

bundle install
# bundle exec rake assets:precompile # These lines are commented out because we have an API only app
# bundle exec rake assets:clean
bundle exec rake db:migrate 
bundle exec rake db:seed
```

Our API-only app doesn't include any assets, so we've commented out the lines to
precompile and clean them.

Then run the following command in the terminal to make sure the script is
executable:

```console
chmod a+x bin/birds-build.sh
```

## Deploying

### Push the Code to GitHub

In order to deploy our app to Render, we first need to create a remote repo and
push our code up. Start by making a commit to save your local changes:

```console
$ git add .
$ git commit -m 'Initial commit'
```

Then, on the repository list page of your GitHub account, click the green "New"
button in the upper right corner. (Alternatively, you can navigate to
[https://github.com/new](https://github.com/new)). In the form that opens, enter
a name for your repo (`bird-app` makes sense) and make sure "Public" is
selected. You can leave everything else as is. Click the "Create repository"
button at the bottom of the page.

On the next page, copy the code in the "push an existing repository from the
command line" section and run it in your terminal:

```console
git remote add origin git@github.com:<your-github-name>/bird-app.git
git branch -M main
git push -u origin main
```

When you refresh the GitHub page, you should see that your code has been pushed
up.

### Create the Database on Render

Go to the [Render dashboard][], click the "New +" button and select
"PostgreSQL". Enter a name for your database — this can be whatever you like.
The remaining fields can be left as is.

![Creating a new database](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/create-database.png)

Scroll to the bottom of the page and click "Create Database". Leave the database
page open — you'll need to copy information from it in the next step.

### Create the Web Service on Render

Open a second tab and navigate back to the Render dashboard. Click the "New +"
button and select "Web Service". Enter a name for your app and make sure the
Environment is set to Ruby:

![Creating a new web service](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/create-web-service.png)

Scroll down and set the Build Command to `./bin/birds-build.sh` and the Start
Command to `bundle exec puma -C config/puma.rb`:

![Update Build and Start commands](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/update-commands.png)

Next, scroll down and click the "Advanced" button, then click "Add Environment
Variable." Enter `DATABASE_URL` as the key, then navigate back to the tab you
left open with your database information. Click the "Connect" button in the
upper right corner, copy the Internal Database URL, and paste it into the value
box.

Click "Add Environment Variable" again. Add `RAILS_MASTER_KEY` as the key. The
value is in the `config/master.key` file in your app's files. Copy the value and
paste it into the web form.

![Add environment variables](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/add-env-variables.png)

Scroll down to the bottom of the page and click "Create Web Service". The deploy
process will begin automatically. Warning: this process can take a while! You
might want to go get a snack or go for a walk.

When the deployment is complete, you should see something like this in the log:

![Log showing successful build and deploy](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/successful-deploy-log.png)

Click on your app's URL in the upper left corner of the screen (just below the
name of the app). Once the page has loaded (which may take a few moments), you
should once again see the JSON for the list of birds. If you get a "Page not
found" error, wait a few minutes then try refreshing the page.

## Adding New Features

Since Render integrates the deploying process with GitHub, it's straightforward
to add new features to your code and deploy them. Let's start by adding a new
controller action in the `BirdsController`:

```rb
def show
  bird = Bird.find(params[:id])
  render json: bird
rescue ActiveRecord::RecordNotFound
  render json: "Bird not found", status: :not_found
end
```

Test your code locally by running `rails s` and visiting
[http://localhost:3000/birds/1](http://localhost:3000/birds/1).

After adding this code, make a commit and push it to GitHub:

```console
$ git add app/controllers/birds_controller.rb
$ git commit -m 'Add show action'
$ git push
```

After pushing the new code, Render should automatically being re-deploying it,
although there might be a bit of a delay. You can also start the deploy
manually: return to the app's page on Render, click the "Manual Deploy" button
in the upper right corner and select "Deploy latest commit." You will now see
the deploy from the new commit in progress:

![Deploying new action](https://curriculum-content.s3.amazonaws.com/phase-4/deploying-rails-api/deploying-new-content.png)

Once the deploy is complete, refresh the page and verify that the show action is
working. Remember that it may take a few minutes for the new content to become
available.

## Troubleshooting

If you ran into any errors along the way, here are some things you can try to
troubleshoot:

- If you're on a Mac and got a server connection error when you tried to run
  `rails db:create`, one option for solving this problem for Mac users is to
  install the Postgres app. To do this, first uninstall `postgresql` by running
  `brew remove postgresql`. Next, download the app from the
  [Postgres downloads page][] and install it. Launch the app and click
  "Initialize" to create a new server. You should now be able to run
  `rails db:create`.

- If you're using WSL and got the following error running `rails db:create`:

  ```txt
  PG::ConnectionBad: FATAL:  role "yourusername" does not exist
  ```

  The issue is that you did not create a role in Postgres for the default user
  account. Check [this video](https://www.youtube.com/watch?v=bQC5izDzOgE) for
  one possible fix.

- If your app failed to deploy at the build stage, make sure your local
  environment is set up correctly by following the steps at the beginning of
  this lesson. Check that you have the latest versions of Ruby and Bundler, and
  ensure that PostgreSQL was installed successfully.

- If you deployed successfully, but you ran into issues when you visited the
  site, make sure you migrated and seeded the database. Also, make sure that
  your application works locally and try to debug any issues on your local
  machine before re-deploying. You can also check the deployment log on the
  app's page in the Render dashboard.

## Conclusion

Congrats on deploying your first Rails app to the world wide web! Understanding
the deployment process and what it takes to run your application on another
computer is an important step toward becoming a full-stack developer. Like
anything new, this process can be daunting the first time you try it, but with
practice and exposure, you'll build confidence over time.

In the next lesson, we'll work on deploying a more complex application with a
Rails API backend and a React frontend, and talk through some of the challenges
of running these two applications together.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. When creating a new Rails app from the terminal, what additional flag do you
   need to use to be able to deploy it on Render?
2. What familiar process is used for deploying code to Render?

## Resources

- [Getting Started with Ruby on Rails on Render][getting started with rails]
- [Render Databases Guide][databases guide]

[Render signup]: https://dashboard.render.com/register
[postgresql wsl]: https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-database#install-postgresql
[getting started with rails]: https://render.com/docs/deploy-rails
[Render dashboard]: https://dashboard.render.com/
[databases guide]: https://render.com/docs/databases
[postgres downloads page]: https://postgresapp.com/downloads.html
