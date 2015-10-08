## Msg ##

  * fix that named property bug. (done) check this, i think the way i did it may have been sub-optimal.
  * msg parse tests.

## Pst ##

  * have a look at PST format. By-and-large, if you're going to be migrating your mail, you'll have a pst rather than a bunch of msg files. (work-in-progress)

## Mapi ##

  * tidy up and separate from msg.
  * look into corner cases covered so far, and work on the mime code. fix random charset encoding issues, in the various weird mime ways, do header wrapping etc etc. check fidelity of conversions, and capture some more properties as headers, such as importance which i don't do yet (initial work done)
  * non mail conversions (look further into vcard, ical et al support for other types of msg)
  * extend conversion to make better html. this is longer term. as i don't use the rtf, i need to make my html better. emulating some rtf things. harder, not important atm.
  * add option to have "text/rtf" section output. add some options to control the automatic "text/rtf" -> "text/html" / "text/plain" conversions that are done automatically etc.
  * rather than extend my rudimentary mime class, i've finally seen a worthwhile equivalent library - TMail. see if there is a gem available (to keep installation easy), and whether actionmailer fixes re encoding are going into it. submit relevant patches if needed.