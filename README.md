# .authrc

Centralized authentication configuration and storage for network-based resources

`Draft version, work in process`

[![No more passwords!](http://img191.imageshack.us/img191/466/v6wa.png)](https://github.com/adesisnetlife/authrc) 

## About

`.authrc`provides a generic and centralized configuration file for authentication credentials management and storage, that can be used by any application or service for network-based resources. It aims to be a standard adopted by the community

## Stage

Current version: 0.1

Note that the specification is currently under active designing, which means that notorious changes can be done in a few future.

## Goals

- Simple and easy to configure
- Centralized
- Locally stored
- Secure (encrypted passwords with solid symmetric ciphers)
- Full URI/URL supporting any network resource type
- Resource matching based on regular expressions or string matching
- Any type of application can easily support it
- Platform independent

# Specification

## Table of contents

- [Configuration file](#configuration-file)
  * [Name](#file-name)
  * [Format](#file-format)
  * [Location](#file-location)
  * [Encoding](#file-encoding)
  * [Discovery algorithm](#file-discovery-algorithm)
- [Data schema](#data-schema)
  * [Main schema](#main-schema)
  * [The `host` value](#host-value)
    * [Host regular expression](#host-definition-like-regular-expression)
    * [Host naming examples](#host-naming-examples)
    * [Host naming considerations](#host-naming-considerations)
    * [Host matching algorithm](#host-matching-algorithm)
    * [Host config object](#host-config-object)
      * [Raw password](#raw-password)
      * [Encrypted password](#encrypted-password)
- [Password encryption](#password-encryption)
  * [Supported ciphers algorithms](#supported-ciphers-algorithms)
    * [Encoding](#encoding)
    * [Encryption output format](#encryption-format)
    * [Block cipher operation mode](#block-cipher-operation-mode)
  * [Security](#security)
  * [Security recommendations](#security-recommendations)
- [Implementations](#implementations)
- [FAQ](#faq)
- [Contributing](#Contributing)
- [License](#license)

### Configuration file

#### File name

The file must always be named `.authrc`.

This is important because it removes the ambiguity and helps to discover the file based on a standardized name.

#### File format

The file content must be a well formed [JSON][4]

JSON is a general purpose well known data change information format. 
It is simple, easy to read, edit and parse.

Like `.authrc` it can be edited manually. It's necessary to prevent format ambiguity syntax.
The strict part of the JSON syntax it is ideal in order to prevent problems when the file is parsed and when it is edited manually, maybe like other formats like `YAML` or `ini`.

There is not a plan to support another format. Please, read the [FAQ](#faq) for more information

#### File location

The general standard pretension is to have just a unique `.authrc` file for the whole system, 
however in some scenarios this is not the best choice or it’s simply impossible.

By default, the `.authrc` file should be located at `$HOME` or `%USERPROFILE%` path directory, 
but you can store it wherever you want.

Take into account that if you are using a non-global located file for specific purposes, 
all stored `hosts` values will override the global `.authrc` config.

#### File encoding

The file must be read and written using [UTF-8][3] encoding

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

### Data schema

#### Main schema

The file must be a [JSON][4] object containing a set of 
properties containing the `Host` value and which implement the `Host` config object.

```
{
    "<hostValue>": <hostConfig>,
    "<hostvalue>": <hostConfig>,
    "<hostValue>": <hostConfig>
}
```

Here’s a basic file example

```json
{
    "my.host.org": { ... },
    "my.server.org": { ... },
    "10.0.0.1": { ... }
}
```

##### Host value

The `host` value must be a `string`.
The `string` can contain any type of value, but usually you should define a full or partial URI-like format

Aditionally, a valid [regular expression][6] is supported like the `host` value.

##### Host definition like regular expression

To define a host like regular expression, you need to wrap the `host` value with `/` characters.

The hosts matching must be always case-insensitive.

`regulars expressions` support was added in order to avoid redundancy in your `.authrc`
and to support more complex `host` definitions for particular cases.

You should be careful when it comes to defining a host-like `regular expression`. 
It is recommended that you define as more explicit `regex` as you can in order 
to prevent ambiguity during the host matching process, 
for example avoiding the wild card helper operator `(*.)`

##### Host naming examples

Here there are some valid `host` values examples

```
http://my.server.org
my.server.org:8443
my.server.org/some/resource
amazing.host
10.0.0.1
https://172.16.0.1:8080
ftp://ftp.server.org
git://my.repo.org/path/to/repo.git
/http[s]?://(\w.).server.org/
/[a-z0-9]+.server.org/
```

##### Host naming considerations

In some scenarios you need to have different authentication credentials for the same 
`hostname`, for example if there are a couple of services running in different ports.

In order to prevent incorrect use of authentication credentials for incorrect network 
resources, it is recommended to define the `host` value in the more explicit way as 
possible in order to prevent issues when the `host` matching algorithm tries to discover 
the appropriated credentials in your `authrc` file.

Imagine you have a service to downloads resources via HTTP and a Git repository, 
both running on `my.server.org`, both of them are accesible via HTTP, under the same resolution hostname
and the same TCP port, but the only difference between them is that they are accessible 
under different server path names, so your `.authrc` should look like:

```json
{
    "http://my.server.org/downloads": { ... },
    "http://my.server.org/git": { ... }
}
```

#### Host matching algorithm

`Note this is a draft version, improvement and contributions are welcome`

The matching algorithm uses both partial comparison of individual parts of 
an URI schema and, secondly, a more deep matching based on a string 
comparison letter by letter, according with the proposed algorithm 
[An O(ND) Difference Algorithm][5] by Eugene W. Myers

What follows explains the detailed process of how the matching 
host algorithm must be implemented:

```
0.
  Get all the existent hosts in `.authrc` file (first level object properties)
     If there is no hosts, exit the process

1. 
  Iterate all the existent properties

1.1
  Check if the `host` string value is a regex-like expression (starts and ends with '/')
    If it is a regex expression, validate it
      If it is not valid, discard the host
      If it is valid, continue
    If it is NOT a regex `string` value 
      Parse the string like a valid URI (if it is not a valid URI, assume the existent value is the hostname)
        If it is valid and the value is not empty
           Then extract the `hostname`
        If it is invalid, discard the host

1.2 
  Iterate each valid host, matching with the search string to match
    If it is a regex, test it with the search string to match
      If the regex test succeed
        Set the host as matched
      If the regex test fails
        Discard the host
    If it is NOT a regex
      Parse like a valid URI
        Parse both host value and search string to match like a valid URI
          Extract the port and protocol form the URI in both
            Performs a comparison between port and protocol values
              If at least one value is equal
                Set as matched host
              If there is not any equality
                Discard the host
    Continue with the next host

1.3
  Get the list of the current matched hosts
    If there is not any matched host from 1.2 process
      Get the valid hosts from 1.1 process
        Discard the regex type host values
          Performs a string comparison by letter for each host (using An O(ND) Difference Algorithm)
            Get the host with minor letters differences returned by the algorithm
              Mark it as the found host
    If there is one matched host from 1.2 process
      Mark it as the found host
    If there is more than 1 matched host from 1.2 process
      Discard the regex type host values
        Get the valid host from 1.2 process
          Performs a string comparison letter by letter for each host (using An O(ND) Difference Algorithm)
            Get the host with minor letters differences returned by the algorithm
              Mark it as the found host

2. 
  Finally, return the matched host value

```

#### Host config object

The `host` config object it’s where the authentication credentials will be stored. 
Basically it is an object with two properties: `username` and `password`

Each `host` object must implement one of the following interfaces

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

### Password encryption

Password encryption must be supported by all the implementations.
The encrypted value, as it is defined above, can be stored in `.authrc` at the `password.value` property,
or stored in an environment variable at the `password.envValue` property

#### Supported ciphers algorithms

Only well-tested symmetric ciphers algorithms are supported. 
Choose whatever you prefer :)

- AES 128 (if not defined, must use this by default) 
- AES 256
- Camellia 128
- Camellia 256
- Blowfish
- CAST
- IDEA
- SEED

All the above ciphers algorithms are well supported by [OpenSSL](http://www.openssl.org)

##### Encoding

The encryption/decryption process must be always done using [UTF-8][3] encoding

##### Encryption output format

All the password encrypted values must be post-encoded as `hexadecimal` value string

##### Block cipher operation mode

The encryption/decryption process for all the ciphers algorithms must be performed using the [CBC][1] operation mode.

There is no plan to support [initialization vectors][2], it just adds an unnecessary level of complexibility

#### Security

First of all, if you use `.authrc`, it is assumed that your machine is a secure environment

`.authrc` proposal does not guarantee any type of additional level of security, it all depends
on your personal security prevention and how you store your decryption keys

`.authrc` was created to be practical in the first place while security comes after

If you are paranoic and you want to add an additional level of security,
you can encrypt/decrypt the whole file by your own way.

#### Security recommendations

- Avoid store passwords in plain format, use encryption instead
- For environment variable decryption key, define it by prompt input at each new shell session
- You should be careful about how and where you store your password decryption keys
- Set your `authrc` file permissions to be only readable by you
- Use different ciphers and different keys for shared passwords across different hosts
- You should avoid that your `.authrc` be tracked by any SCM

[Here][8] is more information about password strength and recommendations

## Implementations

- [Node.js](https://github.com/h2non/node-authrc)

Currently looking for Java, Python, Ruby and C/C++ implementations

## FAQ

- **Is it planned to encrypt the whole .authrc file?**

Not at the moment, but the idea is open. However, you can encrypt/decrypt 
the whole file yourself adding the additional desired security level

- **Is it planned to support another file format?**

At the moment only JSON format is supported, however in a future it will support more.

If you have a good one in mind for this proposal, please share it :)

- **How can I do if I have the same credentials for more than one host?**

You should copy the whole credentials.
Maybe `host` alias will be supported in a near future. Proposals for this are welcome. 

- **How my application can use the `.authrc` file?**

You can use any of the current official specification implementations for your programming language or make your own.

- **Is there a command-line support?**

Yes. Most of the implementations should have a full featured CLI support for basic `.authrc` file manipulation

See the [implementations](#implementations) section to see which are available.

- **Can host names values be full `regex` compatible?**

Yes. Just be careful about escape characters or bad formed expressions 
and keep in mind to define your expression as more explicit as possible

You can check you expression [here][7]

- **Is the host matching algorithm robust?**

Probably it is not the best solution to the problem, but it is a first good solution.

The current algorithm was well tested with some different URIs types and schemas and it works as expected,
however, more test cases are required. 

Note that this still a draft specification. 
If you have a proposal to improve it, please feel free to make it

- **What about the 160, 192, 224 bits symmetric keys support?**

As you probably already know, in a practise world there is not very usual to use different encryptions keys than 128 or 256 bits.

128 bits key length provides a strong level of security, but if you are paranoid you can use 256 bits key length.

- **Can I encrypt the user name also?**

No. Really?

##### More questions?

Feel free to [open][9] an issue with your question

## Contributing

We would be very happy if you wanted to contribute improving and expanding this standard

You can contribute in any of the next ways:

- Using and expanding it :)
- Opening an [issue](https://github.com/adesisnetlife/authrc/issues) with your proposals or improvements
- Forking the repository and opening a PR with your proposals or improvements
- Implementing the standard in a language (looking for a Ruby, Python, Perl or C/C++ implementation)

## License

Copyright (c) 2013 Adesis Netlife S.L

Specification documentation under the [GNU Free Documentation License](http://www.gnu.org/licenses/fdl-1.3-standalone.html).

```
Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.3
or any later version published by the Free Software Foundation;
with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
A copy of the license is included in the section entitled "GNU
Free Documentation License".
```

[1]: http://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Cipher-block_chaining_.28CBC.29
[2]: http://en.wikipedia.org/wiki/Initialization_vector
[3]: http://en.wikipedia.org/wiki/UTF-8
[4]: http://es.wikipedia.org/wiki/JSON
[5]: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.4.6927
[6]: http://en.wikipedia.org/wiki/Regular_expression
[7]: https://www.debuggex.com/
[8]: http://en.wikipedia.org/wiki/Password_strength
[9]: https://github.com/adesisnetlife/authrc/issues
