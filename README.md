# stl-to-cnc-workflow
Objective is to take a 3D file (in the form of STL) and CNC it out using only opensource software.


Whilst capability 
Ideally this will eventually be done using [Freecad's](https://www.freecadweb.org) [3D Surface](https://wiki.freecadweb.org/Path_3DSurface) tool in the [Path workbench](https://wiki.freecadweb.org/Path_Workbench) , its not quite ready.

To benchmark the approach, I'm use the RAMbone slingshot design by Jörg Sprave as it is an organic shape and useful afterwards.

<img width="964" alt="STL in blender" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/blender.png">

## Approach
The [CNC](https://miscpro.com/product/moot_one-desktop-cnc-machine/) my machine is based on is 3-Axis, restricting cutting to occur on X (left/right), Y (forward/back) or Z (up/down); So the model will need to be split into layers to allow workpiece flipping or rejoining of layers.

Models will then be converted to an image [depth map](https://en.wikipedia.org/wiki/Depth_map) and from the depth map to [g-code](https://en.wikipedia.org/wiki/G-code). 

G-Code will then be executed on my [ESP32 based GRBL CNC controller](https://github.com/bdring/Grbl_Esp32)


### Slicing the STL image

[Slic3r](https://slic3r.org/) provided the most trouble free experience to both slice at given depths, fill remaining voids left by the cut and flip the models where required such that the flat surface was down.

```
$ slic3r --cut <depth> <file.stl>
```

The model I used was ~53mm deep, so needed at 3 sheets of 19mm ply to complete. Dividing the model by 3 made the first cut 17.66mm, to which I then cut the new `lower` stl with the same parameters (resulting in `model-upper.stl`, `model-lower-upper.stl` and `model-lower-lower.stl`).


as `model-lower-lower`'s flat side faces down, the model had to be flipped 
```
$ slic3r --rotate-x 180 <model-lower-lower.stl>
```

#### Alternatives for slicing
* [Freecad](https//www.freecadweb.org)
* [blender](https://www.blender.org)

### Creating the Depth map
[stl2png](https://fenrus75.github.io/FenrusCNCtools/javascript/stl2png.html) is a quick win. It was so quick that I did it directly in javascript via the browser version. It turns out there's a [command line version](https://github.com/fenrus75/FenrusCNCtools/tree/master/stl2png) which I didn't discover until writing this document.

A width of 2048 pixels resulted in good enough resolution.

<img width="964" alt="Top slice as depthmap" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/top.png">

#### Alternatives for depth map
* [blender](https://www.blender.org)

### Adding hold-down-tags / reduce waste

All of the image to gcode tools I came across lacked some handy features such as `hold down tags` to ease clamping and `profiling` to reduce waste. I resulted in simulating through image manipulation in [GIMP](https://www.gimp.org)

Firstly I created a profile by 
* Fuzzy selecting the background
* Reducing selection size by ~10 pixels
* Painting white to avoid cutting

Hold down tags were then created by using brush with a black-grey-black gradient stroke and bridged the model via the profile to the newly white surroundings. I did this where ever I wanted hold downs.

As my model slice depths were slightly shallower than my plywood thickness, this wasn't strictly as there was supporting material I sanded back later. It does however give the options to have the cut complete all the way through and remove this step.

<img width="964" alt="Depthmap profiled with tags" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/top-b.png">

### Depth map to gcode
[image-to-gcode](http://www.linuxcnc.org/docs/2.4/html/gui_image-to-gcode.html) from [linuxcnc](http://linuxcnc.org/) worked well enough here. 

Important notes
* Calculate `Pixel Size` to match the original STL model dimensions. `<model width>/<image width (2048)>` did the job
* Set `Depth` to match the the cut size used when slicing each layer

and for a nice finish
* `Lace Bounding` Full
* `Scan Pattern` Rows then Columns gives a nice finish
* `Scan direction` Alternating
* `Tolerance` 0.001
* `Stepover` 3

<img width="964" alt="Image2gcode" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/image2gcode.png">

#### alternatives to image-to-gcode
* [robomechs/image-to-gcode](https://github.com/robomechs/image-to-gcode) - Same as linuxCNC1s image-to-gcode with tool shape options for tapered bits

### G-Code tidy up and preview
All of my gcode [CAM](https://en.wikipedia.org/wiki/Computer-aided_manufacturing) tidy-up and prep is completed with [bCNC](https://github.com/vlachoudis/bCNC) 

<img width="964" alt="bCNC" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/bCNC.png">

Previewing toolpaths is done with [Camotics](https://camotics.org/)


# Findings
This seems to produce a nice result however machinine time is quite lengthy as it isn't an optimal path.

Most tools seem to be designed for debian based systems and python2, so much fun was had transposing to fedora and python3. For times where I just wanted to use the tool, building against the native operating system via [containers](https://en.wikipedia.org/wiki/List_of_Linux_containers) was the go.


<img width="964" alt="Result" src="https://github.com/jelsdon/stl-to-cnc-workflow/blob/main/images/finished_rambone.jpg">
_note: I added a hand-cut mild steel layer for extra strength so no one needs to eat plywood if they ever used it_
