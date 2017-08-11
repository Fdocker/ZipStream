# ZipStream Service

A microservice to build and stream dynamically zipped bundles of remote files. The service does not store files, rather it stores references to files which it can stream in a zipped package to users. Designed to be backend-swappable, it is capable of working with just about any database and any filestore. It aims to be fast, have a low memory footprint, and to support a tens-of-thousands of concurrent connections <sup>[[citation needed]](https://github.com/Cadasta/ZipStream/issues/8)</sup>.

## Use case

Imagine that you store thousands of files on Amazon S3 for a client. The client occasionally asks you to make a subset of their files available for download. A simple solution to this would be to manually download these files, bundle them up as a zip file, upload the zip file to S3, and send a link of zip on S3 to your client. ZipStream aims to simplify this by allowing you to submit references of the client's files to ZipStream and send your client a link the bundle's identifier. When your client visits the link, a zip file of the S3 assets will be generated on the fly and streamed to the user. Naturally, if the zipped bundle is to be downloaded many times over, it's likely more efficient to actually generate the zip file once, store it in a filestore, and send the client a link to that file. In this case, ZipStream may still be of use for the one-time generation of the to-be-shared zip file (ie a script could stream the zip from ZipStream to the filestore).

## Supported backends

The system relies on two backend types: Database and Filestore.

### Database backend

The database backend is responsible for managing data that represent ZipStream [`Bundle`] instances.

To add support for a new database backend, a file should be placed in the `backends/db` directory and offer four functions: `create`, `read`, `update`, `delete`. Each function must take in a [`FileRef`] object and return a Promise object.

#### Currently supported database backends:

- Amazon DynamoDB

To see what's on the database backends radar, see our [Issues](https://github.com/Cadasta/ZipStream/issues?q=is%3Aopen+label%3Aenhancement+label%database).

### Filestore backend

The filestore backend is responsible for returning files in a [Readable stream](https://nodejs.org/api/stream.html#stream_readable_streams) format.

To add support for a new filestore backend, a file should be placed in the `backends/fs` directory and offer a single function: `getStream`. This functino should take in a `base` and `path` argument and return a readable stream.

#### Currently supported filestore backends:

- Amazon S3

To see what's on the filestore backends radar, see our [Issues](https://github.com/Cadasta/ZipStream/issues?q=is%3Aopen+label%3Aenhancement+label%3Afilestore).


## Models

### `FileRef`

- `base`: `String`, base directory or base location of file (depending on backend), such as S3 Bucket
- `path`: `String`, path to file, relative to base value
- `dest`: `String`, _optional_, desired path of file when in bundle, defaults to `path` value

### `Bundle`

- `id`: `String`, UUIDv4
- `secret`: `String`, UUIDv4
- `files`: `Array`, [`FileRef`] objects
- `expirationDate`: `Number`, unix timestamp representing date at which record should be deleted. Note that the codebase does not manage the removal of expired bundles. It is up to individual backends to set up expiration logic via TTL or database trigger settings
- `filename`: `String`, desired filename of bundle (should end with .zip)


## Usage

### Create Bundle

Creates a bundle.

**URL** : `/`

**Method** : `POST`

**Data constraints**

```json
{
    "filename": "[desired filename of bundle (should end with .zip)]"
}
```

Or, optionally:

```json
{
    "filename": "[desired filename of bundle]",
    "files": [
        {
            "base": "[base directory or base location of file (depending on backend), such as S3 Bucket]",
            "path": "[path to file, relative to base value]",
            "dest": "[optional, desired path of file when in bundle, defaults to 'path' value ]"
        }
    ]
}
```

**Data example**

```json
{
    "filename": "`",
    "files": [
        {
            "base": "my-aws-bucket-1",
            "path": "path/to/foo.jpg",
            "dest": "foo.jpg"
        },
        {
            "base": "some-other-bucket-2",
            "path": "bar.gif"
        }
    ]
}
```

#### Success Response

JSON representation of created bundle.

**Code** : `201 CREATED`

**Content example**

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "base": "my-aws-bucket-1",
            "path": "path/to/foo.jpg",
            "dest": "foo.jpg"
        },
        {
            "base": "some-other-bucket-2",
            "path": "bar.gif"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```

### Download Bundle

Download a bundle.

**URL** : `/:id/`

**Method** : `GET`

#### Success Response

Streaming download of zipped bundle.

**Code** : `200 OK`

_Download of `my-awesome-bundle.zip` containing `foo.jpg` and `bar.gif`._

### Get Bundle Information

Retrieves bundle information.

**URL** : `/:id/:secret/`

**Method** : `GET`

#### Success Response

JSON representation of retrieved bundle.

**Code** : `200 OK`

**Content example**

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "base": "my-aws-bucket-1",
            "path": "path/to/foo.jpg",
            "dest": "foo.jpg"
        },
        {
            "base": "some-other-bucket-2",
            "path": "bar.gif"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```

### Update Bundle

Append files a bundle.

**URL** : `/:id/:secret/`

**Method** : `PUT`

**Data constraints**

```json
{
    "files": [
        {
            "base": "[base directory or base location of file (depending on backend), such as S3 Bucket]",
            "path": "[path to file, relative to base value]",
            "dest": "[optional, desired path of file when in bundle, defaults to 'path' value ]"
        }
    ]
}
```

**Data example**

```json
{
    "filename": "`",
    "files": [
        {
            "base": "one-more-bucket-3",
            "path": "another/file.pdf",
            "dest": "another-one.pdf"
        }
    ]
}
```

#### Success Response

JSON representation of bundle with newly-appended data.

**Code** : `200 OK`

**Content example**

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "base": "my-aws-bucket-1",
            "path": "path/to/foo.jpg",
            "dest": "foo.jpg"
        },
        {
            "base": "some-other-bucket-2",
            "path": "bar.gif"
        },
        {
            "base": "one-more-bucket-3",
            "path": "another/file.pdf",
            "dest": "another-one.pdf"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```

### Delete Bundle

Used to create an empty bundle.

**URL** : `/:id/:secret/`

**Method** : `DELETE`

#### Success Response

JSON representation of deleted bundle.

**Code** : `200 OK`

**Content example**

```json
{
   "expirationDate" : 1503029550,
   "secret" : "bd1be533-3c5c-4395-bda9-73dd288c5487",
   "filename" : "my-awesome-bundle.zip",
    "files": [
        {
            "base": "my-aws-bucket-1",
            "path": "path/to/foo.jpg",
            "dest": "foo.jpg"
        },
        {
            "base": "some-other-bucket-2",
            "path": "bar.gif",
        },
        {
            "base": "one-more-bucket-3",
            "path": "another/file.pdf",
            "dest": "another-one.pdf"
        }
   ],
   "id" : "c4f6f218-afc4-4af1-ae1b-b22e9b058f26"
}
```


## Getting Started

Clone the repo:
```sh
git clone git@github.com:Cadasta/zipstream.git
cd zipstream
```

Install yarn:
```js
npm install -g yarn
```

Install dependencies:
```sh
yarn
```

Set environment (vars):
```sh
cp .env.example .env
```

Start server:
```sh
# Start server
yarn start

# Selectively set DEBUG env var to get logs
DEBUG=zipstream:* yarn start
```
Refer [debug](https://www.npmjs.com/package/debug) to know how to selectively turn on logs.


Tests:
```sh
# Run tests written in ES6
yarn test

# Run test along with code coverage
yarn test:coverage

# Run tests on file change
yarn test:watch

# Run tests enforcing code coverage (configured via .istanbul.yml)
yarn test:check-coverage
```
_NOTE: Full test coverage [to come shortly](https://github.com/Cadasta/ZipStream/issues/9)_

Lint:
```sh
# Lint code with ESLint
yarn lint

# Run lint on any file change
yarn lint:watch
```

Other gulp tasks:
```sh
# Wipe out dist and coverage directory
gulp clean

# Default task: Wipes out dist and coverage directory. Compiles using babel.
gulp
```

##### Deployment

```sh
# compile to ES5
1. yarn build

# upload dist/ to your server
2. scp -rp dist/ user@dest:/path

# install production dependencies only
3. yarn --production

# Use any process manager to start your services
4. pm2 start dist/index.js
```

In production you need to make sure your server is always up so you should ideally use any of the process manager recommended [here](http://expressjs.com/en/advanced/pm.html).


## Logging

Universal logging library [winston](https://www.npmjs.com/package/winston) is used for logging. It has support for multiple transports.  A transport is essentially a storage device for your logs. Each instance of a winston logger can have multiple transports configured at different levels. For example, one may want error logs to be stored in a persistent remote location (like a database), but all logs output to the console or a local file. We just log to the console for simplicity, you can configure more transports as per your requirement.

#### API logging
Logs detailed info about each api request to console during development.
![Detailed API logging](https://cloud.githubusercontent.com/assets/4172932/12563354/f0a4b558-c3cf-11e5-9d8c-66f7ca323eac.JPG)

#### Error logging
Logs stacktrace of error to console along with other details. You should ideally store all error messages persistently.
![Error logging](https://cloud.githubusercontent.com/assets/4172932/12563361/fb9ef108-c3cf-11e5-9a58-3c5c4936ae3e.JPG)


## Code Coverage
Get code coverage summary on executing `yarn test`
![Code Coverage Text Summary](https://cloud.githubusercontent.com/assets/4172932/12827832/a0531e70-cba7-11e5-9b7c-9e7f833d8f9f.JPG)

`yarn test` also generates HTML code coverage report in `coverage/` directory. Open `lcov-report/index.html` to view it.
![Code coverage HTML report](https://cloud.githubusercontent.com/assets/4172932/12625331/571a48fe-c559-11e5-8aa0-f9aacfb8c1cb.jpg)


## Docker

[_TODO_](https://github.com/Cadasta/ZipStream/issues/4)


## FAQ

> Why isn't this written an AWS Lambda service?

This service would indeed be a good use case for AWS Lambda. In fact, we initially began building it out as a [Serverless](serverless.com) app. Ultimately, Lambda's 5 minute max-lifetime turned us off of the idea as we cater towards clients in remote, low-bandwidth regions where a 5+ minute download is likely. If you're interesting in running this codebase on AWS Lambda, [we'd love to hear how it goes!](https://github.com/Cadasta/ZipStream/issues/7)

## Contributing

Pull Requests are very welcome! If you would like to add a new feature, it is recommended that you create an Issue to first discuss the idea, however this is not mandatory.

## Attributions

Inspired by [Teamwork's s3zipper](https://github.com/Teamwork/s3zipper). Built from the [express-mongoose-es6-rest-api boilerplate](https://github.com/KunalKapadia/express-mongoose-es6-rest-api).


[`FileRef`]: #fileref
[`Bundle`]: #bundle
