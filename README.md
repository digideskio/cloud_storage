# `cloud_storage`

[![Build Status](https://travis-ci.org/leafo/cloud_storage.svg?branch=master)](https://travis-ci.org/leafo/cloud_storage)

A library for connecting to [Google Cloud Storage](https://cloud.google.com/products/cloud-storage) through Lua.

## Tutorial

You can learn more about authenicating with Google Cloud Storage here:
<https://cloud.google.com/storage/docs/authentication>. Here's a quick guide on
getting started:

The easiest way use this library is to create a service account for your
project. You'll need to download a private key store it alongside your
configuration.


Go to the APIs console, <https://console.cloud.google.com>. Enable Cloud
Storage if you haven't done so already. You may also need to enter billing
information.

Navigate to **IAM & admin** » **Service accounts**, located on the sidebar. Click **Create
service account**.

![Create service account](http://leafo.net/shotsnb/2018-03-31_20-54-45.png)

Select *Furnish a new private key* and select *JSON* as the type.  After
creating the service account your browser will download a `.json` file.  Store
that securely, since it grants access to your Google cloud account.

The `.json` is used to create a new API client with this library:

```lua
local google = require "cloud_storage.google"
local storage = google.CloudStorage:from_json_key_file("path/to/my-key.json")

-- you can now use any of the methods on the storage object
local files = assert(storage:get_bucket("my_bucket"))
```

## Using a p12/pem secret key

Use these directions if you have a key created with the `P12` type.


The private key file you will download is a  `.p12` file. The key uses a hard
coded password `notasecret`.

In order to use it we need to convert it to a `.pem`. Run the following command
(replacing `key.p12` and `key.pem` with the input filename and desired output
filename). Enter `notasecret` for the password.

```bash
openssl pkcs12 -in key.p12 -out key.pem -nodes -clcerts
```

We'll need one more piece of information, the service account email address.
You'll find it labeled **Service account ID** on the service account list. It
might look something like `cloud-storage@my-project.iam.gserviceaccount.com`.

You can create a new client like so:

```lua
local oauth = require "cloud_storage.oauth"

-- replace with your service account ID
o = oauth.OAuth("cloud-storage@my-project.iam.gserviceaccount.com", "path/to/key.pem")

local google = require "cloud_storage.google"

-- use your id as the second argument, everything before the @ in your service account ID
local storage = google.CloudStorage(o, "cloud-storage")

local files = storage:get_bucket("my_bucket")
```

## Reference

### cloud_storage.oauth

Handles OAuth authenticated requests. You must create an OAuth object that will
be used with the cloud storage API.

```lua
local oauth = require "cloud_storage.oauth"
```

#### `ouath_instance = oauth.OAuth(service_email, path_to_private_key)`

Create a new OAuth object.

### cloud_storage.google

Communicates with the Google cloud storage API.

```lua
local google = require "cloud_storage.google"
```

#### Error handling

Any methods that fail to execute will return `nil`, an error message, and an
object that represents the error. Successful responses will return a Lua table
containing the details of the operation.

#### `storage = oauth.CloudStorage(ouath_instance, project_id)`

```lua
local storage = google.CloudStorage(o, "111111111111")
```

#### `storage:get_service()`

<https://cloud.google.com/storage/docs/xml-api/get-service>

#### `storage:get_bucket(bucket)`

<https://cloud.google.com/storage/docs/xml-api/get-bucket>

#### `storage:get_file(bucket, key)`

<https://cloud.google.com/storage/docs/xml-api/get-object>

#### `storage:delete_file(bucket, key)`

<https://cloud.google.com/storage/docs/xml-api/delete-object>

#### `storage:head_file(bucket, key)`

<https://cloud.google.com/storage/docs/xml-api/head-object>

#### `storage:copy_file(source_bucket, source_key, dest_bucket, dest_key, options={})`

<https://cloud.google.com/storage/docs/xml-api/put-object-copy>

Options:

* `acl`: value to use for `x-goog-acl`, defaults to `public-read`
* `headers`: table of additional headers to include in request

#### `storage:compose(bucket, key, source_keys, options={})`

<https://cloud.google.com/storage/docs/xml-api/put-object-compose>

Composes a new file from multiple source files in the same bucket by
concatenating them.

`source_keys` is an array of keys as strings or an array of key declarations as
tables.

The table format for a source key is structured like this:

```lua
{
  name = "file.png",
  -- optional fields:
  generation = "1361471441094000",
  if_generation_match = "1361471441094000",
}
```

Options:

* `acl`: value to use for `x-goog-acl`, defaults to `public-read`
* `mimetype`: sets `Content-type` header for the composed file
* `headers`: table of additional headers to include in request

#### `storage:put_file_string(bucket, key, data, opts={})`

> **Note:** this API previously had `key` as an option but it was moved to
> second argument

Uploads the string `data` to the bucket at the specified key.

Options include:

 * `mimetype`: sets `Content-type` header for the file, defaults to not being set
 * `acl`: sets the `x-goog-acl` header for file, defaults to `public-read`
 * `headers`: an optional array table of any additional headers to send

```lua
storage:put_file_string("my_bucket", "message.txt", "hello world!", {
  mimetype = "text/plain",
  acl = "private"
})
```

#### `storage:put_file(bucket, fname, opts={})`

Reads `fname` from disk and uploads it. The key of the file will be the name of
the file unless `opts.key` is provided. The mimetype of the file is guessed
based on the extension unless `opts.mimetype` is provided.

```lua
storage:put_file("my_bucket", "source.lua", {
  mimetype = "text/lua"
})
```

All the same options from `put_file_string` are available for this method.

> **Note:** This method is currently inefficient with uploads. The whole file
> is loaded into memory and sent at once in the request. An alternative
> implementation will be added in the future.

#### `storage:signed_url(bucket, key, expiration)`

Creates a temporarily URL for downloading an object regardless of it's ACL.
`expiration` is a unix timestamp in the future, like one generated from
`os.time()`.

```lua
print(storage:signed_url("my_bucket", "message.txt", os.time() + 100))
```

## Changelog

# Mar 31, 2018

* **Changed** `put_file_string` now takes `key` as second argument, instead of within options table
* Replace `luacrypto` with `luaossl`
* Add support for `json` private keys
* Better error handling for all API calls. (Failures return `nil`, error message, and error object)
* Add `copy_file` and `compose` methods
* Add `upload_url` method to create upload URLs
* Add `start_resumable_upload` method for creating resumable uploads
* Headers are canonicalized and sorted to make them consistent between calls
* Fix bug where some special characters were not being encoded for signed URLs
* Rewrite documentation tutorial

# Sep 29, 2013

* Initial release

  [0]: https://developers.google.com/storage/docs/accesscontrol
