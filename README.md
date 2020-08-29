<p align="center">
    <img src="pics/logo.png">
</p>

----

# hillarymail: draft one

## goals:
- should be easy to install/use.
- should not need to assume trusty vps admin.

## non-goals:
- compatibility with smtp/pop/imap/people that can't use address books or
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
when we throw hillarymail mailballs at each other, each ball is seen as 2
files only:

    keys.tar
    msg.tar.gz.enc

this also means that when a person submits a new hillarymail mail, he does
not look any different than a relay.  because mails are always like that,
be it newly submitted or relayed.

the voodoo happens inside `msg.tar.gz.enc` where things get recursively
onioned until you reach the actual messages.  this onioning is nice, so
that you don't need to trust that each relay is honest in not changing
previous relays' headers.

key fun things in hillarymail (for those with limited time):
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


## initial authentication
user with public key `PK` wants to authenticate to a server.  the user
could be a user authenticating to his local mail server, but the user could
also be a relay authenticating to another relay or server.  fundamentally
they are indistinguishable.

technically, the identity of a user is his public key.  but public keys are
needlessly too long to let the server, which already has the full public
key, know that it is him.  therefore, we only send the `sha3_224` checksum
of `PK`.

first such user needs to get a challenge string.  he does so by:

    RESP = https://HOST/hillarymail/get_challenge?user=sha3_224(PK)

where:

    RESP = {"status": some int code (0 = ok),
            "challenge": "some random string"}

then, to have the user prove his identity, he needs to solve the challenge
by encrypting the challenge by his private key as follows (aka signing):

    ANS = asym_enc(RESP["challenge"], sender_privatekey)

then, later on, when `ANS` is submitted to the server (some other urls
depending on the api call) the server will attempt to decrypt `ANS` by the
public key with the same `sha3_224` hash given earlier.  if the server
recovers the original challenge string above, then the challenge is
successfully solved and the user is henceforth authenticated.

**note:** this is only the initial authentication.  the same happens with
other api calls.  e.g. for every subsequent api call that you make, the
returned `RESP` structure will contain a new challenge string that you will
have to solve to get a new `ANS` code for the next.  this is because each
`ANS` is valid for only once (after which the challenge expires and becomes
no longer useful for subsequent calls).  this is to prevent replay attacks.


## composing a new message
this happens when a hillarymail client creates a new message to send it to
another person.  we have this directory layout:

    keys/
        48837a787f07673545d9c610bcbcd8d46a2691a71966d856c197e69e.key
        30ebe8c208f6470e4b751d91daf8d62bef074353bafef67806663706.key
        154f98464dcec35cda24c2151c695f59c5bc54e020aa227ab628cf55.key
        316a5fa18f51cc21d44e886636f2649d77e128bf79013d92f5beedd2.key
        f11e32f81b08b60bc89112ef318aa42461b9ac9c3ef57a172d01782c.key
        a27ebadc5be4f2a79078f0432a00f5585bab1860f5b8555c15065625.key
    msg/
        sender/
            PK
            HOST
        reply-to/
            PK
            HOST
        recipients/
            to/
                1/
                    PK
                    HOST
                2/
                    PK
                    HOST
            cc/
                1/
                    PK
                    HOST
            bcc/            # read note on bcc
                1/          #
                    PK      #
                    HOST    #
        msgid.txt
        subject.txt
        timestamp.txt
        body.txt
        attachments/
            lol.png
            app.tar.gz
            puppy_run_green_grass.mp4
        hash.txt
        hash.txt.sig

where:

    keys/*.key  = the file name (*) is the sha3_224 hash of user's public
                   key.  its content is a shared key (KEY) encrypted to
                   user's public key (CKEY).

    HOST         = end-to-end address (e.g. ip, domain, onion address)
                   where end user's mailbox exists.

    PK           = public key of ith party.  this is what uniquely
                   identifies the person.

    msgid.txt    = utf-8 encoded random string that identifies this email.
                   this is used to allow mail clients figure out which
                   reply is to which email, so that when they list emails,
                   they can list them into threads.

    subject.txt  = utf-8 encoded text used as subject.

    timestamp.txt= utf-8 encoded text containing seconds since epoch for
                   around the time the message was sent.

    body.txt     = utf-8 encoded message body.  only text is supported for
                   body text.  no self-hating formats like html/js/css/etc.
                   utf-8 is already fancy enough and has lots of emojies.

                   using other formats will cause rendering issues with
                   other clients and may cause your email to be rejected by
                   them, or they may ask you to pay bitcoins to them in
                   order to read your malformated mail (e.g. malformat
                   fee).  so you better be nice.  if you choose to be not
                   nice, then expect to pay extra fees as recipients will
                   probably charge you for it.  this is only fair.  if
                   you're not nice, and you didn't pay fees to rectify your
                   vandalism, then you've became a thief.

    hash.txt     = sha3_512(msg/sender/PK
                            + msg/sender/HOST
                            + msg/reply-to/PK
                            + msg/reply-to/HOST
                            + msg/recipients/to/1/PK
                            + msg/recipients/to/1/HOST
                            + msg/recipients/to/2/PK
                            + msg/recipients/to/2/HOST
                            + msg/recipients/cc/1/PK
                            + msg/recipients/cc/1/HOST
                            + msg/recipients/bcc/1/PK    # read note on bcc
                            + msg/recipients/bcc/1/HOST  #
                            + msg/subject.txt
                            + msg/timestamp.txt
                            + msg/body.txt
                            + msg/attachments/lol.png
                            + msg/attachments/app.tar.gz.
                            + msg/attachments/puppy_run_green_grass.mp4)

    hash.txt.sig = asym_enc(hash.txt, sender_privatekey)

to send this message structure, we pack it as follows:

    keys.tar = tar(keys/)  # no compression, since keys are random, and
                           # random sequence cannot be compressed.
    msg.tar.gz.enc = sym_enc(tarball(msg/), KEY)

where `KEY` is randomly chosen string by the sender.  it is also this `KEY`
that is encrypted to the public key of each recipient and is stored in
`keys/*.key`.

so it's part of the standard that each mail is comprised of 2 files with
the same names.  e.g. one cannot pick other file names.  case sensitive.

finally, we submit the files to some hillarymail server.  it could be our
own local hillarymail server, or it could be directly the hillarymail
servers of the recipients.  this is a decision that we need to make as
users.  most people would probably prefer using a local hillarymail server
in order to centralize the backup of their hillarymail messages.

**notes on bcc:** 
- messages get packed several times.  once without `bcc` addresses, which
  will be the same message that's sent to recipients in `to` and `cc`.
  then, the message will get packed several times, each time with only a
  single `bcc` address and gets sent only to that single `bcc` address.
  this way `bcc` will work as normally one would expect.  
- each `bcc` recipient must also have a different `KEY` encrypted to him.
  this way if an encrypted mailbox is stolen, one cannot try to decrypt
  with his own keys to extract whether the stolen mailbox belongs to a
  person in the `bcc` of some email that he got.  if any client
  implementation does not behave so, then it's a security vulnerability bug
  and must be fixed.
- honoring the semantics of `bcc`, by performing the steps above, is the
  duty of the sender.  because the relays don't know which recipient's
  public key is bcc or not, plus relays can't modify the content of `msg/*`
  (duh).


## wrapping a mail
this happens when a hillarymail server receives a hillarymail mail (either
from a sender, or from another hillarymail server (aka relay).  

hillarymail servers will wrap the received mails as follows:

    keys/
        48837a787f07673545d9c610bcbcd8d46a2691a71966d856c197e69e.key
        30ebe8c208f6470e4b751d91daf8d62bef074353bafef67806663706.key
        154f98464dcec35cda24c2151c695f59c5bc54e020aa227ab628cf55.key
        316a5fa18f51cc21d44e886636f2649d77e128bf79013d92f5beedd2.key
        f11e32f81b08b60bc89112ef318aa42461b9ac9c3ef57a172d01782c.key
        a27ebadc5be4f2a79078f0432a00f5585bab1860f5b8555c15065625.key
    msg/
        note.txt        # new (optional)
        timestamp.txt   # new (mandatory iff `note.txt`)
        keys.tar        # as received
        msg.tar.gz.enc  # as received
        hash.txt        # newly calculated
        hash.txt.sig    # newly signed by relay's private key

where

    keys/*.key     = the file names (*) is identical to those in the
                      recieved `keys.tar` file (i.e. `sha3_224(PK)` of
                      recipients), except that their content contains an
                      updated encrypted key `CKEY` that's chosen by the
                      relay and encrypted to the public key of the
                      recipients.

    note.txt        = some note by the relay (optional)
    timestamp.txt   = mandatory only if `relay.txt` is used.  this
                      timestamp should be filled with utf-8 encoded seconds
                      since epoch at about the time the server put the
                      note.txt message.

    hash.txt        = sha3_512(msg/note.txt
                               + msg/timestamp.txt
                               + msg/keys.tar
                               + msg/msg.tar.gz.enc)

    hash.txt.sig = asym_enc(hash.txt, relay_privatekey)

then we pack the hillarymail message similarly into an encrypted zip file:

    keys.tar = tar(keys/)
    msg.tar.gz.enc = sym_enc(tarball(msg/), KEY)

where `KEY` is randomly chosen string by the relay.  it is also this `KEY`
that is encrypted to the public key of each recipient and is stored in
`keys/*.key`.


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
submitting a newly composed mail, or submitting a received mail by a
hillarymail relay.

1st [authenticate](#authentication) to get session `ANS`.

then check if we can submit a mail:

    RESP_1 = https://HOST/hillarymail/can_submit?
        & msgid     = ID  # relays will re-use received ID.  composes will
                          # generate a new one
        & dests     = [host1, host2, ...]
        & size      = size(pks)
                      + size(host1, host2, ...)
                      + size(keys.tar)
                      + size(msg.tar.gz.enc)
        & auth      = ANS

if cool (i.e. `RESP_1["status"] = 0`), then actually send it (but 1st make
sure to calculate a new `ANS` with the `RESP_1["challenge"]`):

    RESP_2 = https://HOST/hillarymail/submit?
        & msgid = ID # relays will re-use received ID.  composes will
                     # generate a new one
        & dests = [host1, host2, ...]
        & pks   = pk1, pk2, ...
        & keys  = keys.tar
        & msg   = msg.tar.gz.enc
        & auth  = ANS_1

where

    ID = some unique randomly generated string to uniquely identify this
         message for the purpose of loop detection.  e.g. relays will drop
         the mail if they have seen a message with ID that is seen in the
         past, say, 1 month.

further error checking can be done by reading `RESP_2["status"]`.  but
let's not discuss this now since this is a quickie draft.  e.g. delivery
failures for some recipients should trigger a notification message to the
sender informing him of it.


## relay a received mail
always check if "from" `PK` @ `HOST` is allowed to submit to us a mail, has
passed the challenge, all recipients are permitted to receive the message,
check if file sizes are cool (during `can_submit`), and also check if
message `ID` is not seen in the past, say, month (loop detection).  else
return error and exit.

if things are cool, then:

* if relay's address is in `dests`:
  - remove relay's address from `dests`.
  - for every user in `keys.tar` that corresponds to a local account:
    * store the mail
       ```
       USER/i/
           keys.tar
           msg.tar.gz.enc
       ```
       where `USER = sha3_224(PK)` and `i` is a logical clock (counter) such
       that, for any positive `n`, mail `i+n` implies that it is older than
       mail `i`.

    * remove user's key from `keys.tar` (which is named
      `sha3_224(PK).key`) as well as associated public key from `pks`.

* if the relay is configured to add any notes (e.g. based on the
  authenticated sender), the relay can add add a `note.txt` and
  `timestamp.txt` as explained [here](#wrapping-a-mail), and update shared
  keys in `keys.tar` accordingly.  if no notes are added, then the relay
  does not need to wrap the mail, but forward it as is, because wrapping is
  only needed to securely send relay's notes.

* for the non-local hosts in `dests` (if any), consult the routing table,
  and figure out the right next-hop relays for them.  if several next
  relays are chosen for different `dest` hosts, make sure that the hosts in
  `dest` are mutually exclusive across the next relays (e.g. if relay1 is
  chosen for host1, then relay2 must not get host1).  then, finally, submit
  the mail to the next hop [as discussed here](submit-a-mail).


## get new mail(s) from server
if client did not receive any mail so far, it will have `i=0`.  if it had
received emails in the past, then `i` will be updated to match the counter
of the last received mail.

1st [authenticate](#authentication) to get session `ANS`.

then, obtain list of new mails:

    RESP = https://HOST/hillarymail/check_new?
        & since    = i
        & answer   = ANS

where `RESP` contains a list of counters that represent all mails newly
received after the logical clock `i`.  to be exact, suppose `i=4`:

    RESP = {"status": some int code (0 = ok),
            "new_mails": [5, 9, ...]   # note: list doesn't have to be
                                       # sorted, and items don't have to
                                       # be continuous.  e.g.:
                                       # [500, 8, 100, ...] is possible.
                                       # plus this can contain ranges.
                                       # e.g.: [[5,100], 105, ..]
            "challenge": "new random challenge"}

then the hillarymail client will send requests to download all files for
those mails:

```python
files = ['keys.tar', 'msg.tar.gz.enc']
for i in indexes(RESP["new_mails"]): # these 2 loops can be parallelized
    for f in files:         #
        ANS = asym_enc(RESP["challenge"], sender_privatekey)
        url = f'https://{HOST}/hillarymail/get_mail?' \
              f'&i={i}&file={f}&auth={ANS}'
        saveto = f'~/.hillarymail/caveman/{i}/{f}'
        RESP = download(url, saveto)
```
where `indexes` is a generator that keeps returning the next index number
(abstracting ranges in `RESP["new_mails"]` into single numbers).

the hillarymail server will verify `ANS`, and then simply start sending
those requested files as mere _static_ files using efficient functions like
`sendfile(2)`.

if hillarymail is a uwsgi web application running behind a normal web
server, like nginx, then hillarymail can use
[`X-Accel-Redirect`](https://www.nginx.com/resources/wiki/start/topics/examples/xsendfile/).
this way files are transferred by nginx without hillarymail's involvement
(hillarymail only authenticates `ANS` and then approves by sending
`X-Accel-Redirect: /protected/USER/i/whatever_file`).

then what?  then you have your own private key, and you can unwrap the
onion recursively until you reach the final message.  you can see the
entire journey of _your_ mail with high confidence of not being tampered
with unfairly, not even the header information injected by past relays.  in
fact, next relays won't even know what previous relays have injected in the
onion.  heck, they wouldn't even know if the relay is relaying a mail or
sending a new mail of its own.


## delete mails
1. `MAILS = [1, 5, 100, 7]`.
2. [authenticate](#authentication) to get session `ANS`.
3. `RESP = https://HOST/hillarymail/delete?mails=MAILS&auth=ANS`


## set/unset tags to mails
since servers hardly sees any information about the emails (e.g. can't know
the sender, subject, etc), only the client can tag.  which is fine, we
dont' really need the server to tag on the fly.  when we open our
hillarymail clients, we get mails, and the software syncs tags them
immediately and syncs any tag changes to the server.

how tags are stored locally by the hillarymail client, is purely up to the
client.

but hillarymail specifies how the client syncs tags with the server.  it
happens by creating a text file, `tags.txt`, with each tag separated by a
new line.  e.g.

    tags.txt = unread
               didn't pay yet
               ...

then to sync this with the server, this happens:

1. randomly generate a shared key `KEY`.
2. `tags.txt.gz.enc = sym_enc(gz(tags.txt), KEY)`.
3. `tags.key = asym_enc(KEY, user_privatekey)`
4. [authenticate](#authentication) to get `ANS`.
5. submit this request

    RESP = https://HOST/hillarymail/set_tags?
        & i     = i  # mail's index
        & key   = tags.key
        & tags  = tags.txt.gz.enc
        & auth  = ANS

to get tags of a specific email:

    RESP = https://HOST/hillarymail/get_tags?
        & i     = i  # mail's index
        & auth  = ANS

how to merge changes, or resolve conflicts, is up to the client.  but right
now i personally think it suffices to follow this approach:

1. get latest tags for an email.
2. apply changes to the tags.
3. upload final tags to server.
4. get latest tags for the same email again, but store it in a tmp
   directory, just to double check that server's version is the latest.
   - if server's version is differnt, it means that a racing condition
     happened.  if so, then randomly wait for some time between 0 and 10
     seconds, then merge changes against this tmp tags list, and start over
     from step (2).
   - TODO TODO


## whitelist/blacklist senders
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
