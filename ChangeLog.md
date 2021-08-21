2021-08

- Note: scrolling with spectrogram gets pretty slow.
  Is there a way to speed it up?
- enable spectrogram for any zoom.
  However, for now, destroying the spectrogram when zoom is changed
  as it seems the zoom change doesn't propagate to the spectrogram.
- allow some spectrogram params
- recreate wavesurfer upon loading file.
  This seems to fix some misbehavior when trying to reuse the
  same instance, despite associated cleanup prior to load file
- upgrade wavesurfer.js to 5.2.0
- upgrade project to quasar2 (2.0.3)

2020-04

- quasar upgrade (now 1.6.1) as a general way to also address
  recently reported vulnerability re "kind-of".
  Let's see with a "kind-of=6.0.3" resolution,
  Let's see with a "minimist=1.2.3" resolution,
  App seems working fine even with several warnings like
  `warning Resolution field "kind-of@6.0.3" is incompatible with requested version "kind-of@^3.0.2"`

2019-09

- initial version
