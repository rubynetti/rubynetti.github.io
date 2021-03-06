---
layout: post
title: Running Rails inside a Container
categories: docker rails
author: Giacomo Bertoldi
---

In the
[previous_step]({% post_url 2018-04-16-rails-basic-dockerfile %})
we built the image and run a container ready for a Rails app.
Now we are going to run the app inside it.
We won't dive into
_[Compose](https://docs.docker.com/compose/overview/)_
yet and use just Docker to do that.


#### Binding the working directory

First thing to do is copying the application files inside the container.
We could do that by using an _ADD_ instruction in the _Dockerfile_,
but rebuilding the image after any change during development would not be sustainable.

So we bind the working directory of the project to the one in the container with the _-v (source:destination)_ option.
When _source_ is a path it is bound to the _destination_ in the container
(you can use _$PWD_ as the source path if you _cd_ to the application path)
```
$ docker run --rm -it -v /path-to-my-awesome-app:/app my-awesome-app bash
rails@container-id:/app$
```
If you ```ls``` you should see all the application files, including the _Gemfile_.


#### Start the application

You can now bundle as you usually would
```
rails@container-id:/app$ bundle
Fetching gem metadata from https://rubygems.org/.........
Fetching rake 12.3.1
Installing rake 12.3.1
Fetching concurrent-ruby 1.0.5
.
.
.
Bundle complete! 18 Gemfile dependencies, 78 gems now installed.
Bundled gems are installed into `/usr/local/bundle`
```

Migrate the database
```
rails@container-id:/app$ rails db:migrate
```

And finally run the application
```
rails@container-id:/app$ rails s
=> Booting Puma
=> Rails 5.2.0 application starting in development
=> Run `rails server -h` for more startup options
Puma starting in single mode...
* Version 3.11.4 (ruby 2.5.0-p0), codename: Love Song
* Min threads: 5, max threads: 5
* Environment: development
* Listening on tcp://0.0.0.0:3000
Use Ctrl-C to stop
```

**Note**
To receive requests from outside the container rails server should not be listening on _localhost_. If that is the case you should bind the server to all interfaces with the option _-b '0.0.0.0'_.
I found it is generally better to always specify binding and port for the server, for example ``` rails s -b 0.0.0.0 -p 3000```


#### Publish the ports

If you open a browser and navigate to localhost:3000 you'll find you can't connect to the service.
That's because right now the container is running isolated from the host machine. To connect to the service inside it, we need to publish the port used by it.

The _-p host-port:container-port_ option does that, mapping a port on the container to one on the host machine.
So we can stop rails server
```
^C- Gracefully stopping, waiting for requests to finish
=== puma shutdown: 2018-05-01 08:25:17 +0000 ===
- Goodbye!
Exiting
```

Exit the container
```
rails@4a4a10dbde3e:/app$ exit
exit
$
```

And run again it with the ports option, using rails default port both on the container and the host machine (you can actually use whatever ports you want, as long as the second one is the rails server one)
```
$ docker run --rm -it -v /path-to-my-awesome-app:/app -p 3000:3000 my-awesome-app bash
rails@container-id:/app$
```


#### Bundle volume

Trying to start the server won't work now
```
rails@container-id:/app$ rails s
bash: rails: command not found
```
What's happening here is we need to _bundle_ again since every time we _docker run_, a new container is spawned that does not share anything with the previous (or future) ones.

I can feel the pain waiting for nokogiri to download, again.

Docker lets sharing files between containers using named volumes, with the _-v name:destination_ option. When the first argument is a string instead of a path, _-v_ creates a named volume and mount it in the _destination_ path inside the container.

Named volumes are persisted by default when the container is stopped and even removed. This can be used to share bundle files between runs of the container.

Running _bundle config_ we check the path for the bundle
```
rails@container-id:/app$ bundle config
Settings are listed in order of priority. The top value will be used.
path
Set via BUNDLE_PATH: "/usr/local/bundle"
...
```

Then we can exit the container, and run it adding the volume option (_-v app-bundle:bundle-path_)
```
docker run --rm -it -v $PWD:/app -v app-bundle:/usr/local/bundle -p 3000:3000 my-awesome-app bash
```


#### Check it out

We can finally bundle and start the rails server.
Connection to the service should be available on host at _localhost:3000_.

If we remove and run again the container, with the same options as last time, _bundler_ should now use the gems from the volume when possible, completing the bundle in no time.


#### A little verbose

We just started adding options and the command
_```docker run --rm -it -v $PWD:/app -v app-bundle:/usr/local/bundle -p 3000:3000 my-awesome-app bash```_
is already starting to become long, clumsy and difficult to read.

To overcome this, _docker-compose_ can be used to set docker configuration in a file, instead of the command.


<hr/>

**Next step:**
Compose _(coming soon)_

**Previous step:** [Adding the Dockerfile]({% post_url 2018-04-16-rails-basic-dockerfile %})
