---
title: Ruby on Rails Background Processing on Google App Engine with ActiveJob and Sidekiq
description: Learn how to run background jobs using Ruby on Rails' ActiveJob.
author: chingor13
tags: App Engine, Ruby, Ruby on Rails, ActiveJob, Sidekiq
date_published: 2017-06-08
---
This tutorial shows how to create and configure a [Ruby on Rails](http://rubyonrails.org/) application to run
background processing jobs on Google App Engine Flexible Environment using
[ActiveJob](http://guides.rubyonrails.org/active_job_basics.html) and [Sidekiq](http://sidekiq.org/).

## Objectives

* Create a Ruby on Rails application
* Create a background processing job
* Deploy your application to Google App Engine Flexible Environment
* Verify background jobs are running

## Before you begin

You'll need the following:

* A Google Cloud Console project. You can use an existing project or click the button to create a new project
* [Ruby 2.2.2+ installed](https://www.ruby-lang.org/en/documentation/installation/)
* [Google Cloud SDK installed](https://cloud.google.com/sdk/downloads)
* A Redis instance running in your project. Follow [this guide](setting-up-redis.md)
  to set up Redis on Google Compute Engine. This tutorial assumes the Redis instance is running in the *default*
  network so that the App Engine services can access it without restriction.

## Costs

This tutorial uses billable components of Cloud Platform including:

* Google App Engine Flexible Environment

Use the [pricing calculator](https://cloud.google.com/products/calculator/)
to generate a cost estimate based on your projected usage. New Cloud Platform users might be eligible for a
[free trial](https://cloud.google.com/free-trial).

## Installing Rails and creating your application

If you already have a Rails application, you can skip to [Creating your background job](#creating-your-background-job).
For this tutorial, we will be using the latest version of Rails (currently 5.1.1).

1. Install the `rails` gem:

        gem install rails

1. Create a new rails application called `rails-background-jobs`:

        rails new rails-background-jobs

1. The rest of the tutorial command line tools will run from the Rails project root. Change directories into
   the Rails project root:

        cd rails-background-jobs

Congratulations, you have created a new skeleton Rails application called `rails-background-jobs`.

## Creating your background job

Starting with version 4.2, Rails includes an abstraction layer around background job processing called
[ActiveJob](http://guides.rubyonrails.org/active_job_basics.html). This abstraction allows you to write the queuing
and execution logic independently of queue and job runner implementations.

Ruby on Rails provides command-line tools for generating templated skeletons for things such as database migrations,
controllers, and even background jobs.

You will create a job named `HelloJob` that will accept a `name` argument and "Hello #{name}" to standard output.

1. Use the rails generator feature to create `HelloJob`:

        bin/rails generate job Hello

   Rails creates stub files from templates:

        invoke  test_unit
        create    test/jobs/hello_job_test.rb
        create  app/jobs/hello_job.rb
        create  app/jobs/application_job.rb

1. Edit your `app/jobs/hello_job.rb` with the following:

        class HelloJob < ApplicationJob
          queue_as :default

          def perform(name)
            # Do something later
            puts "Hello, #{name} from ActiveJob!"
          end
        end

## Create a test url to queue the job

1. Create a controller/action for queuing `HelloJob`. Create a file `app/controllers/hello_controller.rb`
   with the following:

        class HelloController < ApplicationController
          def say
            HelloJob.perform_later(params[:name])
            render plain: 'OK'
          end
        end

   This action will queue our `HelloJob` with the parameter name provided.

1. Create a route to this action. In `config/routes.rb`, add:

        get '/hello/:name', to: 'hello#say'

   When you make an HTTP GET request to `/hello/Jeff`, the `HelloController` will handle the request using the `say`
   action with parameter `:name` as "Jeff"

1. Add a route for Google App Engine health checks. In `config/routes.rb`, add:

        get '/_ah/health', to: proc { [200, {}, ['ok']]}

   Without this route, Google App Engine will think your application is crashing and will not send any traffic to it.

## Configuring your background worker to use Sidekiq

ActiveJob can be configured with various different background job runners. This tutorial will cover Sidekiq which
requires a Redis instance to manage the job queue.

1. Obtain the internal address of your redis instance. In the Cloud Platform Console, go to the
   **[VM Instances](https://console.cloud.google.com/compute/instances)** page and find the internal IP address of
   your Compute Engine instance with Redis installed. This IP address will be provided via environment variables
   at deploy time to configure Sidekiq.

1. Add `sidekiq` gem to your `Gemfile`:

        bundle add sidekiq

1. Configure ActiveJob to use Sidekiq as its queue adapter. In `config/application.rb`:

        class Application < Rails::Application
          # ...
          config.active_job.queue_adapter = :sidekiq
        end

## Deploying to App Engine Flexible Environment

### Option A: Shared worker and web application

For this option, the App Engine service will run both the web server and a worker process via a process manager called
[foreman](https://ddollar.github.io/foreman/). If you choose this method, App Engine will scale your web and worker
instances together.

1. Add `foreman` gem to your `Gemfile`:

        bundle add foreman

1. Create a `Procfile` at the root of your application:

        web: bundle exec rails server -p 8080
        worker: bundle exec sidekiq

1. Create an `app.yaml` for deploying the application to Google App Engine:

        runtime: ruby
        env: flex

        entrypoint: bundle exec foreman start

        env_variables:
          REDIS_PROVIDER: REDIS_URL
          REDIS_URL: redis://[REDIS_IP_ADDRESS]:6379
          SECRET_KEY_BASE: [SECRET_KEY]

   Be sure to replace the `[REDIS_IP_ADDRESS]` with the internal IP address of your Redis instance. Also be sure to
   replace the `[SECRET_KEY]` with a secret key for Rails sessions.

1. Deploy to App Engine

        gcloud app deploy app.yaml

### Option B: Separate worker and web application

For this option, you are creating 2 App Engine services - one runs the web server and one runs worker processes. Both
services use the same application code. This configuration allows you to scale background workers independently of
your web workers at the cost of potentially using more resources.

1. Create an `app.yaml` for deploying the web service to Google App Engine:

        runtime: ruby
        env: flex

        entrypoint: bundle exec rails server -p 8080

        env_variables:
          REDIS_PROVIDER: REDIS_URL
          REDIS_URL: redis://[REDIS_IP_ADDRESS]:6379
          SECRET_KEY_BASE: [SECRET_KEY]

   Be sure to replace the `[REDIS_IP_ADDRESS]` with the internal IP address of your Redis instance. Also be sure to
   replace the `[SECRET_KEY]` with a secret key for Rails sessions.

1. Create a `workers.yaml` for deploying the workers service to Google App Engine:

        runtime: ruby
        env: flex
        service: workers

        entrypoint: bundle exec sidekiq

        env_variables:
          REDIS_PROVIDER: REDIS_URL
          REDIS_URL: redis://[REDIS_IP_ADDRESS]:6379
          SECRET_KEY_BASE: [SECRET_KEY]

        health_check:
          enable_health_check: False

   Be sure to replace the `[REDIS_IP_ADDRESS]` with the internal IP address of your Redis instance. Also be sure to
   replace the `[SECRET_KEY]` with a secret key for Rails sessions.

   Note that the health check is disabled here because the workers service is not running a web server and cannot
   respond to the health check ping.

1. Deploy both services to App Engine

        gcloud app deploy app.yaml workers.yaml

## Verify your background queuing works

1. In the Cloud Platform Console, go to the
   **[App Engine Services](https://console.cloud.google.com/appengine/services)** page. Locate the service that is
   running your background workers (if option A, it should be the *default* service, if option B, it should be
   the *workers* service). Click Tools -> Logs for that service.

1. On the Logs dashboard, click the Play icon to start streaming the logs.

1. In a separate window, navigate to your deployed Rails application at:

        https://[YOUR_PROJECT_ID].appspot.com/say/Jeff

   Be sure to replace `[YOUR_PROJECT_ID]` with your Google Cloud Platform project ID.

1. Navigate back to the Logs dashboard and you should see a logging statement like:

        13:13:52.000 Hello, Jeff from ActiveJob!

Congratulations, you have successfully set up background job processing on Google App Engine with Sidekiq.

## Cleaning up

After you've finished this tutorial, you can clean up the resources you created on Google Cloud Platform
so you won't be billed for them in the future. The following sections describe how to delete or turn off these
resources.

### Deleting the project

The easiest way to eliminate billing is to delete the project you created for the tutorial.

To delete the project:

1. In the Cloud Platform Console, go to the **[Projects](https://console.cloud.google.com/iam-admin/projects)** page.
1. Click the trash can icon to the right of the project name.

**Warning**: Deleting a project has the following consequences:

If you used an existing project, you'll also delete any other work you've done in the project.
You can't reuse the project ID of a deleted project. If you created a custom project ID that you plan to use in the future, you should delete the resources inside the project instead. This ensures that URLs that use the project ID, such as an appspot.com URL, remain available.

### Deleting App Engine services

To delete an App Engine service:

1. In the Cloud Platform Console, go to the **[App Engine Services](https://console.cloud.google.com/appengine/services)** page.
1. Click the checkbox next to the service you wish to delete.
1. Click the Delete button at the top of the page to delete the service.

If you are trying to delete the *default* service, you cannot. Instead:

1. Click on the number of versions which will navigate you to App Engine Versions page.
1. Select all the versions you wish to disable and click the Stop button at the top of the page. This will free
   all of the Google Compute Engine resources used for this App Engine service.

## Next steps

* Explore [the example code](https://github.com/chingor13/rails-background-jobs) used in this tutorial.