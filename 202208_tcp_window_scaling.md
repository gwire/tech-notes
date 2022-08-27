# tcp\_parse\_options: Illegal window scaling value

I’ve recently (Aug 2022) noticed the following in logs on unconnected Linux servers hosted in different environments.

```
TCP: tcp_parse_options: Illegal window scaling value 15 > 14 received
```

[TCP window scaling](https://en.wikipedia.org/wiki/TCP_window_scale_option) is an option introduced in 1992 to allow an increase the amount of data being transmitted “on the network”.

According to the current spec, [RFC 2723](https://datatracker.ietf.org/doc/html/rfc7323#section-2):

> The maximum scale exponent is limited to 14 for a maximum permissible receive window size of 1 GiB (2^(14+16)).

So this log appears to have been generated as a result of a connection configured to attempt to exceed that 1 GB limit.

I don’t have packet logs, but my assumption is that this is probably mass network scanning, fingerprinting host and networking equipment operating systems in how they respond. (The usual behaviour is, I believe, to log an error and then continue as though the scaling value 14 was specified.)
