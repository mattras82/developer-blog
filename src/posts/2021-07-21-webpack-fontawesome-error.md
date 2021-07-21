---
title: "Fix for webpack's CSS Loader error with resolving FontAwesome font files"
description: "This post shows how to easily fix an error with webpack's CSS Loader not being able to find the FontAwesome EOT or WOFF files."
date: 2021-07-21T13:00:00Z
tags:
 - webpack
 - fontawesome
 - sass
---

```bash
npm start

...

ERROR in ./scss/styles.scss
Module build failed (from ./node_modules/mini-css-extract-plugin/dist/loader.js):
ModuleBuildError: Module build failed (from ./node_modules/css-loader/dist/cjs.js):
Error: Can\'t resolve '../webfonts/fontawesome/fa-solid-900.eot' in '/home/Projects/my-site/scss'
```

If you're using FontAwesome webfonts in your webpack-powered web project, then you may run in to the above error. The cause of this error is that the CSS Loader attempts to resolve the relative URLs for the font files, but it can't find the files in webpack's array of known assets.  Fortunately, there is an easy fix. Especially if you've already copied the appropriate font files into your project.

## The Unoptimized Fix

Most of the info you can find online at [StackOverflow](https://stackoverflow.com/questions/57243755/webpack-module-not-found-error-cant-resolve-webfonts-fa-solid-900-eot) or [blog posts](https://newbedev.com/webpack-module-not-found-error-can-t-resolve-webfonts-fa-solid-900-eot) that pop up from searching the error will tell you to use webpack's [file-loader](https://webpack.js.org/loaders/file-loader/). That process works by updating your webpack configuration file to find the font files in your `node_modules` directory, then copy them to your output folder. My main issue with this method is that including this process in your main webpack flow will run that process every time that webpack is run. I prefer to copy static files once and forget about it.

## Set Up Webpack to Copy Static Files Once

If you're new to webpack, then you might just want to manually copy the FontAwesome font files into your project's public assets folder. If you do this, you can skip to the next section to find the fix for the error. However, since you're already using webpack, let's make webpack do the work for you!

First off, you'll need to install the [CopyWebpackPlugin](https://webpack.js.org/plugins/copy-webpack-plugin/).

```bash
npm i -D copy-webpack-plugin
```

The Copy plugin is exactly what we need for this solution. You can also use the plugin to copy other static files in your project that you install with NPM (ie jQuery's minified JS file, images, other font files). Most of my projects have a handful of static assets that I only need to copy into my `dist` folder when I start the project, or when the package has a new version. So I'll set up a specific webpack process to copy those files over and then connect that process to a script in my `package.json` file. Here's what that looks like:

```json
{
  "scripts": {
    "static": "webpack --env static"
  }
}
```

Webpack's CLI has a handy `--env` flag that allows you to pass data to the `env` object that is passed to your config file. In order to use that object, you must export a function in your `webpack.config.js` file.

```javascript
// Include our plugins
const CopyWebpackPlugin = require('copy-webpack-plugin');

module.exports = env => {
  if (env.static) {
    // This is where we'll copy our static files
    return {
      plugins: [
        new CopyWebpackPlugin({
          patterns: [
            {
              from: './node_modules/@fortawesome/fontawesome-free/webfonts',
              to: 'webfonts'
            }
          ]
        })
      ],
      entry: './_src/main/index.js',
      output: './dist/assets'
    }
  } else {
    // This is where we'll add our default configuration
  }
};
```

Now that our webpack file is configured to run our static process, we can run the NPM script in a terminal.

```bash
npm run static
```

That will copy the files over. However, you may notice that your JavaScript file is also processed by WebPack. If you're fine with this functionality, you can ignore the next section. If you want webpack to copy your files and not output anything else then follow along.

## Use Last Call to Remove JS Output

For this step, you'll need to install the [Last Call Webpack Plugin](https://www.npmjs.com/package/last-call-webpack-plugin).

```bash
npm i -D copy-webpack-plugin last-call-webpack-plugin
```

This plugin allows you to modify any assets just before webpack outputs them. What we want to do is intercept our JavaScript output and make it go away. This is how I handle this process in my configuration:

```javascript
// Include our plugins
const CopyWebpackPlugin = require('copy-webpack-plugin');
const LastCallPlugin = require('last-call-webpack-plugin');

module.exports = env => {
  if (env.static) {
    // This is where we'll copy our static files
    return {
      plugins: [
        new CopyWebpackPlugin({
          patterns: [
            {
              from: './node_modules/@fortawesome/fontawesome-free/webfonts',
              to: 'webfonts'
            }
          ]
        }),
        new LastCallPlugin({
          assetProcessors: [
            {
              regExp: /dummy/,
              processor: (assetName, asset, assets) => {
                  assets.setAsset('dummy.js', null);
                  return Promise.resolve();
              }
            }
          ]
        })
      ],
      entry: {
        'dummy': './_src/main/index.js'
      },
      output: {
        path: './dist/assets',
        filename: '[name].js'
      }
    }
  } else {
    // This is where we'll add our default configuration
  }
};
```

Notice how we've renamed our entrypoint to "dummy" and the output has been updated to use the filename. This allows us to target the JavaScript file and return an empty promise in the LastCallPlugin. Now when you run `npm run static` you'll get your font files copied over, but no JS file is emitted anymore. Perfect!

Now let's solve the original error message..

## Fixing URL Resolution in the Sass File
Now that you have your FontAwesome webfont files copied into your project's distribution folder, you don't need the CSS Loader to resolve the relative URLs anymore. We know the files are there, so all we need to do is tell CSS Loader to ignore the `url()` function.

Your default webpack configuration probably looks something like this:
```javascript
// Include our plugins
const CopyWebpackPlugin = require('copy-webpack-plugin');
const LastCallPlugin = require('last-call-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = env => {
  if (env.static) {
    // This is where we'll copy our static files
    ...
  } else {
    // This is our default configuration
    return {
      entry: './_src/main/index.js',
      output: './dist',
      mode: 'development',
      plugins: [
        new MiniCssExtractPlugin({
          filename: '[name].css'
        })
      ],
      module: {
        rules: [
          {
              test: /\.(js|jsx)$/,
              loader: 'babel-loader'
          },
          {
            test: /.(sa|sc|c)ss$/,
            use: 
              [
                {
                    loader: MiniCssExtractPlugin.loader
                },
                {
                    loader: 'css-loader'
                },
                {
                    loader: 'postcss-loader',
                    options: {
                        postcssOptions: {
                            plugins: function () {
                                return [
                                    require('autoprefixer')
                                ];
                            }
                        }
                    }
                }, 
                {
                    loader: 'sass-loader'
                }
              ]
          }
        ]
      }
    };
  }
};
```

If you take a look at the [CSS Loader documentation](https://webpack.js.org/loaders/css-loader/), you'll notice that there are a handful of options you can pass to the loader. The option we'll need here is the `url` setting; it's a simple boolean setting that tells the loader whether or not to process any `url()` or `image-set()` functions within your CSS code and resolve the URLs for you. By setting this option to `false`, we can fix our webpack error and get back to coding.

```javascript
// Include our plugins
const CopyWebpackPlugin = require('copy-webpack-plugin');
const LastCallPlugin = require('last-call-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = env => {
  if (env.static) {
    // This is where we'll copy our static files
    ...
  } else {
    // This is our default configuration
    return {
      ...
      module: {
        rules: [
          ...
          {
            test: /.(sa|sc|c)ss$/,
            use: 
              [
                ...
                {
                    loader: 'css-loader',
                    options: {
                        url: false // Set the URL option to false to fix the webpack error
                    }
                },
                ...
              ]
          }
        ]
      }
    };
  }
};
```

## Wrap Up
And that's it! You've now set up your webpack process to copy your static font files into your project with one simple command, and you've updated the configuration to fix this pesky error.

## Next Steps
You can clean up your `webpack.config.js` file even more by using the [Webpack Merge](https://www.npmjs.com/package/webpack-merge) tool. This package makes it easy to extract any common configuration to a base object, then dynamically add certain options depending on the `env` variable in your function, then simply merge all of the config objects into a singular config that is sent to webpack.

```javascript
const { merge } = require('webpack-merge');
// Detect whether this is a production build or not
const production = process.env.NODE_ENV === 'production';
module.exports = env => {
  const build = [];
  const base = {
    mode: 'development'
  };
  build.push(base);
  if (env.static) {
    build.push({
      // Add your static files configuration here
    });
  } else {
    build.push({
      // Add your default configuration here
    });
    if (production) {
      build.push({
        mode: 'production',
        // Add any production-only configuration here
      })
    }
  }
};
```

### Happy Coding! :)
