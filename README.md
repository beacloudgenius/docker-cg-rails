#### Create a dummy project:

    docker run -it --rm \
        --user "$(id -u):$(id -g)" \
        -v "$PWD":/usr/src/app \
        -w /usr/src/app \
        rails:4 \
        rails new --skip-bundle dummy


`--­­user "$(id -­u):$(id -­g)"` This flag ensures that you own the files instead of root. Use the user and group for the current user.

`-v "$PWD":/usr/src/app -w /usr/src/app` allows the container to write the Rails scaffolding to our work station. This segment connects our local working directory with the /usr/src/app path within the Docker image.

`rails new --skip-bundle dummy` is the command we're passing to the Rails image.

You should see a `dummy` project in your working folder

#### Delete the dummy

    rm -rf dummy

#### Create a real project

    docker run -it --rm \
        --user "$(id -u):$(id -g)" \
        -v "$PWD":/usr/src/app \
        -w /usr/src/app \
        rails:4 \
        rails new --skip-bundle cloudgenius

#### Examine your newly create app skeleton

    cd cloudgenius

#### Setup the foundation

Add a few gems to our `Gemfile` and make a few adjustments to our application to make it production ready.

Add the following lines to the bottom of your `Gemfile`:

    gem 'unicorn', '~> 4.9'
    gem 'pg', '~> 0.18.3'
    gem 'sidekiq', '~> 4.0.1'
    gem 'redis-rails', '~> 4.0.0'

Remove `sqlite` gem from the `Gemfile`

#### Change the Database Configuration

We will be using environment variables to configure our application. This change allows us to use the `DATABASE_URL`, while also allowing us to name our databases based on the environment in which they are being run.

Change `config/database.yml` to look like this.

```
---

development:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_development?') %>

test:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_test?') %>

staging:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_staging?') %>

production:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_production?') %>
```

#### Change the Secrets File

We are setting each environment to use the same SECRET_TOKEN environment variable. This is fine, since it's value will be different in each environment.

Change your `config/secrets.yml` to look like this:

```
---

development: &default
  secret_key_base: <%= ENV['SECRET_TOKEN'] %>

test:
  <<: *default

staging:
  <<: *default

production:
  <<: *default
```

#### Update the Application Configuration

Add the following lines to your `config/application.rb` right after this lines `class Application < Rails::Application`

```

    # We want to set up a custom logger which logs to STDOUT.
    # Docker expects your application to log to STDOUT/STDERR and to be ran
    # in the foreground.
    config.log_level = :debug
    config.log_tags  = [:subdomain, :uuid]
    config.logger    = ActiveSupport::TaggedLogging.new(Logger.new(STDOUT))

    # Since we're using Redis for Sidekiq, we might as well use Redis to back
    # our cache store. This keeps our application stateless as well.
    config.cache_store = :redis_store, ENV['CACHE_URL'],
                         { namespace: 'cloudgenius::cache' }

    # If you've never dealt with background workers before, this is the Rails
    # way to use them through Active Job. We just need to tell it to use Sidekiq.
    config.active_job.queue_adapter = :sidekiq

```

#### Create the Unicorn Config for use in production

Create the `config/unicorn.rb` file and add the following content to it:

```
# Heavily inspired by GitLab:
# https://github.com/gitlabhq/gitlabhq/blob/master/config/unicorn.rb.example

# Go with at least 1 per CPU core, a higher amount will usually help for fast
# responses such as reading from a cache.
worker_processes ENV['WORKER_PROCESSES'].to_i

# Listen on a tcp port or unix socket.
listen ENV['LISTEN_ON']

# Use a shorter timeout instead of the 60s default. If you are handling large
# uploads you may want to increase this.
timeout 30

# Combine Ruby 2.0.0dev or REE with "preload_app true" for memory savings:
# http://rubyenterpriseedition.com/faq.html#adapt_apps_for_cow
preload_app true
GC.respond_to?(:copy_on_write_friendly=) && GC.copy_on_write_friendly = true

# Enable this flag to have unicorn test client connections by writing the
# beginning of the HTTP headers before calling the application. This
# prevents calling the application for connections that have disconnected
# while queued. This is only guaranteed to detect clients on the same
# host unicorn runs on, and unlikely to detect disconnects even on a
# fast LAN.
check_client_connection false

before_fork do |server, worker|
  # Don't bother having the master process hang onto older connections.
  defined?(ActiveRecord::Base) && ActiveRecord::Base.connection.disconnect!

  # The following is only recommended for memory/DB-constrained
  # installations. It is not needed if your system can house
  # twice as many worker_processes as you have configured.
  #
  # This allows a new master process to incrementally
  # phase out the old master process with SIGTTOU to avoid a
  # thundering herd (especially in the "preload_app false" case)
  # when doing a transparent upgrade. The last worker spawned
  # will then kill off the old master process with a SIGQUIT.
  old_pid = "#{server.config[:pid]}.oldbin"
  if old_pid != server.pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end

  # Throttle the master from forking too quickly by sleeping. Due
  # to the implementation of standard Unix signal handlers, this
  # helps (but does not completely) prevent identical, repeated signals
  # from being lost when the receiving process is busy.
  # sleep 1
end

after_fork do |server, worker|
  # Per-process listener ports for debugging, admin, migrations, etc..
  # addr = "127.0.0.1:#{9293 + worker.nr}"
  # server.listen(addr, tries: -1, delay: 5, tcp_nopush: true)

  defined?(ActiveRecord::Base) && ActiveRecord::Base.establish_connection

  # If preload_app is true, then you may also want to check and
  # restart any other shared sockets/descriptors such as Memcached,
  # and Redis. TokyoCabinet file handles are safe to reuse
  # between any number of forked children (assuming your kernel
  # correctly implements pread()/pwrite() system calls).
end
```


Create the Sidekiq Initialize Config

also create the `config/initializers/sidekiq.rb` file and add the following code to it:

```
sidekiq_config = { url: ENV['JOB_WORKER_URL'] }

Sidekiq.configure_server do |config|
  config.redis = sidekiq_config
end

Sidekiq.configure_client do |config|
  config.redis = sidekiq_config
end
```

#### Create the Environment Variable File

Create the `.cloudgenius.env` file and add the following code to it:

```
# You would typically use `rake secret` to generate a secure token. It is
# critical that you keep this value private in production.
SECRET_TOKEN=asecuretokenwouldnormallygohere


# Unicorn is more than capable of spawning multiple workers, and in production
# you would want to increase this value but in development you should keep it
# set to 1.
#
# It becomes difficult to properly debug code if there's multiple copies of
# your application running via workers and/or threads.
WORKER_PROCESSES=1


# This will be the address and port that Unicorn binds to. The only real
# reason you would ever change this is if you have another service running
# that must be on port 8000.
LISTEN_ON=0.0.0.0:8000


# This is how we'll connect to PostgreSQL. It's good practice to keep the
# username lined up with your application's name but it's not necessary.
#
# Since we're dealing with development mode, it's ok to have a weak password
# such as `pass` but in production you'll definitely want a better one.
#
# Eventually we'll be running everything in Docker containers, and you can set
# the host to be equal to `postgres` thanks to how Docker allows you to link
# containers.
#
# Everything else is standard Rails configuration for a PostgreSQL database.

DATABASE_URL=postgresql://cloudgenius:pass@postgres:5432/cloudgenius?encoding=utf8&pool=5&timeout=5000


# Both of these values are using the same Redis address but in a real
# production environment you may want to separate Sidekiq to its own instance,
# which is why they are separated here.
#
# We'll be using the same Docker link trick for Redis which is how we can
# reference the Redis hostname as `redis`.
CACHE_URL=redis://redis:6379/0
JOB_WORKER_URL=redis://redis:6379/0
```

This `env` file allows us to configure the application without having to dive into the application code. This is a very important step to making your application production ready.

This file would also hold information like mail login credentials or API keys. You should also add this file to your .gitignore.


#### Dockerize this application

Create a `Dockerfile` with this content

```
# Use the barebones version of Ruby 2.2.3.
FROM ruby:2.2.3-slim

# Optionally set a maintainer name to let people know who made this image.
MAINTAINER Nick Janetakis <nick.janetakis@gmail.com>

# Install dependencies:
# - build-essential: To ensure certain gems can be compiled
# - nodejs: Compile assets
# - libpq-dev: Communicate with postgres through the postgres gem
# - postgresql-client-9.4: In case you want to talk directly to postgres
RUN apt-get update && apt-get install -qq -y build-essential nodejs libpq-dev postgresql-client-9.4 --fix-missing --no-install-recommends

# Set an environment variable to store where the app is installed to inside
# of the Docker image.
ENV INSTALL_PATH /cloudgenius
RUN mkdir -p $INSTALL_PATH

# This sets the context of where commands will be ran in and is documented
# on Docker's website extensively.
WORKDIR $INSTALL_PATH

# Ensure gems are cached and only get updated when they change. This will
# drastically increase build times when your gems do not change.
COPY Gemfile Gemfile
RUN bundle install

# Copy in the application code from your work station at the current directory
# over to the working directory.
COPY . .

# Provide dummy data to Rails so it can pre-compile assets.
RUN bundle exec rake \
    RAILS_ENV=production \
    DATABASE_URL=postgresql://cloudgenius:pass@127.0.0.1/cloudgenius \
    SECRET_TOKEN=pickasecuretoken \
    assets:precompile

# Expose a volume so that nginx will be able to read in assets in production.
VOLUME ["$INSTALL_PATH/public"]

# The default command that gets ran will be to start the Unicorn server.
CMD bundle exec unicorn -c config/unicorn.rb
```

#### Create a .dockerignore files with this content

```
.git
.dockerignore
Gemfile.lock
```
This is similar in concept to `.gitgnore` as it excludes matching files and folders from being built into your Docker image.

#### Create Docker Compose configuration

Create `docker-compose.yml` file with the following content:

```
postgres:
  image: postgres:9.4.5
  environment:
    POSTGRES_USER: cloudgenius
    POSTGRES_PASSWORD: pass
  ports:
    - '5432:5432'
  volumes:
    - cloudgenius-postgres:/var/lib/postgresql/data

redis:
  image: redis:3.0.5
  ports:
    - '6379:6379'
  volumes:
    - cloudgenius-redis:/var/lib/redis/data

cloudgenius:
  build: .
  links:
    - postgres
    - redis
  volumes:
    - .:/cloudgenius
  ports:
    - '8000:8000'
  env_file:
    - .cloudgenius.env

sidekiq:
  build: .
  command: bundle exec sidekiq -C config/sidekiq.yml
  links:
    - postgres
    - redis
  volumes:
    - .:/cloudgenius
  env_file:
    - .cloudgenius.env
```
In this docker compose description:

postgres and redis use Docker volumes to manage persistence
postgres, redis and cloudgenius all expose a port
cloudgenius and sidekiq both use volumes to mount in app code for live editing
cloudgenius and sidekiq both have links to postgres and redis
cloudgenius and sidekiq both read in environment variables from .cloudgenius.env
sidekiq overwrites the default CMD to run sidekiq instead of Unicorn.

#### Create the Volumes

In the `docker-compose.yml` file, we're referencing volumes that do not exist.

Create them by running:

    docker volume create --name cloudgenius-postgres
    docker volume create --name cloudgenius-redis

When data is saved in PostgreSQL or Redis, it is saved to these volumes on your work station. This way, you won't lose your data when you restart the service because Docker containers are stateless.

#### Run Everything

Start up our stack by running the following:

    docker-compose up

The first time this command runs it will take quite a while because it needs to pull down all of the Docker images that our application requires.

This operation is mostly bound by network speed, so your times may vary.

At some point, it's going to begin building the Rails application. You will eventually see the terminal output, including lines similar to these:

    postgres_1    | ...
    redis_1       | ...
    cloudgenius_1 | ...
    sidekiq_1     | ...

You will notice that the cloudgenius_1 container throws an error saying the database doesn't exist.
```
cloudgenius_1  | E, ERROR -- : could not connect to server: Connection refused
cloudgenius_1  | 	Is the server running on host "postgres" (172.17.0.2) and accepting
cloudgenius_1  | 	TCP/IP connections on port 5432?
cloudgenius_1  |  (PG::ConnectionBad)

```
This is a completely normal error to expect when running a Rails application because we haven't initialized the database yet.

#### Initialize the Database

Hit CTRL+C in the terminal to stop everything. If you see any errors, you can safely ignore them.

Run the following commands to initialize the database:

    docker-compose run --user "$(id -u):$(id -g)" cloudgenius rake db:reset

    docker-compose run --user "$(id -u):$(id -g)" cloudgenius rake db:migrate

The first command should warn you that db/schema.rb doesn't exist yet, which is normal. Run the second command to remedy that. It should run successfully.
```
db/schema.rb doesn't exist yet. Run `rake db:migrate` to create it, then try again. If you do not intend to use a database, you should instead alter /cloudgenius/config/application.rb to limit the frameworks that will be loaded.
```

#### Run the stack again

Now that our database is initialized, try running the following:

    docker-compose up

Visit http://localhost:8000/ where you should be greeted with the typical Rails `Welcome aboard` page.


#### Working with the Rails Application

Now that we've Dockerized our application, let's start adding features to it to exercise the commands you'll need to run to interact with your Rails application.

Right now the source code is on your work station, and that source code is being mounted into the Docker container in real time through a volume.

This means that if you were to edit a file, the changes would take effect instantly, but right now we have no routes or any CSS defined to test this.

#### Generate a Controller

Run the following command to generate a Pages controller with a home action:

    docker-compose run \
        --user "$(id -u):$(id -g)" \
        cloudgenius \
        rails g controller Pages home

In a second or two, it should provide everything you would expect when generating a new controller.

This type of command is how you'll run future Rails commands. If you wanted to generate a model or run a migration, you would run them in the same way.

#### Modify the Routes File

Edit the `config/routes.rb`

Remove the `get 'pages/home'` line near the top and replace it with the following:

    root 'pages#home'

If you go back to your browser http://localhost:8000, you should see the new home page we set up.

#### Add a New Job

Use the following to add a new job:

    docker-compose run --user "$(id -u):$(id -g)" \
        cloudgenius \
        rails g job counter

#### Modify the Counter Job

Look for `app/jobs/counter_job.rb` and replace the perform function. The result should look like this:

```
class CounterJob < ActiveJob::Base
  queue_as :default

  def perform(*args)
    # Do something later
    21 + 21
  end
end
```

#### Modify the Pages Controller

In the file `app/controller/pages_controller.rb`, replace the home action to look like this:
```
class PagesController < ApplicationController
  def home
    # We are executing the job on the spot rather than in the background to
    # exercise using Sidekiq in a trivial example.
    #
    # Consult with the Rails documentation to learn more about Active Job:
    # http://edgeguides.rubyonrails.org/active_job_basics.html
    @meaning_of_life = CounterJob.perform_now
  end
end
```

#### Modify the Home View
Edit the file `app/views/pages/home.html.erb` and replace make it look like this:
```
<h1>The meaning of life is <%= @meaning_of_life %></h1>
```

#### Restart the Rails Application

Restart the Rails server to pickup new jobs, so hit CTRL+C to stop everything, and then run `docker-compose up` again.

If you reload the website http://localhost:8000, you should see the changes we made.
