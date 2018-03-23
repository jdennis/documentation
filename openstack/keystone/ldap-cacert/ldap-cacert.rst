This bug report has several unrelated issues. To help engineering it
is important to limit a bug report to just one issue. This bug and
every bug cloned from it should be split to address the specific issue.

Let me try to address the topic of the bug "SSL connection to active
directory in keystone works not with crt file or tls_cacertdir option"
which is not what most of the bug comments are discussing.

I plan on turning this material into a properly formatted document
for publication as a Knowledge Base article.

-------------------------------------------------------------------------

At this juncture there is little reason to believe the tls_cacertfile
and tls_cacertdir options do not work. Of course there is always the
possibility there is a bug with these options but it is much more
likely the failure is caused by a user configuration
problem. Therefore the best approach is to first verify the CA cert
the customer is using works with their AD instance *outside* of
Keystone. This will help identify if the problem is with the data
being fed to Keystone or if Keystone is failing to use the data
correctly. For the impatient skip to "Diagnosing the Problem".

Background Information
======================

Certificate Formats
-------------------

Certificates are binary data but can be represented in a textural format
called PEM. The binary form is called DER (Distinguished Encoding
Rule) from the ASN.1 binary encoding specification. The SSL/TLS
protocol, as well as other binary protocols, always exchange certs in
the binary DER format on the wire. However, binary data can be awkward
to store and transmit in other contexts. To solve this problem the PEM
format was introduced which encodes the binary data as base64 text
wrapped with a header and footer around the base64 text. The header
declares what type of data will be found in the base64 data and the
footer marks the end of the base64 data block. A cert in PEM format
will look like this (in our examples we are using the ca-cert-768.pem
file from the openssl test data):

$ cat ca-cert-768.pem 
-----BEGIN CERTIFICATE-----
MIICRDCCASygAwIBAgIBAjANBgkqhkiG9w0BAQsFADASMRAwDgYDVQQDDAdSb290
IENBMCAXDTE2MDMyMDA2MjcyN1oYDzIxMTYwMzIxMDYyNzI3WjANMQswCQYDVQQD
DAJDQTB8MA0GCSqGSIb3DQEBAQUAA2sAMGgCYQC3wNLc1A9gAjz1H94ozPrLOhE2
R8c6RQjkUIALCOuw8xbZV+AEDSqP11Bw8MVzvmpksR9s1idJhLOugwMNTHfTXJjV
DWoQh9ofR51J5sOph4yDhQBXRmiuvqMDj+a81UkCAwEAAaNQME4wHQYDVR0OBBYE
FKrzei/LKJop6yShiJupKskW0ZQcMB8GA1UdIwQYMBaAFI71Ja8em2uEPXyAmslT
nE1y96NSMAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAFr4hjVtLuZz
gxLILAOREEtanckfnapUrhTLukog9Q8uzqMUE+YDEhkcP4YAVjcab6HaXrbcxXsn
zn+v+GPszD9G3doGbUjuwEEAHz+k/9sjsn8QAGw/XslYhd5dktaRRCqaTNiWT+Ks
xKntAsgXcgWNIpvGikzTB/W7IrjIV8/S1JjLABtoY88tFUX81Ohr3bFFsRc9EHVS
MtGnEwfoBOSlCUjaTWBNHHi1HstK9sG2SNT/nhN1HATk/aiCiQRKr/bm6ezPC2If
6mRidaNiQN8+vzvtn86BqtRJOEi8jj5CBax6IqwfE+lDZIwT7H9C9Cu8Yp4mTM0x
wwzRDnFVisM=
-----END CERTIFICATE-----

The header and footer look like this and identify the base64 data as
being an X509 certificate.

-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----

We can ask openssl to show us the contents of the cert in human
readable format by using the x509 command in the openssl command line
tool.

$ openssl x509 -text -inform PEM -in ca-cert-768.pem 
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 2 (0x2)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = Root CA
        Validity
            Not Before: Mar 20 06:27:27 2016 GMT
            Not After : Mar 21 06:27:27 2116 GMT
        Subject: CN = CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (768 bit)
                Modulus:
                    00:b7:c0:d2:dc:d4:0f:60:02:3c:f5:1f:de:28:cc:
                    fa:cb:3a:11:36:47:c7:3a:45:08:e4:50:80:0b:08:
                    eb:b0:f3:16:d9:57:e0:04:0d:2a:8f:d7:50:70:f0:
                    c5:73:be:6a:64:b1:1f:6c:d6:27:49:84:b3:ae:83:
                    03:0d:4c:77:d3:5c:98:d5:0d:6a:10:87:da:1f:47:
                    9d:49:e6:c3:a9:87:8c:83:85:00:57:46:68:ae:be:
                    a3:03:8f:e6:bc:d5:49
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                AA:F3:7A:2F:CB:28:9A:29:EB:24:A1:88:9B:A9:2A:C9:16:D1:94:1C
            X509v3 Authority Key Identifier: 
                keyid:8E:F5:25:AF:1E:9B:6B:84:3D:7C:80:9A:C9:53:9C:4D:72:F7:A3:52

            X509v3 Basic Constraints: 
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         5a:f8:86:35:6d:2e:e6:73:83:12:c8:2c:03:91:10:4b:5a:9d:
         c9:1f:9d:aa:54:ae:14:cb:ba:4a:20:f5:0f:2e:ce:a3:14:13:
         e6:03:12:19:1c:3f:86:00:56:37:1a:6f:a1:da:5e:b6:dc:c5:
         7b:27:ce:7f:af:f8:63:ec:cc:3f:46:dd:da:06:6d:48:ee:c0:
         41:00:1f:3f:a4:ff:db:23:b2:7f:10:00:6c:3f:5e:c9:58:85:
         de:5d:92:d6:91:44:2a:9a:4c:d8:96:4f:e2:ac:c4:a9:ed:02:
         c8:17:72:05:8d:22:9b:c6:8a:4c:d3:07:f5:bb:22:b8:c8:57:
         cf:d2:d4:98:cb:00:1b:68:63:cf:2d:15:45:fc:d4:e8:6b:dd:
         b1:45:b1:17:3d:10:75:52:32:d1:a7:13:07:e8:04:e4:a5:09:
         48:da:4d:60:4d:1c:78:b5:1e:cb:4a:f6:c1:b6:48:d4:ff:9e:
         13:75:1c:04:e4:fd:a8:82:89:04:4a:af:f6:e6:e9:ec:cf:0b:
         62:1f:ea:64:62:75:a3:62:40:df:3e:bf:3b:ed:9f:ce:81:aa:
         d4:49:38:48:bc:8e:3e:42:05:ac:7a:22:ac:1f:13:e9:43:64:
         8c:13:ec:7f:42:f4:2b:bc:62:9e:26:4c:cd:31:c3:0c:d1:0e:
         71:55:8a:c3

Lets look at what the arguments mean:

openssl x509
    This tells openssl we want to do x509 certificate operations.

-text
    This says to output the data in human readable form

-inform PEM
    -inform means "input format". Here we are indicating the input
    data is in PEM format.

-in ca-cert-768.pem
    -in is the input filename, in our case its the ca-cert-768.pem
    file.

The other common certificate format is DER binary data. openssl x509
can convert between the DER and PEM formats. To convert the PEM
formatted data in the file ca-cert-768.pem to DER binary and output it
to the file ca-cert-768.der we can do this:

$ openssl x509 -inform PEM -in ca-cert-768.pem \
  -outform DER -out ca-cert-768.der

The two new arguments used here are:

-outform DER
    -outform means "output format" Here we are indicating the output
    data will be DER binary data.

-out ca-cert-768.der
    -out is the output filename, in our case its the ca-cert-768.der
    file.

Now we have two files representing the same CA certificate but in two
different formats, PEM and binary. The file command tries to identify
the contents of a file, lets see what it tells us about these two
files:

$ file ca-cert-768.pem ca-cert-768.der 
ca-cert-768.pem: PEM certificate
ca-cert-768.der: data

Earlier we used the -text argument to see a human readable dump of the
certification in PEM format. We can do the same thing on the DER
formatted file by specifying DER for the -inform, for example:

$ openssl x509 -text -inform DER -in ca-cert-768.der

and you'll get the exact same dump as before.

Certificate Filename Extensions
-------------------------------

In practice you'll see certificates with a variety of filename
extensions, two common ones are .cer and .crt. Unfortunately these two
filename extensions are often used for *both* DER binary format and
PEM text format. Therefore you cannot tell just from the filename
extension what format the file is in. We saw earlier how the `file`
command can be used to ascertain the file format.

These web articles give additional information on certificate file
formats, how to convert them and how they might be named:

Various SSL/TLS Certificate File Types/Extensions
https://blogs.msdn.microsoft.com/kaushal/2010/11/04/various-ssltls-certificate-file-typesextensions/

DER vs. CRT vs. CER vs. PEM Certificates and How To Convert Them
https://support.ssl.com/Knowledgebase/Article/View/19/0/der-vs-crt-vs-cer-vs-pem-certificates-and-how-to-convert-them

Certificate filename extensions
https://en.wikipedia.org/wiki/X.509#Certificate_filename_extensions

Only Use PEM Format
-------------------

Most tools and libraries that accept certificates in files expect they
will be formatted in PEM. This includes OpenStack and OpenLDAP
(OpenStack uses OpenLDAP to connect to LDAP servers). Therefore you
should *always* use PEM when creating a file containing a certificate
to be read by an X509 library.

CA Certs and Cert Chains
------------------------

This article is mostly focused on providing a CA cert used when validating
a TLS connection (e.g. tls_cacertfile). There are many types of X509
certificates, but the cert passed in tls_cacertfile *must* be a CA
cert. So how do you determine if the cert is indeed a CA cert?

A CA cert will have at a minimum the "Basic Constraints"
extension. The "Basic Constraints" extension includes a boolean flag
named "CA". That flag *must* be TRUE for the cert to be a CA cert. See
the above CA cert dump for an example.

It is possible to have a CA hierarchy where subordinate CA's are
children of a parent CA. Each subordinate CA certificate is signed by
it's parent CA. A certificate chain exists when you have one or more
levels of subordinate CA's. To validate a cert signed by a
subordinate CA one must first validate the subordinate CA cert and
then walk the "chain" upwards validating each parent CA cert in
succession until you reach a root certificate (sometimes referred to a
trust anchor). A root certificate has no parent and is sometimes
called a self-signed cert. You may be familiar with self-signed certs
you generate yourself, the difference between a self-signed cert you
generate and a root CA cert is the root CA cert has the CA flag set,
in other words it is a CA. These root CA certs are extremely important
because they provide the entire trust for PKI. Your operating system
(and possibly your browser) maintains a set of trusted CA certs which
are known to be valid. Certificates do not pass the validation check
unless the final cert checked in the cert chain is one of these
trusted CA's.

Thus when you pass a CA cert via the tls_cacertfile it *must* also
contain any intermediate certs in the chain until they terminate in a
trusted root anchor.

Explicitly passing a CA cert or CA cert chain to a PKI component
(e.g. TLS) is equivalent to saying "I trust this CA" as if it were
present in your host operating systems set of trusted CA's.

Two Kinds of LDAP TLS
---------------------

TLS encrypted connections to an LDAP server can be done in one of two
ways, using the ldaps URI scheme or the StartTLS extended
operation. The difference is how the TLS connection is
established. With the ldaps scheme an indpendent port (default 636)
for LDAP over TLS is utilizied which is different than the standard
LDAP port (default 389). With ldaps a TLS connection is first
established with the host and then this TLS connection carries the
LDAP protocol. The disadvange is it requires a seperate port.

The advantage of StartTLS is the default LDAP port of 389 can still be
used. A non-encrypted connection is first established using the
conventional LDAP port. Then a special StartTLS command is issued over
the non-encrypted connection and a TLS connection handshake is then
established. No special port is required.

StartTLS is the preferred mechanism.

The choice between ldaps and StartTLS in Keystone is controlled by
the `use_tls` configuration option. When `use_tls` is True then
StartTLS will be used instead of ldaps. The Keystone `use_tls`
default is False.

Diagnosing the Problem
======================

What Can Go Wrong Using the tls_cacertfile Parameter?
-----------------------------------------------------

If the CA cert file you specify in tls_cacertfile is not working it
could be due to any one of these reasons:

* The pathname you provided does not exist.
* The file is not in PEM format
* The certificate is expired.
* The cert is not a CA cert.
* The CA cert is a subordinate CA but the intermediate certs in the
  chain are missing.
* The file is not readable (wrong permissions, bad SELinux labeling
  that prevents the process from being able to read the file resulting
  in an AVC denial).
* The `use_tls` (e.g. StartTLS) configuration option is incompatible
  with your LDAP host URI.
* The port you specified in your LDAP host URI is incompatible with
  your choice of ldaps vs. StartTLS, and/or it's just an incorrect
  port. 

The above background material gives you enough information to answer
these questions.

How Can You Verify Your CA Cert Works With AD?
----------------------------------------------

If you are having trouble getting Keystone LDAP to successfully
connect to your Active Directory server using TLS one of the best
diagnostics you can perform is to removed Keystone from the equation
and simply verify the file you pass to Keystone tls_cacertfile will
work with any other LDAP tool. The OpenLDAP command line tool
`ldapsearch` is a good choice. Using `ldapsearch` you can point it at
your AD server and ask that it make a TLS connection and return some
innocuous information that does not require any authentication or
authorization. We are just trying to test the connection in this
exercise. Try this:

$ LDAPTLS_CACERT=$CA_CERT_FILE ldapsearch -x -ZZ -H $LDAP_URL \
  -s base -b "" "objectclass=*" currenttime

Where CA_CERT_FILE and LDAPURL are respectively the same values
specified for tls_cacertfile and url in the ldap section of your
keystone.conf file.

Here are what the command arguments mean:

-x
    Use simple authentication

-ZZ
    Issue StartTLS and require that StartTLS be successful (do not use
    with an ldaps:: URI scheme)

-H $LDAP_URL
    The internet address of your LDAP server, either a hostname,
    hostname:port, or a URI that includes a ldaps scheme
    (e.g. ldaps::active_directory.example.com). Do not use -ZZ with
    ldaps.

-s base
    -s specifies the LDAP search type.

-b ""
    -b specifies the search base, here it is empty.

 "objectclass=*"
     Any type of object will be returned.

currenttime
    The name (i.e Distinguished Name or DN) of what we are searching
    for. Here it is the current time value.

If the search is successful you know the CA cert you specified is
working. You should get back something that looks like this:

dn:
currentTime: 20141022050611.0Z

If the search fails you can get more diagnostic information by adding
the -v verbose argument that dumps diagnostics to stdout or even the
-d debuglevel argument. Adding the -v verbose flag is probably a good
idea to assure a TLS connection was actually established.

    

