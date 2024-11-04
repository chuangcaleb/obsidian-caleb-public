---
collection:
  - technology
published: '2024-08-06'
up:
  - technology
created: '2024-08-06T00:00:00.000Z'
modified: '2024-11-04T00:00:00.000Z'
slug: making-my-website-guestbook
---
# Making My Website Guestbook

I’ve made a website guestbook: where people can write a message, and it’ll be displayed on this site at <https://chuangcaleb.com/guestbook>! This is a quick overview of how I made it.

## TL;DR

- Built a Google Form
	- Manually+personally distribute the link to people
- Google Form writes to a Google Sheet spreadsheet
- Use a package to (make an authenticated) read from the Google Sheet
- Render results on site
- Click [this skip link](#Implementation) to skip past the requirements specs to the implementation

## Requirements

The guestbook consists of three parts:

1. 📝 A public web form
2. 💾 A persistent storage/database
3. 🖼️ A web page to render entries from the storage

I already had my website `chuangcaleb.com`, so it’s just a matter of the first two!

One particular general requirement was to be able to **review a form entry** before marking it as “safe” for building on the webpage. This is to protect against malicious actors who somehow get a hold of the web form link, and writes malicious content that automatically gets published to the site. Think of those YouTube comment bots. Yucks.

Then there’s also **security**. The workflow needs to be airtight, with no leaks of any private data and personal identifiers.

### The public web form

- needed to be publicly accessible + shareable via link
- user input handling: text, radio single-choice, checkbox multi-choice, etc.
- simple validation, especially for max input length
- some protection against spam attacks

### Persistent Storage

- persistent (lol), not ephemeral
- able to store various data types
- restricted authorised access (by me, only)
	- authorised writes to verify entries
- able to access via API by the website build runner (and during local dev, ofc)
	- secure authorised access, not just publicly available

## Implementation

I played around with different combinations. I had thought out a full-blown system using [Formbricks](https://formbricks.com/) hooked up to a Postgres database hosted on [Railway](https://railway.app/), where the db is exposed via a public URL, and we get the passwords, etc. But after brainstorming all that, I realised…

> That’s over-engineering.

It’s not even a standalone repo, it’s a minor feature that only gets used a few times in a given season.

So, time to scale it back _as simple as possible._

### Google Forms/Sheets

At work, we use Google Sheets to manage internationalisation/translation text variants and feature flags, published as JSON endpoints. I realised that this was the key to the simple workflow that I minimally needed!

I'm using simple [Google Forms](https://support.google.com/a/users/answer/9303071?hl=en) to collect responses and save them in a Google Sheet. That’s basically the form and storage, sorted. I think they’re common and intuitive, so I won’t go into details why they meet the criteria. But they **completely** do.

To verify people, I enable the `Collect email addresses of participants` and `Collect verified emails` form options. This means that they must sign in with an authenticated Google account to submit the form. After people write and submit their posts, I did enable sending a notification on form entry.

I created an extra column `isVerified`. When new entries come in, they will ignore columns with mismatching headers. And if the form schema is modified to have a new column, the custom column will be ignored, and the new column will be created in the next available column!

The `isVerified` column has a data validation for check boxes, for better soft-“typed” validation and ease of modification.

```
'Form responses 1'!A2:A109
Criteria: Check Box
✅ Use custom cell values
Ticked: TRUE
Unticked: FALSE
```

My spreadsheet would look something like this (but transposed, of course):

| key                  | value                                                              |
| -------------------- | ------------------------------------------------------------------ |
| isVerified           | TRUE                                                               |
| Timestamp            | 12/07/2024 21:22:46                                                |
| Email address        | example@email.com                                                  |
| 📇 Display Name      | caleb                                                              |
| 🤝 Real Name         | Chuang Caleb                                                       |
| 👥 Relation to me    | myself                                                             |
| 🏷️ Tag              | Other                                                              |
| 📃 Content Body      | 🎤 Testing, testing. Is this thing on?                             |
| 🔗 Social Link       | <https://chuangcaleb.com>                                          |
| 📸 Avatar Image Link | <https://www.gravatar.com/avatar/29d863c08e05a20bab30479ecae823eb> |

### Spreadsheet as API + Web Render

I use the [theoephraim/node-google-spreadsheet: Google Sheets API wrapper for Javascript / Typescript](https://github.com/theoephraim/node-google-spreadsheet) package to grab the rows of the spreadsheet.

I first created a service account on <https://console.cloud.google.com> and registered my spreadsheet to the account. I will have three environment variables:

```json
GOOGLE_SERVICE_ACCOUNT_EMAIL=guestbook@xxxxxx-xxxxxx-000000.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nxxxxxxxxxxxxxxx\n-----END PRIVATE KEY-----\n"
GUESTBOOK_GOOGLE_SHEET_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
```

In my source code, I like to decouple and make functions as generic as possible — what if I have another Google Sheet, or one day I want to read from Google Docs? The relevant files look like this filetree:

```
.
├── .env
├── lib/
│   └── google-sheets/
│       ├── jwt.ts
│       ├── sheet.ts
│       └── types.ts
└── src/
    └── pages/
        └── guestbook/
            ├── _GuestPost.astro
            └── index.astro
```

#### jwt.ts

I first export a reusable `jwt` instance. Two notes:

- I would like to move off `dotenv` for something more type-secure. But IT WORKS FOR NOW
- You do need to do `.replace(/\\n/g, '\n')` on the `GOOGLE_PRIVATE_KEY` , because while it may work locally on your machine, some build runners like Cloudflare (which I’m using) will read the escaped-newline as literally `\n`, ending up with a different invalid `GOOGLE_PRIVATE_KEY`. Just include this regex to be safe, it won’t break anything if it reads it correctly.

```ts
// jwt.ts
import process from 'node:process';
import {JWT, type JWTOptions} from 'google-auth-library';
import dotenv from 'dotenv';

dotenv.config();

const jwtOptions: JWTOptions = {
	email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
	key: process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
	scopes: ['https://www.googleapis.com/auth/spreadsheets'],
};
const jwt = new JWT(jwtOptions);

export {jwt as default};
```

#### sheet.ts

Then this next file actually has the method for reading.

- I do just select the first sheet (`document.sheetsByIndex`) of the spreadsheets. Don’t expect to have more than one lol
- You want an output of Array of Javascript Objects, but you get a custom Array of [GoogleSpreadsheetRow](https://theoephraim.github.io/node-google-spreadsheet/#/classes/google-spreadsheet-row)s, so you do need to do a map and get `r -> r.toObject()`.

```ts
// sheet.ts
import process from 'node:process';
import dotenv from 'dotenv';
import {GoogleSpreadsheet} from 'google-spreadsheet';
import jwt from './jwt';
import type {GuestPost} from './types';

dotenv.config();

export async function getGuestbookPosts() {
	try {
		const id = process.env.GUESTBOOK_GOOGLE_SHEET_ID;
		if (!id) {
			throw new Error('Missing GUESTBOOK_GOOGLE_SHEET_ID in .env');
		}

		const document = new GoogleSpreadsheet(id, jwt);

		await document.loadInfo();
		const sheet = document.sheetsByIndex[0];
		const rows = await sheet.getRows();

		return rows.map(r => r.toObject()) as GuestPost[];
	} catch (error) {
		throw error;
	}
}
```

#### types.ts

I assert the type of the results with the following Typescript type:

```ts
// types.ts

// commented the fields we don't want to expose
export type GuestPost = {
	isVerified?: 'TRUE';
	'Timestamp': string;
	// 'Email address' ;
	'📇 Display Name': string;
	// '🤝 Real Name':;
	'👥 Relation to me': string;
	'🏷️ Tag': 'Work' | 'School' | 'Other';
	'📃 Content Body': string;
	'🔗 Social Link'?: string;
	'📸 Avatar Image Link'?: string;
};
```

So `await getGuestbookPosts()` should return a result like following:

```json
// faked data, for example purposes
[
  ...
  {
    isVerified: 'TRUE',
    Timestamp: '12/07/2024 21:22:46',
    'Email address': 'email@address.com',
    '📇 Display Name': 'Chuang Caleb',
    '🤝 Real Name': 'caleb',
    '👥 Relation to me': 'myself',
    '🏷️ Tag': 'Other',
    '📃 Content Body': '🎤 Testing, testing. Is this thing on?',
    '🔗 Social Link': 'https://chuangcaleb.com',
    '📸 Avatar Image Link': 'https://www.gravatar.com/avatar/29d863c08e05a20bab30479ecae823eb'
  },
  ...
]
```

You _could_ use `zod` or `yup` some other runtime validation library to enforce that the external data read is type-secure. But:

> That’s over-engineering!

#### guestbook/index.astro

In the frontend render, I do to preprocessing steps:

1. Filter the posts out for `isVerified` to be true (means that I have vetted the post)
2. Reverse the list, so that it’s sorted by latest entry first

```tsx
---
import { getGuestbookPosts } from "lib/google-sheets/sheet";
import NoteLayout from "~/components/layout/NoteLayout/NoteLayout.astro";
import Post from "./_GuestPost.astro";

const title = "Guestbook";

const guestbookPosts = (await getGuestbookPosts())
  .filter((post) => post.isVerified)
  .reverse();
---

<NoteLayout title={title}>
  <h1>Guestbook</h1>
  <p>A collection of post-it notes, signed by people who know me!</p>
  {guestbookPosts.map((post) => <Post post={post} />)}
</NoteLayout>
```

In the `<Post />` component, you will have to parse the `Timestamp` date string into a Javascript `Date`, since it doesn’t do that automatically. Make sure to watch out for `dd/MM/yyyy` vs `MM/dd/yyyy` format!

### Security Considerations

The only sensitive data required is the email address, which could be one’s personal address. These are the procedures I take to not expose it:

- I will keep the Google Sheet fully private, no one can view or edit, even with the link (except for the Service Account, see below)
- The Google Form's responses are also not shared with any collaborators. Just me
- Only my Google account has access. I practice basic account security measures, like password manager and 2FA.
- I use Cloudflare Pages for building and deploying my website source code
- I use a reliable npm package to grab the Sheets data (see the source)
- I use the [Service Account](https://developers.google.com/identity/protocols/oauth2/service-account) (recommended) method of authenticating reads from the Google Sheet
- I don’t expose ANY identifier for the Sheets. Everything traceable is stored in environment variables instead of being hard-coded:
	- Google service account email
	- Google private key (of course)
	- The ID of the Google Sheet itself!
- And then only expose the relevant fields in rendering the site contents
	- Technicallyyyy, the the full form data object is exposed to the Cloudflare build runner, but it’s all server-side pre-rendered and never client-side, and Cloudflare is industry-standard reliable, so this is ok
- The site's source code is made open-source for inspection

## Reflection

I’m actually quite proud of how it turned out. How much simpler everything was, compared to how I initially thought it would take. I’ve really learnt in practice, that **there’s elegance in simplicity.**

See the latest source code at [github.com/chuangcaleb/chuangcaleb.com](https://github.com/chuangcaleb/chuangcaleb.com).

If you got this far, and you know me personally, please contact me and I’ll send you the link to the form! I’d like for you to leave me a message! I won’t modify whatever you write, just tell a joke or share a funny story!
