# GPG / Mutt / Gmail

## About

This is a collection of snippets, not a comprehensive guide. I suggest you start with [Operational PGP](https://gist.github.com/grugq/03167bed45e774551155).

Here is an incomplete list of things that are different from other approaches:

- I don't use keyservers. Ever.
- Yes, I use Gmail instead of some bespoke hipster freedom service
- I use mutt. It is minimal, available for all unix-alikes and not written by Mozilla
- I [don't sign](#footnote---no-signatures)

This approach to GPG is different from almost every traditional guide.

### BYO Threat Model

If you don't know what a threat model is then just stop. If you're a threatened user and you don't know this stuff then learn it before you do anything else. These settings might not be right for you! Hell, using email at all might not be right for you. Don't outsource understanding.

In particular, I'd like to remind you that unless you have a reasonable level of endpoint security, compartmentalisation and operational discipline then your chances of withstanding any kind of targeted attack are probably close to zero. Get that stuff in order before you start messing about with geeky email setups.

## Gmail and IMAP

Both "Operational PGP" and this guide are probably going to be fairly uncomfortable for the "non-technical user". Since talking about Gmail is going to raise a few eyebrows, here are some things I like about it.

### Blend In

An IMAPS connection to Gmail is about the most innocuous looking thing possible, from a network monitoring perspective. This is also the reason I'm not recommending Tor and such. If your location blocks access to Gmail then you'll have to use something to get that access back. I don't cover that here.

### Not NSA-Proof, But Not Bad

Google's overall system security against attackers that aren't completely terrifying is pretty good. In any case, if one of those attackers is personally interested in you then you shouldn't be flailing about on the Internet trusting random guides. From a jargon perspective, what we're aiming at is:
- A reasonable approach to frustrate dragnet surveillance, casual network monitoring and forensics
- Data at rest is protected by Google, from both a legal and technical security standpoint
- Data in motion is protected by Gmail's TLS settings (which are strong) and their Certificate
- No data remains on your disk

### Relatively Easy Pseudonymity

To get a "fairly" anonymous Gmail account you need an unlinkable phone number, and a safe Internet connection, once. While these things may not be easy to come across for all people, if you can't get them then you have bigger problems to worry about.

## Creating a muttrc

[Here](https://blog.bartbania.com/raspberry_pi/consolify-your-gmail-with-mutt/) are the basics. You should read that link and then come back.

__(You can find a sanitized version of my final muttrc in the [Appendix](#appendix---muttrc))__

__NOTE: If you're about to skip ahead, make sure to ALSO use the [gpg.rc](#gpg-gpgrc)__

Make sure you're using TLS
```
set folder="imaps://imap.gmail.com:993"
```

Now, the part where they use
```
set imap_user = 'yourusername@gmail.com'
set imap_pass = 'yourpassword'
```

... don't do that. Do [this](https://pthree.org/2012/01/07/encrypted-mutt-imap-smtp-passwords/) instead - it uses `source` to load the passwords from a GPG encrypted file at startup (which will prompt for passphrase). Except the part where they do
```bash
% gpg -r your.email@example.com -e ~/.mutt/passwords
% ls ~/.mutt/passwords*
/home/user/.mutt/passwords /home/user/.mutt/passwords.gpg
% shred ~/.mutt/passwords
% rm ~/.mutt/passwords
```
... don't do that. Modern filesystems and disks are gigantic snitches and you shouldn't trust `shred`. Instead just do:
```bash
gpg -r your.email@example.com -e -a > /home/user/.mutt/passwords.gpg
```

And the part where they do:
```
# LOCAL FOLDERS FOR CACHED HEADERS AND CERTIFICATES
set header_cache =~/.mutt/cache/headers
set message_cachedir =~/.mutt/cache/bodies
```
... don't do that either. Caching headers is OK, as long as your subject lines follow the guidelines in [Operational PGP](https://gist.github.com/grugq/03167bed45e774551155), TL;DR use meaningless subject lines. Caching message bodies, however, leaves local copies of encrypted messages which I don't want.

### Use vim

I don't have 100% faith in this approach with vim, but I trust it more than other editors. Sorry, but you'll need to learn how to use vim. It's not hard (not for composing email, anyway). Set it not to use the disk, like this:
```
set editor="vim +13 -c 'set nobackup' -c 'set noswapfile' -c 'set nowritebackup' -c 'set tw=72 ft=mail noautoindent'"
```

### SSL Hardening

My general view is that no client or server should be supporting SSLv3 or earlier EVER, and no client should support TLSv1.1 or earlier. Given that we're only connecting to Google, we can disable everything except TLSv1.2. This section is fairly important, because the mutt defaults here are insecure.

```
# SSL hardening
set ssl_force_tls=yes
set ssl_starttls=yes
set ssl_use_sslv2=no
set ssl_use_sslv3=no
set ssl_use_tlsv1=no
set ssl_use_tlsv1_1=no
set ssl_use_tlsv1_2=yes
set ssl_verify_dates=yes
set ssl_verify_host=yes
```

### TOFU

If you're reading this, you're probably the kind of person that would be worried about someone MitMing your TLS connection to Gmail. Chrome uses HSTS to stop MitM via a rogue root cert, but (by default) your system lib probably trusts a whole ton of dodgy root certs. Set these options:
```
# On some configurations this option is not recognised. It should
# default to "" anyway. If this is a valid option, uncomment this.
# unset ssl_ca_certificates_file
# Don't trust the system.
unset ssl_usesystemcerts
set certificate_file=~/.mutt/gmailcerts
```

Now when you first start mutt you will see something like:
```
This certificate belongs to:
   GeoTrust Global CA
   Unknown
   GeoTrust Inc.
   Unknown
   Unknown

This certificate was issued by:
   Unknown
   Unknown
   Equifax
   Equifax Secure Certificate Authority
   Unknown

This certificate is valid
   from May 21 04:00:00 2002 GMT
     to Aug 21 04:00:00 2018 GMT

Fingerprint: 2E7D B2A3 1D0E 3DA4 B25F 49B9 542A 2E1A

-- Mutt: SSL Certificate check (certificate 1 of 3 in chain)
(r)eject, accept (o)nce, (a)ccept always
```

In theory you should manually verify the fingerprint of all 3 certs in the chain. You're almost certainly going to find this a harrowing and frustrating experience - which is more or less the permanent state of anyone who has peeked under the sheets of the global PKI. In practice, as long as you're fairly sure you're not being actively MitM'd by that exact root RIGHT NOW you can just trust the certs. The key benefit of this step is that if they ever change you will see this screen again and you can freak out.

Once you have (a)ccepted all three certs, you should be able to see them saved on disk:
```bash
$ grep "BEGIN CERTIFICATE" .mutt/gmailcerts | wc -l
       3
```

(One more thing - as soon as you try to _send_ an email you're going to get asked one more time to trust the Gmail SMTP server's cert for TLS. It's ok. That's good.)

**NOTE**

One of the reviewers of this document pointed out that it would be even better if we could pin directly on the Google intermediate CA - that way we wouldn't have to trust the Geotrust root cert. The way that SHOULD work would be to have:
```
set ssl_ca_certificates_file=~/.mutt/google.crt
```
According to `man muttrc`

> This  variable  specifies a file containing trusted CA certificates.  Any server certificate
> that is signed with one of these CA certificates is also automatically accepted.

Unfortunately, that doesn't seem to work - so they opened a Debian [bug](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=792396) about it.

## Mutt gpg.rc

The mutt wiki has a lot of basic information: http://dev.mutt.org/trac/wiki/MuttGuide/UseGPG

The parts I care about:
```
# Don't sign!
# set pgp_autosign=yes
set pgp_sign_as=0xDEADBEEF
# Encrypt replies to PGP mails by default
set pgp_replyencrypt=yes
# Set this to something comfortable
set pgp_timeout=1800
```

## GPG gpg.rc

As well as the mutt GPG options we need to harden the actual GPG settings.

Here is my whole `.gnupg/gpg.rc`. Note that ALL the keyserver cruft is not there:
```
# These first three lines are not copied to the gpg.conf file in
# the users home directory.
# $Id$
# Options for GnuPG
# Copyright 1998, 1999, 2000, 2001, 2002, 2003,
#           2010 Free Software Foundation, Inc.
# 
# This file is free software; as a special exception the author gives
# unlimited permission to copy and/or distribute it, with or without
# modifications, as long as this notice is preserved.
# 
# This file is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY, to the extent permitted by law; without even the
# implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#
# Unless you specify which option file to use (with the command line
# option "--options filename"), GnuPG uses the file ~/.gnupg/gpg.conf
# by default.
#
# An options file can contain any long options which are available in
# GnuPG. If the first non white space character of a line is a '#',
# this line is ignored.  Empty lines are also ignored.
#
# See the man page for a list of options.

# Uncomment the following option to get rid of the copyright notice

no-greeting
default-key DEADBEEF
default-recipient-self
require-cross-certification

# IMPORTANT! GPG still defaults to bad choices for digests and symmetric
# ciphers. This doesn't totally eliminate the risk of a peer deliberately
# downgrading to weaker algorithms, but it makes the defaults strong when
# you communicate with good faith actors with up-to-date software.

personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256 SHA224

# Don't include key-id metadata! The makes it a bit harder to link
# the parties that are communicating.
throw-keyids

# More like "Web of Mistrust", amirite??
trust-model always
```

## Aliases

Yo, dawg, I heard you like aliases so I made some aliases for your aliases.

Using the `-F` switch to mutt we can create multiple copies of the muttrc we just created, and use them for different accounts. If you like you can use completely different keyrings, but you definitely want different private keys. Now, you just need to create aliases in your `.bash_aliases` (or whichever file is appropriate for your OS) like:
```
alias sally='mutt -F ~/.mutt/muttrc_sally'
alias trevor='mutt -F ~/.mutt/muttrc_trevor'
```
Having aliases like this might take a tiny bit more work to set up, but it's a lot harder to mess it up down the road. You should also consider using visual cues in your terminal (background colors, different workspaces or desktops...) to help you stay "in character". 

## The End

Once again, this isn't a comprehensive guide, it is just a few things that I think are important and run counter to a lot of "conventional wisdom" on the Internet. A lot of people that read this won't like it because it is "too hard" for the fantasy "non-technical user" we condescendingly assume is too dim to learn about things that are directly relevant to their continued liberty or safety. If you're reading this and fall into the "this is unrealistic11!1!" category, please direct your comments to Hacker News. If you feel that there are material errors or omissions in the configuration options, please add comments below.

Baby Seals,

ben

### Footnote - No Signatures??

Crypto people are always dismayed that I don't use signatures - on the face of it I am taking perfectly good Authenticity and throwing it away - an attacker could spoof my email address, send email to someone with whom they know I communicate and then encrypt with that persons key. This is, undeniably, a risk. The thing you should consider, though, is the other side of the coin. Quite simply, if you plan, at any point, to say something in encrypted email that you wouldn't want read out in court, DON'T SIGN ANYTHING EVER. The person you are sending to is about to have a plaintext copy of the communication, where you have undeniably linked the text with your private key. If that message leaks, gets snitched, if the other peer gets owned by HackingTeam (or whatever) then it's more or less Game Over. This is not just hypothetical. Things can go poorly if, say, you're found with the private key that matches this:
```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA1

This message is to prove I am the Dread Pirate Roberts

Date: 10/26/2013
Time: 18:37 UTC

This message is only a sample. To prove my identity, please encrypt a message to me and ask I decrypt it.
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1.4.12 (GNU/Linux)

iQIcBAEBAgAGBQJSbAxsAAoJEPMoGw8w0+tzllwQAKJx0aH7MTp5I/4emwUnxAhw
GoVBR+a4QXiGFe4U+XAwCdj4+gJMrcbx9LG+F0sUhWnnfp62RWhPGEgWAviCKiGl
QEIFNwE6/ArQvEDHV+wVkJOJWpxa9xoxPZgclMkohlPfivd0j7lqRyQiAncW5FSo
WUqqmuWneFNu/2rns5skWnjfxpfgR0SUN7DP+RjqOFmoonzQaSy1n8RIb1ZW5fwo
U3i9pIV1OruWGKeNimYYsHXalFsZMNOHiKXhr7D5N3uF+SikX0YAC3lGLdTZa2DI
yJrQKpnFPzqGiYV2wDyKx3eLwQngQBUXbq8BGM8mx7zKURdbvMxrfmwxLY34LrRm
SDDpLUuuJTdyoytn+B2AuGihJoj9EsDWwADX+WrSfb8KpvBrwdLSMfibBT66FGYQ
kotA9QqfP3pHjRaKP18h3KM1lzo+yKD6pSrY3JIYvQPV382Q1C0W9PNkQuDVAcOx
rntCcQtYRDcrKHv9zUH0jGj0BDm1BaT6ZPELIqLNXmFr5tH0B0vTTkcRSWBxaDM6
s4awygbc+Q3RJ0NMLx3g+7Pe7mM42LhNTKadf3vkerrwr0LrSeF6pRHEVPG+PH4H
iEsgbMqJwCQ9hShH6Trb5zKTg47DLxjd1vxijZXTp49mfnGYjPnudoMfBP0E6VBg
UU6zIZGwlOyMHmzATIaL
=G4Nr
-----END PGP SIGNATURE-----
```

## Appendix - muttrc

```
# -*- muttrc -*-
#
# rc file for mutt

# protect imap and sendmail passwords with GPG
source "gpg -d ~/.mutt/muttpasswd.gpg |"

set realname="David Vincenzetti"

set imap_user="hackingteam@gmail.com"
set imap_keepalive=60
set imap_passive=no
set imap_check_subscribed=yes
set imap_idle=yes
set mail_check=60

# Sidebar Stuff
set sidebar_visible=no
set sidebar_width=30
set sidebar_folderindent=yes
set sidebar_sort=yes
set sidebar_shortpath=yes
set sidebar_delim=" | "
color sidebar_new brightred default
# b toggles sidebar visibility
macro index b '<enter-command>toggle sidebar_visible<enter>'
macro pager b '<enter-command>toggle sidebar_visible<enter>'
# Remap bounce-message function to Ã¢â‚¬Å“BÃ¢â‚¬Â
bind index B bounce-message

set smtp_url="smtp://hackingteam@smtp.gmail.com:587/"
set from="d.vincenzetti@hackingteam.com"
set folder="imaps://imap.gmail.com:993"
set spoolfile="+INBOX"
set postponed="+[Gmail]/Drafts"

# unset ssl_ca_certificates_file
unset ssl_usesystemcerts

# SSL hardening
set ssl_force_tls=yes
set ssl_starttls=yes
set ssl_use_sslv2=no
set ssl_use_sslv3=no
set ssl_use_tlsv1=no
set ssl_use_tlsv1_1=no
set ssl_use_tlsv1_2=yes
set ssl_verify_dates=yes
set ssl_verify_host=yes

set header_cache=~/.mutt/gmailcache
set certificate_file=~/.mutt/gmailcertificates
# looks like you must explicitly do this to make sure you
# don't save local copies of sent mail >:(
unset record

set move = no
set hostname=hackingteam                  # Name of our local host.
set hidden_host                           # Hide host details.
set envelope_from                         # set the envelope-from information
set reverse_name=yes                      # build From: in the reply based on the To: address (must have
set alias_file=~/.mutt/aliases            # Keep aliases in this file.
set postpone=ask-no                       # Ask about postponing.
set print=ask-yes                         # Ask before printing.
set delete=no                             # Ask before doing a delete.
set include                               # Include the message in replies.
set sort=threads                          # always sort by thread
set sort_aux=date-received                # Sort threads by date received.
set charset=iso-8859-1                    # One of those days in England...
set noallow_8bit                          # 8bit isn't safe via Demon.
set ascii_chars=yes                       # use ascii characters when displaying trees
set meta_key=yes                          # allow to use alt or ESC
set attribution="* %n <%a> [%{%Y-%m-%d %H:%M:%S %Z}]:\n"
set edit_headers                          # I want to edit the message headers.
set fast_reply                            # skip initial prompts when replying
set nohelp                                # don't show the help line at the top
set editor="vim +13 -c 'set nobackup' -c 'set noswapfile' -c 'set nowritebackup' -c 'set tw=72 ft=mail noautoindent'"
set nomark_old                            # Don't mark unread new msgs as old.
set nobeep                                # We don't need no beeping software.
set nosmart_wrap                          # Don't want smart wrapping.
set nomarkers                             # Don't want any wrap markers.
set mime_forward                          # Forward message as MIME attachments.
set pager_context=3                       # Display 3 lines of context in pager.
set pager_index_lines=20
set nostrict_threads                      # Lets have some fuzzy threading.
set nopipe_decode                         # Don't decode messages when piping.
set text_flowed                           # label messages as format-flowed
set print_command="enscript --font=Times-Roman10 --pretty-print"
set tilde                                 # Fill out messages with '~'.
set read_inc=100                          # Read counter ticks every 100 msgs.
set write_inc=100                         # Write counter ticks every 100 msgs.
set noconfirmappend                       # Just append, don't hassle me.
set pager_stop                            # Don't skip msgs on next page.

macro index <esc>m "T~N<enter>;WNT~O<enter>;WO\CT~T<enter>" "mark all messages read"

# What we consider to be a quote.
set quote_regexp="^( {0,4}[>|:#%]| {0,4}[a-z0-9]+[>|]+)+"
set to_chars=" +TCF "                     # Drop the "L".

source ~/.mutt/gpg.rc         # Use GPG
source ~/.mutt/solarized_light_16.muttrc
source ~/.mutt/headers        # Configure header display.

# Last, but not least, get mutt to display its version on startup.
push <show-version>
message-hook '!(~g|~G) ~b"^-----BEGIN\ PGP\ (SIGNED\ )?MESSAGE"' "exec check-traditional-pgp"
auto_view text/html                       # eg with links --dump
```


Source: https://gist.github.com/bnagy
