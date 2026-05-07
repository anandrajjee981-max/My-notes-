ImageKit.md (Hinglish Notes)

1. ImageKit Kya Hai?

ImageKit ek media optimization + CDN service hai jo images/videos ko fast deliver karta hai.
Node.js SDK ka use backend mein upload, transform, signed URL, webhook verification etc. ke liye hota hai.


---

2. Installation

npm install @imagekit/nodejs

Import Karna

import ImageKit from '@imagekit/nodejs'


---

3. Client Setup

Definition

ImageKit client = SDK ka main object jisse saare operations karte hain.

const client = new ImageKit({
  privateKey: process.env.IMAGEKIT_PRIVATE_KEY
})

Important

privateKey secret hoti hai

Backend mein hi use karo

Frontend mein kabhi mat bhejo



---

4. File Upload

Definition

Server se image/video upload karna.


---

Basic Upload

import fs from 'fs'

await client.files.upload({
  file: fs.createReadStream('./photo.jpg'),
  fileName: 'photo.jpg'
})


---

Upload Methods

1. fs.ReadStream (Recommended)

fs.createReadStream('./photo.jpg')

2. File Object

new File(['data'], 'file')

3. Fetch Response

await fetch('https://abc.com/image.jpg')

4. toFile Helper

import { toFile } from '@imagekit/nodejs'

await client.files.upload({
  file: await toFile(Buffer.from('hello'), 'demo'),
  fileName: 'demo.txt'
})


---

5. URL Generation

Definition

Image ka optimized URL banana.


---

Basic URL

const url = client.helper.buildSrc({
  urlEndpoint: 'https://ik.imagekit.io/your_id',
  src: '/image.jpg'
})

Output

https://ik.imagekit.io/your_id/image.jpg


---

6. Transformations

Definition

Image ko resize/compress/format change karna.


---

Resize + WebP

const url = client.helper.buildSrc({
  urlEndpoint: 'https://ik.imagekit.io/your_id',
  src: '/image.jpg',
  transformation: [
    {
      width: 400,
      height: 300,
      quality: 80,
      format: 'webp'
    }
  ]
})


---

Common Transformations

Property	Meaning

width	image width
height	image height
quality	compression quality
format	webp/jpg/png
crop	crop type



---

7. Overlay

Definition

Image ke upar image ya text add karna.


---

8. Image Overlay

overlay: {
  type: 'image',
  input: '/logo.png',
  position: {
    x: 10,
    y: 10
  }
}

Use Case

Watermark

Logo

Sticker



---

9. Text Overlay

overlay: {
  type: 'text',
  text: 'Hello Anand',
  transformation: [
    {
      fontSize: 40,
      fontColor: 'FFFFFF'
    }
  ]
}

Use Case

Meme text

Thumbnail title

Quotes



---

10. Multiple Overlay

Definition

Ek hi image mein multiple overlays.

transformation: [
  {
    overlay: {
      type: 'text',
      text: 'Header'
    }
  },
  {
    overlay: {
      type: 'image',
      input: '/watermark.png'
    }
  }
]


---

11. Signed URL

Definition

Secure URL jo expire ho sakta hai.


---

Example

const signedUrl = client.helper.buildSrc({
  urlEndpoint: 'https://ik.imagekit.io/your_id',
  src: '/private.jpg',
  signed: true,
  expiresIn: 3600
})

Use Case

Premium content

Private files

Temporary access



---

12. Authentication Parameters

Definition

Frontend upload secure banane ke liye auth values generate karna.

const auth = client.helper.getAuthenticationParameters()

Returns

{
  token,
  expire,
  signature
}

Use Case

Frontend upload without exposing private key.


---

13. Webhook Verification

Definition

Check karna webhook ImageKit se hi aaya hai ya fake hai.


---

Setup

const client = new ImageKit({
  privateKey: process.env.PRIVATE_KEY,
  webhookSecret: process.env.WEBHOOK_SECRET
})


---

Verify Webhook

const event = client.webhooks.unwrap(
  webhookBody,
  {
    headers: webhookHeaders
  }
)


---

14. Error Handling

Definition

API fail hone pe error catch karna.

try {

} catch(err) {

}


---

Example

try {
  await client.files.upload({...})
} catch(err) {

  if(err instanceof ImageKit.APIError){
    console.log(err.status)
  }
}


---

15. Error Types

Status	Error

400	BadRequestError
401	AuthenticationError
403	PermissionDeniedError
404	NotFoundError
429	RateLimitError
500+	InternalServerError



---

16. Retries

Definition

Request fail ho toh automatically dobara try.


---

Default

2 retries hoti hain.


---

Custom Retry

const client = new ImageKit({
  maxRetries: 5
})


---

17. Timeout

Definition

Kitna time wait kare request ke liye.


---

Example

const client = new ImageKit({
  timeout: 20000
})

Meaning

20 seconds wait karega.


---

18. Raw Response

Definition

Original fetch response access karna.


---

asResponse()

const response = await client.files
  .upload({...})
  .asResponse()

Use Case

Headers access karna.


---

withResponse()

const {data , response} =
await client.files
.upload({...})
.withResponse()

Use Case

Data + raw response dono chahiye.


---

19. Logging

Definition

Debug information console mein show karna.


---

Log Levels

Level	Meaning

debug	sab kuch
info	info + warning
warn	warning
error	only errors
off	no logs



---

Example

const client = new ImageKit({
  logLevel: 'debug'
})


---

20. Custom Logger

Definition

Own logger use karna like pino/winston.

import pino from 'pino'

const logger = pino()

const client = new ImageKit({
  logger,
  logLevel: 'debug'
})


---

21. Custom API Request

Definition

Undocumented endpoint hit karna.

await client.post('/some/path', {
  body: {
    hello: 'world'
  }
})


---

22. Fetch Customization

Definition

Custom fetch use karna.


---

Example

import fetch from 'my-fetch'

const client = new ImageKit({
  fetch
})


---

23. Fetch Options

const client = new ImageKit({
  fetchOptions: {}
})

Use Case

Proxy

Headers

Custom request config



---

24. Proxy Setup

Node.js Proxy

import * as undici from 'undici'

const proxyAgent =
new undici.ProxyAgent(
  'http://localhost:8888'
)

const client = new ImageKit({
  fetchOptions: {
    dispatcher: proxyAgent
  }
})


---

25. Semantic Versioning

Meaning

Version update rules.

Important

Minor update mein bhi breaking change aa sakta hai especially MCP package mein.


---

26. MCP Server

Definition

AI agents ko direct ImageKit API access dene wala package.

@imagekit/api-mcp

Use Case

AI automation

AI assistants

Agent workflows



---

27. Important Interview Points

Must Remember

Image Upload

client.files.upload()

URL Generate

client.helper.buildSrc()

Signed URL

signed: true

Webhook Verify

client.webhooks.unwrap()

Auth Params

client.helper.getAuthenticationParameters()


---

28. Best Practices

DO ✅

Private key backend mein rakho

Signed URL use karo secure files ke liye

WebP format use karo

Quality compress karo

Retry + timeout configure karo



---

DON'T ❌

Private key frontend mein expose mat karo

Large original images directly serve mat karo

Webhook verify kiye bina trust mat karo



---

29. Real World Use Cases

Feature	Use

Transformations	fast loading
Overlay	watermark
Signed URL	premium files
Webhook	upload events
Auth params	secure frontend upload
Compression	performance



---

30. Short Revision

Image Upload -> files.upload()

Image URL -> helper.buildSrc()

Secure URL -> signed: true

Webhook Verify -> webhooks.unwrap()

Frontend Upload Security ->
getAuthenticationParameters()
