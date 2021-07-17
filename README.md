# CloudFlare Workers `FormData`

CloudFlare Workers supports getting a `FormData` instance from the request using
`request.formData()`. However, the resulting `FormData` instance, when used with
`formData.get('file')`, does not return a `File` instance (inclusive of the file
name and type), but a string representation of the file's contents. While it may
be possible to use `charCodeAt` to construct a `Uint8Array` from the string and
recover the file's binary contents this way, access to the file name and type is
seemingly impossible.

I've opened a thread on the CloudFlare Community site for this:
https://community.cloudflare.com/t/worker-formdata-get-file-instance/155009

This might get fixed, it might not, I don't know. Meanwhile, I've prototyped a
basic MIME multipart parser, which is able to recover the file binary contents as
well as the metadata (name and type). Find it here:

https://github.com/TomasHubelbauer/mime-multipart

The contents of the script (in `index.html`) need to be pasted to the worker script
directly, it cannot be referenced in any other way, because Workers does not support
ESM (https://community.cloudflare.com/t/esm-modules-in-cloudflare-workers/155721)
and `eval` is disabled in the worker script context.

A full example (including the `parseMimeMultipart` function as of the time of writing)
of a worker which accepts a file to be uploaded and serves it right back:

```javascript
addEventListener('fetch', event => event.respondWith(serve(event.request)))

async function serve(request) {
  try {
    const url = new URL(request.url);
    const action = request.method + ' ' + url.pathname;
    if (action === 'GET /') {
      const html = `
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Test</title>
  </head>
  <body>
    <form method="POST" enctype="multipart/form-data">
      <input type="file" multiple name="files[]" />
      <input type="submit" />
    </form>
  </body>
</html>
`;

      return new Response(html, { headers: { 'Content-Type': 'html' } });
    }

    const uint8Arrray = await request.arrayBuffer();
    const parts = [...parseMimeMultipart(uint8Arrray)];
    if (parts.length === 0) {
      return new Response('No parts!');
    }

    if (parts.length > 1) {
      return new Response('Too many parts!');
    }

    const [part] = parts;
    const type = part.headers.find(h => h.name === 'Content-Type')?.values[0] || 'application/octet-stream';
    const blob = uint8Arrray.slice(part.index, part.index + part.length);
    return new Response(blob, { headers: { 'Content-Type': type } });
  } catch (error) {
    return new Response(error.message);
  }
}

// https://github.com/TomasHubelbauer/mime-multipart
function* parseMimeMultipart(/** @type {Uint8Array} */ uint8Array) {
  const textDecoder = new TextDecoder();
  /** @typedef {{ name: string; values: string[]; }} Header */
  /** @typedef {{ type: 'boundary'; boundary: string; }} Boundary */
  /** @typedef {{ type: 'header-name'; boundary: string; name: string; headers: Header[]; }} HeaderName */
  /** @typedef {{ type: 'header-value'; boundary: string; name: string; value: string; values: string[]; headers: Header[]; }} HeaderValue */
  /** @typedef {{ type: 'content'; boundary: string; headers: Headers[]; index: number; length: number; }} Content */
  /** @type {Boundary | HeaderName | HeaderValue | Content} */
  let state = { type: 'boundary', boundary: '' };
  let index = 0;
  let line = 0;
  let column = 0;
  for (; index < uint8Array.byteLength; index++) {
    const character = textDecoder.decode(uint8Array.slice(index, index + 1));
    if (character === '\n') {
      line++;
      column = 0;
    }

    column++;

    switch (state.type) {
      case 'boundary': {
        // Check Windows newlines
        if (character === '\r') {
          if (textDecoder.decode(uint8Array.slice(index + 1, index + 2)) !== '\n') {
            throw new Error(`At ${index} (${line}:${column}): found an incomplete Windows newline.`);
          }

          break;
        }

        if (character === '\n') {
          state = { type: 'header-name', boundary: state.boundary, name: '', value: '', headers: [] };
          break;
        }

        state.boundary += character;
        break;
      }
      case 'header-name': {
        // Check Windows newlines
        if (character === '\r') {
          if (textDecoder.decode(uint8Array.slice(index + 1, index + 2)) !== '\n') {
            throw new Error(`At ${index} (${line}:${column}): found an incomplete Windows newline.`);
          }

          break;
        }

        if (character === '\n') {
          if (state.name === '') {
            state = { type: 'content', boundary: state.boundary, headers: state.headers, index: index + 1, length: 0 };
            break;
          }
          else {
            throw new Error(`At ${index} (${line}:${column}): a newline in a header name '${state.name}' is not allowed.`);
          }
        }

        if (character === ':') {
          state = { type: 'header-value', boundary: state.boundary, name: state.name, value: '', values: [], headers: state.headers };
          break;
        }

        state.name += character;
        break;
      }
      case 'header-value': {
        // Check Windows newlines
        if (character === '\r') {
          if (textDecoder.decode(uint8Array.slice(index + 1, index + 2)) !== '\n') {
            throw new Error(`At ${index} (${line}:${column}): found an incomplete Windows newline.`);
          }

          break;
        }

        if (character === ';') {
          state.values.push(state.value);
          state.value = '';
          break;
        }

        if (character === ' ') {
          // Ignore white-space prior to the value content
          if (state.value === '') {
            break;
          }
        }

        if (character === '\n') {
          state.values.push(state.value);
          state = { type: 'header-name', boundary: state.boundary, name: '', value: '', headers: [{ name: state.name, values: state.values }, ...state.headers] };
          break;
        }

        state.value += character;
        break;
      }
      case 'content': {
        // If the newline is followed by the boundary, then the content ends
        if (character === '\n' || character === '\r' && textDecoder.decode(uint8Array.slice(index + 1, index + 2)) === '\n') {
          if (character === '\r') {
            index++;
          }

          const boundaryCheck = textDecoder.decode(uint8Array.slice(index + '\n'.length, index + '\n'.length + state.boundary.length));
          if (boundaryCheck === state.boundary) {
            const conclusionCheck = textDecoder.decode(uint8Array.slice(index + '\n'.length + state.boundary.length, index + '\n'.length + state.boundary.length + '--'.length));
            if (conclusionCheck === '--') {
              index += '\n'.length + state.boundary.length + '--'.length;
              yield { headers: state.headers, index: state.index, length: state.length };

              if (index !== uint8Array.byteLength) {
                const excess = uint8Array.slice(index);
                if (textDecoder.decode(excess) === '\n' || textDecoder.decode(excess) === '\r\n') {
                  return;
                }

                throw new Error(`At ${index} (${line}:${column}): content is present past the expected end of data ${uint8Array.byteLength}.`);
              }

              return;
            }
            else {
              yield { headers: state.headers, index: state.index, length: state.length };
              state = { type: 'boundary', boundary: '' };
              break;
            }
          }
        }

        state.length++;
        break;
      }
      default: {
        throw new Error(`At ${index} (${line}:${column}): invalid state ${JSON.stringify(state)}.`);
      }
    }
  }

  if (state.type !== 'content') {
    throw new Error(`At ${index} (${line}:${column}): expected content state, got ${JSON.stringify(state)}.`);
  }
};
```

## To-Do

### See if with Durable Objects and ESM this script could be a dependency

https://developers.cloudflare.com/workers/cli-wrangler/configuration#modules

Cloudflare Workers now seem to support ESM, but the questions remains if this is
true ESM or some Wrangler bundling, and if so, if it supports pulling in HTTP
modules as opposed to file protocol modules only.

Would be good to try this while I still have Workers subscription.
