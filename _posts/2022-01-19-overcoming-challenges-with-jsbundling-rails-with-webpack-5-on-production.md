---
title: jsbundling-rails with Webpack 5 on Production
description: Challenges we faced when migrating to jsbundling-rails along with Webpack 5 and running it on production.
tags: ["Ruby on Rails", "jsbundling-rails", "Webpack 5"]
author: Sri Vishnu Totakura
layout: post
---

At [Zavvy](https://www.zavvy.io), we have been wanting to migrate to Webpack 5
for a long time now.
Our app was running on Rails 6.1 with
[Webpacker](https://github.com/rails/webpacker).
Our development environment is setup with Docker and running Webpacker
in Docker with webpack-dev-server has been very suboptimal and slow.
Webpack itself, is also very slow and we wanted to migrate away from it and
move to other bundlers like Esbuild.

Rails team has
[announced](https://world.hey.com/dhh/rails-7-will-have-three-great-answers-to-javascript-in-2021-8d68191b)
that Webpacker will be retired and that
[`jsbundling-rails`](https://github.com/rails/jsbundling-rails) is their answer
to bundling Javascript in Rails apps going forward.
Soon after `jsbundling-rails` gem was ready, we decided to migration to it
from Webpacker.
Our decision was to first migrate to jsbundling-rails along with Webpack 5 and
later consider migrating to another, faster bundler like esbuild.

After a few months of slow progress amidst demanding product features, we
have now successfully deployed to production with Webpack 5 and
`jsbundling-rails`.
This post documents some of the challenges we faced and how we resolved them.

Let's start with a simple overview of our final configuration after the
migration.
Webpack entrypoint files are under `app/javascript/entrypoints` directory.
Webpack bundles these files and other chunks or asset files
(JS, CSS, images etc.) and places them in `app/assets/builds/` directory.
Rails' Asset pipeline picks them from this directory and compiles them when
`rake assets:precompile` is run.

Generally, the entrypoints bundled by Webpack are bundled without any
digest appended in the filename.
When Sprockets precompiles these assets, it appends the digest and places them
in `public/assets/` directory.
This way, we simply can using `javascript_include_tag` Rails helper with the
entrypoint name Asset pipeline correctly includes the compiled file.

For example:

```erb
# source entrypoint
app/javascript/entrypoints/app.js

# after Webpack bundling
app/assets/builds/app.js

# after Asset pipeline
public/assets/app-5698cfb0cae1ccdeeba796484254062f529d8b5fc574c77bbb98ff9b6c3104bd.js

# include in HTML with
<%= javascript_include_tag app.js %>
```

This works fine for the basic cases.
However, no Webpack build configuration is simple and neither is ours.

## Chunks, Sourcemaps and Asset pipeline (Sprockets)

First issue we faced was with chunks configuration.
Webpack generates chunks with incremental file names like `app-chunk-0.js`,
`app-chunk-1.js` etc., and there is no way for us to know how many chunks
are created and how they are named.
So, we cannot simply use `javascript_include_tag` for each chunk.

Our solution to this is to use lazy-loaded chunks.
In case of lazy-loaded chunks, we can simply include the main entrypoint file
using `javascript_include_tag app.js` and when the page loads, Javascript
asynchronously loads the chunks.
Since webpack is bundling `app.js`, it will inject the logic necessary to load
the required chunks.
However, Webpack is unaware of Rails asset pipeline and the digest that it
generates to load these assets correctly.

Luckily, Sprockets has been updated to workaround such cases.
The goal is to tell Webpack to generate the chunk file with a digest already
and make Sprockets not digest these files again.

Note: At the time of this writing, Sprockets has not yet released a version
with the required patches for this to work correctly.
See [pull request](https://github.com/rails/sprockets/pull/714).
However, the required changes are merged into `main` branch.
So, we installed Sprockets from it's `main` branch and updated
Webpack config to output the chunk with the digest hash and `.digested` suffix
as follows:

```ruby
# Gemfile

# Temporarily reference directly to the commit until
# https://github.com/rails/sprockets/pull/714 changes are released.
gem "sprockets", github: "rails/sprockets", ref: "4aa1c55e66463f982c05cc85b94375be52d0d3f9"
```

```javascript
# webpack.config.js

module.exports = {
  //...
  output: {
    // NOTE: filename is not touched. We don't want digest included in it's name
    //...
    chunkFilename: "[name]-[contenthash].digested.js",
    //...
  },
};

```

We faced the same issue with Sourcemaps as well.
Webpack adds the `sourceMappingURL` comment with the URL to the source map file.
However, just like with the chunks, Webpack needs to generate them with the
digest already so that Asset pipeline doesn't add it's own digest to it.

```javascript
module.exports = {
  //...
  output: {
    // NOTE: filename is not touched. We don't want digest included in it's name
    //...
    chunkFilename: "[name]-[contenthash].digested.js",
    sourceMapFilename: "[file]-[fullhash].digested.map",
    //...
  },
};
```

After, we had chunks working with lazy loading and sourcemaps working correctly.

## After deployment, Asset precompilation is not moving assets generated by Webpack to public directory

As we started deploying to our staging server with Capistrano, we noticed that
Webpack is bundling our files  but somehow the files from `builds` directory
are not picked up by the precompilation process.
If we run `rake assets:precompile` second time, it seems to work fine.

After 5 hours of debugging, we realised that the `app/assets/builds` directory
is gitingored and is being cleared by Webpack for every build.

The fix was to add a `.keep` file to `app/assets/builds` and configure Webpack
to not clear this directory completely so that the directory exists before
the build process actually starts.
Turns out `jsbundling-rails` needs the directory to be present before the
precompilation process is triggered.

```
# .gitignore

/app/assets/builds/*
!/app/assets/builds/.keep
```

```javascript
module.exports = {
  //...
  output: {
    //...
    clean: {
      keep: /.keep/,
    },
  },
};
```

See [this issue](https://github.com/rails/jsbundling-rails/issues/23)
for reference.

## Sprockets breaks Sourcemaps with a semicolon

Once we deployed, everything was working well except for Sourcemaps.
We noticed that the browsers were requesting sourcemaps with a `;` in the end
of the URL.

After inspection, it was clear that Sprockets (this is what Rails uses for
Asset compilation) was appending a `;` to the end of the `sourceMappingURL`
comment during the asset precompile process.

Example:

```js
//# sourceMappingURL=app-5698cfb0cae1ccdeebd.js.map;
```

Turns out, there is no immediate solution available for this.
See [the issue on jsbundling-rails repo](https://github.com/rails/jsbundling-rails/issues/24).

#### Configure nginx to remove the semicolon

Our solution to this was to configure nginx with a `rewrite` regex rule to
redirect all sourcemap requests with a trailing `;` to the same URL without the
`;`.

```nginx
server {
  # ...

  # We have an issue with sprockets adding a trailing semi-colon to the
  # sourcemap URLs when the JS files are bundled by external bundlers like
  # Webpack.
  # So, we rewrite the path to remove the semi-colon
  # See:
  #   - https://github.com/rails/jsbundling-rails/issues/24
  location ~ "^\/assets\/.*\.map;$" {
    rewrite ^/assets/(.*).map\;$ /assets/$1.map permanent;
  }

  # ...
}
```

## Deployments without rebundling with Webpack when no changes to assets

After fixing all the challenges above, we were able to successfully deploy the
app to our staging instance and tested it thoroughly.

With subsequent deployments, we noticed that Webpack is rebuilding JS bundles
even though there are no changes to be rebuilt.
This adds a few minutes to the deployment time and few is a lot in case our
app is down and we have to deploy a fix urgently.

We first tried using [Webpack cache](https://webpack.js.org/configuration/cache/)
to improve the build time.
We setup a `filesystem` cache and made sure all the directories are added to the
`linked_dirs` in Capistrano configuration.
Webpack still would use cache during deployments for some reason.
Instead of trying figure out the solution to that, we took a different approach.

We decided to use `git diff` to see if any of the files are changed compared to
the previously deployed version and run `assets:precompile` only if there are
changes to files that affect the Javascript builds.
There is a
[Makandra card](https://makandracards.com/makandra/52744-capistrano-+-rails-automatically-skipping-asset-compilation-when-assets-have-not-changed)
about the same.
It defines a new Capistrano task that will look for the file changes and removes
the tasks from the `assets:precompile` deploy stage.
We adapted it to our setup by adding all the files and directories that are to
be checked for changed.
This is what we added to our `config/deploy.rb`:

```ruby
# Skip assets if nothing that effects them changes:
namespace :deploy do
  desc "Automatically skip asset compile if possible"
  task :auto_skip_assets do
    # rubocop:disable Layout/LineLength
    locations_affecting_assets = %r(^(Gemfile\.lock|app/assets|app/javascript|lib/assets|vendor/asset|\.browserslistrc|\.nvmrc|babel\.config\.js|globalSetup\.js|package\.json|postcss\.config\.js|tailwind\.config\.js|tsconfig\.build\.json||tsconfig\.json|webpack\.common\.js||webpack\.prod\.js|yarn\.lock))
    # rubocop:enable Layout/LineLength

    revisions = []
    on roles :app do
      within current_path do
        revisions << capture(:cat, "REVISION").strip
      end
    end

    # Never skip asset compile when servers are running on different code
    next if revisions.uniq.length > 1

    changed_files = `git diff --name-only #{revisions.first}`.split
    if changed_files.grep(locations_affecting_assets).none?
      warn "** Assets have not changed since last deploy. Will skip asset:precompile."
      warn "If you think this is a mistake, looks for +locations_affecting_assets+ variable "\
           "capistrano config or tasks and make sure it includes all the files "\
           "and directories that are to be checked."

      invoke "deploy:skip_assets"
    end
  end
  before "assets:precompile", "auto_skip_assets"

  desc "Skip asset compile"
  task :skip_assets do
    warn Airbrussh::Colors.yellow("** Skipping asset compile.")
    Rake::Task["deploy:assets:precompile"].clear_actions
  end
end
```

Now, assets are precompiled only when there are changes to the files that affect
them and we saved a few minutes in our deployment.

The downside to this approach is that we have to keep
`locations_affecting_assets` updated according to the changes to our Javascript
setup.
If we add new configuration files, we have to make sure that they are added to
the check; otherwise, the assets won't be bundled correctly.
Luckily, we will notice that immediately as we first always deploy to Staging.

## A note on Hot Module Replacement (HMR)

Unfortunately, with this setup, we can not use HMR in development.
This is because Rails is now made unaware of the Javascript bundler and there
are no helpers built for this purpose.

We accepted not having HMR for now and if this becomes an absolute necessasity,
we think we understand the internals of how HMR works and we will be able to
build a solution ourselves by extending Rails' helpers.
There might also be new open-source libraries that enable that functionality
in the future as more Rails app take this approach with `jsbundling-rails`.


## Summary

This has been our journey with upgrading to jsbundling-rails and Webpack 5.
We are loooking forward to using this setup because it eliminates a lot of
inconsistencies that our front-end developers face while developing with
Webpacker and having to regularly restart Docker engine because Rails server
from Docker container won't communicate correctly with the Webpack dev server.

This setup also enables us to later move away from Webpack 5 independently from
Rails to faster alternatives.

We hope this article helps others to easily plan a migration to
`jsbundling-rails`.
Please do let us know your experiences in the comments.

Thank you for reading!
