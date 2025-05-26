# Linking Curl to Apple LibreSSL

TLDR: this shouldn't be so complicated.


## Curl TLS backends on MacOS

MacOS includes a hacked version of LibreSSL, which [by default sets the CA trust store to the MacOS keychain](https://daniel.haxx.se/blog/2024/03/08/the-apple-curl-security-incident-12604/) instead of the usual `/etc/ssl/cert.pem` file.
Sadly the source code for these LibreSSL modifications is not upstreamed or published on [apple-oss-distributions](https://github.com/apple-oss-distributions/), so this is all we know:


```sh
/usr/bin/openssl version -a
# LibreSSL 3.3.6

curl --version
# curl 8.7.1 (x86_64-apple-darwin24.0) libcurl/8.7.1 (SecureTransport) LibreSSL/3.3.6 zlib/1.2.12 nghttp2/1.64.0
```

Afaik, the sole use for this modded Apple LibreSSL is as TLS backend for curl. Apple does this because the `certs.pem` bundle is outdated on many systems, as it cannot be auto-updated once the user has touched it, leading to [connection problems](https://github.com/curl/curl/issues/17432) and security liabilities with outdated certs.

An alternative to LibreSSL is to use curl with the MacOS "Secure-Transport" TLS backend, which also gets the CA from the keychain trust store. When enabled, this can be flipped on usin the `CURL_SSL_BACKEND` environment variable:

```sh
# Switches active TLS backend
CURL_SSL_BACKEND="Secure-Transport" curl --version
# curl 8.7.1 (x86_64-apple-darwin24.0) libcurl/8.7.1 SecureTransport (LibreSSL/3.3.6) zlib/1.2.12 nghttp2/1.64.0
```

However sadly Secure-Transport is deprecated as it does not even support TLS-1.3 and [is about to be removed from curl](https://daniel.haxx.se/blog/2025/01/14/secure-transport-support-in-curl-is-on-its-way-out/), so this will soon no longer be an option.


## Curl on MacOS is broken

The obscure LibreSSL situation would be less of a problem if Apple did not also do a really poor job with their latest libcurl update. As maintainer of libcurl bindings, I get free notices when Apple messes up an update because we see an instant peak in bug reports from MacOS users facing broken or crashing applications.

All currently supported fully-updated MacOS versions (12,13,14,15) ship with a build of libcurl 8.7.1 which has several very unfortunate bugs. For example it borks when a server uses deflate encoding:

```sh
curl --compressed https://httpbin.org/deflate
# ...
# curl: (23) Failed writing received data to disk/application
```

Or it can crash when downloading an ftp and http file simultaneously:


```sh
URL1="https://en.wikipedia.org/wiki/Bioconductor"
URL2="ftp://ftp.ensembl.org/pub/release-71/gtf/caenorhabditis_elegans/Caenorhabditis_elegans.WBcel235.71.gtf.gz"
curl --keepalive-time 60 -OL $URL1 -OL $URL2 -OL $URL2
# segfault here
```

These bugs were fixed upstream in libcurl 8.8.0 (over a year ago), and most linux distros had updated quickly, but as of today Apple is still on their [buggy curl 8.7.1 build](https://github.com/apple-oss-distributions/curl). Somewhat frustrated by bug reports from users, I tried to escalate this to Apple Security, but they did not care, so the libcurl included with MacOS is unusable for the time being.

## Linking against Apple LibreSSL

Because the MacOS stock version of libcurl is unusable, we want to build our own. But to make it work with the Apple keychain trust store, we need to build it against this modified Apple LibreSSL. Turns out Apple has made this also very complicated by not including LibreSSL in the SDK.

In order to link stuff against Apple LibreSSL, we first need to get compatible headers and a linkable library. Let's first lookup the version and shared libs of LibreSSL on MacOS:

```sh
/usr/bin/openssl version
# LibreSSL 3.3.6

otool -L /usr/bin/openssl
# /usr/bin/openssl:
# 	/usr/lib/libssl.48.dylib (compatibility version 49.0.0, current version 49.2.0)
# 	/usr/lib/libcrypto.46.dylib (compatibility version 47.0.0, current version 47.2.0)
```

So we can download and build an actual copy of LibreSSL 3.3.6. But we can also use this total hack to install an old homebrew build of LibreSSL that has the things we need:

```sh
# dont do this at home
curl -OL https://raw.githubusercontent.com/Homebrew/homebrew-core/b60f7a1249aac36e67dc68aec08b8a94195344bf/Formula/libressl.rb
brew install ./libressl.rb
```

Now we can now copy the LibreSSL headers from `/opt/homebrew/opt/libressl/include`. However linking is more complicated.

These days it is no longer possible on MacOS to link to system libraries directly. In fact the library files `/usr/lib/libssl.48.dylib` and `/usr/lib/libcrypto.46.dylib` that we want to link against do not really exist on disk. As of MacOS 11, such libraries get loaded from a "shared cache".

In order to link against system libraries from this "shared cache", Apple created special  `.tbd` (text-based stub) format. When you link an appliction with `-lfoobar`, the linker will look for `libfoobar.tbd` in addition to the usual `libfoobar.a` and `libfoobar.dylib` files. So we can only link applications against libraries in `/usr/lib` if the xcode SDK includes a corresponding `.tbd` file:

```sh
# Here they are! The linkable libraries!
ls -ltr $(xcrun --show-sdk-path)/usr/lib 
```

Among other things, this text based stub contains the (fake) install-name path of the dylib from the shared cache, for example for libcurl it looks like this:

```sh
cd $(xcrun --show-sdk-path)/usr/lib
head -n5 libcurl.tbd
# --- !tapi-tbd
# tbd-version:     4
# targets:         [ x86_64-macos, x86_64-maccatalyst, arm64-macos, arm64-maccatalyst,
#                    arm64e-macos, arm64e-maccatalyst ]
# install-name:    '/usr/lib/libcurl.4.dylib'
```

However, you guessed it, the SDK does not include `libssl.tbd` and `libcrypto.tbd` files we need to link to LibreSSL. Fortunately we can produce these `tbd` files ourselves from our own homebrew LibreSSL files using the xcode `tapi` tool.

```sh
cd /opt/homebrew/opt/libressl/lib
/Library/Developer/CommandLineTools/usr/bin/tapi stubify --filetype=tbd-v4 libcrypto.dylib
/Library/Developer/CommandLineTools/usr/bin/tapi stubify --filetype=tbd-v4 libssl.dylib
```

This generates the `libssl.48.tbd` and `libcrypto.46.tbd` files containing the metadata for LibreSSL 3.3.5 shared libraries. Now open these files in a text editor and replace the `install-name` field to the native MacOS paths `/usr/lib/libssl.48.dylib` and `/usr/lib/libcrypto.46.dylib`. You can also replace the `targets` header to match what you see for e.g. `libcurl.tbd`, because libs under `/usr/lib` are actually multi-arch libs.


## Building libcurl



