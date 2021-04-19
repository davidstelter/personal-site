+++
title = "Dactyl Manuform"
description = "My homemade Dactyl Manuform - a daily typer that inspires befuddlement"
weight = 0

[taxonomies]
tags = ["keyboards", "3dPrinting", "microcontrollers", "qmk"] 
+++

{{img_fit_width(path="dactyl-lit.jpg", width=800)}}

This board is a mashup of the Dactyl and the Manuform, and there are loads of variants of it on github. Like the original, all the ones I've seen are tweakable for layout/number of keys,
curve/contour radii, shell height etc.

It's a great project (if you like point-to-point wiring) and has become my daily typer - it feels pretty natural now that I've settled on a keymap.

{{img_fit_width(path="dactyl-mcu-wiring2.jpg", width=800)}}


## Thumb Cluster

The thumb cluster is so much better than on an Ergodox or Kinesis Advantage. That being said, it could still use improvement!
The two large buttons, which I map to space, backspace, delete, enter and command (⌘), are perfectly well placed, and the first pair of buttons isn't bad either, but the bottom two are a bit of a stretch.
I'm still trying to figure out what arrangement I'd like for these, and custom keycaps might be part of the answer.

## Key Switches

I'm using [Hako Violets](https://input.club/the-comparative-guide-to-mechanical-switches/tactile/hako-violet/),
which are a light tactile switch from a set designed to resemble the feel of a Topre board. These don't nail that perfectly but get close enough to make me happy.

## Keycaps

Keycap sets for Ergodox layouts (often just called "ergo") are a good fit for this board, depending on the particulars of the chosen layout. I went with a layout nearly identical to the Ergodox,
with the notable change that the two innermost columns are all 1u keys, where the Ergodox uses two 1.5u keys.

{{img_fit_width(path="dactyl-unlit.jpg", width=800)}}

Currently I have [Matt3o /dev/tty MT3](https://drop.com/buy/drop-matt3o-devtty-custom-keycap-set) caps installed, which are a good profile for this board and have a very nice surface feel. 

## Firmware & Controllers

For firmware, [QMK](https://qmk.fm) is the obvious choice. Has great tooling, tons of customization and lots of fun features like audio, lighting, LCD/OLED screens, encoders, etc.

I'm using a pair of [QMK Proton C](https://qmk.fm/proton-c/) controllers, which actually turned out to be a little bit of a headache as some of the addon features like internal RGB lighting use
funky drivers on ARM chips that didn't play too nice with the style of serial communication between the two controllers. I ended up making it work by having the RGB
controlled by just one controller, and running the control signal over a spare wire on the cable that connects the two halves.
