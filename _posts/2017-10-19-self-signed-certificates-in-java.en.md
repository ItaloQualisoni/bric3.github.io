---
layout: post
title: HTTPS in Java with a self-signed certificate
date: 2017-10-19
published: true
tags:
- ssl
- tls
- https
- java
- okhttp
- openssl
- trustmanager
- certificate authority
- self-signed certificate
author: Brice Dutheil
language: en
---

Your application should connect to an HTTPS server, usually that's a no brainer,
however as good software craft-man, you may think how do I test this code to make
nothing has been forgotten.

The following code is using **wiremock**, the test assert that the client can 
connect to the HTTPS port of the wiremock server.

```java
public class WireMockSSLTest {
    @Rule
    public WireMockRule wireMock = new WireMockRule(wireMockConfig().dynamicPort()
                                                                    .dynamicHttpsPort());

    @Test
    public void ssl_poke() throws IOException {
        new OkHttpClient.Builder().build()
                                  .newCall(new Request.Builder().get()
                                                                .url("https://localhost:" 
                                                                     + wireMock.httpsPort())
                                                                .build())
                                  .execute();
    }
}
```

When executed the test will raise an exception with the following stacktrace :

```
javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
	at ...
Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
	at ...
	... 10 more
Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
	at ...
	... 16 more
```

Wow that might look overwhelming at the first sight, there's scary acronyms 
(like `PKIX`), `sun` packages. So what does say this stack trace :

* `SSLHandshakeException` : An error happened while establishing the SSL layer
* `ValidatorException: PKIX path building failed` : the client can't build
    the certificate chain by following the **PKIX** standard

In plain english, this server (wiremock) exposes a certificate, but the client 
can't trust this certificate because it cannot rebuild a _path_ to a known 
**authority** that signed the wiremock certificate. 
Why does this exception happen in this case, the very same code connecting 
to `https://google.com` will succeed ; the issue in this case is that wiremock 
is using a **self-signed certificate**, this certificate is not trusted 
because it hasn't been signed by any of the CAs or public certificates 
available in the JVM standard installation.

In this article, we will look at various ways to generate a **self-signed certificate**
and explore how to control the certificate used in test environment.

## First a brief reminder on ~~SSL~~, I mean TLS ?

Nowadays when people speak about _SSL_ they conflate two protocols : 
_TLS_ and his predecessor _SSL_.

* SSL (**S**ecure **S**ockets **L**ayer) protocol was engineered and developed 
    by **Netscape** (Thought for those who remember this pioneer internet browser 
    then Internet Explorer competitor).
    The last version SSLv3 is considered as deprecated since 2015.
* TLS (**T**ransport **L**ayer **S**ecurity) protocol is the direct successor 
    of the SSLv3 protocol, however they are not compatible.

[[source]](https://en.wikipedia.org/wiki/Transport_Layer_Security)

The role of TLS is to ensure that communication between two peers is secured.
In order to do so, TLS [rely among other things on **PKIX**](https://tools.ietf.org/html/rfc5246#section-7)
([**P**ublic **K**ey **I**nfrastructure **X**.509](https://tools.ietf.org/html/rfc5280)),
equipped with these certificate mechanism a client can verify the server identity 
is actually the one it pretends to be with the help of a trusted third party.

For example : 

> Upon connecting to https://github.com, the client will first verify the GitHub 
> certificate, then inspect who signed this certificate, then verify the 
> signer's certificate and so on, until a trusted third party certificate is reached.
> Usually this root certificate comes from a certificate authority.

The certificates are called **X.509** they carry informations such as 
the server name, the signer name, the signature, etc. This certificate mechanism 
is one approach to implement a PKI (**P**ublic **K**ey **I**nfrastructure), 
hence the acronym **PKIX**.

A certificate can be stored in a file, either encoded in its binary form
known as `DER` (**D**istinguished **E**ncoding **R**ules), or in a US-ASCII 
text form which the Base 64 encoding of the binary form known as `PEM` 
(**P**rivacy-enhanced **E**lectronic **M**ail). 

> Note that a file with the `CRT` extension is a certificate but allow 
> the content to be either binary or plain text (i.e. either `DER` or `PEM`).
> This filename extension is an alternative naming form used by Microsoft.


## Fixing the test code

### Trusting every peer

So to accept the self-signed certificate, it is possible to configure the 
SSL socket to accept all certificates.

```java
@Rule
public WireMockRule wireMock = new WireMockRule(wireMockConfig().dynamicPort()
                                                                .dynamicHttpsPort());

@Test
public void ssl_poke() throws IOException {
    X509TrustManager trustManager = TrustAllX509TrustManager.INSTANCE;
    OkHttpClient client = new OkHttpClient.Builder()
            .sslSocketFactory(
                    sslContext(null,
                                new TrustManager[]{trustManager}).getSocketFactory(),
                    trustManager)
            .build();
    try (Response r = client.newCall(new Request.Builder().get()
                                                          .url("https://localhost:"
                                                               + wireMock.httpsPort())
                                                          .build())
                            .execute()) {
        // noop / success
    }
}
```

OkHttp client can be configured with a SSL socket factory that is itself configured
with a custom trust manager. This is the job of the trust manager to validate
or not a certificate. This one is coed to accept all certificate.

Since the security API is a bit tedious to read and use the code has been 
abstracted away in methods that are easier to read and comment.

```java
public static SSLContext sslContext(KeyManager[] keyManagers, TrustManager[] trustManagers) {
    try {
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(keyManagers,
                        trustManagers,
                        null);
        return sslContext;
    } catch (NoSuchAlgorithmException | KeyManagementException e) {
        throw new IllegalStateException("Couldn't init TLS context", e);
    }
}
```

The cryptography code architecture is very very generic and some design 
choices raise questions on its usage.

For example why the method `SSLContext#init` require 
arrays ? While the javadoc of this method [`SSLContext#init`](https://docs.oracle.com/javase/8/docs/api/javax/net/ssl/SSLContext.html#init-javax.net.ssl.KeyManager:A-javax.net.ssl.TrustManager:A-java.security.SecureRandom-) indicate that only the first element of the array will be used.
This also explains why the code above initialize the context with a single 
trust manager.

_Emphasizes in the javadoc are mine._

> ```
> javax.net.ssl.SSLContext
> public final void init(KeyManager[] km,
>                       TrustManager[] tm,
>                       SecureRandom random)
>                throws KeyManagementException
>
> Initializes this context. Either of the first two parameters may be null in which 
> case the installed security providers will be searched for the highest priority 
> implementation of the appropriate factory. Likewise, the secure random parameter 
> may be null in which case the default implementation will be used.
>
> **Only the first instance** of a particular key and/or trust manager implementation 
> type in the array is used. (For example, only the first javax.net.ssl.X509KeyManager 
> in the array will be used.)
>
> Parameters:
>     km - the sources of authentication keys or null
>     tm - the sources of peer authentication trust decisions or null
>     random - the source of randomness for this generator or null
> Throws:
>    KeyManagementException - if this operation fails
> ```

Another thing is that the `TrustManager` interface is empty, that's right there is 
no methods, it's a marker interface. Given that it is about TLS and the only
active specification is based on PKIX (X.509), the code will have to implement
`X509TrustManager` interface (itself a subtype of `TrsutManager`). This interface
declares the required method to implement certificate validation.

In this section this trust manager will accept all certificates, in fact it will
just do nothing.

```java
public static class TrustAllX509TrustManager implements X509TrustManager {
    public static final X509TrustManager INSTANCE = new TrustAllX509TrustManager();

    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) 
    throws CertificateException { }

    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType) 
    throws CertificateException { }

    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return new X509Certificate[0];
    }
}
```

So this code is supposedly working, however on first try there a new exception
showing up upon connecting to wiremock.

```
javax.net.ssl.SSLPeerUnverifiedException: Hostname localhost not verified:
    certificate: sha256//W3v5TDAEE3dl4peiEwpwDKa1OZAna1ITgokQDz0rkQ=
    DN: CN=Tom Akehurst, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
    subjectAltNames: []

	at okhttp3.internal.connection.RealConnection.connectTls(RealConnection.java:308)
	...
```

What is this `SSLPeerUnverifiedException` ? After inspection, the above code
tackle the trust layer of the TLS, but according to the HTTPS 
[RFC 2818](https://tools.ietf.org/html/rfc2818#section-3.1), the client
is required to check the identity of the hostname against the certificate,
this help prevent _man-in-the-middle_ attacks. That means that the TLS layer 
is established, but this failure happens at the upper level.

Indeed in the current setup, wiremock self-signed certificate doesn't provide enough 
data to perform this verification. The `subjectAltNames` field should contain 
server alternative names like `localhost` or the (local) IP address(es).

So let's tell OkHttp to accept all hostnames : 

```java
public static HostnameVerifier allowAllHostNames() {
    return (hostname, sslSession) -> true;
}
```

```java
new OkHttpClient.Builder()
            .sslSocketFactory(
                    sslContext(null, 
                               new TrustManager[] {TrustAllX509TrustManager.INSTANCE}).getSocketFactory(),
                    TrustAllX509TrustManager.INSTANCE)
            .hostnameVerifier(allowAllHostNames())
            .build()
            .newCall(new Request.Builder().get()
                                          .url("https://localhost:" + wireMock.httpsPort())
                                          .build())
            .execute();
```

Finally this code works, we have an HTTPS connection to a wiremock server.

However this way of setting up the code is not right and especially for
production code, because this code just deactivated security for every HTTPS 
connection passing by this http client.

With this approach one should think of a configuration mechanism that will 
enforce sane settings in production code and relax settings for test code.

On a side note this example has been written with backend application in mind,
however this topic is concerning mobile application or IoT as well. 
The [Android documentation](https://developer.android.com/training/articles/security-ssl.html#CommonHostnameProbs) 
explains very well why hostname verification is important to consider.

#### Vanilla Java

For those interested the same issues happen with the JDK way to connect to 
HTTPS via `URL`, this can be configured via the `HttpsUrlConnection` class, 
here's how to configure it, **before establishing the connection** :

```java
// Allow all certificates
HttpsURLConnection.setDefaultSSLSocketFactory(trustAllSslContext().getSocketFactory());
new URL("https://localhost:" + wireMock.httpsPort()).openConnection().connect();
```

And just set a [`HostnameVerifier`](https://docs.oracle.com/javase/8/docs/api/javax/net/ssl/HostnameVerifier.html) 
that does nothing.

```java
// Allow all hostnames
HostnameVerifier allHostsValid = (hostname, session) -> true;
HttpsURLConnection.setDefaultHostnameVerifier(allHostsValid);
HttpsURLConnection.setDefaultSSLSocketFactory(trustAllSslContext().getSocketFactory());
new URL("https://localhost:" + wireMock.httpsPort()).openConnection().connect();
```

Notice that `setDefaultHostnameVerifier` and `setDefaultSSLSocketFactory` are 
static methods, that means these changes affect all connections using the JDK.


### Extending trust to self-signed certificate

So rather than turning off the whole security of HTTPS connections, a 
big improvement would be to ignore verification of self-signed certificate only.
The first question is how to identify a self-signed certificate ?

To answer that question, let's inspect certificates with a reference tool.
OpenSSL has an SSL/TLS client available with the `s_client` sub-command,
we can then connect to the wiremock HTTPS server.

```sh
echo -n | openssl s_client -connect localhost:8443 2>&1
```

Upon connection, the terminal shows a bunch of interesting data...

```
CONNECTED(00000003)
depth=0 C = Unknown, ST = Unknown, L = Unknown, O = Unknown, OU = Unknown, CN = Tom Akehurst
verify error:num=18:self signed certificate
verify return:1
depth=0 C = Unknown, ST = Unknown, L = Unknown, O = Unknown, OU = Unknown, CN = Tom Akehurst
verify return:1
---
Certificate chain
 0 s:/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
   i:/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIDgzCCAmugAwIBAgIEHYkuTzANBgkqhkiG9w0BAQsFADBxMRAwDgYDVQQGEwdV
bmtub3duMRAwDgYDVQQIEwdVbmtub3duMRAwDgYDVQQHEwdVbmtub3duMRAwDgYD
VQQKEwdVbmtub3duMRAwDgYDVQQLEwdVbmtub3duMRUwEwYDVQQDEwxUb20gQWtl
aHVyc3QwIBcNMTUwMjI0MTM1ODUwWhgPMjExNTAxMzExMzU4NTBaMHExEDAOBgNV
BAYTB1Vua25vd24xEDAOBgNVBAgTB1Vua25vd24xEDAOBgNVBAcTB1Vua25vd24x
EDAOBgNVBAoTB1Vua25vd24xEDAOBgNVBAsTB1Vua25vd24xFTATBgNVBAMTDFRv
bSBBa2VodXJzdDCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAIAIvMUo
vy4ufnWKxMU0tBdXqtX6RzKYgQvj/82qPAmRiNki8PpPGrF70Lb3WzsUDYB9CsXw
m5VWc9l1XBdGh6zZVFkkSzBtRjyHy8Z8azsIv/YzQF5bRxE2Cvruh7o01Sq1qz5B
kxt0u/NbUUErxKZeA0li1W/op7RC94h0dzob7auruHUvb56NXAJZcu8r2G9jxh9w
WBPC6lSozuCzwfdS4v2ZOQBYpmMz9oJm3ElQUbOrhnVQtgxQicU2oDETwz37IIEw
FV12la+qNIMSOTe6uJj1jEZP22NL2IYq06BT/ZnK6HYIOXAtwURSsf0MN0b8NKBB
NOLQN2juRj+vn6UCAwEAAaMhMB8wHQYDVR0OBBYEFDZ6soXRxD/N2n5b++CVrWbr
XLKWMA0GCSqGSIb3DQEBCwUAA4IBAQBiPfCUg7EHz8poRgZL60PzMdyaKLwafGtF
dshmY1y9vzpPJIoFcIH7crSsmUcRk+XSj5WhSr4RT3y15JsfZy935057f0knEXEf
or+Gi8BlDaC33qX+6twiAaub1inEDc028ZFtEwbzJQYgJo1GvLG2o2BMZB1C5F+k
Nm9jawu4rTNtXktXloNhoxrSWtyEUoDAvGgBVnAJwQXcfayWq3AsCr9kpHI3bBwL
J9NAGC4M8j7z9Aw71JGmwBDk1ooAO6L82W7DWBYPzpLXXeXmHRCxpujKWaveAV2T
cgsQaCmzy29i+F03pLl7Vio4Ei+z9XQgZiN4Awiwz9D+lshnKuII
-----END CERTIFICATE-----
subject=/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
issuer=/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
---
No client certificate CA names sent
Peer signing digest: SHA512
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 1387 bytes and written 434 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: 59D642ED8CCB7F4219617B3739CB93E1C294F873854C2A284D28A77D674AC050
    Session-ID-ctx: 
    Master-Key: A43DF026E3FA620BC7CC5207D5BCB87828B7E3D673CFEEF12CAB425619B63610F443FB96FEC33CC50FBBAC73C152572B
    Key-Arg   : None
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1507214061
    Timeout   : 300 (sec)
    Verify return code: 18 (self signed certificate)
---
DONE
```

...typically the first lines :

```
depth=0 C = Unknown, ST = Unknown, L = Unknown, O = Unknown, OU = Unknown, CN = Tom Akehurst
verify error:num=18:self signed certificate
```

The client reports a verification error number `18`, and display the 
associated reason that it is a self-signed certificate. More specifically
the [openssl wiki](https://wiki.openssl.org/index.php/Manual:Verify(1)) 
defines the error 18 as :

> **18 X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT**: self signed certificate

So openssl understand that a certificate chain with a depth of `0` is in 
fact a self-signed-certificate.

Continuing on the output, there's the certificate chain section itself, 
it declares each certificate that are presented by the server.

For the certificate with a `0` depth, there's two lines :

 - the first prefixed by **`s`** that print the _subject_ of the certificate
 - the second prefixed by **`i`** that print the _issuer_ of the certificate

```
Certificate chain
 0 s:/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
   i:/C=Unknown/ST=Unknown/L=Unknown/O=Unknown/OU=Unknown/CN=Tom Akehurst
```

There is additional information, but the above information leads to think 
it is possible to write a trust manager that will skip verification for
self-signed certificates only. The code below checks the number of 
certificate, if there's only one element in the array, it means the 
depth is zero.

```java
public static class TrustSelfSignedX509TrustManager implements X509TrustManager {
    private X509TrustManager delegate;

    private TrustSelfSignedX509TrustManager(X509TrustManager delegate) {
        this.delegate = delegate;
    }

    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        delegate.checkClientTrusted(chain, authType);
    }

    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        if (isSelfSigned(chain)) {
            return;
        }
        delegate.checkServerTrusted(chain, authType);
    }

    private boolean isSelfSigned(X509Certificate[] chain) {
        return chain.length == 1;
    }

    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return delegate.getAcceptedIssuers();
    }

    public static X509TrustManager[] wrap(X509TrustManager delegate) {
        return new X509TrustManager[]{new TrustSelfSignedX509TrustManager(delegate)};
    }
}
```

Notice how this code don't actually perform certificate verification but 
delegates this task to an existing `TrustManager`.

This decorator can wrap the JVM trust manager, which happens to be able 
to verify _normal_ certificates.

```java
X509TrustManager trustManager = TrustSelfSignedX509TrustManager.wrap(systemTrustManager());
new OkHttpClient.Builder()
            .sslSocketFactory(
                    sslContext(null, 
                               new TrustManager[] { trustManager }).getSocketFactory(),
                    trustManager)
            .hostnameVerifier(allowAllHostname())
            .build()
            .newCall(new Request.Builder().get()
                                          .url("https://localhost:" + wireMock.httpsPort())
                                          .build())
            .execute();
```

Here's how to gain access to the default manager :

```java
public static X509TrustManager systemTrustManager() {
    TrustManager[] trustManagers = systemTrustManagerFactory().getTrustManagers();
    if (trustManagers.length != 1) {
        throw new IllegalStateException("Unexpected default trust managers:"
                                        + Arrays.toString(trustManagers));
    }
    TrustManager trustManager = trustManagers[0];
    if (trustManager instanceof X509TrustManager) {
        return (X509TrustManager) trustManager;
    }
    throw new IllegalStateException("'" + trustManager + "' is not a X509TrustManager");
}


private static TrustManagerFactory systemTrustManagerFactory() {
    try {
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init((KeyStore) null);
        return tmf;
    } catch (NoSuchAlgorithmException | KeyStoreException e) {
        throw new IllegalStateException("Can't load default trust manager", e);
    }
}
```


`TrustManagerFactory.getInstance(String)` is asking for the name of trust
implementation _algorithm_. And at this time `TrustManagerFactory.getDefaultAlgorithm()` 
should only return `PKIX` because that's the only trust implementation 
of TLS at this time.

However one should note that JVM implementers load their own algorithm 
implementation of PKIX, this implementation is available under the 
implementation name like Sun did with `SunX509`, of course this implementation
is not available when using IBM J9 or when using Android, they have their own.
All should implement `X509TrustManager`. Avoid using those names those
implementation may change for various reason, could renamed to `OracleX509` 
one day, or be replace by some other thing.
So when looking at code on the internet that rely on implementation specific 
one should take this opportunity to use the most generic name unless something 
specific is needed, in this case we want the default `PKIX` algorithm.

Notice again how this JDK security API is very generic, the trust interface 
just does not make any assumption about the trust implementation. It could be 
_public key infrastructure_ based (_i.e X.509 or something else_) or 
something else entirely.

Also `TrustManagerFactory.getTrustManagers()` returns an array for supposedly for
each type of trust management.
Here's the javadoc:

> ```
> public final TrustManager[] getTrustManagers()
>
> Returns one trust manager for each type of trust material.
>
> Returns:
>     the trust managers
> Throws:
>     IllegalStateException - if the factory is not initialized.
> ```

Give the javadoc of `SSLContext#init` and that in practice today the only
available kind of way to connect over TLS is PKIX / X.509. So this 
factory only creates a single trust manager instance, of type 
`X509Trustmanager`.


So the above described approach at least doesn't degrade the security for 
the HTTPS calls to website properly relying on root certificate authorities.
Yet the security for self-signed signed certificates is still not handled, 
just dismissed. In my opinion this is acceptable for test code, but barely,
security is a major concern, if we cannot get it right in tests what about 
production code that must speak to self signed certificate.

Just because it is self-signed, doesn't mean there's no way to verify this 
certificate.

## How to properly validate a self-signed certificate ?

What to do if the production code has to establish to a server whose
certificate is self-signed ?

We saw that the JVM doesn't grant trust to certificates whose no known 
certification authority (_CA_) signed the peer certificate. A simple 
workaround would be install the unknown certificate in question. 

But before installing anything, let's get a hold of it.

### Get the self-signed certificate

Upon the TLS connection handshake, the server sends his certificate chain,
as we saw earlier when using the `s_client` of `openssl`.

It his possible to get the certificates in a `PEM` format using another 
command of `openssl`.

```sh
echo -n | \
  openssl s_client -prexit -connect host:port 2>&1 | \
  openssl x509 -outform pem \
  > certificate-host.pem
```

The above command indicate to `openssl` to establish a connection to 
`host:port` then to pass the results to the `x509` command in order to
decode the certificate and to encode it in a `PEM` format.

It is also possible to print on the console a human readable representation
(not PEM):

```sh
echo -n | \
  openssl s_client -connect host:port 2>&1 | \
  openssl x509 -text
```

To read the certificate file in the `PEM` format there either the standard
tool `openssl` or with `keytool` coming with the base install of the JDK :

```sh
keytool -printcert -file certificate-host.pem
openssl x509 -inform pem -in certificate-host.pem -text
```

### Using the self-signed certificate on the JVM

#### _Importing_ the certificate in the JVM's own _cacerts_

The default JVM install comes with _cacerts_ file that stores the _certificates_
of _certificate authorities_. A self-signed certificate wasn't signed (issued)
by any of these authority, but since this certificate is signing itself, it is
possible to inject it in the _cacerts_ key store. 
It is possible to achieve this with `keytool` ; prepended with `sudo` 
if the actual file system permission requires it.
At JVM installation time this key store is configured with password `changeit`, 
but a good practice in production is to change this password to prevent any 
tampering with the _cacerts_ file.

```sh
keytool -import \
        -alias wiremock \
        -file wiremock.pem \
        -keystore $JAVA_HOME/jre/lib/security/cacerts
```

The wiremock certificate has been extracted to file `wiremock.pem` in the 
**PEM** format. The following command will import this certificate with
the alias `wiremock` in the `cacerts` key store.

> Eventually check the import went well by querying the content of the 
> `cacerts` key store :
> 
> ```sh
> keytool -list -keystore $JAVA_HOME/jre/lib/security/cacerts
> ```
>
> After the injection of the certificate, a `wiremock` entry should appear
> next to Verisign, Digital Certs, Geo Trust, etc.

Once done, when trying a simpler version of the HTTPS calling code without
the whole SSL setup, there's no error, the standard security configuration 
can match the wiremock certificate because it now knows about it.

```java
new OkHttpClient.Builder()
            .hostnameVerifier(allowAllHostNames())
            .build()
            .newCall(new Request.Builder().get()
                                          .url("https://localhost:8443")
                                          .build())
            .execute();
```

Note however there's still a need of a dedicated hostname verifier that 
allows every host, because the TLS protocol handshake is different than
host or IP verification that is part of HTTPS 
([RFC 2818](https://tools.ietf.org/html/rfc2818#section-3.1)).

Regardless injecting certificates in the JDK installation requires a 
deployment, that is a pain for developers, that is a pain for CI, and 
that is a pain for OPS. We can do better.


#### Using an alternate key store

_Sometime this specific **key store** is referred to as a **trust store**._

In this part let's craft a key store from the ground using the same tool
as above `keytool`.

THe first contact with the key store was when we were meddling with the 
_cacerts_ file. To create our own the usual extension is `jks`, that acronym 
stands for Java **key** - **store** is a file, that's where the `PEM` 
certificate will be imported.

```sh
keytool -import \
        -alias wiremock \
        -file wiremock.pem \
        -keystore ./wiremock-truststore.jks
```

This commands takes a certificate file stored in the `PEM` format and stores
it in a newly created Java Key Store file named `wiremock-truststore.jks`.

> The above commands requires a user interaction, this can be avoided
> by passing a few options, especially `-noprompt` and `-storepass <password>`.

```sh
keytool -import \
        -noprompt \
        -storepass changeit \
        -alias wiremock \
        -file wiremock.pem \
        -keystore ./wiremock-truststore.jks
```

Then the `java` commands allow to change the runtime trust store, which 
is the by default the _cacerts_ file, with the following options : 

* `-Djavax.net.ssl.trustStore=./wiremock-truststore.jks`
* `-Djavax.net.ssl.trustStorePassword=changeit`

The full command :

```sh
java -Djavax.net.ssl.trustStore=./wiremock-truststore.jks \
     -Djavax.net.ssl.trustStorePassword=changeit \
     OkSSLConnect localhost 8443
```

In the above command `OkSSLConnect` is the program that will establish the
TLS connection to the wiremock HTTPS server with the given host and port. 
The `main` function consists only of the simplest code above (no special 
TLS configuration, just the _hostname verifier_). 
If the program succeeds then the alternate trust store has been used.

However using this alternate trust store won't work if the same program
must connect to some _well known_ website (i.e. like `github.com`) whose
certificates are issued by well known certificate authorities. In this
case the same `SSLHandshakeException` will be raised.

```sh
java -Djavax.net.ssl.trustStore=./wiremock-truststore.jks \
     -Djavax.net.ssl.trustStorePassword=changeit \
     OkSSLConnect google.com 443
```

Without setting the alternate trust store, this command 
`java OkSSLConnect google.com 443` succeeds.

So this approach is also lacking in some areas, the simple fact these options
change the JVM's trust store is wrong, unless there's specific use cases.
Also this can become a deployment issue in local development environment 
or on servers. Moreover it restrict the servers the JVM can connect to 
servers that have the very specific certificates.
Not an acceptable solution in my opinion.

#### Importing programmatically the self-signed certificate

##### From JKS key store

Lets begin with the JKS file created in the section above with `keytool`.
The code below is split in three different method with each their 
own concerns :

* one to load `TrustManager` factory, using the default algorithm 
  [`PKIX`](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#TrustManagerFactory)
* one to load the trust store itself, using the default type 
  `KeyStore.getDefauktType()` returns `jks` 
  <sup>
    [[1]](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#KeyStore)
    [[2]](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStore.html#getDefaultType--)
  </sup>.
* one to build the trust manager instance from the trust store

```java
public static X509TrustManager trustManagerFor(KeyStore keyStore) {
    TrustManagerFactory tmf = trustManagerFactoryFor(keyStore);

    TrustManager[] trustManagers = tmf.getTrustManagers();
    if (trustManagers.length != 1) {
        throw new IllegalStateException("Unexpected number of trust managers:"
                                                + Arrays.toString(trustManagers));
    }
    TrustManager trustManager = trustManagers[0];
    if (trustManager instanceof X509TrustManager) {
        return (X509TrustManager) trustManager;
    }
    throw new IllegalStateException("'" + trustManager + "' is not a X509TrustManager");
}


public static TrustManagerFactory trustManagerFactoryFor(KeyStore keyStore) {
    try {
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(keyStore);
        return tmf;
    } catch (KeyStoreException | NoSuchAlgorithmException e) {
        throw new IllegalStateException("Can't load trust manager for keystore : " + keyStore, e);
    }
}


public static KeyStore readJavaKeyStore(Path javaKeyStorePath, String password) {
    try (InputStream inputStream = new BufferedInputStream(Files.newInputStream(javaKeyStorePath))) {
        KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
        ks.load(inputStream, password.toCharArray());
        return ks;
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    } catch (CertificateException | NoSuchAlgorithmException | KeyStoreException e) {
        throw new IllegalStateException(e);
    }
}
```

Note in the code above the password parameter given to read the trust store.
Usually it is advised to array for password because a program have a better 
control on their lifecycle (`String` are immutable and on the heap, while 
array are mutable thus allowing to erase the password, if not the array 
object).

Then the OkHttp client builder can be configured with the custom SSL 
socket factory :

```java
X509TrustManager trustManager = trustManagerFor(readJavaKeyStore(Paths.get("./wiremock-truststore.jks"), "changeit"));
new OkHttpClient.Builder()
        .sslSocketFactory(sslContext(null, new TrustManager[]{trustManager}).getSocketFactory(),
                          trustManager)
        .hostnameVerifier(allowAllHostNames())
        .build()
        .newCall(new Request.Builder().get()
                                        .url("https://localhost:8443")
                                        .build())
        .execute();
```

This code is quite nice because it can server for the majority of HTTP usage
in the application. The code allows a approach that is a bit more configurable 
and under control.

##### From a `PEM` certificate

Since the trust manager factory can only be built with a key store, this 
approach will build a key store in memory. This key store
will be injected with the `X.509` certificate that was extracted previously 
with the command `openssl x509 -outform pem`.

Since the only type of _key store_ available in the JVM is `jks`, and the
scope of this article is limited to SSL/TLS mechanism of the JVM only, 
the following code will then build a JKS in memory. This key store
will inject the `X.509` certificate that was extracted previously with
the command `openssl x509 -outform pem`.

```java
public static KeyStore makeJavaKeyStore(Path certificatePath) {
    try (BufferedInputStream bis = new BufferedInputStream(Files.newInputStream(certificatePath))) {
        CertificateFactory cf = CertificateFactory.getInstance("X.509");

        KeyStore ks = KeyStore.getInstance(KeyStore.getDefaultType());
        ks.load(null, null);
        int certificate_counter = 0;
        for (X509Certificate certificate : (Collection<X509Certificate>) cf.generateCertificates(bis)) {
            ks.setCertificateEntry("cert_" + certificate_counter++, certificate);
        }

        return ks;
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    } catch (CertificateException e) {
        throw new IllegalStateException("Can't load certificate : " + certificatePath, e);
    } catch (KeyStoreException | NoSuchAlgorithmException e) {
        throw new IllegalStateException("Can't create the internal keystore for certificate : " + certificatePath, e);
    }
}
```

As a reminder notice this code use the key store default type, i.e. `jks`, yet 
the JVM also supports other types like `PKCS12` 
([see more](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#KeyStore))

Also notice how the call `(Collection<X509Certificate>) cf.generateCertificates(bis)`
is **generic** and that depending on the instance type the generated 
certificate object could have different type, here the type is `"X.509"` 
so according to the javadoc objects can be cast to `X509Certificate` type.

One other thing to note is that `(Collection<X509Certificate>) cf.generateCertificates(bis)`
returns actually a collection of certificates. That means the file 
represented by this input stream may contain multiple certificates. Indeed the
[`CertificateFactory.generateCertificate(InputStream)`](https://docs.oracle.com/javase/8/docs/api/java/security/cert/CertificateFactory.html#generateCertificate-java.io.InputStream-) 
javadoc specifies that for **X.509** type the input stream content must be 

* `PEM` formatted, which is means a payload encoded to ASCII base64 within 
    the following boundaries or markers
    `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` 

* `DER` formatted which is the binary representation of the certificate
    (It is possible to get one using this command 
    `openssl x509 -in wiremock.pem -outform der > wiremock.der`).
 
> In the case of a certificate factory for X.509 certificates, the certificate 
> provided in inStream must be DER-encoded and may be supplied in binary or 
> printable (Base64) encoding. If the certificate is provided in Base64 encoding, 
> it must be bounded at the beginning by `-----BEGIN CERTIFICATE-----`, and must be 
> bounded at the end by `-----END CERTIFICATE-----`.

_The certificate entry name is not important in the context of this blog._

Finally this code can be integrated by instructing the code to create a 
trust manager using this key store :

```java
X509TrustManager trustManager = trustManagerFor(makeJavaKeyStore(Paths.get("./wiremock.pem")));
new OkHttpClient.Builder()
        .sslSocketFactory(sslContext(null, new TrustManager[]{trustManager}).getSocketFactory(),
                          trustManager)
        .hostnameVerifier(allowAllHostNames())
        .build()
        .newCall(new Request.Builder().get()
                                        .url("https://localhost:8443")
                                        .build())
        .execute();
```

That's even nicer because there's once less step, no `keytool` action required.

Yet there again once this client is configured it still cannot connect to HTTPS 
servers using other certificates, like `github.com`.


### Programmatically use both the system trust chain, and the self signed certificate

If the HTTPS client must connect to various third parties having either a
certificate signed from a _known_ certificate authority or a self signed 
certificate, then it is needed to tweak the custom trust manager.

This approach simply craft a composite trust manager delegating certificate 
verification to the configured delegate trust managers :


```java
public class CompositeX509TrustManager implements X509TrustManager {
    private final List<X509TrustManager> trustManagers;

    public CompositeX509TrustManager(X509TrustManager... trustManagers) {
        this.trustManagers = Arrays.asList(trustManagers);
    }

    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        new MultiException<>(new CertificateException("This certification chain couldn't be trusted"))
                .collectFrom(trustManagers.stream(),
                             trustManager -> trustManager.checkClientTrusted(chain, authType))
                .scream(UNLESS_ANY_SUCCESS);
    }

    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        new MultiException<>(new CertificateException("This certification chain couldn't be trusted"))
                .collectFrom(trustManagers.stream(),
                             trustManager -> trustManager.checkServerTrusted(chain, authType))
                .scream(UNLESS_ANY_SUCCESS);
    }

    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return trustManagers.stream()
                            .map(X509TrustManager::getAcceptedIssuers)
                            .flatMap(Arrays::stream)
                            .toArray(X509Certificate[]::new);
    }
}
```

`MultiException` is small _helper_ that allows to collect several exceptions 
that are raised by a collection of calls, while still raising a single 
parent exception, it relies on `Throwable.addSuppressed` that was added
in Java 1.7 for _try-with-resources_ statements.

```java
public class MultiException<E extends Exception> {
    private final E parent;
    private boolean successMarker = false;

    public MultiException(E parent, Exception... exceptions) {
        this.parent = parent;
        Arrays.stream(exceptions).forEach(parent::addSuppressed);
    }

    public <T> MultiException<E> collectFrom(Stream<T> stream, ThrowingConsumer<T> invocation) {
        stream.forEach(t -> collect(t, invocation).ifPresent(parent::addSuppressed));
        return this;
    }

    private <T> Optional<Exception> collect(T type, ThrowingConsumer<T> throwing) {
        try {
            throwing.accept(type);

            successMarker = true;
            return Optional.empty();
        } catch (Exception e) {
            return Optional.of(e);
        }
    }

    public void scream(Mode mode) throws E {
        if (Mode.UNLESS_ANY_SUCCESS == mode && successMarker) {
            return;
        }
        if (parent.getSuppressed().length > 0) {
            throw parent;
        }
    }

    @FunctionalInterface
    public interface ThrowingConsumer<T> {
        void accept(T type) throws Exception;
    }

    public enum Mode {
        UNLESS_ANY_SUCCESS,
        ANY_FAILURE
    }
}
```

The composite manager can be used this way :

```java
X509TrustManager compositeTrustManager = new CompositeX509TrustManager(
        trustManagerFor(makeJavaKeyStore(Paths.get("./wiremock.pem"))),
        systemTrustManager());
OkHttpClient okHttpClient = httpClient(sslContext(null,
                                                  new TrustManager[]{compositeTrustManager}),
                                        compositeTrustManager)
        .newBuilder()
        .hostnameVerifier(allowAllHostname())
        .build();
```

This client can now access both kind of HTTPS servers.

## Server side

This blog post explored a bit the client side, but form the beginning the code
dealt with wiremock's certificate. There's something to do on this side
to allow a complete control in the tests. The first thing would be to generate 
one.

### Generate a self-signed certificate with `keytool`

The JDK's `keytool` offers various commands and one, especially useful in the 
current case, that allows to generate a certificate.

```sh
keytool -genkey \
        -keyalg RSA \
        -alias bric3 \
        -keystore bric3.jks \
        -storepass the_password \
        -validity 360 \
        -keysize 2048
```

With the above command `keytool` will generate a Java keystore file with 
the brand new certificate. It will interact with the user, asking for
the following certificate's attributes :

* `CN` (**C**ommon **N**ame)
* `OU` (**O**rganizational **U**nit)
* `O` (**O**rganization)
* `L` (**L**ocality)
* `ST` (**ST**ate)
* `C` (**C**ountry)

then it will ask for the certificate password, not to be mixed with the 
keystore password (which is given in the above command with 
`-storepass the_password`). This can work in batch mode (non-interactive)
if the command has the following options : 

* `-keypass password`
* `-dname 'CN=Brice Duhteil, OU=Arkey, O=Arkey, L=Paris, ST=France, C=FR'`

> `dname` means **D**istinguished **N**ames

However this new generated certificate and the one of wiremock have the 
same problems leading to the same obstacle that had to be dealt with in 
the first half of this blog :

* it is self-signed
* it will require a custom host name verifier ([RFC 2818](https://tools.ietf.org/html/rfc2818#section-3.1))

Let's first fix this last point first, what this certificate need is a
[`SAN` (**S**ubject **A**lternative **N**ames)](https://tools.ietf.org/html/rfc5280#section-4.2.1.6)
section that can contain DNS names or IP addresses. `keytool` can be tell
to add this information with the `-ext` option along with the **SAN** 
_extension_ that needs to be formatted with almost any number of `dns` or 
`ip` elements.

```
-ext SAN=dns:domain.com,dns:localhost,ip:127.0.0.1
```

For example if the certificate needs to be valid for the following hostnames

* `blog.arkey.fr`
* `blog`
* `127.0.0.1`
* `::1`

`keytool` needs to be told to generate the certificate with the following options :

```sh
keytool -genkey \
        -keyalg RSA \
        -alias bric3 \
        -keystore bric3.jks \
        -storepass the_password \
        -validity 360 \
        -keysize 2048 \
        -keypass the_password \
        -dname 'CN=Brice Duhteil, OU=Arkey, O=Arkey, L=Paris, ST=France, C=FR' \
        -ext 'SAN=dns:blog.arkey.fr,dns:blog,dns:localhost,ip:127.0.0.1,ip:::1'
```

--------------------------------------------------------------------------------

**Note #1:**

1.  `keytool` generate a certificate and stores it immediately in the **J**ava 
     **K**ey **S**tore
2.  wiremock in this version only allow to configure a single password that 
    will be used for both the certificate and the **J**ava **K**ey **S**tore ; 
    that means for our tests the command should specify the same password for 
    both JKS and the certificate 
    e.g. : `-storepass the_password` and `-keypass the_password`.

**Note #2:**

`keytool` do validate the DNS entry, but it doesn't understand all possible 
allowed characters of a domain name for this reason it would be preferable 
to use `openssl` or similar tools to generate the certificates. 
An interesting part of the PKIX / X509 specification in version 3 is the 
_Subject Alt Names_ ; this extension allows to list the _names_ of the 
servers for for which this certificate has been issued, this completes
the limited _Common Name_.
One of the advantages of this extension is the ability to enter 
[wildcard](https://tools.ietf.org/html/rfc5280#section-4.2.1.6) names, 
although the RFC does cover how a client should interpret and 
act on those wildcards :

> Finally, the semantics of subject alternative names that include
> wildcard characters (e.g., as a placeholder for a set of names) are
> not addressed by this specification.  Applications with specific
> requirements MAY use such names, but they must define the semantics.

E.g. **`google.com`** certificate is configured with domain names with 
wildcard:

```sh
echo -n | openssl s_client -showcerts -connect google.com:443 2>&1 | openssl x509 -text
```

```
X509v3 Subject Alternative Name: 
    DNS:*.google.com, DNS:*.android.com, DNS:*.appengine.google.com, DNS:*.cloud.google.com, DNS:*.db833953.google.cn, DNS:*.g.co, DNS:*.gcp.gvt2.com, DNS:*.google-analytics.com, DNS:*.google.ca, DNS:*.google.cl, DNS:*.google.co.in, DNS:*.google.co.jp, DNS:*.google.co.uk, DNS:*.google.com.ar, DNS:*.google.com.au, DNS:*.google.com.br, DNS:*.google.com.co, DNS:*.google.com.mx, DNS:*.google.com.tr, DNS:*.google.com.vn, DNS:*.google.de, DNS:*.google.es, DNS:*.google.fr, DNS:*.google.hu, DNS:*.google.it, DNS:*.google.nl, DNS:*.google.pl, DNS:*.google.pt, DNS:*.googleadapis.com, DNS:*.googleapis.cn, DNS:*.googlecommerce.com, DNS:*.googlevideo.com, DNS:*.gstatic.cn, DNS:*.gstatic.com, DNS:*.gvt1.com, DNS:*.gvt2.com, DNS:*.metric.gstatic.com, DNS:*.urchin.com, DNS:*.url.google.com, DNS:*.youtube-nocookie.com, DNS:*.youtube.com, DNS:*.youtubeeducation.com, DNS:*.yt.be, DNS:*.ytimg.com, DNS:android.clients.google.com, DNS:android.com, DNS:developer.android.google.cn, DNS:developers.android.google.cn, DNS:g.co, DNS:goo.gl, DNS:google-analytics.com, DNS:google.com, DNS:googlecommerce.com, DNS:source.android.google.cn, DNS:urchin.com, DNS:www.goo.gl, DNS:youtu.be, DNS:youtube.com, DNS:youtubeeducation.com, DNS:yt.be
```

--------------------------------------------------------------------------------

### Generate a self-signed certificate with `openssl`

`keytool` is nice, but it requires the JVM, and let's use the standard 
tool, the _Swiss army knife_ `openssl` (or the new forks **BoringSSL** or 
**LibreSSL**).

The `keytool` command that was used was generating a self-signed certificate 
in one shot, it is also possible with `openssl` but let's understand the 
different steps that are needed under the hood to generate a self-signed 
certificate.

1. Generate a private key

    ```sh
    openssl genrsa \
        -out bric3-private.key \
        2048
    ```

    The above command creates a private key of 2048 bits using the RSA 
    algorithm. This key is **not** protected by a password, so keep it 
    safe (to do that use the `-des3` option). 

2. Request a new certificate for the domain(s).

    ```sh
    openssl req \
        -new \
        -outform pem \
        -out bric3-self.csr \
        -keyform pem \
        -key bric3-private.key \
        -sha256 \
        -config <(cat <<-EOF

    [req]
    prompt = no
    req_extensions = bric3_req_ext
    distinguished_name = dn
    
    [dn]
    CN=Brice Dutheil
    O=Arkey
    OU=Arkey
    L=Paris
    ST=France
    C=FR

    [bric3_req_ext]
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = localhost
    DNS.2 = arkey.fr
    DNS.3 = *.arkey.fr
    DNS.4 = arkey.pro
    DNS.5 = *.arkey.pro
    DNS.6 = blog
    IP.1 = 127.0.0.1
    IP.2 = ::1

    EOF
    )
    ```

    The above `req` command works in batch mode (i.e. non-interractive). 
    It will generate a `CSR` file (**C**ertificate **S**ign **R**equest) 
    stored in the PEM format. This command needs the private key of the 
    server's owner. In order to pass additional parameters to generate 
    the CSR are passed via a shell stream and _[here document](http://tldp.org/LDP/abs/html/here-docs.html)_ 
    `<(cat <<-MARKER ... MARKER)`.

    In the _document_ there's a `[req]` section with
    
      - an option that instruct the program to run in non-interactive 
        mode : `prompt = no`, 
      - a _link_ to the **D**istinguished **N**ame section named `dn` : 
        `distinguished_name = dn`, 
      - a _link_ to the **S**ubject **A**lt **N**ame extension section :
        `req_extensions = bric3_req_ext`, without this field it would be 
        necessary to pass the the following option to the command 
        `-reqexts bric3_req_ext`

    The `[dn]` section defines fields that compose what is a _DN_, e.g. 
    the **C**ommon **N**ame, the **O**rganization, etc. 
    
    The _extension section_ (`bric3_req_ext`) defines where is located 
    the _alternative name section_ : `subjectAltName = @alt_names`, in 
    this case only the `@` prefix is necessary.
    
    The _alternative name section_ (`alt_names`) will define the wanted 
    entries of `DNS` type, `IP` type, etc.

    For more on this `req` command and how it can be configured, just deep 
    dive in the [man (master branch)](https://www.openssl.org/docs/manmaster/man1/req.html) 
    page.


3. Sign the _certificate sign request_ to generate the actual certificate

    ```sh
    openssl x509 \
        -req \
        -days 3650 \
        -inform pem \
        -in bric3-self.csr \
        -signkey bric3-private.key \
        -outform pem \
        -out bric3-self.pem \
        -extensions bric3_ext \
        -extfile <(cat <<-EOF
    [bric3_ext]
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = localhost
    DNS.2 = arkey.fr
    DNS.3 = *.arkey.fr
    DNS.4 = arkey.pro
    DNS.5 = *.arkey.pro
    DNS.6 = blog
    IP.1 = 127.0.0.1
    IP.2 = ::1
    EOF
    )
    ```

    The command `x509` gnerate a new X.509 certificate from a CSR (`-req`). 
    As an intput it obvisouly need the request file `-in bric3-self.csr`,
    and since it is a self-signed certificate **the command uses the same 
    private key** that was used to generate the request `-signkey bric3-private.key`,
    otherwise it would be the private key of the certificate authority. 
    The new certificate will be valid for ten years (`-days 3650`).

    Unfortunately this `openssl` command don't use the extensions of the 
    request, consequently it his needed to provide them as well via 
    configuration file or via a stream (and _[here docs](http://tldp.org/LDP/abs/html/here-docs.html)_)
    `-extfile <(cat <<-EOF ... EOF)`, however in this case the extension 
    section name has to be passed via an option `-extensions bric3_ext`.

    If everything is correct a new `bric3-self.pem` certificate will be issued.

    For more on this `x509` command and how it can be configured, just deep 
    dive in the [man (master branch)](https://www.openssl.org/docs/manmaster/man1/x509.html) 
    page.


The configuration shown above is quite simple for this blog, in practice
it will be more complicated with a _certificate authority_ and a more 
elaborate certificate chain.

As said earlier the 3 above steps that can be reduced to the the following 
command to generate a self-signed certificate.

```sh
openssl req \
    -new \
    -nodes \
    -sha256 \
    -newkey rsa:2048 \
    -keyform pem \
    -keyout bric3-openssl.key \
    -x509 \
    -days 3650 \
    -outform pem \
    -out bric3-openssl.crt \
    -config <(cat <<-EOF
[req]
prompt = no
distinguished_name = dn
x509_extensions = bric3_ext

[dn]
CN=Brice Dutheil
O=Arkey
OU=Arkey
L=Paris
ST=France
C=FR

[bric3_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = arkey.fr
DNS.3 = *.arkey.fr
DNS.4 = arkey.pro
DNS.5 = *.arkey.pro
DNS.6 = blog
IP.1 = 127.0.0.1
IP.2 = ::1
EOF
)
```

The command is `req` but tweaked with these options :

- `-newkey rsa:2048` option that will generate the RSA 2048 private key.
  The key will be stored with the PEM format in the `bric3-openssl.key` file.
- `-x509` that tell the command the output won't be a certificate sign 
  request but a X.509 signed certificate. The certificate will be valid 
  for ten years `-days 3650` and will be stored with the PEM format in 
  the `bric3-openssl.crt` file. Since `-x509` option is used the extension 
  section location field changes from `req_extensions` to `x509_extensions = bric3_ext`.

--------------------------------------------------------------------------------

Note the `subjectAltName = @alt_names` line again, the `@` allows to reference 
a _vertical_ list declared within the `alt_names` section. For example

```
[bric3_ext]
subjectAltName = DNS.1 : localhost, DNS.3 : arkey.fr, DNS.3 : *.arkey.fr, DNS.4 : arkey.pro, DNS.5 : *.arkey.pro, DNS.6 : blog, IP.1 : 127.0.0.1, IP.2 : ::1
```

is equivalent to

```
[bric3_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = arkey.fr
DNS.3 = *.arkey.fr
DNS.4 = arkey.pro
DNS.5 = *.arkey.pro
DNS.6 = blog
IP.1 = 127.0.0.1
IP.2 = ::1
```

There's however a few syntax changes
- the use of a colon `:` next to the the alternative name (`IP`, `DNS`, etc.)
  for the horizontal list, an equal sign `=` for the vertical one.

- the post-fix _index_ notation is optional for the horizontal list, but 
  mandatory in the vertical list.


--------------------------------------------------------------------------------

Finally they need to be packaged in order to use the certificate and the key 
with the JVM. Standard libraries of the JDK can load a [**J**ava **K**ey 
**S**tore or `P12` (**PKCS12**) file](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#KeyStore).

```sh
openssl pkcs12 \
    -export \
    -in bric3-openssl.crt \
    -inkey bric3-openssl.key \
    -passout pass:cadeau \
    -out bric3.p12
```

As a side note here's how to create a Java Keystore from a PKCS12 store :

```sh
keytool -importkeystore \
        -srckeystore bric3.p12 \
        -srcstoretype PKCS12 \
        -srcstorepass cadeau \
        -deststorepass the_password \
        -destkeypass the_password \
        -destkeystore bric3-openssl.jks
```


### Loading the generated certificate

In order to use the certificate wiremock need to be configured with the 
location of the key store (and of course in the client code).


```java
@Rule
public WireMockRule wireMockRule = new WireMockRule(wireMockConfig().dynamicPort()
                                                                    .keystorePath("./bric3.jks")
                                                                    .keystorePassword("the_password")
                                                                    .dynamicHttpsPort());

@Test
public void my_precious_self_signed_certificate() throws IOException {
    X509TrustManager compositeTrustManager = new CompositeX509TrustManager(
            trustManagerFor(readJavaKeyStore(Paths.get("./bric3.jks"), "the_password")),
            systemTrustManager());
    OkHttpClient okHttpClient = httpClient(sslContext(null,
                                                      new TrustManager[]{compositeTrustManager}),
                                           compositeTrustManager)
            .newBuilder()
            .build();
    try (Response response = okHttpClient.newCall(new Request.Builder().get()
                                                                       .url(format("https://%s:%d",
                                                                                   "localhost",
                                                                                   wireMockRule.httpsPort()))
                                                                       .build())
                                         .execute()) {
        // successfully established connection
    }
}
```

Notice the use of the JKS key store instead of the P12 file, while I just
wrote that 
[all JVM implementations are supposed to support PKCS12 type key store](https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#impl), 
what happened ? Unfortunately wiremock uses Jetty under the hood and 
[Jetty doesn't allow to be configured with a PKCS12 file]((https://wiki.eclipse.org/Jetty/Howto/Configure_SSL#Loading_Keys_and_Certificates_via_PKCS12)).
Meaning that our P12 file have to be converted in a **J**ava **K**ey **S**tore.

## Wrap up

This was the last piece of code of this blog entry. In this article I hope
you discovered how to manage simple self-signed certificates with Java,
and how to generate self-signed certificates with Java tools and a almost 
standard tool `openssl`. I also you better understand how Java architectured
their security infrastructure classes.

There's way more thing that needs to dive in with TLS, for example 
- how to build his own certificate authority
- how to handle a real certificate chain
- how to handle mutual authentication (clients is also sending his own certificate)
- how to use third party providers like [BouncyCastle](https://www.bouncycastle.org/java.html)

### Versions

This article has been elaborated with the following versions

* Java 1.8.0u144
* okhttp 3.9.0
* wiremock 2.8.0
* High Sierra / OSX 10.13 / Darwin 17.0.0 / 17A405
* openssl => LibreSSL 2.2.7 (High Sierra comes with LibreSSL)

### Some references

* https://www.openssl.org
* https://www.libressl.org
* http://wiki.cacert.org/FAQ/subjectAltName
* https://www.digitalocean.com/community/tutorials/how-to-create-a-ssl-certificate-on-apache-for-ubuntu-14-04
* https://tools.ietf.org/html/rfc5280
* https://tools.ietf.org/html/rfc2818
* https://tools.ietf.org/html/rfc5246
* https://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html






{% if notes %}

https://security.stackexchange.com/questions/107240/how-to-read-certificate-chains-in-openssl
https://security.stackexchange.com/questions/72077/validating-an-ssl-certificate-chain-according-to-rfc-5280-am-i-understanding-th/72085#72085
https://security.stackexchange.com/questions/83372/what-is-the-difference-of-trustmanager-pkix-and-sunx509
https://www.digitalocean.com/community/tutorials/how-to-create-a-ssl-certificate-on-apache-for-ubuntu-14-04
http://blog.palominolabs.com/2011/10/18/java-2-way-tlsssl-client-certificates-and-pkcs12-vs-jks-keystores/index.html
https://typesafehub.github.io/ssl-config/CertificateGeneration.html
https://stackoverflow.com/a/33844921/48136

https://stackoverflow.com/questions/906402/how-to-import-an-existing-x509-certificate-and-private-key-in-java-keystore-to-u/8224863#8224863
http://blog.endpoint.com/2014/10/openssl-csr-with-alternative-names-one.html
https://security.stackexchange.com/a/166645/30540

https://security.stackexchange.com/questions/73156/whats-the-difference-between-x-509-and-pkcs7-certificate
http://apetec.com/support/GenerateSAN-CSR.htm
https://gist.github.com/jchandra74/36d5f8d0e11960dd8f80260801109ab0
https://www.pixelstech.net/article/1420427307-Different-types-of-keystore-in-Java----PKCS12

{% endif %}
