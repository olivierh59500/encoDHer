[encoDHer] [1] - ephemeral Diffie-Hellman for email communications
===

  [1]: https://github.com/rxcomm/encoDHer/  "encoDHer"

### Introduction

encoDHer is a python utility to facilitate symmetric encryption of email
messages using the Diffie-Hellman (DH) protocol for generating a shared
secret between the sender and recipient of an email message.

Keys are stored in the database file keys.db.

encoDHer implements perfect forward secrecy (PFS) in email communications.
It is essentially ephemeral Diffie-Hellman for email, where dynamic key changes
are provided by the ```--mutate-key``` option in encoDHer.

### Why symmetric encryption? Isn't using public key encryption good enough if we use ```--throw-keyids``` as an option with GPG?

The biggest argument in favor of
symmetric encryption has to do with the difficulty required to generate new keys.  It is
much easier to generate a new set of DH keys than it is to generate a new pair of RSA keys,
for example.
So, when we want to rotate keys to preserve perfect forward secrecy, the key-generation
process is very quick.

An important second argument
in favor of using the DH key exchange protocol is that for
a given key length, DH protocols are stronger than public key protocols. The additional
strength comes from the fact that the best known algorithm for cracking either is the
[General Number Field Sieve] [2].  The matrix algebra part of this algorithm involves
a large matrix with bits as entries for public key protocols, and with large integers as
entries for DH protocols. In essence, this makes the DH protocols stronger by a factor
of the length of the integers in the matrix (the length of the large prime used in DH).
More detail on the above can be found
[here] [3].

  [2]: https://en.wikipedia.org/wiki/General_number_field_sieve "General Number Field Sieve"
  [3]: http://security.stackexchange.com/questions/35471/is-there-any-particular-reason-to-use-diffie-hellman-over-rsa-for-key-exchange" "here"

One further advantage of DH protocols is that both sender and receiver are automatically
authenticated.
It takes the sender's DH secret key and receiver's DH public key to encrypt a message
and the sender's DH public key and receiver's DH private key to decrypt the message.
In contrast, public key protocols only authenticate the receiver - anyone can encrypt
a message using the receiver's public key. While it is true that the sender can sign the
message to authenticate it, this requires a separate operation and is not a part of
the basic public key encryption protocol.

There are some additional minor advantages that arise due to the nature of the standards
specifying these two encryption methodologies - DH is free and the public-key algorithms
have historically been protected by patents.

The primary disadvantage of using DH for generating shared secrets is, of course,
that you need a separate DH secret key for each email route that you want to use.
EncoDHer provides a simple way to manage all of these keys.

EncoDHer also provides a mechanism for anonymous communication between parties.
In this case, communication takes place via the public usenet newsgroup mailbox
alt.anonymous.messages rather than by standard email.

### Installation requirements

The encodher package requires python-gnupg-0.3.5 or greater to function.
The latest version is documented at:

http://pythonhosted.org/python-gnupg/

Since version 0.3, encoDHer uses OpenSSL (through the M2Crypto library)
for key generation and Diffie Hellman parameter management.
Details regarding the use of OpenSSL are explained further
below in this README. We note the following for those desiring legacy operation:

* Legacy (pre-version 0.3) operation of encoDHer made use of the DiffieHellman() class from [Mark Loiseau] [4]. This Diffie Hellman class is still included in the distribution and
can be used by removing the dhparams.pem file.

 [4]: http://blog.markloiseau.com/2013/01/diffie-hellman-tutorial-in-python/ "Mark Loiseau"

Finally, this package requires the use of gnupg to sign DH public keys for
export and to identify DH public keys for import. You will need the
GPG secret key to sign your DH keys, and the receiver's GPG public key to
verify and import DH public keys.

You should also edit the constants.py file to reflect your parameters.

Install the encoDHer module using the command:

    sudo python setup.py install

from the encoDHer directory. An executable named encodher will be created
and copied to /usr/local/bin/encodher.  The setup.py command should also
automatically install python-gnupg with most linux variants if you have ```setuptools```
installed (for example, ```sudo apt-get install python-setuptools``` in ubuntu).

### Startup

To use the program, you must first initialize the keys database.  This is
accomplished with the following command:

    encodher --init

Once your database is created, you can generate keys for specific email routes
using the command:

    encodher --gen-key sender@example.org receiver@otherexample.org

where your email address is sender@example.org and your intended recipient's
email address is receiver@otherexample.org. Note that using the DH shared
secret protocol requires generating a DH secret key for each sender -> receiver
route.  So you will probably end up with several keys in the database.

### Key distribution

You can then export your signed DH public key with the command:

    encodher --sign-pub sender@example.org receiver@otherexample.org

The output will be a GPG signed version of your DH public key.  You can then
send that public key to a recipient over a non-secret channel.  An
anonymized key tablet will be developed later to facilitate DH public key
exchanges for the encodher utility, but any nonsecure channel will work.

When the recipient receives your DH public key, she can then import your
key into her database using the command:

    encodher --import textfile

where textfile is a text file containing your signed DH public key. Note
that the GPG public key used to sign your DH public key must be in her
keychain in order to verify the signature on the DH public key.

Your intended recipient should then export her signed DH public key and
send it to you.  You can then import her signed DH public key and you
are ready to communicate using symmetric key encryption with a DH shared
secret key.

### Message encryption and exchange

To encrypt a message, first create the plaintext in a file.  You can
then encrypt the contents of that message with your DH shared secret
key using the command:

    encodher --encode-email file sender@example.org receiver@otherexample.org

The email will be encrypted using symmetric AES256 encryption and output
to file.asc.  The file is an ascii-armored pgp-format file.

Upon receipt of the encrypted message, the receiver can then decrypt the
file using the command:

    encodher --decode-email file.asc sender@example.org receiver@otherexample.org

The plaintext message will then be displayed.  It is probably good to note
here that the first email address is always the message sender, and the second
is always the receiver.  So for example, if I received a message at
alice@alice.com from my buddy bob@bob.com, the correct syntax to decrypt the
message would be:

    encodher --decode-email file.asc bob@bob.com alice@alice.com

because bob was the sender of the email and alice was the receiver.  When alice
encrypts an email to bob, the correct syntax is:

    encodher --encode-email file alice@alice.com bob@bob.com

because in this case alice is the sender and bob the receiver.

### Key cloning

It is possible to clone your DH secret key to another route. This functionality
is useful when, for example you want to post your DH public key on a
key table or other public place to enable multiple people to send you
encrypted email.  You clone keys by using the command:

    encodher --clone-key old1@first.org old2@second.org new1@third.org new2@fourth.org

This will clone your secret key from the route old1@first.org -> old2@second.org
to the new route new1@third.org -> new2@fourth.org.  When someone wants
to use your posted DH public key to send you encrypted email, they will send
you their DH public key over any insecure channel.  You can then import their
public key and the new route will have a completely different shared secret
than the old route does.

### Other options

Various options for key management exist.  The following list of options
can be obtained by executing encodher without any arguments.

    Options:
     --init, -i: initialize keys.db database
     --import, -m: import signed public key
     --mutate-key, -a: mutate DH secret key
     --sign-pub, -s: sign your own public key
     --change-toemail, -t: change toEmail on key
     --change-fromemail, -f: change fromEmail on key
     --change-pubkey, -p: change public key for fromEmail -> toEmail
     --encode-email, -e: symmetrically encode a file for fromEmail -> toEmail
     --decode-email, -d: symmetrically decode a file for fromEmail -> toEmail
     --list-routes, -l: list all routes in database
     --gen-secret, -c: generate shared secret for fromEmail -> toEmail
     --gen-key, -n: generate a new key for fromEmail -> toEmail
     --get-key, -g: get key for fromEmail -> toEmail from database
     --fetch-aam, -h: fetch messages from alt.anonymous.messages newsgroup
     --clone-key, -y: clone key from one route to another
     --rollback, -b: roll back the a.a.m last read timestamp
     --change-dbkey, -k: change keys.db encryption key

### Perfect forward secrecy

The primary reason for using symmetric encryption with DH shared secrets
for email exchanges is to provide for perfect forward secrecy (PFS).  If, after
an exchange, the DH keys are destroyed by both parties, any messages
encrypted with these keys can no longer be read, achieving PFS. Users can
change their keys to take advantage of PFS by executing:

    encodher --mutate-key sender@example.org receiver@example.org

This will destroy sender@example.org's old DH secret key, generate a new
DH secret key and export the corresponding new DH public key encrypted with the
old DH shared secret.  This encrypted file can be emailed to the receiver,
who can then decrypt the file and import the key using the ```--import``` option.
If an unencrypted version of the new signed DH public key is desired,
the ```--sign-pub``` option can be used. Once the new DH public key has been
imported by the receiver, no one (including the original sender and receiver)
will be able to read the old messages. Change keys carefully.

### Anonymous communication

If anonymous communication is desired, encodher presents an option to encode
the email for transmission using the mixmaster network.  Once the appropriate
mixmaster headers are added (by answering affirmatively to the question about
sending anonymously), the message can be dispatched to the
alt.anonymous.messages newsgroup with the command:

    mixmaster -c 1 < message.asc

where message.asc contains the encrypted message and plaintext headers. Note
that you must have the mixmaster https://github.com/crooks/mixmaster package
installed and stats updated for this to work. Most linux flavors have this
package available.

To receive messages sent to you at a.a.m using a DH key of yours, you can
execute the command:

    encodher --fetch-aam

This will fetch and decrypt all a.a.m messages for each of the keys in your
database.

### Building an executable

To install the executable, change to the encoDHer directory and execute the command:

    sudo python setup.py install

This will create the executable named 'encodher' and copy it to /usr/local/bin/encodher.
You will need the python-gnupg library
(version 0.3.5 or greater - see the link above) before the executable will run.
This should be automatically installed by setup.py for most linux variants.

If you don't want to download the entire source, the executable should run on
most linux systems without any changes.  So you can grab a copy using wget:

    wget https://github.com/rxcomm/encoDHer/raw/master/encodher

make it executable:

    chmod +x encodher

and it should run if you have python-gnupg installed. To extract the python source
from the executable, rename it as a .zip archive and unzip it, e.g.:

    mv encodher encodher.zip
    unzip encodher.zip

### encoDHer with OpenSSL

Since version 0.3, encoDHer has used OpenSSL for key generation. 
The Diffie-Hellman parameters are imported from an OpenSSL dhparams.pem file.
encoDHer interacts with OpenSSL through the M2Crypto library.
In most linux systems, the M2Crypto library can be installed through the
command:

```
sudo apt-get install python-m2crypto
```

or similar, depending on your particular flavor of linux.
If M2Crypto is not present, encoDHer will use a version of the
DiffieHellman() class that contains the same prime, generator, and keysize
that is distributed in dhparams.pem. If the dhparams.pem file is not present, encoDHer will
gracefully roll back to the pre-version 0.3 DiffieHellman() class from Mark Loiseau.

The dhparams.pem file that is distributed with encoDHer uses an 8192 bit
prime and a generator of 5.  This file is NOT secret, and may be re-used.

*For* *the* *truly* *paranoid* - if you don't trust the dhparams.pem file provided by
me and want to replace it with your own, the command to generate
a new set of DH parameters with an 8192 bit prime is:

```
openssl dhparam -[2,5] -out dhparams.pem 8192
```

Using 2 in the command will force a generator of 2, and 5 will force a
generator of 5. Keep in mind that everyone you communicate with needs to use
the same set of DH parameters, so you will need to distribute those somehow.
Smaller prime sizes (4096, 2048, 1024) will also work, although you will see
a lot of zeros in your exported DH public keys.

To find the prime in your new dhparams.pem file, execute the command:

```
./dh_m2crypto.py
```

This will run a key-matching test, and both the prime and generator are output
as a part of this test.  If you do change dhparams.pem, you should also edit
the prime and generator definitions in dh\_pydhe.py to reflect your new prime and
generator. The prime should be copied directly (incluing the 0x hex prefix).
If you change the prime size, you should also change the DH private key size in
the \_\_init\_\_() method of dh\_pydhe.py to match your new prime.
