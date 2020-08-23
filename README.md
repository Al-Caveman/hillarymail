<p align="center">
    <img src="pics/logo.png">
</p>

----

# hillarymail: draft zero

## goals:
- should be easy to install/use.
- should not need to assume trusty vps admin.

## non-goals:
- compatibility with smtp/pop/imap/idiots that can't use address books or
  bitcoin.


## tasks hillarymail should do
- submit a new mail.
- relay a received mail.
- get new mail(s) from server.
- delete mails.
- set/unset tags to mails (e.g. read, unread, deleted, work, personal,
  important, urgent, gentoo-user, etc).
- whitelist/blacklist senders.


## how hillarymail mails look like in mid-air
when we throw hillarymail mailballs at each other, each ball is seen as 4
files:

    parties.json
    msg.zip.enc
    hash.txt
    hash.txt.sig

this also means that when a person submits a new hillarymail mail, he does
not look any different than a relay.  because mails are always like that,
be it newly submitted or relayed.

the voodoo happens inside `msg.zip.enc` where things get recursively
onioned until you reach the actual messages.  this onioning is nice, so
that you don't need to trust that each relay is honest in not changing
previous relays' headers.

key fun things for those with limited time:
- [here](#wrapping-a-mail) shows how the recursive onioning is made by each
  server/relay.  this onioning also makes it impossible for relays in the
  middle to edit/remove any information that previous relays have inserted.
  basically, if a user gets a mail, it guarantees that no relay could
  _independently_ make a lie about a previous relay.
- [here](#composing-a-new-message) shows how the final message looks like
  (after fully unwrapping the onion).  this is also part of a happy story
  that reveals how your mails belong only to you.
- [here](#get-new-mails-from-server) shows how mails can be served as
  static files over https through standard web servers like nginx.
- [here](#whitelistblacklist-senders) shows a neat and fair solution to
  spam.  this way, "spam" ceases being spam, but becomes a fair trade.


## authentication
user with public key `PK` (be it an end user wanting to send an email, or a
server wanting to relay to another server) wants to authenticate to HOST:

    RESP = https://HOST/hillarymail/get_challenge?user=sha3_224(PK)

where:

    RESP = {"status": some int code (0 = ok),
            "challenge": "some random string"}

then authentication code is `ANS` in later requests:

    ANS = asym_enc(RESP["challenge"], sender_privatekey)


## deauthentication
this will cause the server to expire the ANS session:

    RESP = https://HOST/hillarymail/bye?auth=ANS

client has the responsibility to deauthentication asap.


## composing a new message
this happens when a hillarymail client creates a new message to send it to
another person.  1st we start with this is the directory layout:

    parties.json
    msg/
        header.json
        body.txt
        attachments/
            lol.png
            app.tar.gz
            puppy_run_green_grass.mp4
    hash.txt
    hash.txt.sig

where:

    parties.json = {
        "from": [HOST1, PK1, CKEY1],
        "to"  : [[HOST21, PK21, CKEY21], [HOST22, PK22, CKEY22], ...],
        "cc"  : [[HOST31, PK31, CKEY31], [HOST32, PK32, CKEY32], ...],
        "bcc" : [[HOST41, PK41, CKEY41], [HOST42, PK42, CKEY42], ...]}
    HOSTi        = end-to-end address (e.g. ip, domain, onion address)
                   where end user's mailbox exists.
    PKi          = public key of ith party.  this is what uniquely
                   identifies the person.
    CKEYi        = shared key encrypted to the ith public key PKi.
    KEY          = securely generated random string used as a symmetric
                   cipher to encrypt the message.
    header.json  = {"subject" : "hey look at my new puppy!",
                    "time"    : 1598253247}
    body.txt     = actual message in utf-8.  only text is supported for
                   body text.  no self-hating formats like html/js/css/etc.
                   utf-8 is already fancy enough and has lots of emojies.
    hash.txt     = sha3_512(parties.json
                            + msg/header.json
                            + msg/body.txt
                            + msg/attachments/lol.png
                            + msg/attachments/app.tar.gz.
                            + msg/attachments/puppy_run_green_grass.mp4)
    hash.txt.sig = asym_enc(hash.txt, sender_privatekey)

to send this message structure, we need to pack it somehow.  we do it by
compressing `msg` directory into a zip file, and encrypting it:

    msg.zip.enc = sym_enc(zip(msg), KEY)

in summary, now we have the mail fully packed.  so we got these files:

    parties.json
    msg.zip.enc
    hash.txt
    hash.txt.sig

so it's part of the standard that each mail is comprised of 4 files with
the same names.  e.g. one cannot pick other file names.  case sensitive.

finally, we submit the files to some hillarymail server.  it could be our
own local hillarymail server, or it could be directly the hillarymail
servers of the recipients.  this is a decision that we need to make as
users.  most people would probably prefer using a local hillarymail server
in order to centralize the backup of their hillarymail messages.


## wrapping a mail
this happens when a hillarymail server receives a hillarymail mail (either
from a sender, or from another hillarymail server (aka relay).  

hillarymail servers will wrap the received mails as follows:

    parties.json     # new parties header
    msg/
        server.json  # new (optional)
        parties.json # as received
        msg.zip.enc  # as received
        hash.txt     # as received
        hash.txt.sig # as received
    hash.txt     # newly calculated
    hash.txt.sig # newly signed

where

    server.json = {"note" : "sender is our salesman", # optional
                   "time" : 1598253248}               #
    parties.json = {
        "from" : [HOST_server, PK_server, CKEY_server], # note:  "from" is
                                                        # updated to be the
                                                        # server's.

        "to"   : [[HOST22, PK22, CKEY22], ...],  # note: destinations with
        "cc"   : [[HOST32, PK32, CKEY32], ...],  # local hillarymail boxes
        "bcc"  : [[HOST42, PK42, CKEY42], ...]}  # are removed, since they
                                                 # are delivered by this
                                                 # local mail server.  so
                                                 # no need to put them into
                                                 # the new parties list.
                                                 # the parties list will
                                                 # shrink as relays keep
                                                 # forwarding the message.
                                                 #
                                                 # plus, if this server
                                                 # will use multiple other
                                                 # relays for different
                                                 # receipients, the server
                                                 # will need to create
                                                 # multiple `parties.json`
                                                 # suitable for each next
                                                 # relay.
    CKEYi        = NEW shared key encrypted to the ith public key PKi.
    KEY          = NEWLY securely generated shared password.
    hash.txt     = sha3_512(parties.json
                            + msg/parties.json
                            + msg/msg.zip.enc
                            + msg/server.json)
    hash.txt.sig = asym_enc(hash.txt, server_privatekey)

then we pack the hillarymail message similarly into an encrypted zip file:

    msg.zip.enc = sym_enc(zip(msg), KEY)


so now we have these files again (except modified and wrapped):

    parties.json # the new one with updated "from", removed delivered
                 # destinations and updated CKEYi with newly generated KEY.

    msg.zip.enc  # the newly wrapped one
    hash.txt     # new hash
    hash.txt.sig # new sign

actually, depending on how many relays we'd like to talk to, we will have
several different versions of the `parties.json` file (each with its own
recipients).  meaning while we will have a single `msg.zip.enc`, we will
have several `parties.json`, `hash.txt`, and `hash.txt.sig` for each next
server.

finally, we submit the mail to the next HOST(s) depending on the routing
table of the server.  the next HOST(s) could be the final destination
server where the parties hillarymail boxes exist, or could be other relays.


## submit a mail
here, we talk about how to submit a mail, regardless of how it is created.
so, this part is consistent everywhere throughout hillarymail, be it
submitting a new mail, or submitting a received mail by a hillarymail
relay.

1st [authenticate](#authentication) to get session `ANS`.

then check if we can submit a mail:

    RESP_1 = https://HOST/hillarymail/can_submit?
        & msg_id    = ID
        & parties   = parties.json
        & cmsg_size = size(msg.zip.enc)
        & auth      = ANS

if cool (i.e. `RESP_1["status"] = 0`), then actually send it:

    RESP_2 = https://HOST/hillarymail/submit?
        & msg_id  = ID
        & parties = parties.json
        & cmsg    = msg.zip.enc
        & hash    = hash.txt
        & sig     = hash.txt.sig
        & auth    = ANS

where

    ID = some unique randomly generated string to uniquely identify this
         message for the purpose of loop detection.  e.g. relays will drop
         the mail if they have seen a message with ID that is seen in the
         past, say, 1 month.

further error checking can be done by reading `RESP_2["status"]`.  but
let's not discuss this now since this is a quickie draft.  e.g. delivery
failures for some recipients should trigger a notification message to the
sender informing him of it.

finally, [deauthenticate](#deauthentication) the `ANS` session.


## relay a received mail
always check if "from" `PK` @ `HOST` is allowed to submit to us a mail, has
passed the challenge, all recipients are permitted to receive the message,
check if file sizes are cool (during `can_submit`), and also check if
message `ID` is not seen in the past, say, month (loop detection).  else
return error and exit.

if things are cool, then:

* if `"from": [HOST, PK, CKEY]` corresponds to a local hillarymail box,
  then copy the message into the `sent` directory under him:
    ```
    USER/sent/i/
        parties.json
        msg.zip.enc
        hash.txt
        hash.txt.sig
    ```
    where `USER = sha3_224(PK)` and `i` is a logical clock (counter) such
    that, for any positive `n`, mail `i+n` implies that it is older than
    mail `i`.

* if any parties in "to", "cc" or "bcc" exist locally, then store their
  mails locally in the `inbox` dir:
    ```
    USER/inbox/i/
        parties.json
        msg.zip.enc
        hash.txt
        hash.txt.sig
    ```

* for any non-local parties (those on other servers) consult the routing
  table, and figure out the right next-hop relays for them.  this may lead
  to several `parties.json` files (and corresponding `hash.txt`s and
  `hash.txt.sig`s).

  1. wrap the mail [as discussed here](#wrapping-a-mail).
  2. submit the mail to the next hop [as discussed here](submit-a-mail).


## get new mail(s) from server
if client did not receive any mail so far, it will have `i=0`.  if it had
received emails in the past, then `i` will be updated to match the counter
of the last received mail.

1st [authenticate](#authentication) to get session `ANS`.

then, obtain list of new mails:

    RESP = https://HOST/hillarymail/check_new?
        & since    = i
        & box      = "inbox" # or "sent" if client wnats to sync sent items
        & answer   = ANS

where `RESP` contains a list of counters that represent all mails newly
received after the logical clock `i`.  to be exact, suppose `i=4`:

    RESP = {"status": some int code (0 = ok),
              "new_mails": [5, 6, ...]}  # note: list doesn't have to be
                                         # sorted, and items don't have to
                                         # be continuous.  e.g.:
                                         # [500, 8, 100, ...] is possible

then the hillarymail client will send requests to download all files for
those mails:

```python
files = ['parties.json', 'msg.zip.enc', 'hash.txt', 'hash.txt.sig']
for i in RESP["new_mails"]: # these 2 loops can be parallelized
    for f in files:         #
        url = f'https://{HOST}/hillarymail/get_mail?' \
              f'box={box}&i={i}&file={f}&auth={ANS}'
        saveto = f'~/.hillarymail/caveman/{box}/{i}/{f}'
        download(url, saveto)
```

the hillarymail server will verify `ANS`, and then simply start sending
those requested files as mere _static_ files using efficient functions like
`sendfile(2)`.

if hillarymail is a uwsgi web application running behind a normal web
server, like nginx, then hillarymail can use
[`X-Accel-Redirect`](https://www.nginx.com/resources/wiki/start/topics/examples/xsendfile/).
this way files are transferred by nginx without hillarymail's involvement
(hillarymail only authenticates `ANS` and then approves by sending
`X-Accel-Redirect: /protected/USER/inbox/i/whatever_file`).

finally, [deauthenticate](#deauthentication).

then what?  then you have your own private key, and you can unwrap the
onion recursively until you reach the final message.  you can see the
entire journey of _your_ mail with high confidence of not being tampered
with unfairly, not even the header information injected by past relays.  in
fact, next relays won't even know what previous relays have injected in the
onion.  heck, they wouldn't even know if the relay is relaying a mail or
sending a new mail of its own.


# delete mails
1. `MAILS = {"sent": [1, 5, 100, 7], "inbox": [99, 5, 1]}`.
2. [authenticate](#authentication) to get session `ANS`.
3. `RESP = https://HOST/hillarymail/delete?mails=MAILS&auth=ANS`
4. [deauthenticate](#deauthentication).


# set/unset tags to mails
this is a local business.  the server has nothing to do with it.  the
client locally should have a rule that tags things locally.  as mails come,
or as whatever however the owner of the mails pleases.

if user has several machines where he wishes to sync his tags, then he
_actually_ needs to synchronize his tag-assignment policy/configuration.
so this is basically a configuration synchronization problem.  some users
like to synchronize their configs via `git`, some may like `rsync`, some
may like something else.

at least for now there is no tagging synchronization.  we'll see how this
goes in the future.


# whitelist/blacklist senders
this is also none of server's business.  basically what will happen is
this:
- the server has no clue who can, or cannot, message you.  the server
  stupidly simply throws messages at you.  in fact, the server is morally
  _forbidden_ to filter anything.
- but how will you filter spam, right?  the answer is that your client will
  have a local whitelist containing those that can message you.  the
  _default_ is that everyone is blacklisted unless they pay you some
  bitcoin.  let's be more specific:
  - if you had already messaged a person, the person will get automatically
    whitelisted (you can remove him from the whitelist him later on if you
    end up regretting it).
  - if a person, who is not whitelisted by your local hillarymail client,
    messaged you, then:

   1. your hillarymail client will automatically generate a response
   telling the person that he has to pay `X` many bitcoins to your
   wallet's address with `C`-many confirmations in order to get
   whitelisted.

   2. the local hillarymail client then tracks the bitcoin network somehow
   (e.g. query some btc monitor server, or query local btc installation) to
   check if the sender has already sent to your wallet the `X` bitcoins
   with `C` confirmations.

   3. if `X` btcs with `C` confirmations is found, then the person is
   automatically whitelisted and notified, and his message then
   automatically leaves your local `spam` quarantine box, moving to your
   local `inbox`.

**notes:** 
- a person has an incentive to whitelist people that pay him the
  whitelisting fee.  because if he lies, then people won't bother him,
  since he is literally a thief.
- a mailing list is basically a person that acts like a very good snitch.
  basically, the mailing list will have a `PK` @ `HOST` address that, once
  you mail him, he will decrypt it and then encrypt it again to the public
  key of list's members and send it to them.  the recipients in the list
  will see that the message is signed by you, because the list didn't
  remove your signature (which implies the list cannot change the content,
  nor the header.json which defines the subject and the timestamp).
- when a person replies to a mailing list, it's up to the person if he
  wants to reply to the list's `PK` @ `HOST`, or the individual addresses
  in the list.  it is just that it is a bad idea, because the person will
  have to pay some bitcoin to get whitelisted.  see, this is an elegant fix
  again spam, where spam suddenly becomes a bad idea thanks to the fact
  that hillarymail _honestly_ faces the monetary consequences of
  interrupting people.
- this way spam becomes a fair business deal.  you can think of this spam
  solution as _"goog ads done right"_ where advertisers _directly_ pay
  viewers, which in this case are mail readers (instead of paying goog).
