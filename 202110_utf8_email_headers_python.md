# Looking at utf-8 encoded email headers in python

I was recently investigating a phishing email sent from "Amazon Web Services".

Looking at the raw email, the From header was encoded. This is unnecessary for latin-based alphabets, so it probably indicates obfuscation. (Email address changed in the example.)

```email
From: =?UTF-8?B?4oCq4oCq4oCq4oCu4oCuc2VjaXZyZVMgYmXigK5X4oCuIG5vemFtQQ==?= <fake@example.invalid>
```

Email is still, in 2021, a 7-bit transmission medium. This means characters outside the "lower ascii" range need to be encoded into a 7-bit form in order to be sent. Many servers do, in fact, support 8-bit transmission - but because email uses a store-and-forward delivery model, a sender can’t know if every point in the chain supports 8-bit, so needs to stick to the lowest-common denominator for reliable delivery.

The encoding format is described in [RFC 2047](https://datatracker.ietf.org/doc/html/rfc2047) (which replaced 1992’s  [RFC 1342](https://datatracker.ietf.org/doc/html/rfc1342)).

So just eyeballing the first part of the string, we can see it’s a BASE64-encoded UTF-8 string.

You can copy out the bit between the third and forth question marks and pipe it through a base64 decoder, or you might play around in python’s [email.header](https://docs.python.org/3/library/email.header.html). Decode the header into a list containing the byte string and character encoding (`dh`), then decode the byte string into the specified utf-8 string (`dhu`).

```python
>>> from email.header import decode_header
>>> dh = decode_header("""=?UTF-8?B?4oCq4oCq4oCq4oCu4oCuc2VjaXZyZVMgYmXigK5X4oCuIG5vemFtQQ==?=""")
>>> print(dh)
[(b'\xe2\x80\xaa\xe2\x80\xaa\xe2\x80\xaa\xe2\x80\xae\xe2\x80\xaesecivreS be\xe2\x80\xaeW\xe2\x80\xae nozamA', 'utf-8')]
>>> dhu = dh[0][0].decode(dh[0][1])
>>> dhu
'\u202a\u202a\u202a\u202e\u202esecivreS be\u202eW\u202e nozamA'
```

Printing the unicode string *should* produce the output we see in the email client:

```python
>>> print(dhu)
Amazon Web Services
```

However, if your terminal is set up like mine, you’d see something that looks like it’s been molested by the *Unicode Riddler*.

```python
>>> print(dhu)
 ��secivreS be�W� nozamA
```

What we’re seeing, if we look at the byte string encoding, is the use of U+202E [right-to-left-override](https://www.unicodepedia.com/unicode/general-punctuation/202e/right-to-left-override/) and U+202A [left-to-right embedding](https://www.unicodepedia.com/unicode/general-punctuation/202a/left-to-right-embedding/). Basically the `From:` name had been written in reverse, and then prefixed with a right-to-left text control so that email clients would reverse the characters. But since my terminal setup doesn’t currently support directional text, I just see the “[unprintable character](https://en.wikipedia.org/wiki/Specials_\(Unicode_block\)#Replacement_character)” glyphs.

You can see what’s happening by replacing the control characters with [←](https://www.unicodepedia.com/unicode/arrows/2190/leftwards-arrow/)  and [→](https://www.unicodepedia.com/unicode/arrows/2192/rightwards-arrow/).

```python
>>> print(dhu.replace("\u202e","\u2190 ").replace("\u202a","\u2192 "))
→ → → ← ← secivreS be← W←  nozamA
```

As far as I can tell, only one right-to-left control character would have been necessary for this effect, so presumably the other characters are there to fool anti-spam analysis.

You can play about with testing how email clients deal with unusually encoded utf-8 strings by generating them with the `Header` class.

```python
>>> from email.message import Message
>>> from email.header import Header
>>> msg = Message()
>>> h = Header(dhu, 'utf-8')
>>> msg['From'] = h
>>> msg.as_string()
'From: =?utf-8?b?4oCq4oCq4oCq4oCu4oCuc2VjaXZyZVMgYmXigK5X4oCuIG5vemFtQQ==?=\n\n'
```
