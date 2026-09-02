# active_storage_encryption

This library enables use of per-blob encryption keys with ActiveStorage, with a separate encryption key for every `Blob`. To implement encryption, it enables the use of [CSEK](https://cloud.google.com/storage/docs/encryption/using-customer-supplied-keys) on Google Cloud, [SSE-C](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerSideEncryptionCustomerKeys.html#specifying-s3-c-encryption) on AWS, and [block_cipher_kit](https://rubygems.org/gems/block_cipher_kit) for files on disk.

During streaming download, either the cloud provider or a Rails controller will decrypt the requested chunk of the file as it gets served to the client.

## Contents

* [Requirements](#requirements)
* [Installation](#installation)
* [How it works](#how-it-works)
* [Where the encryption keys live and how they get generated](#where-the-encryption-keys-live-and-how-they-get-generated)
* [How this gem integrates with ActiveStorage](#how-this-gem-integrates-with-activestorage)
* [Picking a service](#picking-a-service)
  * [AWS S3](#aws-s3---encrypteds3service)
  * [Google Cloud Storage](#google-cloud-storage---encryptedgcsservice)
  * [Local disk](#local-disk---encrypteddiskservice)
  * [Mirroring](#mirroring---encryptedmirrorservice)
* [Private URL constraints](#private-url-constraints)
* [Direct uploads](#direct-uploads)
* [Downloading the plaintext file contents in full or in part](#downloading-the-plaintext-file-contents-in-full-or-in-part)
* [Constraints with encrypted Blobs](#constraints-with-encrypted-blobs)
* [What this gem can and cannot do](#what-this-gem-can-do)
* [Security considerations / notes](#security-considerations--notes)
* [Migrating your blobs into an encrypted store](#migrating-your-blobs-into-an-encrypted-store)
* [Development](#development)

## Requirements

* Ruby >= 3.1
* Rails >= 7.2.2.1 (with ActiveStorage and ActiveRecord encryption available)
* For AWS: the `aws-sdk-s3` gem and a bucket that supports SSE-C
* For Google Cloud: the `google-cloud-storage` gem and a bucket that supports CSEK

## Installation

There are five steps: install the gem, run the migration, mount the engine, configure ActiveRecord encryption, and declare an encrypted service.

### 1. Install the gem and run the migration

```shell
bundle add active_storage_encryption
bin/rails g active_storage_encryption:install
bin/rails db:migrate
```

The generated migration adds a single `encryption_key` string column to your `active_storage_blobs` table. That column is where the per-blob key material is kept — see [Where the encryption keys live](#where-the-encryption-keys-live-and-how-they-get-generated).

### 2. Mount the engine

The gem ships a Rails engine with the controllers used for streaming downloads and for direct uploads into a named (non-default) service. Both are required for normal use, so mount it in `config/routes.rb`:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount ActiveStorageEncryption::Engine => "/active-storage-encryption"
end
```

If you skip this step, generating a private URL (`Blob#url`, `rails_blob_path`) with the default `private_url_policy: stream` will fail, because there is no route for the streaming controller to point at.

### 3. Configure ActiveRecord encryption

The `encryption_key` column contains raw key material. This gem already declares `encrypts :encryption_key` on `ActiveStorage::Blob` for you (non-deterministic encryption, which is what you want here), but ActiveRecord encryption itself needs keys to be configured in your app, or every `Blob` save will raise:

```shell
bin/rails db:encryption:init
```

Copy the generated `active_record_encryption` block into your credentials:

```shell
bin/rails credentials:edit
```

```yaml
active_record_encryption:
  primary_key: ...
  deterministic_key: ...
  key_derivation_salt: ...
```

See the [ActiveRecord encryption guide](https://guides.rubyonrails.org/active_record_encryption.html) for key management, key rotation and multi-environment setups.

> [!WARNING]
> Do not remove the `encrypts :encryption_key` behaviour or store the column in plaintext. The value in that column is key material: an attacker who obtains a database dump would immediately have unfettered access to the encryption keys of every `Blob` that you have.

### 4. Declare an encrypted service

Encryption is enabled by using one of the encrypted `Service` classes in `config/storage.yml`. Your existing ActiveStorage services will not magically start encrypting anything — it is recommended to define *separate* services for your sensitive data, since the behaviour of an encrypted store differs from a standard one in subtle ways.

The `EncryptedDisk` service is the easiest one to play with in development:

```yaml
# config/storage.yml
encrypted_local_disk:
  service: EncryptedDisk # this is the service implementation you need to use
  private_url_policy: stream
  root: <%= Rails.root.join("storage", "encrypted") %>
```

For S3 and GCS the configuration is exactly the same as for the stock ActiveStorage services, with the addition of the `private_url_policy` [parameter](#private-url-constraints). See [Picking a service](#picking-a-service) for complete per-provider examples, IAM permissions and CORS configuration.

### 5. Point your attachments at that service

```ruby
class User < ApplicationRecord
  has_one_attached :id_document_scan, service: :encrypted_local_disk
end
```

And.. that's it. Attaching, downloading, `Blob#open`, variants of the plaintext and so on keep working as usual — no call site has to pass an encryption key explicitly.

## How it works

This gem protects from a relatively common data breach scenario - cloud account access. Should an attacker gain access to your cloud storage bucket, the files stored there will be of no use to them without them also having a separate, specific encryption key for every file they want to retrieve.

The standard implementation of this usually works like this:

* You generate an encryption key which satisfies the provider's requirements
* You then send that key in a header along with your upload PUT request. The PUT request sends the file unencrypted to the provider, along with the key you have generated and signed.
* Before depositing your file in the bucket, the provider applies its encryption (and usually - some form of chunking, but that is transparent to you) to the data it receives from you
* Once the encrypted file is in the cloud storage, there is no plaintext version present anywhere
* Every read access to the file requires that you provide the encryption key

With this gem, you configure encrypted storage services in your `storage.yml` config, and run the included migration - which adds the `encryption_key` column to your `active_storage_blobs` table. Interactions with the cloud storage will then add the encryption key to all requests.

Once a `Blob` destined to be stored on an encrypted storage service (you set the `service:` where the blob should go in your `has_attached` calls) gets created in Rails, the blob will get a random encryption key generated for it. All further operations with the `Blob` are going to use that generated encryption key, automatically. None of the calls to the Blob will require the encryption key to be provided explicitly - everything should work as if you were dealing with a standard `ActiveStorage::Blob`.

This enables enhanced security for sensitive data, such as uploaded documents of all sorts, identity photos, biometric data and the like.

## Where the encryption keys live and how they get generated

This is the part that differs most from stock ActiveStorage, so it is worth going through in detail.

### Generation

Keys are generated **on the server**, by the gem, using `SecureRandom`:

```ruby
ActiveStorage::Blob.generate_random_encryption_key # => 48 random bytes
```

* The key is a high-entropy bag of random bytes — **not** a password or a passphrase.
* 48 bytes get generated (`ENCRYPTION_KEY_LENGTH_BYTES = 16 + 32`), which is more than the 32 bytes AES-256 needs. See [key truncation](#key-truncation) for why.
* Every `Blob` gets its own key. Keys are never shared between blobs and never derived from the blob key/path/filename.

A key gets generated automatically, and only if the destination service is an encrypted one, whenever a blob is created through one of the standard entry points, which this gem overrides:

| Entry point | Used by |
| --- | --- |
| `ActiveStorage::Blob.create_and_upload!` | `attach`, direct `create_and_upload!` calls |
| `ActiveStorage::Blob.build_after_unfurling` / `create_after_unfurling!` | `attach` with an IO |
| `ActiveStorage::Blob.create_before_direct_upload!` | direct uploads |
| `ActiveStorage::Blob.compose` | composing blobs |

All of them accept an optional `encryption_key:` argument, so you can supply your own key if you have a reason to (for example when [migrating existing blobs](#migrating-your-blobs-into-an-encrypted-store)). If you don't pass one, a random key is generated for you.

Whether a key gets generated is decided by `ActiveStorage::Blob.service_encrypted?(service_name)`, which simply asks the resolved service whether it responds to `encrypted?` with a truthy value. Blobs headed for a non-encrypted service keep `encryption_key` at `nil`.

### Storage

The key is stored in the `encryption_key` column of `active_storage_blobs`:

* The column is added by the install generator's migration as a `:string` column.
* The gem declares `encrypts :encryption_key` on `ActiveStorage::Blob`, so ActiveRecord encrypts the value with your application's ActiveRecord encryption keys (non-deterministic) before it is written to the database, and decrypts it on read. This is why [step 3](#3-configure-activerecord-encryption) of the installation is not optional.
* A validation requires `encryption_key` to be present for blobs whose service is encrypted, so a blob can never silently end up on an encrypted service without a key.
* `Blob#serializable_hash` always excludes `encryption_key`, so the key does not leak into `to_json` output, API responses or logs of serialized blobs.
* The key is **not** stored in the provider's KMS, and the cloud provider does not retain it. Losing the database row means losing access to the file, permanently.

This means the trust boundary is: `secret_key_base` / ActiveRecord encryption keys (your app secrets) + the database + the bucket. An attacker needs *all three* to read a file.

### How the key reaches the storage provider

| Service | Mechanism | Headers used |
| --- | --- | --- |
| `EncryptedS3Service` | SSE-C, key sent per request by the AWS SDK | `x-amz-server-side-encryption-customer-algorithm`, `x-amz-server-side-encryption-customer-key`, `x-amz-server-side-encryption-customer-key-MD5` |
| `EncryptedGCSService` | CSEK, key sent per request | `x-goog-encryption-algorithm`, `x-goog-encryption-key`, `x-goog-encryption-key-sha256` |
| `EncryptedDiskService` | encryption performed in your app process by `block_cipher_kit` | `x-active-storage-encryption-key` (direct upload to our controller only) |

For downloads through the provided streaming controller the key travels inside the URL token — but that token is *encrypted*, not merely signed, using `ActiveStorageEncryption.token_encryptor` (an `ActiveSupport::MessageEncryptor` keyed from `Rails.application.secret_key_base`). The raw key is therefore not exposed in the URL. See [key exposure on download](#key-exposure-on-download).

### Key truncation

In practice, all services use some form of AES-256 and therefore use a 32-byte encryption key. However, we can't exclude the possibility that there will be support for newer encryption schemes in the future with a longer encryption key being available. Because we potentially may need to allow a `Blob` to be encrypted and decrypted by service A, and then encrypted by service B, there is a possibility that key length requirements for those services could differ. Therefore, we generate a longer `encryption_key` (from the same random byte source) than strictly necessary for AES-256 and save it in your `active_storage_blobs` table. This key then gets truncated by each service to satisfy its own key length requirements:

* `EncryptedS3Service` uses the first 32 bytes as the SSE-C key
* `EncryptedGCSService` uses the first 32 bytes as the CSEK key (and raises if fewer than 32 bytes are available)
* `EncryptedDiskService` uses the first 32 bytes for the current `v2` scheme

This has an important implication for how the Service classes are written: they need to truncate the encryption keys to their conformant length themselves.

Important, once again: **do not use passwords for this encryption key.** If you really want to use passwords, use something like PBKDF (available via [ActiveSupport::KeyGenerator](https://api.rubyonrails.org/classes/ActiveSupport/KeyGenerator.html)) to derive a high-entropy key from your passphrase in a safe manner.

## How this gem integrates with ActiveStorage

There is no separate model, no separate attachment macro and no separate API. The gem plugs into ActiveStorage in four places.

### 1. Service classes

Encrypted services are ordinary `ActiveStorage::Service` subclasses that inherit from their stock counterparts and answer `encrypted? == true`:

| `service:` value in `storage.yml` | Class | Inherits from |
| --- | --- | --- |
| `EncryptedDisk` | `ActiveStorageEncryption::EncryptedDiskService` | `ActiveStorage::Service::DiskService` |
| `EncryptedS3` | `ActiveStorageEncryption::EncryptedS3Service` | `ActiveStorage::Service::S3Service` |
| `EncryptedGCS` | `ActiveStorageEncryption::EncryptedGCSService` | `ActiveStorage::Service::GCSService` |
| `EncryptedMirror` | `ActiveStorageEncryption::EncryptedMirrorService` | `ActiveStorage::Service::MirrorService` |

Because they subclass the stock services, all of the configuration keys you already know (`bucket`, `region`, `credentials`, `root`, `upload:` options, ...) keep working. The service classes take an extra `encryption_key:` keyword on the operations that need it (`upload`, `download`, `download_chunk`, `compose`, `url`, `url_for_direct_upload`, `headers_for_direct_upload`), which the `Blob` supplies for you.

### 2. Patches on `ActiveStorage::Blob` and friends

An initializer (re-applied on every reload) mixes the following into ActiveStorage:

| Target | What it does |
| --- | --- |
| `ActiveStorage::Blob` (class methods) | `encrypts :encryption_key`, the presence validation, `generate_random_encryption_key`, `service_encrypted?`, and the key-aware versions of `create_and_upload!` / `create_before_direct_upload!` / `build_after_unfurling` / `create_after_unfurling!` / `compose` |
| `ActiveStorage::Blob` (instance methods) | `upload_without_unfurling`, `download`, `download_chunk`, `open`, `compose`, `url`, `service_url_for_direct_upload`, `service_headers_for_direct_upload`, `serializable_hash` — each one passes `encryption_key` down to the service when the service is encrypted, and calls `super` when it is not |
| `ActiveStorage::Blob::Identifiable` | `download_identifiable_chunk` — content type sniffing reads the first 4 KB, which needs the key |
| `ActiveStorage::Downloader` | `open`/`download` accept `encryption_key:` and raise a clear error if it is missing |

Every one of these overrides is a no-op for blobs on non-encrypted services, so adding this gem does not change the behaviour of your existing services.

### 3. The engine, its routes and controllers

Mounting the engine (see [step 2](#2-mount-the-engine)) adds three routes:

| Route | Controller | Purpose |
| --- | --- | --- |
| `PUT /blob/:token` | `EncryptedBlobsController#update` | Receives direct uploads for `EncryptedDiskService` (or any encrypted service that you do not want to PUT to directly) |
| `POST /blob/direct-uploads` | `EncryptedBlobsController#create_direct_upload` | Creates the `Blob` record before a direct upload **into a named service** — the URL helper is `create_encrypted_blob_direct_upload_url` |
| `GET /blob/:token/*filename` | `EncryptedBlobProxyController#show` | Streams the decrypted blob (this is what `private_url_policy: stream` points at); supports HTTP `Range` |

Both controllers are `ActionController::Base` subclasses which read all their parameters from the signed (PUT) or encrypted (GET) token — they never accept a key, service name or object path from unsigned request parameters.

### 4. The URL token encryptor

`ActiveStorageEncryption.token_encryptor` builds an `ActiveSupport::MessageEncryptor` from `Rails.application.secret_key_base` (via `ActiveSupport::KeyGenerator`, with a fixed salt), URL-safe, and is used for the streaming GET tokens and the mirroring job payloads. Note that this encryptor is *only* used for tokens — never for file data.

### Lifecycle of a request

An upload through `attach`:

1. `Blob.create_and_upload!` runs, sees that the target service is encrypted, and generates a 48-byte key.
2. ActiveRecord encrypts that key and stores it in `active_storage_blobs.encryption_key`.
3. `Blob#upload_without_unfurling` calls `service.upload(key, io, encryption_key:, checksum:)`.
4. S3/GCS receive the key in headers and encrypt server-side; `EncryptedDiskService` encrypts in-process before writing the file.

A download through `rails_blob_path` with `private_url_policy: stream`:

1. Rails' own `ActiveStorage::Blobs::RedirectController#show` calls `Blob#url`.
2. `Blob#url` calls `service.url(..., encryption_key:)`, which builds an *encrypted* token containing the object key, service name, content type, disposition, byte size and the encryption key.
3. The browser is redirected to `EncryptedBlobProxyController#show`, which decrypts the token, verifies the key against the object (by reading one byte) and streams the plaintext out, honouring `Range` requests.

## Picking a service

All four services are drop-in replacements for their stock counterparts, but they differ in what they support. Pick per bucket/provider:

| | `EncryptedS3` (AWS) | `EncryptedGCS` (Google) | `EncryptedDisk` |
| --- | --- | --- | --- |
| Provider mechanism | SSE-C | CSEK | `block_cipher_kit`, in your app |
| Where encryption happens | in AWS, before the write | in GCP, before the write | in your Ruby process |
| Key length used | first 32 bytes | first 32 bytes | first 32 bytes (`v2`) |
| Cipher | AES-256 (AWS-managed) | AES-256 (Google-managed) | AES-256-GCM (CTR for ranges) |
| Upload / download | yes | yes | yes |
| `download_chunk` / `Range` requests | yes | yes | yes |
| Direct upload (presigned `PUT`) | yes, straight to S3 | yes, straight to GCS | yes, through our controller |
| `compose` | yes (streams through your app) | **not implemented** (raises `NotImplementedError`) | yes |
| `public: true` | forbidden (raises `ArgumentError`) | ignored (`public?` is always false) | forbidden (raises `ArgumentError`) |
| Extra CORS headers needed | 3 `x-amz-*` headers | 3 `x-goog-*` headers | `x-active-storage-encryption-key`, `content-md5` |

Every one of them accepts `private_url_policy:` (`stream`, `require_headers` or `disable` — default `stream`), described in [Private URL constraints](#private-url-constraints).

### AWS S3 - EncryptedS3Service

Uses [SSE-C](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerSideEncryptionCustomerKeys.html#specifying-s3-c-encryption). Configuration is that of the stock `S3Service` plus `private_url_policy`:

```yaml
# config/storage.yml
encrypted_s3:
  service: EncryptedS3
  private_url_policy: stream
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: eu-central-1
  bucket: my-app-secure-documents
```

You need the `aws-sdk-s3` gem in your `Gemfile`, exactly as for the stock service. If your app runs on EC2/ECS/EKS with an instance or task role, leave `access_key_id`/`secret_access_key` out and let the SDK pick up the ambient credentials.

Notes and requirements:

* **IAM.** The same actions as for a normal ActiveStorage bucket: `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket` (and `s3:AbortMultipartUpload` if you use `compose`). SSE-C needs no KMS permissions at all — the key never goes to KMS.
* **HTTPS is mandatory.** S3 rejects SSE-C requests over plain HTTP.
* **CORS.** For direct uploads, allow these request headers on the bucket:
  * `x-amz-server-side-encryption-customer-algorithm`
  * `x-amz-server-side-encryption-customer-key`
  * `x-amz-server-side-encryption-customer-key-MD5`
* **S3-compatible providers.** SSE-C is an AWS feature. Other services offering S3-compatible object storage (Minio, Ceph, R2...) may not support it — check the documentation of your provider before pointing this service at them.
* **`exist?` works differently.** The stock service does a `HEAD` request, but retrieving object metadata for an SSE-C object requires the encryption key, and `Service#exist?` has no way to accept one. So we issue a one-byte `GET` instead and distinguish the resulting errors (`InvalidRequest` = exists but encrypted, `NoSuchKey` = absent).
* **`compose` is unchanged in cost.** The stock `S3Service#compose` also streams through your app, so there is no reduction in functionality vis-a-vis the standard `S3Service`.
* **Presigned GET URLs need headers** — see [Private URL constraints](#private-url-constraints).

While S3 allows the `x-amz-server-side-encryption-customer-key-MD5` to be added to the signed URL for PUT, the value of that header gets removed from the signature due to the process called "hoisting" - which occurs during the signing of the URL. So your client _may_ override the encryption key you give it forcibly, by replacing the `x-amz-server-side-encryption-customer-key` and `x-amz-server-side-encryption-customer-key-MD5`. This can produce Blobs encrypted with a key you do not have. If you want to exclude the possibility of this, you need to perform an integrity check on your uploads. The integrity check will fail if the encryption key has been overridden in this manner, and you can then destroy the Blob. This problem has been reported to AWS.

### Google Cloud Storage - EncryptedGCSService

Uses [customer-supplied encryption keys (CSEK)](https://cloud.google.com/storage/docs/encryption/using-customer-supplied-keys). Configuration is that of the stock `GCSService` plus `private_url_policy`:

```yaml
# config/storage.yml
encrypted_gcs:
  service: EncryptedGCS
  private_url_policy: stream
  project_id: my-gcp-project
  bucket: my-app-secure-documents
  credentials: <%= Rails.root.join("config/gcs_keyfile.json") %>
```

You need the `google-cloud-storage` gem in your `Gemfile`. As with the stock service, `credentials:` can be a path to a service account JSON keyfile, a hash of its contents, or omitted entirely when the app runs with [application default credentials](https://cloud.google.com/docs/authentication/application-default-credentials) (GKE Workload Identity, Cloud Run, or `GOOGLE_APPLICATION_CREDENTIALS` in the environment). If you sign URLs through the IAM API rather than with a local private key, the stock `iam: true` / `gsa_email:` options are honoured for the signing of both direct upload and `require_headers` URLs.

Notes and requirements:

* **IAM.** `roles/storage.objectAdmin` on the bucket (or the equivalent `storage.objects.create` / `get` / `delete` / `list` permissions) is enough. CSEK requires no Cloud KMS permissions.
* **CORS.** For direct uploads, allow these request headers on the bucket:
  * `x-goog-encryption-algorithm`
  * `x-goog-encryption-key`
  * `x-goog-encryption-key-sha256`
* **`compose` is not supported.** GCP's `compose` RPC requires all source objects to be encrypted with the *same* key, and with this gem every `Blob` has its own. Composing would therefore mean downloading, decrypting and re-uploading, which the service does not currently do — `EncryptedGCSService#compose` raises `NotImplementedError`. If you need composed blobs, use S3 or the disk service.
* **The key must be at least 32 bytes** — the service raises `ArgumentError` otherwise. Keys generated by this gem are 48 bytes, so this only matters if you supply your own.
* **Presigned GET URLs need headers** — see [Private URL constraints](#private-url-constraints).

### Local disk - EncryptedDiskService

Can be used instead of the cloud services in development, or on the server if desired. Unlike the cloud services, encryption happens inside your Ruby process.

```yaml
# config/storage.yml
encrypted_disk:
  service: EncryptedDisk
  private_url_policy: stream
  root: <%= Rails.root.join("storage", "encrypted") %>
```

Implementation details:

* The schemes for encryption are in the `block_cipher_kit` gem. The current scheme (`v2`) uses AES-256-GCM for blobs, with the authentication tag verified in case of a full download. Random access uses AES-256-CTR, since GCM cannot be authenticated without reading the whole message.
* Files get an `.encrypted-v<N>` filename extension, where `v<N>` is the version of the encryption scheme applied. New files are always written with the newest scheme; older files keep being read with the scheme their extension names, so an added scheme does not invalidate existing data.
  * `v2` (current): AES-256-GCM, using the first 32 bytes of the blob key, with a random IV per blob and the auth tag at the end of the ciphertext.
  * `v1` (legacy, read for existing files only): AES-256-CFB, taking the first 16 bytes of the blob key as the IV and the following 32 bytes as the AES key.
* A SHA2 digest of the encryption key is stored at the beginning of the encrypted file. This is used as a key check value, to deny a download rapidly (and without tying up server resources) if an incorrect encryption key is provided — an `ActiveStorageEncryption::IncorrectEncryptionKey` gets raised.
* Direct uploads go to `EncryptedBlobsController#update` (`PUT /blob/:token` under the engine mount point) rather than to a cloud endpoint. The signed token pins the object key, service, content length, checksum and the SHA256 of the encryption key; the client sends the key itself in the `x-active-storage-encryption-key` header and the MD5 in `Content-MD5`. Mismatches are rejected with `422`.
* Presigned URLs are subject to the [same constraints](#private-url-constraints) as the other providers, and are always served by the provided streaming controller.

### Mirroring - EncryptedMirrorService

We provide a version of the `EncryptedMirrorService` which is going to use the same encryption key when mirroring to multiple services:

```yaml
# config/storage.yml
encrypted_mirrored:
  service: EncryptedMirror
  primary: encrypted_s3
  mirrors:
    - encrypted_gcs
```

It needs some modifications in comparison to a standard `MirrorService` because if any services it mirrors to use encryption, it needs an encryption key to be provided upstream for all write operations - so that it can be passed on to downstream Services. The key is handed to the background mirroring job inside an *encrypted* token (using `ActiveStorageEncryption.token_encryptor`), so it does not sit in plaintext in your job queue.

* `private_url_policy` is delegated to the primary service and cannot be set on the mirror service itself (doing so raises `ArgumentError`).
* Mirror targets that are *not* encrypted services would receive the plaintext bytes, which defeats the purpose of using this gem. Use encrypted services for the primary and for every mirror.

## Private URL constraints

Both major cloud providers (S3 and GCP cloud storage) disallow using signed GET URLs unless you also supply the encryption key (and encryption parameters, as well as the key checksum) in the GET request headers. This has _severe_ implications for use of `Blob#url` and `rails_blob_path` / `rails_blob_url`, namely:

* You cannot redirect to a signed GET URL for an ActiveStorage blob (for downloading). Standard Rails `blob_path` helpers lead to an ActiveStorage controller which will try to redirect your browser to such a URL, but the browser will then receive a `403 Forbidden` response.
* You can no longer use a signed GET URL for an ActiveStorage blob as the `src` attribute for an `img`, `video` or `audio` element

Cloud providers presumably disallow supplying the encryption key inside the URL itself because they want to prevent those URLs getting saved in web server / load balancer logs, and from being shared. This is a valid security concern, as most URL signing schemes are just for _signing_ but not for _encryption._ An encryption key of this nature could also be retained by a malicious party and reused.

However, for practical purposes you _may_ want to permit such URLs to be generated by your application, with very limited expiry time. We allow for this, with an associated limitation that the blob binary data **is then going to be streamed.** In that setup your Rails app functions as a streaming proxy, which will perform the request to cloud storage - passing along the requisite credentials - and stream the output to your client. This may not be the most performant way to stream data, but when per-file encryption is required this usually concerns sensitive files, which are not very widely shared anyway. We believe streaming to be a sensible compromise. Note that you want the streaming URLs to be short-lived!

To configure this facility, every encrypted `Service` we provide supports the `private_url_policy` configuration parameter. The possible values are as follows:

* `private_url_policy: stream` (**the default**) will stream the decrypted Blob through our Rails controller. The URLs to that controller will not expose the encryption key. `rails_blob_path` will work, and generate a URL to the stock Rails `ActiveStorage::Blobs::RedirectController#show` action. That action, in turn, will generate a URL leading to `ActiveStorageEncryption::EncryptedBlobProxyController#show`. That action will stream out your file from whichever encrypted service your Blob is using. This requires the [engine to be mounted](#2-mount-the-engine).
* `private_url_policy: require_headers` will generate signed URLS, and you will need to ensure these URLs are only requested with the correct HTTP headers. The URLs will not expose the encryption key. When trying to use `rails_blob_path` you will end up receiving a 403 from the cloud storage provider after the redirect. You still may want to generate those URLs if you want to use them elsewhere and will be willing to manually add HTTP headers to the request. Note that `EncryptedDiskService` has no cloud endpoint to sign for, so it keeps serving through the streaming controller — but it will then insist on the `x-active-storage-encryption-key` header being present, to mimic the cloud services.
* `private_url_policy: disable` will make every call to `Blob#url` raise `ActiveStorageEncryption::StreamingDisabled`. This will be raised by the stock Rails `ActiveStorage::Blobs::RedirectController#show` too, so use `Blob#download` / `Blob#open` instead.

For using the `require_headers` option you may want to use the service's `headers_for_private_download(key, encryption_key:)` method - it will return you a `Hash` of headers that have to be supplied along with your request to the signed URL of the cloud service:

```ruby
service = blob.service
service.headers_for_private_download(blob.key, encryption_key: blob.encryption_key)
# => {"x-amz-server-side-encryption-customer-key" => "<base64>"}
```

It is currently implemented by `EncryptedS3Service` and `EncryptedDiskService`. For `EncryptedGCSService` the headers are the CSEK trio listed in [How the key reaches the storage provider](#how-the-key-reaches-the-storage-provider), with the key truncated to 32 bytes.

## Direct uploads

Our recommended setup is to have your encrypted ActiveStorage service as an additional service configuration in `storage.yml`:

```yaml
# config/storage.yml
development:
  public: true
  service: Disk
  root: <%= Rails.root.join("storage") %>

encrypted_disk:
  service: EncryptedDisk
  root: <%= Rails.root.join("storage", "encrypted") %>
  private_url_policy: stream
```

To upload into a named service that is non-default, you will need to use a different method for generating your presigned upload URL, as the standard Rails controller that creates the `Blob` records prior to upload does not allow you to set it. If you follow the [official Rails guide](https://guides.rubyonrails.org/active_storage_overview.html#direct-uploads) you will need to use a different URL helper for generating a URL to create blobs, which _does_ accommodate the service name. The URL you need to use is `create_encrypted_blob_direct_upload_url` (instead of `rails_direct_uploads_url`), and it takes the service name as a parameter:

```erb
<%= form.file_field :id_document_scan,
      direct_upload: true,
      data: {direct_upload_url: active_storage_encryption.create_encrypted_blob_direct_upload_url(service_name: :encrypted_s3)} %>
```

This is not going to be necessary once the corresponding [Rails issue](https://github.com/rails/rails/issues/38940) will get addressed.

All supported services accept the encryption key as part of the headers for the PUT direct upload. The provided Service implementations will generate the correct headers for you. However, your upload client _must_ use the headers provided to you by the server, and not invent its own. The standard ActiveStorage JS bundles honor those headers - but if you use your own uploader you will need to ensure it honors and forwards the headers too.

We also configure the services to generate _and_ require the `Content-MD5` header. The client doing the PUT will need to precompute the MD5 for the object before starting the PUT request or getting a PUT url (as the checksum gets signed by the cloud SDK or by our `EncryptedDiskService`). This is so that any data transmission errors can be intercepted early (we know of one case where a bug in Ruby, combined with a particular HTTP client library, led to bytes getting uploaded out of order). The `create_direct_upload` action rejects requests that do not carry a `checksum` for the blob.

Note that the encryption headers may require amending your CORS configuration - see the documentation per service regarding that.

## Downloading the plaintext file contents in full or in part

Data will be automatically decrypted using the correct key on `Blob#open` or `Blob#download`. `EncryptedDiskService` will also apply the GCM validation check at the end of download/read (authenticate the cipher).

All of our encrypted Services support `download_chunk` (`Blob#download_chunk(range)`) for random access to the blob's plaintext. The decryption will be done in a streaming manner, without buffering:

* Cloud providers give you access to plaintext segments of the file using the HTTP `Range:` header (ranges are in plaintext offsets)
* `EncryptedDiskService` provides the same, but inside the app will access OS files - decrypting them in a streaming manner.

The streaming controller supports `Range` requests too (using the `serve_byte_range` gem), so `<video>`/`<audio>` seeking works against a streaming private URL.

## Constraints with encrypted Blobs

There are also subtle differences in how cloud providers treat encrypted objects in their object storage vs. how other objects are treated, as well as which facilities change or become unavailable once your object is encrypted. Additionally, some ActiveStorage operations change semantics or start requiring an `encryption_key:` argument. Understanding those limitations is key to using active_storage_encryption correctly and effectively.

Key differences are as follows:

* If a Service supports encryption, _every_ blob stored on it will be encrypted. No exceptions. You cannot supply an encryption key of `nil` to bypass encryption on that service.
* A blob stored onto - or retrieved from - an encrypted Service must have an `#encryption_key` that is not `nil`
* Most operations performed on an encrypted Service must supply the encryption key, or multiple encryption keys (in case of `Service#compose`)
* An encrypted Service cannot be `public: true` - no CDNs we are aware of can proxy objects with per-object encryption keys.
* An encrypted Service cannot generate a signed GET URL unless you let that URL go through a streaming controller (we provide one), or you are going to send headers to the cloud providers' download endpoint. We default to using the streaming controller, which may cause a performance impact on your app due to slow clients. See more [here.](#private-url-constraints)
* Objects using per-object encryption are usually **inaccessible for cloud automation** - for example, scripts that load files into a cloud database using paths to the bucket will likely not work, as data is no longer readable for them. There will also be limitations for CLI clients. For example, the `gcloud` CLI only allows you to supply 100 encryption keys, and thus will only be able to download 100 objects at once. The same goes for the cloud console (web UI): objects cannot be previewed or downloaded from it.

### Additional information

* The stored `digest` (the Base64-encoded MD5 of the blob) will be for the plaintext file contents. This reduces security somewhat, because MD5 has known collisions and facilitates (to some extend) mounting a "known plaintext" attack.
* The stored `filesize` will be for the plaintext. This, again, facilitates an attack somewhat.

## What this gem _can_ do

It is a tool for additional protection of sensitive files.

Normally, your cloud storage used for binary data will already support some form of encryption-at-rest, usually using the could provider's KMS (key management service). This is a sensible default, however it does not protect you from one important attack vector: a party obtaining access to your cloud storage bucket. If they do manage to obtain that access, they usually also have access to the KMS (by virtue of having access to a cloud account you control) and can bulk-download all of your data, unencrypted. All an attacker needs are cloud credentials for an account with "read" and "list" permissions.

With per-object encryption, however, just access to the bucket does not give the attacker much. Since every object is encrypted with a separate key, they need to have a key for every single file. That key is not stored in the provider's KMS, so even an account with KMS access won't be able to decrypt them.

Additionally, neither the cloud console (web UI) nor the API client will be able to download those objects without the keys.

The only way to obtain access would be for the attacker to have access to:

* A database dump of the `active_storage_blobs` table
* Your application's secrets (to decrypt the values in the table).
* The cloud storage bucket

It's way more work, and way more hassle. This is great for sensitive files, and increases security considerably.

## What this gem _cannot_ do

This gem does not provide an E2E encrypted solution. The file still gets encrypted by your cloud provider, and decrypted by your cloud provider. While it offers a strong protection _at rest_ it does not offer extra protection _in transit._ If you need that level of protection, you may want to look into [S3 client encryption](https://ankane.org/activestorage-s3-encryption) or other similar tech.

## Security considerations / notes

### Key exposure on upload

Both implementations of customer-supplied encryption keys (S3 and GCP) sign the checksum of the encryption key issued to the uploading client, so that the client may not alter the encryption key your application has issued. However, neither of them support key wrapping - encrypting the key before giving it to the client for performing the upload. GCP does support key wrapping, but only for its Compute Platform, and not for Cloud Storage. Therefore, the uploading client (the one that performs the PUT to the cloud storage or to our controller) is going to be able to decode and retain the raw encryption key.

You can, of course, rewrite the object in storage to decrypt it and re-encrypt it with a new encryption key, which the original uploader does not possess. This takes extra resources, however.

### Key exposure on download

When we stream through the controller, we encrypt the token instead of just signing it. This conceals the encryption key of the Blob, and uses standard Rails encryption. For our purposes we consider this configuration sufficiently secure.

### Notes

* It is imperative that you let the server generate the encryption key, and not generate one on the client. Where possible, we add the headers for the encryption key to the signed URL parameters, so client generated keys will deliberately _not_ function. Letting the client generate the keys can lead to key reuse (unintentional or - worse - intentional).
* The key used is _not_ a passphrase or a password. It is a high-entropy bag of bytes, generated for every blob separately using `SecureRandom`. We generate more bytes than the cloud providers usually expect (32 bytes for AES) and take the starting bytes off that longer key - that is to allow mirroring to services that have varying key lengths, using the same encryption key.
* To the best of our knowledge, both S3 and GCS use AES-256-GCM (or a variation thereof) for encryption. Random access to GCM blocks requires dropping the auth tag validation (cipher authentication) of GCM, and downgrades it to CTR. We find this an acceptable tradeoff to enable random access.
* Your `encryption_key` column **must** be an encrypted ActiveRecord attribute (this gem declares `encrypts :encryption_key` for you, but you have to [configure ActiveRecord encryption](#3-configure-activerecord-encryption)), otherwise your data is not really safe in case a database dump gets exfiltrated.
* active_storage_encryption has not been verified by information security experts or cryptographers. While we do our best to have it be robust and secure, mishaps may happen. You have been warned.
* While cloud providers do publish some information about their encryption approaches, we do not know several crucial details allowing one to say that "this encryption scheme is secure enough for our needs". Namely:
  * Neither GCP nor AWS say how the IV gets generated. A compromised IV (repeated IV) simplifies breaking a block cipher considerably.
  * Neither GCP nor AWS say when they reset the IV. Counter-based IVs have a limit on the number of counter values and blocks that they support. For GCM and CTR the practical limit is around 64GB per encrypted message. To add extra safety, it is sometimes advised to "stitch" the message from multiple messages with the IV getting regenerated at random, for every message. If providers do use such "chunking", we could not find information about the size of the chunks nor the mechanics by which the IV gets generated for them.

Finally: this gem has not been verified by an information security expert or a cryptographer. While we did take all the possible precautions with regards to producing a secure design, we are all humans and might have omitted something.

## Migrating your blobs into an encrypted store

We do not provide a built-in method for this, but you can easily implement this yourself:

* Generate an `encryption_key` for the blob in question (`ActiveStorage::Blob.generate_random_encryption_key`)
* Stream the plaintext data into a copy on the encrypted service, applying encryption. The way to do this depends on the encrypted service used - `service.upload(key, io, encryption_key:, checksum:)` is the lowest common denominator.
* Transactionally store the encryption key on the `Blob` _and_ switch its `service_name` to the encrypted service.

Since the encrypted services keep using the stock ActiveStorage key/path layout, you do not need to change the `key` of the blob.

## Development

```shell
bundle install
bundle exec rake # runs app:test in test/dummy
```

The suite runs against SQLite and the `EncryptedDiskService` out of the box. The cloud service tests skip themselves unless credentials are present in the environment:

* `EncryptedS3Service`: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (the test bucket and region are set in `test/lib/encrypted_s3_service_test.rb`)
* `EncryptedGCSService`: `GOOGLE_APPLICATION_CREDENTIALS` pointing at a service account JSON keyfile (project and bucket are set in `test/lib/encrypted_gcs_service_test.rb`)

`test/dummy` is a small Rails app used by the integration tests, and it doubles as a worked example of the configuration described above: see `test/dummy/config/storage.yml`, `test/dummy/config/routes.rb` and `test/dummy/app/models/user.rb`. Use `bundle exec appraisal install && bundle exec appraisal rake` to run the suite against the supported Rails versions (see `Appraisals`), and `bundle exec standardrb` for linting (`rake format` fixes what it can).

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
