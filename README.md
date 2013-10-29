# .authrc

Centralized authentication configuration and storage for network-based resources

`Draft version, work still in process`

[![No more passwords!](http://img191.imageshack.us/img191/466/v6wa.png)](https://github.com/adesisnetlife/authrc)

## About

`.authrc` aims to be a standard community well supported which provides a generic and centralized configuration file for authentication credentials management and storage that can be used by applications or services for network-based resources

## Stage

Current version:  0.1-beta

Note that the specification is currently under active designing, which means that notorious changes can be done during the process

Please take the above into account if you want to make your own implementation based on the standard

## Goals

- Easy to configure
- Centralized
- Local installed
- Secure (encrypted passwords with solid symmetric ciphers)
- Full URI/URL/URN supporting any network resource type
- Applications can support it (transparent to the end user)
- Platform independent

## Specification

### Configuration file

#### File name

The file must always be named `.authrc`

#### File content format

The file must be a well formed [JSON][4] object.

There is not plan to support another format. Read the [FAQ](#faq) for more information

#### File location

The general standard pretension is to have just a unique `.authrc` file for the whole system, 
however in some scenarios this is not the best choice or is simply impossible.

By default, the `.authrc` file should be located at `$HOME` or `%USERPROFILE%` path directory, 
but you can store it wherever you want.

Take into account that if you are using a non-global located file for specific purposes, 
all stored `hosts` values will override the global `.authrc` config.

`TODO`

#### File discovery algorithm

The `.authrc` file search discovery algorithm should do what follows:

```
Try to find .authrc file on the current working directory
  If it exists, read and parse it
  If it doesn’t exist, fallback to $HOME
Try to find .authrc file in $HOME directory
  If it exists, read and parse it
  If it doesn’t exist, finish the process
```

#### File encoding

The file must be read and written using [UTF-8][3] encoding

#### File data schema 

The file must be a well formed [JSON][4] object containing a set of 
properties which implement the `Host` config object.

Here an basic file example

```json
{
    "my.host.org": { ... },
    "my.server.org": { ... },
    "10.0.0.1": { ... }
}
```

##### Host value

The `host` value can be any `string`, that means any type of URI, URL or URN 
must be supported, with full a partial resource path.

Here is some examples of possible `hosts` values

```
http://my.server.org
my.server.org:8443
my.server.org/some/resource
amazing.host
10.0.0.1
https://172.16.0.1:8080
ftp://ftp.server.org
git://my.repo.org/path/to/repo.git
file://home/user/server
urn:issn:3613
```

##### Host naming considerations

In some scenarios you need to have different authentication credentials for the same 
`hostname`, for example if there is a couple of services running in different ports.

In order to prevent incorrect use of authentication credentials in incorrect network 
resources, it is recommended to define the `host` value in the more explicit way as 
possible in order to prevent issues when the `host` matching algorithm tries to discover 
the appropriate credentials in your `authrc` file.

Supossing you have some service to downloads resources via HTTP and a Git repository 
running on `my.server.org` and both are accesible via HTTP in the same hostname 
and the same TCP port but in different path names, your .authrc should looks like:

```json
{
    "my.server.org/downloads": { ... },
    "my.server.org/git": { ... }
}
```

#### Host matching algorithm

The maching algorithm uses both partial comparison of the URI schema and a
string comparison letter by letter according with the proposed algorithm 
[An O(ND) Difference Algorithm][5] by Eugene W. Myers

The following explains by detailed process about how the matching host algorithm must works:

`Note this is a draft version`

```
1.
  Parse the string to match like a valid [URI](http://en.wikipedia.org/wiki/URI_scheme)
  Extract the `hostname`

2.
  Iterate all the existent hosts in `.authrc` file.
  Parse the `host` string value like a valid URI
  Extract the `hostname`

3.
  Performs a string comparison with both hostnames values (possible regex support?)
  Filter by the matched hostname and discard the others

4.
  If there is no any `host` matched, exit
  If there is only one `hostname` matched, use it and exit
  If there are more than one matched `hosts`, continue the process

5. 
  Performs a string comparison between the URI port
    If there is no present port in both URIs, discard the process 
  Performs a string comparison between the URI protocol
    If there is no present port in both URIs, dicard the process

TODO...
```

#### Host config object

Each first level `host` property must implement one of the following interfaces:

##### Raw password

```
interface HostAuthRaw {
    attribute String username;
    attribute String password;
}
```

Implementation example

```json
{
    "my.server.org": {
        "username": "john",
        "password": "$sup3r-p@sSw0rd"
    }
}
```

##### Encrypted password

```
interface HostAuthEncrypted {
    attribute String username;
    attribute HostConfigPassword password;
}
```
The password property must implement at least one of the following interfaces

Encrypted password with explicit cipher algorithm

```
interface HostPasswordCipher {
    attribute String value;
    attribute String cipher;
}
```

Encrypted password with default cipher algorithm

```
interface HostPasswordEncrypted {
    attribute String value;
    attribute Boolean encrypted;
}
```

Encrypted password with environment variable value store

```
interface HostPasswordEncryptedEnviroment {
    attribute String envValue;
    attribute String cipher;
}
```

Encrypted password with environment variable decryption key value store

```
interface HostPasswordEncryptedEnviromentKey {
    attribute String value;
    attribute String cipher;
    attribute String envKey;
}
```

Implementations examples

```json
{
    "my.server.org": {
        "username": "john",
        "password": {
            "value": "41b717a64c6b5753ed5928fd8a53149a7632e4ed1d207c91",
            "cipher": "aes256"
        }
    }
}
```

Environment variable encrypted password store
```json
{
    "my.server.org": {
        "username": "john",
        "password": {
            "envValue": "MY_ENCRYPTED_PASSWORD",
            "cipher": "aes256"
        }
    }
}
```

In this case, it should use the default algorithm `AES128`
```json
{
    "my.server.org": {
        "username": "john",
        "password": {
            "value": "41b717a64c6b5753ed5928fd8a53149a7632e4ed1d207c91",
            "encrypted": true
        }
    }
}
```

See [examples/](https://github.com/adesisnetlife/authrc/tree/master/examples) for full featured example files

### Encryption

#### Supported ciphers algorithms

Only well-tested symmetric ciphers algorithms are supported. 
Choose whatever you prefer :)

- AES 128 (default) 
- AES 256
- Camellia 128
- Camellia 256
- Blowfish
- CAST
- IDEA
- SEED

All the above ciphers algorithms are suppored by [OpenSSL](http://www.openssl.org)

##### Encryption format

All the password encrypted values must me post-encoded as `hexadecimal` based string

##### Block cipher operation mode

The encryption/decryption process for all the ciphers algorithms must be performed using the [CBC][1] operation mode.

There is no plan to support [initialization vectors][2], it just adds an unnecessary level of complexibility

#### Possible security concerns

`TODO`

#### Password security recommendations

`TODO`

## Implementations

- [Node.js](https://github.com/h2non/node-authrc)

## FAQ

- **Why JSON?**

[JSON][4] is a well supported, simple and consistent format for data interchange.
Its estricness about how the format must be defined is good in order to remove inconsiscy with data.

`TODO` 

- **There is a plan to support another file format?**

At the moment is only JSON format supported, however a possible new format support condidate will be `ini`.

- **How my application can use the .authrc file?**

You can use any of current oficial specification implementations or make your own implementation.

Your application can have it

- **There is a command-line support?**

Yes. Most of the different language implementations have CLI support.

See the [implementations](#implementations) section to see which are available.

- **Hosts names can be `regex` expressions?**

Not at the moment, but it's a really good idea.

If you have a specific proposal about how to implementation it properly, open an issue and explain it :)

- **The host matching algorithm is robust?**

Probably is not the best solution to the problem, but it's a temporal good solution.

The current algorithm was well tested with some different URIs types and schemas and its works fine,
however, more test cases is required. Note this stills a draft specification.

- **What about the 160, 192, 224 bits symmetric keys support?**

As you probably already know, in a practise there is not much usual to use different encryptions keys
than 128 bits.

128 bits key length provides a strong level of security, but if you are paranoid you can use 256 bits key length.

## Contributing

We would be very happy if you wanted to contribute improving and expanding this standard

You can contribute in any of the next ways:

- Using and expanding it :)
- Opening an [issue](https://github.com/adesisnetlife/authrc/issues) with your proposals or improvements
- Forking the repository and opening a PR with your proposals or improvements
- Implementing the standard in a language (currenyl waiting for a Ruby, Python, Perl or C/C++ implementations)

## License

Copyright (c) 2013 Adesis Netfile S.L

Specification documentation under the [GNU Free Documentation License](http://www.gnu.org/licenses/fdl-1.3-standalone.html).

```
Copyright (c) 2013 Adesis Netfile S.L

Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.3
or any later version published by the Free Software Foundation;
with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
A copy of the license is included in the section entitled "GNU
Free Documentation License".
```

Code under MIT license.

[1]: http://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher-block_chaining_.28CBC.29
[2]: http://en.wikipedia.org/wiki/Initialization_vector
[3]: http://en.wikipedia.org/wiki/UTF-8
[4]: http://es.wikipedia.org/wiki/JSON
[5]: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.4.6927
