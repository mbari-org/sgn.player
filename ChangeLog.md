2021-08

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
