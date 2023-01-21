---
layout: post
title: Query Signal Desktop messages locally from SQLite using JavaScript
tags: [javascript,signal]
comments: false
readtime: true
slug: query-signal-desktop-messages-sqlite
---

In this short article, I want to show you how to query the Signal Desktop messages locally using JavaScript. It can be helpful in case you want to build some application or script that, for example, exports or backups Signal messages.

The Signal Desktop app stores all messages locally in an *encrypted* SQLite database. The key to decrypting the database is stored locally as part of the app's configuration files. The location of the database and a key depends on your operation system:

- For Linux, `<home>/.config/Signal`;
- For MacOS, `<home>/Library/Application Support/Signal`;
- For Windows, `<home>/AppData/Roaming/Signal`.

To decrypt the Signal database, you must install an [sqlcipher](https://github.com/sqlcipher/sqlcipher). This takes time and can cause errors. I wanted to find a more straightforward way. I decided to check how the [Signal-Desktop](https://github.com/signalapp/Signal-Desktop) app, written in Electron, handles a database connection.

For the desktop app, the Signal team made [a fork of better-sqlite3](https://github.com/signalapp/better-sqlite3) package and used it inside their desktop application. I assume in the fork, they added sqlcipher by default which simplifies a few steps for us. The forked package is available via `npm/yarn`.

{: .box-note}
If you are curious how Signal Desktop handles the database in code, check [this function](https://github.com/signalapp/Signal-Desktop/blob/2a4166a8360e02e01f343723a65de6f7cb748701/ts/sql/Server.ts#L442) that opens and decrypts the SQLite database.

First, let's install the Signal's fork of better-sqlite3 package:

{: .box-warning}
On macOS using NodeJS v17, installation failed due to some OpenSSL error. I suggest using the NodeJS v18.

```bash
npm install @signalapp/better-sqlite3 
```

Second, let's create a file `signal-db.js`, and write a script that will decrypt the database and execute a few queires:

```javascript
const os = require('os');
const fs = require('fs');
const path = require('path');
const SQL = require('@signalapp/better-sqlite3');
  

// this function works for MacOS, for other OS check paths in the list above
function getFolderPath() {
    return path.join(os.homedir(), 'Library/Application Support/Signal');
}


function getDBPath() {
	return path.join(getFolderPath(), 'sql/db.sqlite');
}


function getDBKey() {
	const config = path.join(getFolderPath(), 'config.json');
	return JSON.parse(fs.readFileSync(config).toString())['key'];
}


// read only, to make sure we will not overwrite anything accidentally
const db = SQL(getDBPath(), { readonly: true });

// decrypt the database using a key
db.pragma(`key = "x'${getDBKey()}'"`);

// list all tables in the database
let stm = db.prepare(`SELECT name FROM sqlite_schema WHERE type="table"`);
console.log(stm.all());

// query UUIDs of all active private conversations in Signal
stm = db.prepare(`SELECT id FROM conversations WHERE type="private" AND active_at IS NOT NULL AND name IS NOT NULL ORDER BY active_at DESC`);
console.log(stm.all());

```

Now, we can run the script:

```bash
node signal-db.js
```

The output should be something like this:

```bash
[
  { name: 'sqlite_stat1' },
  { name: 'sqlite_stat4' },
  { name: 'conversations' },
  { name: 'identityKeys' },
  { name: 'items' },
  { name: 'sessions' },
  { name: 'attachment_downloads' },
  { name: 'sticker_packs' },
  { name: 'stickers' },
  { name: 'sticker_references' },
  { name: 'emojis' },
  { name: 'messages' },
  { name: 'messages_fts' },
  { name: 'messages_fts_data' },
  { name: 'messages_fts_idx' },
  { name: 'messages_fts_content' },
  { name: 'messages_fts_docsize' },
  { name: 'messages_fts_config' },
  { name: 'jobs' },
  { name: 'reactions' },
  { name: 'senderKeys' },
  { name: 'unprocessed' },
  { name: 'sendLogPayloads' },
  { name: 'sendLogRecipients' },
  { name: 'sendLogMessageIds' },
  { name: 'preKeys' },
  { name: 'signedPreKeys' },
  { name: 'badges' },
  { name: 'badgeImageFiles' },
  { name: 'storyReads' },
  { name: 'storyDistributions' },
  { name: 'storyDistributionMembers' },
  { name: 'uninstalled_sticker_packs' },
  { name: 'groupCallRingCancellations' }
]
[
  { id: 'uuid1' },
  { id: 'uuid2' },
  { id: 'uuid3' },
  { id: 'uuid4' },
  { id: 'uuid5' },
  { id: 'uuid6' },
  { id: 'uuid7' },
  { id: 'uuid8' },
  { id: 'uuid9' }
]
```

The tables you will probably be the most interested in are `messages` and `conversations`.

I hope this article was helpful whatever you want to export Signal messages using a custom script, do some advanced search, etc. Thank you, and have a nice day!
