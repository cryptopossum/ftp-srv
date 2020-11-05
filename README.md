<p align="center">
  <a href="https://github.com/autovance/ftp-srv">
    <img alt="ftp-srv" src="logo.png" width="600px"  />
  </a>
</p>

<p align="center">
  Modern, extensible FTP Server
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/ftp-srv">
    <img alt="npm" src="https://img.shields.io/npm/dm/ftp-srv.svg?style=for-the-badge" />
  </a>

  <a href="https://circleci.com/gh/autovance/workflows/ftp-srv/tree/master">
    <img alt="circleci" src="https://img.shields.io/circleci/project/github/autovance/ftp-srv/master.svg?style=for-the-badge" />
  </a>
</p>

---

- [Overview](#overview)
- [Features](#features)
- [Install](#install)
- [Usage](#usage)
  - [API](#api)
  - [CLI](#cli)
  - [Events](#events)
  - [Supported Commands](#supported-commands)
  - [File System](#file-system)
- [Contributing](#contributing)
- [License](#license)

## Overview

`ftp-srv` is a modern and extensible FTP server designed to be simple yet configurable.

## Features

- Extensible [file systems](#file-system) per connection
- Passive and active transfers
- [Explicit](https://en.wikipedia.org/wiki/FTPS#Explicit) & [Implicit](https://en.wikipedia.org/wiki/FTPS#Implicit) TLS connections
- Promise based API

## Install

`npm install ftp-srv --save`

## Usage

```js
// Quick start

const FtpSrv = require('ftp-srv');
const ftpServer = new FtpSrv({ options ... });

ftpServer.on('login', (data, resolve, reject) => { ... });
...

ftpServer.listen()
.then(() => { ... });
```

## API

### `new FtpSrv({options})`

#### url

[URL string](https://nodejs.org/api/url.html#url_url_strings_and_url_objects) indicating the protocol, hostname, and port to listen on for connections.
Supported protocols:

- `ftp` Plain FTP
- `ftps` Implicit FTP over TLS

_Note:_ The hostname must be the external IP address to accept external connections. `0.0.0.0` will listen on any available hosts for server and passive connections.  
**Default:** `"ftp://127.0.0.1:21"`

#### `pasv_url`

The hostname to provide a client when attempting a passive connection (`PASV`).  
If not provided, clients can only connect using an `Active` connection.

#### `pasv_min`

Tne starting port to accept passive connections.  
**Default:** `1024`

#### `pasv_max`

The ending port to accept passive connections.  
The range is then queried for an available port to use when required.  
**Default:** `65535`

#### `greeting`

A human readable array of lines or string to send when a client connects.  
**Default:** `null`

#### `tls`

Node [TLS secure context object](https://nodejs.org/api/tls.html#tls_tls_createsecurecontext_options) used for implicit (`ftps` protocol) or explicit (`AUTH TLS`) connections.  
**Default:** `false`

#### `anonymous`

If true, will allow clients to authenticate using the username `anonymous`, not requiring a password from the user.  
Can also set as a string which allows users to authenticate using the username provided.  
The `login` event is then sent with the provided username and `@anonymous` as the password.  
**Default:** `false`

#### `blacklist`

Array of commands that are not allowed.  
Response code `502` is sent to clients sending one of these commands.  
**Example:** `['RMD', 'RNFR', 'RNTO']` will not allow users to delete directories or rename any files.  
**Default:** `[]`

#### `whitelist`

Array of commands that are only allowed.  
Response code `502` is sent to clients sending any other command.  
**Default:** `[]`

#### `file_format`

Sets the format to use for file stat queries such as `LIST`.  
**Default:** `"ls"`  
**Allowable values:**

- `ls` [bin/ls format](https://cr.yp.to/ftp/list/binls.html)
- `ep` [Easily Parsed LIST format](https://cr.yp.to/ftp/list/eplf.html)
- `function () {}` A custom function returning a format or promise for one.
  - Only one argument is passed in: a node [file stat](https://nodejs.org/api/fs.html#fs_class_fs_stats) object with additional file `name` parameter

#### `log`

A [bunyan logger](https://github.com/trentm/node-bunyan) instance. Created by default.

#### `timeout`

Sets the timeout (in ms) after that an idle connection is closed by the server  
**Default:** `0`

## CLI

`ftp-srv` also comes with a builtin CLI.

```bash
$ ftp-srv [url] [options]
```

```bash
$ ftp-srv ftp://0.0.0.0:9876 --root ~/Documents
```

#### `url`

Set the listening URL.

Defaults to `ftp://127.0.0.1:21`

#### `--pasv_url`

The hostname to provide a client when attempting a passive connection (`PASV`).  
If not provided, clients can only connect using an `Active` connection.

#### `--pasv_min`

The starting port to accept passive connections.  
**Default:** `1024`

#### `--pasv_max`

The ending port to accept passive connections.  
The range is then queried for an available port to use when required.  
**Default:** `65535`

#### `--root` / `-r`

Set the default root directory for users.

Defaults to the current directory.

#### `--credentials` / `-c`

Set the path to a json credentials file.

Format:

```js
[
  {
    "username": "...",
    "password": "...",
    "root": "..." // Root directory
  },
  ...
]
```

#### `--username`

Set the username for the only user. Do not provide an argument to allow anonymous login.

#### `--password`

Set the password for the given `username`.

#### `--read-only`

Disable write actions such as upload, delete, etc.

## Events

The `FtpSrv` class extends the [node net.Server](https://nodejs.org/api/net.html#net_class_net_server). Some custom events can be resolved or rejected, such as `login`.

### `login`

```js
ftpServer.on('login', ({connection, username, password}, resolve, reject) => { ... });
```

Occurs when a client is attempting to login. Here you can resolve the login request by username and password.

`connection` [client class object](src/connection.js)  
`username` string of username from `USER` command  
`password` string of password from `PASS` command  
`resolve` takes an object of arguments:

- `fs`
  - Set a custom file system class for this connection to use.
  - See [File System](#file-system) for implementation details.
- `root`
  - If `fs` is not provided, this will set the root directory for the connection.
  - The user cannot traverse lower than this directory.
- `cwd`
  - If `fs` is not provided, will set the starting directory for the connection
  - This is relative to the `root` directory.
- `blacklist`
  - Commands that are forbidden for only this connection
- `whitelist`
  - If set, this connection will only be able to use the provided commands

`reject` takes an error object

### `client-error`

```js
ftpServer.on('client-error', ({connection, context, error}) => { ... });
```

Occurs when an error arises in the client connection.

`connection` [client class object](src/connection.js)  
`context` string of where the error occurred  
`error` error object

### `RETR`

```js
connection.on('RETR', (error, filePath) => { ... });
```

Occurs when a file is downloaded.

`error` if successful, will be `null`  
`filePath` location to which file was downloaded

### `STOR`

```js
connection.on('STOR', (error, fileName) => { ... });
```

Occurs when a file is uploaded.

`error` if successful, will be `null`  
`fileName` name of the file that was uploaded

### `RNTO`

```js
connection.on('RNTO', (error, fileName) => { ... });
```

Occurs when a file is renamed.

`error` if successful, will be `null`  
`fileName` name of the file that was renamed

## Supported Commands

See the [command registry](src/commands/registration) for a list of all implemented FTP commands.

## File System

The default [file system](src/fs.js) can be overwritten to use your own implementation.  
This can allow for virtual file systems, and more.  
Each connection can set it's own file system based on the user.

The default file system is exported and can be extended as needed:

```js
const {FtpSrv, FileSystem} = require('ftp-srv');

class MyFileSystem extends FileSystem {
  constructor() {
    super(...arguments);
  }

  get(fileName) {
    ...
  }
}
```

Custom file systems can implement the following variables depending on the developers needs:

### Methods

#### [`currentDirectory()`](src/fs.js#L40)

Returns a string of the current working directory  
**Used in:** `PWD`

#### [`get(fileName)`](src/fs.js#L44)

Returns a file stat object of file or directory  
**Used in:** `LIST`, `NLST`, `STAT`, `SIZE`, `RNFR`, `MDTM`

#### [`list(path)`](src/fs.js#L50)

Returns array of file and directory stat objects  
**Used in:** `LIST`, `NLST`, `STAT`

#### [`chdir(path)`](src/fs.js#L67)

Returns new directory relative to current directory  
**Used in:** `CWD`, `CDUP`

#### [`mkdir(path)`](src/fs.js#L114)

Returns a path to a newly created directory  
**Used in:** `MKD`

#### [`write(fileName, {append, start})`](src/fs.js#L79)

Returns a writable stream  
Options:  
 `append` if true, append to existing file  
 `start` if set, specifies the byte offset to write to  
**Used in:** `STOR`, `APPE`

#### [`read(fileName, {start})`](src/fs.js#L90)

Returns a readable stream  
Options:  
 `start` if set, specifies the byte offset to read from  
**Used in:** `RETR`

#### [`delete(path)`](src/fs.js#L105)

Delete a file or directory  
**Used in:** `DELE`

#### [`rename(from, to)`](src/fs.js#L120)

Renames a file or directory  
**Used in:** `RNFR`, `RNTO`

#### [`chmod(path)`](src/fs.js#L126)

Modifies a file or directory's permissions  
**Used in:** `SITE CHMOD`

#### [`getUniqueName()`](src/fs.js#L131)

Returns a unique file name to write to  
**Used in:** `STOU`

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Contributors

- [OzairP](https://github.com/OzairP)
- [TimLuq](https://github.com/TimLuq)
- [crabl](https://github.com/crabl)
- [hirviid](https://github.com/hirviid)
- [DiegoRBaquero](https://github.com/DiegoRBaquero)
- [edin-m](https://github.com/edin-m)
- [voxsoftware](https://github.com/voxsoftware)
- [jorinvo](https://github.com/jorinvo)
- [Johnnyrook777](https://github.com/Johnnyrook777)
- [qchar](https://github.com/qchar)
- [mikejestes](https://github.com/mikejestes)
- [pkeuter](https://github.com/pkeuter)
- [qiansc](https://github.com/qiansc)
- [broofa](https://github.com/broofa)
- [lafin](https://github.com/lafin)
- [alancnet](https://github.com/alancnet)
- [zgwit](https://github.com/zgwit)

## License

This software is licensed under the MIT Licence. See [LICENSE](LICENSE).

## References

- [https://cr.yp.to/ftp.html](https://cr.yp.to/ftp.html)

## TODO

Support this:

```
220 FTP Server ready.

USER
 test

331 Username ok, send password.

PASS
 pass

230 Login successful.

TYPE
 I

200 Type set to: Binary.

PASV


227 Entering passive mode (172,20,0,65,212,110).

CWD
 /.

250 "/" is the current directory.

DELE
 diag-EFLX_023-20200818T154530530.aes

550 No such file or directory.

STOR
 diag-EFLX_023-20200818T154530530.aes

125 Data connection already open. Transfer starting.

QUIT
226 Transfer complete.



221 Goodbye.
```

```
00000000: 3031 302e 3034 302e 3030 322e 3234 352e  010.040.002.245.
00000010: 3032 3632 312d 3031 302e 3134 322e 3030  02621-010.142.00
00000020: 302e 3031 312e 3532 3134 373a 2032 3230  0.011.52147: 220
00000030: 2052 6561 6479 0d0a 0a30 3130 2e31 3432   Ready...010.142
00000040: 2e30 3030 2e30 3131 2e35 3231 3437 2d30  .000.011.52147-0
00000050: 3130 2e30 3430 2e30 3032 2e32 3435 2e30  10.040.002.245.0
00000060: 3236 3231 3a20 5553 4552 204e 4c45 464c  2621: USER NLEFL
00000070: 5a41 5054 4543 5445 5354 3457 5353 0d0a  ZAPTECTEST4WSS..
00000080: 0a30 3130 2e30 3430 2e30 3032 2e32 3435  .010.040.002.245
00000090: 2e30 3236 3231 2d30 3130 2e31 3432 2e30  .02621-010.142.0
000000a0: 3030 2e30 3131 2e35 3231 3437 3a20 3333  00.011.52147: 33
000000b0: 3120 5573 6572 6e61 6d65 206f 6b61 792c  1 Username okay,
000000c0: 2061 7761 6974 696e 6720 7061 7373 776f   awaiting passwo
000000d0: 7264 0d0a 0a30 3130 2e31 3432 2e30 3030  rd...010.142.000
000000e0: 2e30 3131 2e35 3231 3437 2d30 3130 2e30  .011.52147-010.0
000000f0: 3430 2e30 3032 2e32 3435 2e30 3236 3231  40.002.245.02621
00000100: 3a20 5041 5353 2066 3536 3636 3162 3630  : PASS f56661b60
```

```
00000000: 3232 3020 4654 5020 5365 7276 6572 2072  220 FTP Server r
00000010: 6561 6479 2e0d 0a0a 5553 4552 0a20 7465  eady....USER. te
00000020: 7374 0d0a 0a33 3331 2055 7365 726e 616d  st...331 Usernam
00000030: 6520 6f6b 2c20 7365 6e64 2070 6173 7377  e ok, send passw
00000040: 6f72 642e 0d0a 0a50 4153 530a 2070 6173  ord....PASS. pas
00000050: 730d 0a0a 3233 3020 4c6f 6769 6e20 7375  s...230 Login su
00000060: 6363 6573 7366 756c 2e0d 0a0a 5459 5045  ccessful....TYPE
00000070: 0a20 490d 0a0a 3230 3020 5479 7065 2073  . I...200 Type s
00000080: 6574 2074 6f3a 2042 696e 6172 792e 0d0a  et to: Binary...
00000090: 0a50 4153 560a 0d0a 0a32 3237 2045 6e74  .PASV....227 Ent
000000a0: 6572 696e 6720 7061 7373 6976 6520 6d6f  ering passive mo
000000b0: 6465 2028 3137 322c 3230 2c30 2c36 352c  de (172,20,0,65,
000000c0: 3231 342c 3637 292e 0d0a 0a43 5744 0a20  214,67)....CWD.
000000d0: 2f2e 0d0a 0a32 3530 2022 2f22 2069 7320  /....250 "/" is
000000e0: 7468 6520 6375 7272 656e 7420 6469 7265  the current dire
000000f0: 6374 6f72 792e 0d0a 0a44 454c 450a 2064  ctory....DELE. d
00000100: 6961 672d 4546 4c58 5f30 3233 2d32 3032  iag-EFLX_023-202
00000110: 3030 3831 3854 3135 3534 3434 3038 342e  00818T155444084.
00000120: 6165 730d 0a0a 3535 3020 4e6f 2073 7563  aes...550 No suc
00000130: 6820 6669 6c65 206f 7220 6469 7265 6374  h file or direct
00000140: 6f72 792e 0d0a 0a53 544f 520a 2064 6961  ory....STOR. dia
00000150: 672d 4546 4c58 5f30 3233 2d32 3032 3030  g-EFLX_023-20200
00000160: 3831 3854 3135 3534 3434 3038 342e 6165  818T155444084.ae
00000170: 730d 0a0a 3132 3520 4461 7461 2063 6f6e  s...125 Data con
00000180: 6e65 6374 696f 6e20 616c 7265 6164 7920  nection already
00000190: 6f70 656e 2e20 5472 616e 7366 6572 2073  open. Transfer s
000001a0: 7461 7274 696e 672e 0d0a 0a31 3235 2044  tarting....125 D
000001b0: 6174 6120 636f 6e6e 6563 7469 6f6e 2061  ata connection a
000001c0: 6c72 6561 6479 206f 7065 6e2e 2054 7261  lready open. Tra
000001d0: 6e73 6665 7220 7374 6172 7469 6e67 2e0d  nsfer starting..
```
