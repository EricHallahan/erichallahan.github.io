---
title: A Heat-Powered Nitinol-Actuated Motor
publishDate: 2023-04-10T21:00:00-04:00
description: A concept for leverging Nitinol in the application of lightweight actuators.
type: blogpost
cover:
    video: 
        src: https://cdn.discordapp.com/attachments/860184200996716588/1087553103433322527/render.mp4
        type: video/mp4
        width: 1920
        height: 1080
---

<section>

## Introduction

During my Fall 2022 semester at Penn State, one of the electives I was able to complete was <samp>ME 480</samp>: Mechanism Design and Analysis. This course primarily considers the design and analysis of mechanical linkages, including kinematic synthesis and dynamic analysis.

However, for our final semester project, we were offered a variety of topical options to complete (or the ability to propose our own), with the intention of allowing us to explore a bit beyond the linkage-heavy course curriculum. One of these projects considered a widely-recognized existing mechanical design problem: Leveraging [shape-memory alloys](https://en.wikipedia.org/wiki/Shape-memory_alloy) for lightweight actuators. In particular, [Nitinol](https://en.wikipedia.org/wiki/Nickel_titanium) is a widely-accessible <abbr title="shape-memory alloy">SMA</abbr> that is widely produced in a variety of forms for novelty and industry.

While we were asked to select a project topic to complete at mid-semester, as is common for myself, I procratinated on the execution of the design work until a few days before the project submission deadline, when I finally decided to pick myself up by the bootstraps and get the work done. I completed the assignment within two days, going from having minimal knowedge on <abbr title="shape-memory alloys">SMAs</abbr> to a rendered video presentation of what I could produce to solve the task in that time.

</section><section>

## Conceptualization & Design

<section>

### Early design decisions

Something I realized early in the conceptualization process was that, despite hints to consider using Nitinol to provide a large impulse to a device that could moderate that into something more useful (like a large flywheel), the most valuable type of actuator to pursue would one that provided a continuous (or non-impulsive) actuation. This drastically narrowed my solution space.

Another consideration was the forms in which Nitinol is commonly available, as it increased the potential for the design to eventually prototyped on a limited budget. Although students were not required to produce their final semester project designs for the course and were offered the opportuity to pursue prototypes if they desired with the university, my status as a student taking the course remotely meant that it wasn't particularly practical for me to do so. Regardless, it is often preferable to design around what is available off-the-shelf. Nitinol is most commonly available in lengths of straight wire, and is commonly used for orthodontic retainers in this form by leveraging the material's superelasticity. While other designs would have been possible if given the opportunity select any desired shape for the Nitinol, for these reasons I chose the abundantly available form to design around.

</section><section>

### Detailed design and concept

Conceptually, this design is based upon around a thoughtfully-shaped cammed split stator and a rotor.

I used [OpenSCAD](https://openscad.org/) as my parametric modeling software. It is not the most conventional software for modeling, but in this case, its strength in straightforward declaritive specification was very well appreciated—I don't know if I would have been able to produce this design in other software packages given the geometry. It also means that I can share the code I used to produce the both the stator and the rotor. 

<figure>

```openscad
$fn = 180;

z = 14;

difference() {
    translate([0, 0, z/4]) cylinder(h = z/2, r = 2*z, center = true);
    cylinder(h = z+1, r = 11, center = true);

    for(i = [-180:360/$fn:180]) {
        if(i == 0) {
            /*radius = z/2+10;
            rotate([0,0,i]) {
                translate([radius+z,0,0]) {
                    rotate([90,0,0]) { 
                        cube([2*radius, 2*radius, 4*z], center = true);
                    }
                }
            }*/
        } else {
            radius = z/abs(i/180)/2;
            rotate([0,0,i]) {
                translate([radius+z,0,0]) {
                    rotate([90,0,0]) { 
                        cylinder(h = 4*z, r = radius, center = true);
                    }
                }
            }
        }
    }
}
```
<figcaption>OpenSCAD code used to produce the model of the stator cams seen in the video.</figcaption>
</figure>

<figure>

```openscad
$fn = 180;

shaft = 5;

union() {
    difference() {
        cylinder(h = 3.5, r = 14.9, center = true);

        for(i = [0:24]) {
            rotate([0,0,i*360/24]) {
                translate([14.5,0,0]) {
                    cylinder(h = 4, r = 0.5, center = true);
                }
            }
        }
    }
    translate([0,0,shaft/2]) {
        cylinder(h = 14 + 3.5 + shaft + 2, r = 4, center = true);
    }
}
```
<figcaption>OpenSCAD code used to produce the model of the rotor seen in the video.</figcaption>
</figure>

The stator is the interesting piece: It is produced by subtracting a torus of revolution with variable radii, such that the hole remains a circle; on one side of the torodial axis, both the major and minor radii are finite, while in the limit at the opposing side of the toroidal axis, the major and minor radii are infinite—the camming surface profile thus varies contiuously between curved and straight. This bends the Nitonol wires embedded around the circumference of the rotor as they move towards the finite-radius end of the stator, and when heated, their straightening applys a force against the cam to drive the rotor. The OpenSCAD code produces the stator cam shape in the limit of subtracting many cylinders (and optionally a single cube at the single location where the radius is supposed to be infinite). This was a quite inefficent way of acheiving that shape (and consumed a substatial amount of my laptop's system resources in both compute and memory when rendering the mesh), but it did produce the complex shape I wanted without any additional fuss.

The concept shown in the presentation video was designed around the common skateboard bearing. These bearings are press-fit into the two stator cams and support the rotor and its integral output shaft. I imagined the split-stator and rotor both as being manufactured in a [suitably temperature-tolerant thermosoft polymer](https://en.wikipedia.org/wiki/Thermoplastic) or a [thermosetting polymer](https://en.wikipedia.org/wiki/Thermosetting_polymer) to keep the mass low, which is the primary motivation for such an actuator.

The benifits of this arangement are quite good on paper: The output from the motor is continuous, the direction of rotation is controlable by repositioning the heat source, the design is relatively compact and it is not unreasonable to imagine linking multiple such motors together in parallel to increase output. It is theoretically possible (although I do not know if realistic) to apply a substatial current to the Nitinol wires to heat them into the transition (which wouldn't be straightforwardly possible with a design that uses a continuous loop), opening the door to a direct electronic control method for the actuator.

</section><section>

## Future development

While the presented concept is incomplete and not immediately prototypable, there is not much missing from making that possible—the only part truly missing is something to keep the split-stator properly spaced apart and to keep the rotor meaningfully centered between them and axially supported. That task is not difficult to acheive, and with that minimal work it is likely that this could be prototyped on a shoestring budget—the most expensive component to procure would likely be the Nitinol itself, and that is surprisingly affordable when sold in the wire form this project requires.

I might choose to take a crack at building it soon, just because it stimulates my curiousity enough to see this (potentially) work—stay tuned for future updates on this topic.

</section>