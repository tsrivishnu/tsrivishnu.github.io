---
title: Posts
---

## Posts

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url }})
{{ post.description }}

{% for tag in post.tags %}`{{ tag | upcase }}` {% endfor %}

{% endfor %}

### [Resources to get familiar with Webpacker, Webpack and Rails](https://gist.github.com/tsrivishnu/94b6334eefe2a23afbabba1a65591bb6)
When I started a new Rails 6 project, as a primarily backend developer, I found it intimidating to start developing with Webpack.
These are some resources that helped me through to get familiar with Webpack, Webpacker and how to develop with them.

`WEBPACK` `WEBPACKER` `RAILS`

### [Increasing `fs.inotify.max_user_watches` for Docker images](https://gist.github.com/tsrivishnu/9f551ef0098021a913e01d6d594c555d)
You can't set those for an image during build time becasue Docker takes the sysctl configuration from the Host. So, set the config on your host machine...

`DOCKER` `DEVOPS`

### [Debugging sudden increase in Postgres disk usage](https://blog.experteer.engineering/debugging-sudden-increase-in-postgres-disk-usage.html)
Postgres' pg_xlog (Write Ahead Log) grew to take up full disk space. How did we resolve it?

`POSTGRES`

### [What are parents on a merge commit?](https://blog.experteer.engineering/what-are-parents-on-git-merge-commits.html)
Understand parents on a merge commit to be able to revert or cherry-pick.

`GIT`

### [Addressing Kafka's corrupted index files warnings after restarting brokers](https://blog.experteer.engineering/kafka-corrupted-index-file-warnings-after-broker-restart.html)
We were upgrading our Kafka cluster to the newest versions following the rolling upgrade plan according to the documentation. As we restarted brokers, even without upgrading them, they take a lot of time to join the cluster...

`KAFKA`, `DEVOPS`

### [Avoiding extra actions on resourceful Rails controllers](https://blog.experteer.engineering/avoiding-extra-actions-on-resourceful-rails-controllers.html)
Rails advises to keep controllers and its actions resourceful buy providing resourceful routes helpers to create routes with default CRUD actions...

`RAILS`

### [Javascript sourcemaps in Rails with Sprockets 3 and Uglifier.](https://blog.experteer.engineering/generating-sourcemaps-with-sprockets-3-and-uglify.html)
How to generate JS Sourcemaps in Rails

`RAILS`, `JAVASCRIPT`

### [Trackpoint scrolling on Thinkpad T440p running Ubuntu 16.04](https://gist.github.com/tsrivishnu/5b467f07374ce42ad6d97b6a3fdf0ea5)
How to enable Middle button (plus Trackpoint) scrolling on Thinkpad T440p running Ubuntu 16.04

`UBUNTU` `THINKPAD`

### [How to match a pattern and return only a part of the lines from a file](https://gist.github.com/tsrivishnu/d92a34a36cdf4f4e11b16c9be34f2c5e)
How to use `sed` to extract matched strings from a file

`BASH` `SED` `DEV TOOLS`

### [Git: How can I ignore a file that is already committed to the repo?](https://gist.github.com/tsrivishnu/a2f3adbbca9fcad5f3597af301ad1abb)
How to Ignore already checked in files in Git?

`GIT`


