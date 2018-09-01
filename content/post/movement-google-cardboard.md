---
title: "Movement Experimentation in Google Cardboard"
date: "2017-02-01T16:48:56-07:00"
draft: false
---

I love how accessible Google Cardboard is. Got a smartphone and [$7][knox v2]?
You're fully equipped to experience virtual reality in your home.

With that comes some trade-offs, though. The stock Cardboard headsets don't
come with head straps (though solutions [do exist][janky strap]), the interface
consists of one button, and most people aren't going to be running your
application on top-of-the-line specs. Working with limitations can be fun,
though. When used to its fullest extent, I think Cardboard can deliver some
very cool little experiences.

## The Problem

Movement in virtual reality is a very hot topic right now. Developers have to
play a tricky balancing act between comfort, immersiveness, and availability.
Teleportation is the most common movement mechanic in virtual reality, because
it is simple to implement and doesn't cause nausea, but teleportation doesn't
feel natural and it's generally an unimmersive and sometimes jarring
experience.

These issues are well-outlined in other articles,
[like this one][unity movement].

## My Solution

[Here is the video of it in action.][video]

This is the space that my user needs to traverse.

![Landscape](/cardboard-forest/landscape.png)

Clearly, it's pretty small, but I made a few key decisions that make this area
good for my demo and the Cardboard platform.

1. There are points of reference close-by in every direction, in the form of
   trees. This helps the user get a better idea of how they're moving and helps
   prevent motion sickness.
2. The goal is immediately noticeable and visible from any point among the
   trees. This prompts the user to try to move.
3. The graphics are purposely low-poly and use solid colors. Maintaining a
   comfortable (read: very high) frame rate on mobile phones can be difficult.
   You can see Google use the same sort of art style in their demos.

This is what the environment looks like to the user.

![View](/cardboard-forest/view.png)

When the player inevitably tries pressing the trigger on the headset, they will
see a blue object tossed in front of them.

![Thrown Teleporter](/cardboard-forest/thrown-teleporter.png)

The object will roll a ways in front of them, slowing down as it progresses,
like how you might expect a ball to roll.

![Teleporter on the Ground](/cardboard-forest/teleporter-on-ground.png)

After a short period of time, the player will move to the area the object was
at.  The movement is quick and with a consistent velocity, both important
things to keep the player comfortable. Before the move, the player gets to see
where they will end up relative to the vantage points around them, and the
gradual movement towards that point keeps the player from losing track of where
they are.

[knox v2]: https://www.knoxlabs.com/products/knox-v2
[janky strap]: https://www.reddit.com/r/GoogleCardboard/comments/2epbbl/best_method_for_headstrap/ck1vpwq/
[unity movement]: https://unity3d.com/learn/tutorials/topics/virtual-reality/movement-vr
[video]: https://www.youtube.com/watch?v=H0fgeGyrKH4
