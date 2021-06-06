# Middy AJV

<div align="center">
  <img alt="Middy logo" src="https://raw.githubusercontent.com/middyjs/middy/master/docs/img/middy-logo.png"/>
</div>

<div align="center">
  <p><strong>A fast for the middy framework</strong></p>
</div>

<div align="center">
<p>
  <a href="http://badge.fury.io/js/%40willfarrell%2Fmiddy-ajv">
    <img src="https://badge.fury.io/js/%40willfarrell%2Fmiddy-ajv.svg" alt="npm version" style="max-width:100%;">
  </a>
  <a href="https://snyk.io/test/github/willfarrell/middy-ajv">
    <img src="https://snyk.io/test/github/willfarrell/middy-ajv/badge.svg" alt="Known Vulnerabilities" data-canonical-src="https://snyk.io/test/github/willfarrell/middy-ajv" style="max-width:100%;">
  </a>
  <a href="https://standardjs.com/">
    <img src="https://img.shields.io/badge/code_style-standard-brightgreen.svg" alt="Standard Code Style"  style="max-width:100%;">
  </a>
  <a href="https://gitter.im/middyjs/Lobby">
    <img src="https://badges.gitter.im/gitterHQ/gitter.svg" alt="Chat on Gitter"  style="max-width:100%;">
  </a>
</p>
</div>

RDS provides seamless connection with database of your choice.

After initialization your database connection is accessible under:
```javascript
middy((event, context) => {
  const { db } = context;
});
```

Mind that if you use knex you will also need driver of your choice ([check docs](http://knexjs.org/#Installation-node)), for PostgreSQL that would be:
```
yarn add pg
// or
npm install pg
```


## Install

To install this middleware you can use NPM:

```bash
npm install --save middy-ajv
```

Requires: @middy/core:>=2.0.0

## Options
- `client` (function) (required): client that you want to use when connecting to database of your choice. Designed to be used by knex.js. However, as long as your client is run as client(config), you can use other tools.
- `config` (object) (required): configuration object passed as is to client (knex.js recommended), for more details check [knex documentation](http://knexjs.org/#Installation-client)
- `internalData` (object) (optional): Pull values from middy internal storage into `config.connection` object.
- `cacheKey` (string) (default `rds`): Cache key for the fetched data responses. Must be unique across all middleware.
- `cachePasswordKey` (string) (default `rds`):Cache key for the fetched data response related to the password. Must match the `cacheKey` for the middleware that stores it.
- `cacheExpiry` (number) (default `-1`): How long fetch data responses should be cached for. `-1`: cache forever, `0`: never cache, `n`: cache for n ms.


**Note:**
- `config.connection` defaults to:

```javascript
{
  ssl: {
    rejectUnauthorized: true,
    ca, // rds-ca-2019-root.pem
    checkServerIdentity: (host, cert) => {
      const error = tls.checkServerIdentity(host, cert)
      if (error && !cert.subject.CN.endsWith('.rds.amazonaws.com')) {
         return error
      }
    }
  }
}
```

If your lambda is timing out, likely your database connections are keeping the event loop open. Check out [do-not-wait-for-empty-event-loop](https://github.com/middyjs/middy/tree/master/packages/do-not-wait-for-empty-event-loop) middleware to resolve this.

## Sample usage

Minimal configuration

### knex
```javascript
const rds = require('middy-rds')
const knex = require('knex')
const pg = capturePostgres(require('pg')) // AWS X-Ray
const handler = middy(async (event, context) => {
    const { db } = context;
    const records = await db.select('*').from('my_table');
    console.log(records);
  })
  .use(rdsSigner({
    fetchData: {
      rdsToken: {
        region: 'ca-central-1',
        hostname: '*.ca-central-1.rds.amazonaws.com',
        username: 'iam_role',
        database: 'postgres',
        port: 5432
      }
    },
    cacheKey: 'rds-signer'
  }))
  .use(rds({
    internalData: {
      password: 'rdsToken'
    },
    cacheKey: 'rds',
    cachePasswordKey: 'rds-signer',
    client: knex,
    config: {
      client: 'pg',
      connection: {
        host: '*.ca-central-1.rds.amazonaws.com',
        user: 'iam_role',
        database: 'postgres',
        port: 5432
      }
    }
  }))
```

### pg
```javascript
const rds = require('middy-rds/pg')
const pg = capturePostgres(require('pg')) // AWS X-Ray
const handler = middy(async (event, context) => {
    const { db } = context;
    const records = await db.select('*').from('my_table');
    console.log(records);
  })
  .use(rdsSigner({
    fetchData: {
      rdsToken: {
        region: 'ca-central-1',
        hostname: '*.ca-central-1.rds.amazonaws.com',
        username: 'iam_role',
        database: 'postgres',
        port: 5432
      }
    },
    cacheKey: 'rds-signer'
  }))
  .use(rds({
    internalData: {
      password: 'rdsToken'
    },
    cacheKey: 'rds',
    cachePasswordKey: 'rds-signer',
    client: pg.Pool,
    config: {
      host: '*.ca-central-1.rds.amazonaws.com',
      user: 'iam_role',
      database: 'postgres'
    }
  }))
```

## Build step

### Folder Structure
```shell
{project}
|-- handlers
| |-- {enpoint}
| | |-- index.mjs
| | |-- schema.json
```
After the build scripts have been run the `endpoint` folder will contain `schema.js` and `index.js`. The later of which you need to upload to AWS.

### Install
```shell
$ npm install -D ajv-cli rollup @rollup/plugin-commonjs @rollup/plugin-json @rollup/plugin-node-resolve @rollup/plugin-replace rollup-plugin-esbuild esbuild
```

### rollup.config.js
```javascript
import { readdirSync } from 'fs'
import json from '@rollup/plugin-json'
import replace from '@rollup/plugin-replace'
import resolve from '@rollup/plugin-node-resolve'
import commonjs from '@rollup/plugin-commonjs'
import esbuild from 'rollup-plugin-esbuild'

function onwarn (warning, warn) {
  if (warning.code === 'CIRCULAR_DEPENDENCY') return
  warn(warning)
}

import {dirname, join} from 'path'
const handlers = readdirSync('./handlers').filter((dir) => dir !=='.DS_Store')
export default handlers.map((input) => ({
  input: 'handlers/' + input + '/index.mjs',
  output: {
    file: 'handlers/' + input + '/index.js',
    format: 'cjs',
  },
  plugins: [
    replace({
      preventAssignment: false,  // remove warning, will be default in future version
      delimiters: ['',''],
      include: ['./**/index.mjs'],
      values: {
        // use leaner version of `@middy/validator`
        'require(\'@middy/validator\')': 'require(\'./middy-ajv\')',
        'from \'@middy/validator\'': 'from \'./middy-ajv\'',

        // use compiled schemas for `@middy/validator`
        'require(\'./schema.json\')': 'require(\'./schema.js\').default',
        'inputSchema from \'./schema.json\'': 'inputSchema from \'./schema.js\'',
      }
    }),
    json(),
    // Break RDS Signer :(
    resolve({
      // Exclude `aws-sdk`, included in lambda runtime. Why is there React and XML code in the aws-sdk? No idea
      resolveOnly: [/^.*(?<!aws-sdk)(?<!aws-sdk-core-react)(?<!aws-sdk-react-native)(?<!xml2js)$/]
    }),
    commonjs(),
    esbuild({
      minify: true,
      target: 'es2020'
    })
  ],
  onwarn,
}))
```

### Run
```shell
# Compile JSON Schema
$ for dir in handlers/*/; do node ./node_modules/ajv-cli/dist/index.js compile -c ajv-formats -c ajv-formats-draft2019 --strict=true --coerce-types=array --all-errors=true --use-defaults=empty --messages=false -s $dir'schema.json' -o $dir'schema.js'; done
# Bundle
$ rollup --config rollup.config.js --environment INCLUDE_DEPS,BUILD:production
```


## Middy documentation and examples

For more documentation and examples, refers to the main [Middy monorepo on GitHub](https://github.com/middyjs/middy) or [Middy official website](https://middy.js.org).


## Contributing

Everyone is very welcome to contribute to this repository. Feel free to [raise issues](https://github.com/middyjs/middy/issues) or to [submit Pull Requests](https://github.com/middyjs/middy/pulls).


## License

Licensed under [MIT License](LICENSE). Copyright (c) 2017-2021 will Farrell and the [Middy team](https://github.com/middyjs/middy/graphs/contributors).

<a href="https://app.fossa.io/projects/git%2Bgithub.com%2Fmiddyjs%2Fmiddy?ref=badge_large">
  <img src="https://app.fossa.io/api/projects/git%2Bgithub.com%2Fmiddyjs%2Fmiddy.svg?type=large" alt="FOSSA Status"  style="max-width:100%;">
</a>
