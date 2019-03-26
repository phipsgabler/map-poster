# Create your own hipster map poster

This repo desribes how I created [this map poster](oxford/oxford-large-map.pdf) from raw OpenStreetMap data, 
using just open source software.  I tried to mimic the style of the posters available from 
[grafomap](https://grafomap.com/) and [zulumaps](https://grafomap.com/); they probably use an equivalent 
procedure internally.

This example will use a map of Oxford, UK, but it works the same for all other maps.

## Software

As mentioned, you will need some software, but all of it is free and open source.  I used [InkScape](https://inkscape.org/)
for vector graphics and PDF generation, [Osmosis](https://wiki.openstreetmap.org/wiki/Osmosis) for data extraction,
and [Maperitive](https://wiki.openstreetmap.org/wiki/Maperitive) for map rendering and conversion to SVG.

Inkscape works really well, although a full map is noticeably heavy memory load for it.  Osmosis is a usable command line 
tool with consistent interface.  Maperitive has
some quirks -- it uses a lot of memory some times, or freezes, so we have to watch out filter away everything we
don't need first.  Also, it uses local paths internally, so you have to start it with its installation directory
as working directory.


## The process

### Get raw data

First, you have to get the OSM data for the region you want to create a map of.  openstreetmap.com has an 
export function, but that is usually not available, because it is rate limited.  So I downloaded the data
of Great Britain from [Geofabric](https://download.geofabrik.de/europe.html).  This is a rather large file
around 1 GiB.


### Extract bounding box

Since this is much more than we need, we'll cut out just what we need from that map.  For this I found it 
easiest to use the [export function](https://www.openstreetmap.org/export#map=12/51.7536/-1.2400) of the OSM
web page to manually determine the coordinate boundaries I want.  

To avoid cutting the box later in the vector 
graphics, I calculated the coordinates beforehand based on the assumption that I want to have the poster printed
on A3 at scale 1:50000, which means that I want to have an area of 270 x 300 mm (leaving some space for text below --
A3 as a whole is 297 x 420 mm).  This corresponds to 13500 x 15000 m on the scaled map, which we can convert 
quite exactly to degree coordinates by the formula given [here](https://gis.stackexchange.com/a/2964):  a latitude
offset of `13500 / 111111`, which is 0.22°, and a longitude offset of `15000 / (15000 * cosd(51.7))`, which is 0.12° 
(51.7° is the approximate latitude at the center of Oxford, and `cosd` means cosine for angles in degrees).

Given these offsets, I found a bounding box that I liked by fiddling around with the positions a bit.  Then I extracted it
with Osmosis:

```
osmosis --rx file=great-britain-latest.osm --bounding-box bottom=51.6751 left=-1.3458 top=51.8078 right=-1.1481 --wx file=oxford-large.osm
```

The resulting file is [oxford/oxford-large.osm](oxford/oxford-large.osm).


### Feature extraction

Now we have all map data inside this bounding box, which is way too much.  Thus we have to Osmosis once more, to extract only
those things which will actually end up on the printed map.  In reality, I now went forwards and backwards
between rendering and extracting to try out what works best (usually indicated by memory consumption as well as aesthetics).

What I ended up using was this call:

```
osmosis --rx oxford.osm --tf accept-ways highway='*' waterway='*' landuse='*' building='*' natural='*' leisure=park --tf reject-relations --used-node --wx oxford-small.osm
```

This will [give you](oxford/oxford-large-extracted.osm) all streets, bodies of water, some greens, and "residential areas",
which are buildings and similar areas, and looks nice enough.


### Designing the rendering rule

You now have to design how you want the features on your map to look like.  Have a glance at grafomaps and zulumaps
for inspiration.  I wanted to have something very simple, with mostly black and white and some gray areas.

Maperitive has a quite elaborate syntax for how it renders OSM map features.  An introduction is given 
[here](http://maperitive.net/docs/Rulesets.html).  There's a couple of rule sets 
[available in the OSM wiki](https://wiki.openstreetmap.org/wiki/Category:Maperitive/Rules), on some of which
I loosely based the [one I ended up with](simple.mrules).


### Rendering the map

Now you can open and your extrated data file in Maperitive (I only used its terminal, which is a bit tedious, performing
quite brutal auto-complete):

```
change-dir <where your file is>
clear-map
load-source <name of extracted osm file>
use-ruleset <name of rules file>
apply-ruleset
map-zoom-scale 50000
```

`use-ruleset` needs to be followed by `apply-ruleset`, otherwise nothing will show up. `map-zoom-scale` sets the zoom 
level such that the scale is fixed to a certain ratio.

Now you can use `export-svg` or the menu to get your rendered map as an [SVG file](oxford/oxford-large.svg) 
for further processing.


### SVG cleaning

The file you end up with from Maperitive is still quite large and not ideal to work with in Inkscape.  I first opened it in 
Inkscape and removed the stuff I didn't need (like empty layers and the OSM copyright box).  

This cleaned file I ran through [scour](https://github.com/scour-project/scour), an SVG optimizer that comes with Inkscape:

```
scour oxford-large.svg oxford-large-scrubbed.svg
```

The [resulting file](oxford/oxford-large-scrubbed.svg) is about 60 % of the original and makes Inkscape hang much less.


### Designing the poster

Now you can do the actual designing.  Make a new Inkscape file, and set it to A3 portrait.  Import the scrubbed SVG of the 
map, and position it (e.g. center it horizontally, relative to the page).

I moved the scale legend somewhere with a white background, and changed the font of it to match the one I used for the title.
For this and other operations the "Objects" window was really helpful, as it allows you to precisely select the things
you want to change -- and they are usually organized quite messily (scour seems not to like layers...).

You can also edit the map, for example to remove small regions, by using the "Edit path" tool (F2 keyboard shortcut)
and simply deleting the nodes of objects in the map.

I then added a rectangle or the same width as the map with gradient of white to transparent to the bottom of the map,
so that the transition to the text becomes smoother.  The actual text is simply a text field.  I used a combination of
Baskerville (since it looks "classy") and URW Gothic (which is very geometric and constrasts Baskerville's serifs nicely).

The [final file](oxford/oxford-large-map.svg) should be exported as a 
[PDF with fonts rendered to paths](oxford/oxford-large-map.pdf), so that you don't have problems with printing
from a system which does not have the fonts you used.


