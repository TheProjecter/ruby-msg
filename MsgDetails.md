_I should probably add a section about the actual .msg file format. For now, use the source (see last section for links)_

# Msg Details #

A quick look at using the library to examine some components of an msg:

```
require 'mapi/msg'

msg = Mapi::Msg.load open(filename)

# access to the 3 main data stores, if you want to poke with the msg
# internals
msg.recipients
# => [#<Recipient:'\'Marley, Bob\' <bob.marley@gmail.com>'>]
msg.attachments
# => [#<Attachment filename='blah1.tif'>, #<Attachment filename='blah2.tif'>]
msg.properties
# => #<Properties ... normalized_subject='Testing' ... 
# creation_time=#<DateTime: 2454042.45074714,0,2299161> ...>
```

## API ##

You can access the underlying ole object, and see all the gory details of how msgs are serialised. See Ole section for more details.

```
puts msg.ole.root.to_tree
# =>
- #<OleDir:"Root Entry" size=3840 time="2006-11-03T00:52:53Z">
  |- #<OleDir:"__nameid_version1.0" size=0 time="2006-11-03T00:52:53Z">
  |  |- #<OleDir:"__substg1.0_00020102" size=16 data="CCAGAAAAAADAAA...">
  |  |- #<OleDir:"__substg1.0_00030102" size=64 data="DoUAAAYAAABShQ...">
  |  |- #<OleDir:"__substg1.0_00040102" size=0 data="">
  |  |- #<OleDir:"__substg1.0_10010102" size=16 data="UoUAAAYAAQAQhQ...">
  |  |- #<OleDir:"__substg1.0_10090102" size=8 data="GIUAAAYABgA=">
  |  |- #<OleDir:"__substg1.0_100A0102" size=8 data="BoUAAAYABwA=">
  |  |- #<OleDir:"__substg1.0_100F0102" size=8 data="A4UAAAYABAA=">
  |  |- #<OleDir:"__substg1.0_10110102" size=8 data="AYUAAAYAAwA=">
  |  |- #<OleDir:"__substg1.0_10120102" size=8 data="DoUAAAYAAAA=">
  |  \- #<OleDir:"__substg1.0_101E0102" size=8 data="VIUAAAYAAgA=">
  |- #<OleDir:"__substg1.0_001A001E" size=8 data="SVBNLk5vdGU=">
  ...
  |- #<OleDir:"__substg1.0_8002001E" size=4 data="MTEuMA==">
  |- #<OleDir:"__properties_version1.0" size=800 data="AAAAAAAAAAABAA...">
  \- #<OleDir:"__recip_version1.0_#00000000" size=0 time="2006-11-03T00:52:53Z">

     |- #<OleDir:"__substg1.0_0FF60102" size=4 data="AAAAAA==">
     |- #<OleDir:"__substg1.0_3001001E" size=4 data="YXNkZg==">
     |- #<OleDir:"__substg1.0_5FF6001E" size=4 data="YXNkZg==">
     \- #<OleDir:"__properties_version1.0" size=152 data="AAAAAAAAAAAeAA...">
```

Named properties have recently been implemented, and Msg::Properties now allows associated guids. Keys are represented by Msg::Properties::Key, which contains the relevant code.

You can now write code like:
```
props = msg.properties

props[0x0037] # access subject by mapi code
props[0x0037, Msg::Properties::PS_MAPI] # equivalent, with explicit GUID.
key = Mapi::Msg::Properties::Key.new 0x0037 # => 0x0037
props[key] # same again

# keys support being converted to symbols, and then use a symbolic lookup
key.to_sym # => :subject
props[:subject] # as above
props.subject   # still good
```

Under the hood, there is complete support for named properties:
```
# to get the categories as set by outlook
props['Keywords', Msg::Properties::PS_PUBLIC_STRINGS]
# => ["Business", "Competition", "Favorites"]

# and as a fallback, the symbolic lookup will automatically use named properties,
# which can be seen:
props.resolve :keywords
# => #<Key {00020329-0000-0000-c000-000000000046}/"Keywords">

# which allows this to work:
props.keywords # as above
```

With some more work, the property storage model should be able to reach feature
completion.

## More information ##

Until more specifics are added here, have a look at some of these files in the source:

  * [lib/mapi/msg.rb](http://ruby-msg.googlecode.com/svn/trunk/lib/mapi/msg.rb) - top level msg structure and information about the msg property store format.
  * [lib/mapi/property\_set.rb](http://ruby-msg.googlecode.com/svn/trunk/lib/mapi/property_set.rb) - information on the higher level mapi property set wrapper.
  * [data/mapitags.yaml](http://ruby-msg.googlecode.com/svn/trunk/data/mapitags.yaml) - the mapi tag <=> symbolic name mapping that is used (built from aggregation of a few header files).
  * [data/named\_map.yaml](http://ruby-msg.googlecode.com/svn/trunk/data/named_map.yaml) - same sort of thing but for named properties (parsed from a web page i think).