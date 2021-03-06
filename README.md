# [**email-templates**](https://github.com/niftylettuce/email-templates)

[![build status](https://img.shields.io/travis/niftylettuce/email-templates.svg)](https://travis-ci.org/niftylettuce/email-templates)
[![code coverage](https://img.shields.io/codecov/c/github/niftylettuce/email-templates.svg)](https://codecov.io/gh/niftylettuce/email-templates)
[![code style](https://img.shields.io/badge/code_style-XO-5ed9c7.svg)](https://github.com/sindresorhus/xo)
[![styled with prettier](https://img.shields.io/badge/styled_with-prettier-ff69b4.svg)](https://github.com/prettier/prettier)
[![made with lass](https://img.shields.io/badge/made_with-lass-95CC28.svg)](https://lass.js.org)
[![license](https://img.shields.io/github/license/niftylettuce/email-templates.svg)](LICENSE)

> Create, [preview][preview-email], and send custom email templates for [Node.js][node]. Highly configurable and supports automatic inline CSS, stylesheets, embedded images and fonts, and much more! Made for sending beautiful emails with [Lad][].
>
> **NEW**: v3.0.0 is released! See the [2.x branch][2-x-branch] documentation if you're using an older version


## Table of Contents

* [Install](#install)
* [Preview](#preview)
* [Usage](#usage)
  * [Basic](#basic)
  * [Automatic Inline CSS via Stylesheets](#automatic-inline-css-via-stylesheets)
  * [Cache Pug Templates](#cache-pug-templates)
  * [Localization](#localization)
  * [Custom Text Template](#custom-text-template)
  * [Custom Template Engine (e.g. EJS)](#custom-template-engine-eg-ejs)
  * [Custom Default Message Options](#custom-default-message-options)
  * [Custom Rendering (e.g. from a MongoDB database)](#custom-rendering-eg-from-a-mongodb-database)
* [Options](#options)
* [Plugins](#plugins)
* [Tip](#tip)
* [Contributors](#contributors)
* [License](#license)


## Install

> By default we recommend [pug][] for your template engine, but you can use [any template engine][supported-engines].

[npm][]:

```sh
npm install email-templates pug
```

[yarn][]:

```sh
yarn add email-templates pug
```


## Preview

We've added [preview-email][] by default to this package!

This means that (by default) in the development environment (e.g. `NODE_ENV=development`) your emails will be rendered to the tmp directory for you and automatically opened in the browser.

<a target="_blank" href="https://github.com/niftylettuce/preview-email/blob/master/demo.png">View the demo</a>


## Usage

### Basic

> You can swap the `transport` option with a [Nodemailer transport][nodemailer-transport] configuration object or transport instance. We highly recommend using [Postmark][] for your transport (it's the default in [Lad][]).

```js
const Email = require('email-templates');

const email = new Email({
  message: {
    from: 'niftylettuce@gmail.com'
  },
  transport: {
    jsonTransport: true
  }
});

email.send({
  template: 'mars',
  message: {
    to: 'elon@spacex.com'
  },
  locals: {
    name: 'Elon'
  }
}).then(console.log).catch(console.error);
```

The example above assumes you have the following directory structure:

```sh
.
├── app.js
└── emails
    └── mars
        ├── html.pug
        └── subject.pug
```

And the contents of the `pug` files are:

> `html.pug`:

```pug
p Hi #{name},
p Welcome to Mars, the red planet.
```

> `subject.pug`:

```pug
= `Hi ${name}, welcome to Mars`
```

### Automatic Inline CSS via Stylesheets

Simply include the path or URL to the stylesheet in your template's `<head>`:

```pug
link(rel="stylesheet", href="/css/app.css", data-inline)
```

This will look for the file `/css/app.css` in the `build/` folder.

If this asset is in another folder, then you will need to modify the default options when creating an `Email` instance:

```js
const email = new Email({
  // <https://github.com/Automattic/juice>
  juice: true,
  webResources: {
    preserveImportant: true,

    // default path is `build/`:
    relativeTo: path.resolve('build')

    //
    // but you might want to change it to something like:
    // relativeTo: path.join(__dirname, '..', 'assets')
    //

  }
});
```

### Cache Pug Templates

We strongly suggest to follow this example and pre-cache your templates with [cache-pug-templates][] (if you're using the default [Pug][] template engine).

If you do not do this, then your Pug templates will re-compile and re-cache every time you deploy new code and restart your app.

1. Ensure you have [Redis][] (v4.x+) installed:

   * Mac: `brew install redis && brew services start redis`
   * Ubuntu:

     ```sh
     sudo add-apt-repository -y ppa:chris-lea/redis-server
     sudo apt-get update
     sudo apt-get -y install redis-server
     ```

2. Install the packages:

   [npm][]:

   ```sh
   npm install cache-pug-templates redis
   ```

   [yarn][]:

   ```sh
   yarn add cache-pug-templates redis
   ```

3. Configure it to read and cache your entire email templates directory:

   ```js
   const path = require('path');
   const cachePugTemplates = require('cache-pug-templates');
   const redis = require('redis');
   const Email = require('email-templates');

   const redisClient = redis.createClient();
   const email = new Email({
     message: {
       from: 'niftylettuce@gmail.com'
     },
     transport: {
       jsonTransport: true
     }
   });

   cachePugTemplates(redisClient, email.config.views.root);

   // ...
   ```

4. For more configuration options see [cache-pug-templates][].

### Localization

All you need to do is simply pass an [i18n][] configuration object as `config.i18n` (or an empty one as this example shows to use defaults).

```js
const Email = require('email-templates');

const email = new Email({
  message: {
    from: 'niftylettuce@gmail.com'
  },
  transport: {
    jsonTransport: true
  },
  i18n: {} // <------ HERE
});

email.send({
  template: 'mars',
  message: {
    to: 'elon@spacex.com'
  },
  locals: {
    name: 'Elon'
  }
}).then(console.log).catch(console.error);
```

Then slightly modify your templates to use localization functions.

> `html.pug`:

```pug
p= t(`Hi ${name},`)
p= t('Welcome to Mars, the red planet.')
```

> `subject.pug`:

```pug
= t(`Hi ${name}, welcome to Mars`)
```

Note that if you use [Lad][], you have a built-in filter called `translate`:

```pug
p: :translate(locale) Hi #{name}
p: :translate(locale) Welcome to Mars, the red planet.
```

### Custom Text Template

> By default we use `html-to-text` to generate a plaintext version and attach it as `message.text`.

If you'd like to customize the text body, you can pass `message.text` or set `config.htmlToText: false` (doing so will automatically lookup a `text` template file just like it normally would for `html` and `subject`).

```js
const Email = require('email-templates');

const email = new Email({
  message: {
    from: 'niftylettuce@gmail.com'
  },
  transport: {
    jsonTransport: true
  },
  htmlToText: false // <----- HERE
});

email.send({
  template: 'mars',
  message: {
    to: 'elon@spacex.com'
  },
  locals: {
    name: 'Elon'
  }
}).then(console.log).catch(console.error);
```

> `text.pug`:

```pug
| Hi #{name},
| Welcome to Mars, the red planet.
```

### Custom Template Engine (e.g. EJS)

1. Install your desired template engine (e.g. [EJS][])

   [npm][]:

   ```sh
   npm install ejs
   ```

   [yarn][]:

   ```sh
   yarn add ejs
   ```

2. Set the extension in options and send an email

   ```js
   const Email = require('email-templates');

   const email = new Email({
     message: {
       from: 'niftylettuce@gmail.com'
     },
     transport: {
       jsonTransport: true
     },
     views: {
       options: {
         extension: 'ejs' // <---- HERE
       }
     }
   });
   ```

### Custom Default Message Options

You can configure your Email instance to have default message options, such as a default "From", an unsubscribe header, etc.

For a list of all available message options and fields see [the Nodemailer message reference](https://nodemailer.com/message/).

> Here's an example showing how to set a default custom header and a list unsubscribe header:

```js
const Email = require('email-templates');

const email = new Email({
  message: {
    from: 'niftylettuce@gmail.com',
    headers: {
      'X-Some-Custom-Thing': 'Some-Value'
    },
    list: {
      unsubscribe: 'https://niftylettuce.com/unsubscribe'
    }
  },
  transport: {
    jsonTransport: true
  }
});
```

### Custom Rendering (e.g. from a MongoDB database)

You can pass a custom `config.render` function which accepts two arguments `view` and `locals` and must return a `Promise`.

If you wanted to read a stored EJS template from MongoDB, you could do something like:

```js
const ejs = require('ejs');

const email = new Email({
  // ...
  render: (view, locals) => {
    return new Promise((resolve, reject) => {
      // this example assumes that `template` returned
      // is an ejs-based template string
      db.templates.findOne({ view }, (err, template) => {
        if (err) return reject(err);
        if (!template) return reject(new Error('Template not found'));
        resolve(ejs.render(template, locals));
      });
    });
  }
});
```


## Options

For a list of all available options and defaults [view the configuration object](index.js).


## Plugins

You can use any [nodemailer][] plugin. Simply pass an existing transport instance as `config.transport`.

You should add the [nodemailer-base64-to-s3][] plugin to convert base64 inline images to actual images stored on Amazon S3 and Cloudfront.

We also highly recommend to add to your default `config.locals` the following:

* [custom-fonts-in-emails][] - render any font in emails as an image w/retina support (no more Photoshop or Sketch exports!)
* [font-awesome-assets][] - render any [Font Awesome][fa] icon as an image in an email w/retina support (no more Photoshop or Sketch exports!)


## Tip

Instead of having to configure this for yourself, you could just use [Lad][] instead.


## Contributors

| Name           | Website                   |
| -------------- | ------------------------- |
| **Nick Baugh** | <http://niftylettuce.com> |


## License

[MIT](LICENSE) © [Nick Baugh](http://niftylettuce.com)


## 

[node]: https://nodejs.org

[npm]: https://www.npmjs.com/

[yarn]: https://yarnpkg.com/

[pug]: https://pugjs.org

[supported-engines]: https://github.com/tj/consolidate.js/#supported-template-engines

[nodemailer]: https://nodemailer.com/plugins/

[font-awesome-assets]: https://github.com/ladjs/font-awesome-assets

[custom-fonts-in-emails]: https://github.com/ladjs/custom-fonts-in-emails

[nodemailer-base64-to-s3]: https://github.com/ladjs/nodemailer-base64-to-s3

[lad]: https://lad.js.org

[2-x-branch]: https://github.com/niftylettuce/node-email-templates/tree/2.x

[i18n]: https://github.com/ladjs/i18n#options

[fa]: http://fontawesome.io/

[nodemailer-transport]: https://nodemailer.com/transports/

[postmark]: https://postmarkapp.com/

[ejs]: http://ejs.co/

[cache-pug-templates]: https://github.com/ladjs/cache-pug-templates

[redis]: https://redis.io/

[preview-email]: https://github.com/niftylettuce/preview-email
