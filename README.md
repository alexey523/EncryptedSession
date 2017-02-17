# Welcome to EncryptedSession

EncryptedSession is a drop-in replacement for the default Flask session cookie implementation, adding session data encryption to prevent sensitive session information from being leaked. See [this blog](https://blog.miguelgrinberg.com/post/how-secure-is-the-flask-user-session) for more details of how the default Flask session cookies works, but in brief the default implementation signs the session cookie to prevent tampering, but does encrypt it. The result is that a user or man-in-the-middle agent who captures the session cookies can quite simply decode the session data to plain text. That makes a Flask session fundamentally dangerous for storing any sensitive or valuable information.

EncryptedSession provides a simple drop-in replacement for the default Flask session cookie mechanism, and encrypts the session information using AES-256. Encryption is done by the excellent PyCryptodome package, which is a fork and continuation of the original PyCrypto package.

# Limitations

* Currently EncryptedSession only handles simple session variable types which are serializable using Python's JSON encoder. If you need to store binary data in your session then you will have to encode and decode the data in your application when reading or writing the session object.

# Why use EncryptedSession rather than server-side session storage ?

The general rule is that to make your session data secure you should not pass it around in a cookie, since a cookie is by definition passed between the server and client, so can be captured along the way. The usual approach to session storage is to use a unique session ID cookie as a server-side key. The server can then use this key for lookup in a database, a key:value store such as Redis, or any other suitable mechanism. In this model the session data itself is kept on the server and never passed onto the client.

However this approach imposes a requirement on the server for fast key:value storage. Although that can be simply implemented using some sort of in-memory key:value dictionary within the Flask application, that means that session data is local to that server and will not be available to other servers that may be part of a redundant server setup or a server farm configuration. The same is true for the simplest case of a relational database running locally on the server, where session data stored in that database is only visible on that server. If you have a server running as part of a server farm this means that the load balancer must be configured for _sticky sessions_ so that a client with a session on server 'A' is always routed to that server so that the session data is available.

A session implementation where the session data is passed around using cookies has the advantage that the session data is available on all servers as it is passed with each HTTP request.

# Why use server-side session storage instead ?

Firstly because of the insecurity of passing session data around in cookies, particular with the default Flask session cookie implementation. Even with the EncryptedSession implementation there is still the risk that someone who steals your secret encryption key can decrypt the session data, or that at some point an attack might be targeted at the encryption algorithm used.

Secondly because the session cookie increases the size of each HTTP request and response. How much depends on the size and nature of the session data, but if your application stores large data types in the session then passing that data across the network with every request is a very bad idea and server-side session storage is a much better option. However, for an application which stores a moderate amount of textual data the overhead of the session cookie may be acceptable.

## Installation

Since EncryptedSession depends on PyCryptodome you need to install that package for Python first:

`pip install pycryptodome`

then include `encrypted_session.py` in your project.

## Use

To replace the default Flask session cookie implementation with EncryptedSession you need to do only the following:

* Generate an 32 byte encryption key
* Import EncryptedSession into your application
* Store the encryption key against the Flask app
* Tell Flask to use the EncryptedSession implementation

### Generate the encryption key

EncryptedSession requires a 32 byte (256 bit) encryption key, which can be generated using `get_random_bytes()` from `Crypto.Random`. From the Python command line:

`>>> from Crypto.Random import get_random_bytes`

`>>> key = get_random_bytes(32)`

`>>> print (key)`

which will print a Python `bytes` which can be cut&pasted as your `crypto_key` value.

### Importing EncryptedSession 

`from encrypted_session import EncryptedSessionInterface`

### Store the encryption key against the Flask app

For example:

`app.crypto_key = b'\x8e;\xa9=\x11\xf7\r\xf9\x8d\x8a?\x1fM\xac\x94\xa8\xa2F]\x91s#Q\x07\x06\x99\xf2B\xab\x0c9S'`

Note that the default Flask cookie implementation uses `app.secret_key` for the signing key. EncryptedSession does not reuse `secret_key`, to make it easier to switch between implementations.

### Tell Flask to use the EncryptedSession implementation

Somewhere after the Flask app has been created:

`app.session_interface = EncryptedSessionInterface()`

and that's it.

# Notes

* Currently the implementation encrypts the session data using AES encryption in 'EAX' mode. This allows the receiver to detect unauthorized modification to the data, similarly to how `itsdangerous` is used for the default Flask session cookie implementation.

* Session data is compressed using `zlib` (Python Standard Library) if larger than 1KB. This is done at the point where the session data is a JSON serialized copy of the session dictionary, on the grounds that textual data is likely to be more compressible. In practice this seems to result in 1KB of session data compressing down to about half that size.


