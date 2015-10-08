This is intended to be a quick brain dump about the pst file format. Code wise, much will be restructured as I get a better understanding of the file layout - but currently its in flux.

Pretty incomplete, but its a start.

# Pst Details #

First things first, pretty much all the basics have come from the excellent work of `libpst`. Take a look at some docs on the pst file format from one of the forks of that project - http://www.five-ten-sg.com/libpst/rn01re04.html - and the source itself is more illuminating.

One of the things to note about the pst file format, is it is database like. I remember seeing something saying it also used the Jet engine (as in access), but a look at the mdbtools code doesn't reveal any similarities at the file format level to me. It is also an ISAM file structure, which according to wikipedia (http://en.wikipedia.org/wiki/ISAM) is _Indexed Sequential Access Method_. "A secondary set of hash tables known as indexes contain pointers into the tables..." rings a bit of a bell.

## Basics ##

The definition of the header, and the mechanism of loading the "index" and "desc" trees I'm using is unchanged from `libpst`, with the exception that I'm using block sizes of 512 instead of 516.

```
require 'mapi/pst'

pst = Mapi::Pst.new open('../test/test4-o1997.pst')

p pst.header 
# =>  #<Pst::Header:0xb784c0bc @magic=557990990, @size=779264, @index2_count=242, @index1=119808, @encrypt_type=1, @index1_count=239, @index_type=14, @index2=121856>

p pst.idx
# => [#<struct Pst::Index type=:data, id=0x4, offset=0x5800, size=0x64, u1=0x4>, #<struct Pst::Index type=:data, id=0x8, offset=0x5880, size=0xb4, u1=0x6>, ...

pst.instance_variable_get(:@orphans).each { |desc| puts desc.to_tree }
# =>
- #<struct Pst::Desc desc_id=0x21, idx_id=0x180, idx2_id=0x0, parent_desc_id=0x0>
- #<struct Pst::Desc desc_id=0x61, idx_id=0xc2, idx2_id=0xce, parent_desc_id=0x0>
- #<struct Pst::Desc desc_id=0x122, idx_id=0x4c, idx2_id=0x0, parent_desc_id=0x122>
  |- #<struct Pst::Desc desc_id=0x2223, idx_id=0x38, idx2_id=0x0, parent_desc_id=0x122>
  |- #<struct Pst::Desc desc_id=0x8022, idx_id=0x268, idx2_id=0x0, parent_desc_id=0x122>
  |  |- #<struct Pst::Desc desc_id=0x8042, idx_id=0x5c, idx2_id=0x0, parent_desc_id=0x8022>
  |  |- #<struct Pst::Desc desc_id=0x200024, idx_id=0x168, idx2_id=0x166, parent_desc_id=0x8022>
  |  \- #<struct Pst::Desc desc_id=0x200044, idx_id=0x18a4, idx2_id=0x18ae, parent_desc_id=0x8022>
  |- #<struct Pst::Desc desc_id=0x8062, idx_id=0x6c, idx2_id=0x0, parent_desc_id=0x122>
  \- #<struct Pst::Desc desc_id=0x8082, idx_id=0x178, idx2_id=0x0, parent_desc_id=0x122>
- #<struct Pst::Desc desc_id=0x12d, idx_id=0x26c, idx2_id=0x0, parent_desc_id=0x0>
...
```

Couldn't describe properly the point of this yet, but pretty much all data access is done through Index records. Index#type is currently one of :data, :data\_chain\_header, or :id2\_assoc, which correspond to the 3 known types of Index blocks currently.

Basically a data block is one where the (Index#id & 0x2) == 0. These blocks form the core data records used in the format, and appear to have a maximum size of 8190 (not 8192 strangely). The file offsets generally seem to be padded to 64 byte boundaries.

A data\_chain\_header block is simply a mechanism to store large blocks, and is just a (potentially recursive) list of pointers to blocks. Finally, the id2\_assoc block is one which stores a ID2 <=> Index association, the point of which is not completely clear to me yet. These latter two blocks can be differentiated by their first few bytes. Note also, that unlike the data block, these blocks aren't encrypted (with the Mapi::Pst::CompressableEncryption) even if the file is encrypted.

## Message store ##

Anyway, so thats the basics of the file layout. The point of the Pst format, of course, is as a MAPI message store. By-and-large, each Desc above, corresponds to a Pst::Item, whether it be a Folder, a Message, or some other thing. The code doing this is kind of convoluted at the moment (using the Desc parent/child relationship, in addition to parent/child relationships as defined by random mapi properties), but you can see the end result using something like:

```
puts pst.root.to_tree
# =>
- #<Pst::RootFolder desc_id=0x21 display_name="Personal Folders">
  |- #<Pst::WastebasketFolder desc_id=0x8042 display_name="Deleted Items">
  |- #<Pst::Item desc_id=0x200024 subject="draft message test with attachment">
  |- #<Pst::Item desc_id=0x200044 subject="subject">
  \- #<Pst::FinderFolder desc_id=0x8062 display_name="Search Root">
```

An Item can be a Message, or something like an outlook rule, an out-of-office message, etc.

## RawPropertyStore ##

Any item, regardless of its type, has an associated property store, that is loaded when the Item object is created. You can parse these manually:

```
p Mapi::Pst::RawPropertyStore.new(pst.desc_from_id(0x200024)).to_a
# =>
[[2, 11, true],
 [23, 3, 1],
 [26, 30, "IPM.Note"],
 [35, 11, false],
 [38, 3, 0],
 [41, 11, false],
 [54, 3, 0],
 [55, 30, "draft message test with attachment"],
 ...
```

This gives a list of mapi property tuples - key, type, and value. These are loaded into Mapi::PropertySet objects later, which gives named access and handles some mapi peculiarities.

These blocks are serialized to a block, with a signature 0xbcec (see Mapi::Pst::BlockParser) - what libpst refers to as a "type 1" block. The format seems to be mostly understood, if a little complex and convoluted. An additional complication not handled in libpst, (which is treated as a "type 3" block), is the case when the initial block is an indirect (:data\_chain\_header) style block. This requires special handling, where the offset tables are actually split among the individual Index blocks.

This is not that clear yet, but neither is my understanding. Look at Mapi::Pst::BlockParser, and Mapi::Pst::RawPropertyStore to make more sense of this.

## RawPropertyStoreTable ##

A message may have an associate list of recipients and attachments. These are serialized in blocks with a signature 0x7cec - the "type 2" block from libpst. Here, the list of which properties are stored (ie the columns), is stored once, and then a fixed size row format is decoded, where all variable length values are stored indirectly (as for RawPropertyStore). This format isn't completely understood, in cases where the primary block is indirect. See Mapi::Pst::RawPropertyStoreTable, and Mapi::Pst::RawPropertyStoreTable::Row.

Also, the mechanism by which the associated data records for recipients and attachments are found is pretty odd. Take a loot at Mapi::Pst::AttachmentTable, and Mapi::Pst::RecipientTable. Note that the recipient table is not currently parsed in libpst at all, but it was obvious enough given the way attachments were handled.