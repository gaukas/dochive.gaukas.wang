---
layout: default
title: Setup legacy version of Instructure Canvas LMS on Ubuntu 20.04
parent: DevOps
nav_order: 1
---

# Setup legacy version Instructure Canvas LMS on Ubuntu 20.04
This tutorial is about how to set up a specific older version of Canvas LMS made by Instructure. Normally, we do not encourage the use of any legacy software, and this is no exception. Instead this tutorial is the prerequisite for demonstrating [a CVE I have found](https://www.cve.org/CVERecord?id=CVE-2021-36539) in this version of Canvas which is partially patched and hard to be exploited in the latest version.

## Before we get started

{: .warning }
> A huge problem of Canvas's repository is that the tags are almost always unbuildable. Only a few tags can eventually be built into a Canvas instance that does the job. Another problem is with the wiki comes with the repository, where dependencies are not version-pinned. 

So for this guide, we are going to use a specific tag that is known to work: `release/2022-01-19.127`. Alternatively, `release/2021-10-13.36` is also a tag with a working copy of Canvas that I have found. Note that only the later (the Oct 2021 one) is vulnerable to CVE-2021-36539. Apparently the patch has been developed before Jan 2022, and deployed to the hosted Canvas much later than that. 

## Prerequisites

To begin with, we are assuming a fresh installation of Ubuntu 20.04 LTS. The patch version (e.g., 20.04.x) shouldn't matter but it is always recommended to use the latest version possible even when in a testing environment just to be safe. In this guide, I am demonstrating with Ubuntu 20.04.6 LTS. 

## Step 1: Install system dependencies

### APT packages

```bash
sudo apt-get update
sudo apt-get -y install ruby ruby-dev postgresql-12 zlib1g-dev libxml2-dev libsqlite3-dev libpq-dev libxmlsec1-dev curl build-essential
```

### Install Node.js

```bash
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash - # a deprecation warning (and a script deprecation warning) will be shown, ignore it and wait for the script to proceed and finish
sudo apt-get install -y nodejs 
```

### Install Yarn

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo apt-get update && sudo apt-get install yarn=1.19.1-1
```

### Setup PostgreSQL User
This sets up a PostgreSQL user with the same name as the current UNIX user and gives it superuser privileges. If your UNIX username is not `canvas`, it is okay. Later we will come back and set up `canvas` DB user. 

```bash
sudo -u postgres createuser $USER
sudo -u postgres psql -c "alter user $USER with superuser" postgres
```

### Setup Ruby Gems

```bash
sudo gem install bundler -v 2.2.19
```

## Step 2: Clone the repository

```bash
sudo apt-get install git # if you don't have git installed
git clone --single-branch --branch stable/2022-01-19 https://github.com/instructure/canvas-lms.git canvas # canvas repo is huge so we are using --single-branch to save time and space
```

## Step 3: Install Canvas dependencies

```bash
cd canvas # or wherever you cloned the repo
bundle install
yarn install --pure-lockfile --time-out 1000000 # repeat if errored 
```

It is very likely that the last command you executed will error on the first try. If it does, just run it again. Until you see something like this:

```
Done in 43.24s.
running for git dir .git
Done in 301.85s.
```

## Step 4: Configure Canvas

Copy config files: 

```bash
for config in amazon_s3 delayed_jobs domain file_store outgoing_mail security external_migration; \
          do cp -v config/$config.yml.example config/$config.yml; done
cp config/dynamic_settings.yml.example config/dynamic_settings.yml
```

## Step 5: Run Canvas file generation

Despite this is what the official documentation says, depending on the actual version of canvas you installed, directly running this MAY give you an error.  

```bash
bundle exec rails canvas:compile_assets
```

### Could not find json-2.5.1 in any of the sources

You might see some error like "Could not find json-2.5.1 in any of the sources", while the version number might differ. To fix this, run the following command:

```bash
sudo gem install json # You will see a version (likely 2.7.1) gets installed
bundle install
```

### NameError: uninitialized constant Nokogiri::HTML4

This is due to the incompatible version of `loofah` gem being installed. To fix this, add this line to your `Gemfile`:

```ruby
gem 'loofah', '~>2.19.1'
```

Then, refresh the bundle:

```bash
bundle update loofah
bundle install
```

## Step 6: Database setup

```bash
cp config/database.yml.example config/database.yml
createdb canvas_development
```

Remember to edit `config/database.yml` to set up the database user and password. 

```yaml
# do not create a queue: section for your test environment
test:
  adapter: postgresql
  encoding: utf8
  database: canvas_test
  host: localhost
  username: canvas
  password: your_password
  timeout: 5000

development:
  adapter: postgresql
  encoding: utf8
  database: canvas_development
  password: 000000 # edit this to your own password
  timeout: 5000
  secondary:
    username: sysadmin # edit this to your account username or a different name that you will create as step 1

production:
  adapter: postgresql
  encoding: utf8
  database: canvas_production
  host: localhost
  username: canvas
  password: your_password
  timeout: 5000
```

### Troubleshooting: createdb: could not connect to database postgres

According to the official documentation you MIGHT encounter this error:

```
createdb: could not connect to database postgres: could not connect to server: No such file or directory
    Is the server running locally and accepting
    connections on Unix domain socket "/var/pgsql_socket/.s.PGSQL.5432"?
```

See [this wiki page](https://github.com/instructure/canvas-lms/wiki/Quick-Start/19a83bf7191fcd23b55198ef90270bc0902b6ba1#database-configuration) for the solution. 

## Step 7: Database population

```bash
bundle exec rails db:initial_setup
```

This wizard will interactively ask you for the canvas admin email and password. 

## Step 8: Start the server

Now you should be able to start the server:

```bash
bundle exec rails server # binds to 127.0.0.1:3000
```

or, if you want to specify the IP to bind

```bash
bundle exec rails server --binding=IPAddress
```

(Likely you need this if you run it on a remote server/VM, etc.)

### Troubleshooting: Blocked host

Once the server is up and you access it from a browser, you might see an error like this:

```
Blocked host: 192.168.118.134


To allow requests to 192.168.118.134, add the following to your environment configuration:

config.hosts << "192.168.118.134"
```

This is introduced in Rails 6. You may edit `config/environments/development.rb` and add the following line:

```ruby
config.hosts.clear
``` 

to disable this feature. 

At the end of the file, it should look like this:

```ruby
  # allow any additional hosts
  ENV["ADDITIONAL_ALLOWED_HOSTS"]&.split(",")&.each do |host|
    config.hosts << host
  end

  config.hosts.clear # add this line

  # eval <env>-local.rb if it exists
  Dir[File.dirname(__FILE__) + "/" + File.basename(__FILE__, ".rb") + "-*.rb"].each { |localfile| eval(File.new(localfile).rea>
end
```

Now save, exit and restart the server. You just finished setting up Canvas LMS! 
