# CloudFlare Workers `FormData`

CloudFlare Workers supports getting a `FormData` instance from the request using
`request.formData()`. However, the resulting `FormData` instance, when used with
`formData.get('file')`, does not return a `File instance (inclusive of the file
name and type), but a string representation of the file's contents. While it may
be possible to use `charCodeAt` to construct a `Uint8Array` from the string and
recover the file's binary contents this way, access to the file name and type is
seemingly impossible.

I've opened a thread on the CloudFlare Community site for this:
https://community.cloudflare.com/t/worker-formdata-get-file-instance/155009

This might get fixed, it might now, I don't know. Meanwhile, I've prototyped a
basic MIME multipart parser, which is able to recover the file binary contents as
well as the metadata (name and type). Find it here:

https://github.com/TomasHubelbauer/mime-multipart

The contents of the script (in `index.html`) need to be pasted to the worker script
directly, it cannot be referenced in any other way, because Workers does not support
ESM (https://community.cloudflare.com/t/esm-modules-in-cloudflare-workers/155721)
and `eval` is disabled in the worker script context.
