tang(8) -- Network-Based Cryptographic Binding Server
=====================================================

## OVERVIEW

Tang is a service for binding cryptographic keys to network presence. It
offers a secure, stateless, anonymous alternative to key escrow services.

The Tang project arose as a tool to help the automation of decryption.
Existing mechanisms predominantly use key escrow systems where a client
encrypts some data with a symmetric key and stores the symmetric key in a
remote server for later retrieval. The desired goal of this setup is that the
client can automatically decrypt the data when it is able to contact the
escrow server and fetch the key.

However, escrow servers have many additional requirements, including
authentication (so that clients can't get keys they aren't suppossed to have)
and transport encryption (so that attackers listening on the network can't
eavesdrop on the keys in transit).

Tang avoids this complexity. Instead of storing a symmetric key remotely,
the client performs an asymmetric key exchange with the Tang server. Since
the Tang server doesn't store or transport symmetric keys, neither
authentication nor encryption are required. Thus, Tang is completely stateless
and zero-configuration. Further, clients can be completely anonymous.

Tang does not provide a client. But it does export a simple REST API and
it transfers only standards compliant JSON Object Signing and Encryption
(JOSE) objects, allowing you to create your own clients using off the shelf
components. For an off-the-shelf automated encryption framework with support
for Tang, see the Clevis project. For the full technical details of the Tang
protocol, see the Tang project's homepage.

## GETTING STARTED

Getting a Tang server up and running is simple:

    $ sudo systemctl enable tangd.socket --now

That's it. The server is now running with a fresh set of cryptographic keys
and will automatically start on the next reboot.

## CONFIGURATION

Tang intends to be a minimal network service and therefore does not have any
configuration. To adjust the network settings, you can override the
`tangd.socket` unit file using the standard systemd mechanisms. See
`systemd.unit`(5) and `systemd.socket`(5) for more information.

## KEY ROTATION

In order to preserve the security of the system over the long run, you need to
periodically rotate your keys. The precise interval at which you should rotate
depends upon your application, key sizes and institutional policy. For some
common recommendations, see: https://www.keylength.com.

To rotate keys, first we need to generate new keys in the key database
directory. This is typically `/var/db/tang`. For example, you can create
new signature and exchange keys with the following commands:

    # DB=/var/db/tang
    # jose jwk gen -i '{"alg":"ES512"}' -o $DB/new_sig.jwk
    # jose jwk gen -i '{"alg":"ECMR"}' -o $DB/new_exc.jwk

Next, rename the old keys to have a leading `.` in order to hide them from
advertisement:

    # mv $DB/old_sig.jwk $DB/.old_sig.jwk
    # mv $DB/old_exc.jwk $DB/.old_exc.jwk

Tang will immediately pick up all changes. No restart is required.

At this point, new client bindings will pick up the new keys and old clients
can continue to utilize the old keys. Once you are sure that all the old
clients have been migrated to use the new keys, you can remove the old keys.
Be aware that removing the old keys while clients are still using them can
result in data loss. You have been warned.

## HIGH PERFORMANCE

The Tang protocol is extremely fast. However, in the default setup we
use systemd socket activiation to start one process per connection. This
imposes a performance overhead. For most deployments, this is still probably
quick enough, given that Tang is extremely lightweight. But for larger
deployments, greater performance can be achieved.

Our recommendation for achieving higher throughput is to proxy traffic to Tang
through your existing web services using a connection pool. Since there is one
process per connection, keeping a number of connections open in this setup
will enable effective parallelism since there are no internal locks in Tang.

For Apache, this is possible using the `ProxyPass` directive of the `mod_proxy`
module.

## HIGH AVAILABILITY

Tang provides two methods for building a high availability deployment.

1. Client redundency (recommended)
2. Key sharing with DNS round-robin

While it may be tempting to share keys between Tang servers, this method
should be avoided. Sharing keys increases the risk of key compromise and
requires additional automation infrastructure.

Instead, clients should be coded with the ability to bind to multiple Tang
servers. In this setup, each Tang server will have its own keys and clients
will be able to decrypt by contacting a subset of these servers.

Clevis already supports this workflow through its `sss` plugin.

However, if you still feel that key sharing is the right deployment strategy,
Tang will do nothing to stop you. Just (securely!) transfer all the contents
of the database directory to all your servers. Make sure you don't forget the
unadvertised keys! Then set up DNS round-robin so that clients will be load
balanced across your servers.

## COMMANDS

The Tang server provides no public commands.

## AUTHOR

Nathaniel McCallum &lt;npmccallum@redhat.com&gt;

## SEE ALSO

`systemd.unit`(5), `systemd.socket`(5), `jose-jwk-gen`(1)

## FURTHER READING

* Clevis    : https://github.com/latchset/clevis
* Tang      : https://github.com/latchset/tang
* JOSE      : https://datatracker.ietf.org/wg/jose/charter/
* mod_proxy : https://httpd.apache.org/docs/2.4/mod/mod_proxy.html 
