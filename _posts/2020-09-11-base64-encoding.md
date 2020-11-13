---
layout: post
title: "Learning to speak base64"
---

We always use base64 encoded strings in a lot of places while working on several different things. I can spot a base64 encoded string from far away (and I have several sight impairments). It is so widely used today that I decided to understand a bit more about it.

As any techology, it is described in an RFC: [RFC 4648](https://tools.ietf.org/html/rfc4648).

It is also mentioned in the [Multipurpose Internet Mail Extensions RFC](https://tools.ietf.org/html/rfc2045).
- MIME enforces a 76 limit on base64 encoded data
- MIME inherits encoding from [PEM](https://tools.ietf.org/html/rfc1421), which limits a line length of 64
- These limits seem to be required due to SMTP

{% highlight ruby %}
class Base64Encoder
  def initialize(parser)
    @parser = parser
  end

  def encode(data)
    result = []
    data.each do |byte|
      result << byte.wrap(IMPORTANT_CONSTANT)
    end

    return result
  end
end
{% endhighlight %}

{% highlight java %}
class MaskingConverter<ILoggingEvent> extends CompositeConverter<E> {
  @Override
  public String transform(E event, String message) {
    Marker marker = event.getMarker();
    return CONFIDENTIAL.equals(marker.getName()) ? "****" : message;
  }
}
{% endhighlight %}

{% highlight bash %}
#!/bin/bash

echo "Enter any number"
read n

if [[ ( $n -eq 15 || $n  -eq 45 ) ]] then
  echo "You won the game"
else
  echo "You lost the game"
fi
{% endhighlight %}

## Why is it so widely used?

Because every computer in any place uses ASCII to display information on a screen. 

## 64? What about 32? Or even 16?

The 64 is the number which describes the alphabet used to encode. In a base64 encoding, 64 symbols are used. In a base32 encoding, 32 symbols are used. You get the idea.

The alphabet used should be the best one considering the requirements that need to be met.
- Is it going to be read by humans? Well, humans are quite terrible at differetiating `0`s and Os. Or 1s, ls and Is.
- Is it going to be used on URLs or file names? I don't think your alphabet should have a `/` then.

As far as that goes, I want to look at this `Base64Encoder` and be happy.

## Huh, can characters outside that alphabet appear in it?

Oh, yes. Either by simple corruption or by design. Those characters could be used to exploit implementation errors, although I'm not entirely sure on how yet.

## Oh, and what if some symbol not in that alphabet appear in the data?

Well, it depends on the implementation. Some implementations may reject the full encoded data or simply ignore those characters (MIME ignores it, in order to be _be liberal in what you accept_). That means a CRLF is ignored. Also, the pad character (=) is ignored too if it's not at the end of string.

Note: base16 does not padding.
__TODO: Why? What is padding anyway?__

How does this encoding scheme works though?


...