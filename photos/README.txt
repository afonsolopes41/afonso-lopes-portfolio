CHENGDU GALLERY
===============
35 photos, named chengdu-01 .. chengdu-35 in the order they appear on
the site (opening ceremony -> campus -> labs -> city -> excursions ->
cohort -> certificates -> flight home).

Each photo exists twice:

  chengdu-NN.jpg        760px long edge  - the rail thumbnail
  chengdu-NN-full.jpg  1600px long edge  - loaded only when opened

The untouched originals are in _originals/ (gitignored, 43 MB). Thirteen
of them are HEIC, which Chrome and Firefox cannot display at all - that
is why everything is converted rather than served directly.

TO CHANGE THE ORDER OR THE CAPTIONS
Edit the SHOTS array in index.html (search for "var SHOTS"). Each entry
is { f: 'chengdu-NN', c: 'CAPTION', w: <full width>, h: <full height> }.
The w/h are used to reserve the lightbox frame before the image loads.

TO ADD A PHOTO
Save it at both sizes with the next number, then add an entry to SHOTS.
