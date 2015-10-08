Generally, the goal of the project is to enable the conversion of msg and pst files into standards based formats, without reliance on outlook, or any platform dependencies. In fact its currently _pure ruby_, so it should be easy to get running.

It is targeted at people who want to migrate their PIM data from outlook, converting msg and pst files into rfc2822 emails, vCard contacts, iCalendar appointments etc. However, it also aims to be a fairly complete mapi message store manipulation library, providing a sane model for (currently read-only) access to msg and pst files (message stores).

Conversion of msg files into individual emails (this will create some\_email.eml):

```
gem install ruby-msg
mapitool -i some_email.msg
```

(Note that windows users may prefer to install the stand alone msgtool binary which doesn't need ruby, rather than the gem version.)

Or to convert a bunch of msg files into combined formats (this creates Mail.mbox, Contacts.vcf as appropriate):

```
mapitool *.msg
```

See the [wiki](http://code.google.com/p/ruby-msg/wiki/Home) for further details

_I am happy to accept patches, give commit bits etc._

_Please let me know how it works for you, any feedback would be welcomed._

**UPDATE**: version 1.4.0 has been released with _experimental_ pst format support.