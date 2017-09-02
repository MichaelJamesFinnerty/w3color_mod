# w3color_mod.js
###### Fork of w3color.js by W3 Schools for Mountain Oak Design (mod) <br /> fixes hex/ncol input detection bug and adds name property to ColorObject
As part of Mountain Oak Design's HTML Named Colorwheel, we decided to use the w3color.js module developed by the fine folk at W3 Schools.
* Unfortunately, we discovered an issue in how the module interpreted hex value inputs!
* This fork addresses the hexl/ncol input parsing but, and also makes adds a name property to the ColorObject.

## Hex/ncol Input Detection Bug
### w3color.js
The w3color module does a solid job of parsing a variety of inputs (rgb, hsl, cmyk, ncol, &c.), and it can even parse some of them (including hex rgb) without typing syntax through the toColorObject() function.

For the HTML Named Colorwheel, MOD needed to retrieve the full arrays of color names and associated hex values from w3color.js, however I observed that certain colors (specifically FireBrick and MediumVioletRed) were not properly loading from their hex values.

#### ncol -- the 'natural' color standard
>Not familiar with ncol? It is a new color standard that is not yet part of the HTML standard, but which W3 Schools is pushing heavily well in advance. It seems interesting, and I encourage you to check it out, but for our purposes this line presented only problems.

>The ncol standard is formatted as an identifying letter `[RYGCBM]`, then three comma separated percentage values measuring the distance from the initial color, white, and black (white and black cannot combine to greater than 100). For example, FireBrick, a.k.a. `rgb(128,0,0)` would be expressed in ncol syntax as `R50,0,0`

### Identifying the issue
The issue was noticed because two named colors (`FireBrick: #B22222` and `MediumVioletRed: #C71585`) would not load into the main color array.

Tracing the failure back up through the module led to the toColorObject function, specifically the point where the input is scanned for ncol formatting:
```js
if (( x == "R" || x == "Y" || x == "G" || x == "C" || 
      x == "B" || x == "M" || x == "W") && !isNaN(y)
  ) {
    c = "ncol(" + c + ")";
  }
```
This line looks for the necessary ncol identifying letter `[RYGCBM]` and whether the remainder of the input is not a number. This is insufficient because it incorrectly catches hex values such as `B22222`.

Without modifications, the module is unable to return a valid response for the argument `B22222`. The module will try to interpret the input as an ncol value, but to do so it requires a color distance value less than 100. Because the rest of the input is '22222', the module will not be able to interpret this input as valid ncol notation and will return a default "R0" color object.

### Resolving the issue
The key to distinguishing between a hex and ncol input comes down to the presence of commas. A hex value will always be five consecutive hex value characters, while an ncol input (which may contain between 0 and 6 digits after the color indicator) will always contain commas.

To prevent the ncol recognition from triggering on a hex input, I inserted a regular expression check that looks for five consecutive alpha-num characters.
```js
ncolRe = /\w{5,}/; 

if (( x == "R" || x == "Y" || x == "G" || x == "C" || 
      x == "B" || x == "M" || x == "W") && !isNaN(y) && !(ncolRe.exec(y))
  ) {
    c = "ncol(" + c + ")";
  }
```
This allows for value like `B22222` to be bounced for hex processing (FireBrick red) while still allowing `B2,22,22` as an ncol input (a dark periwinkle).

## Adding a name property to the ColorObject
In early drafts of the HTML Named Colorwheel, we were wasting a lot of time looking up the name for each ColorObject after they were created from hex.

We decided to streamline this process by adding a name property to the ColorObject (which defaults to "undefined" if it is not an HTML named color).

> The w3color.prototype includes a toName() method, but this was not convenient for use with the ColorObjects returned by the toColorObject() function.

### Determine the color's named array index
The first step is to determine the index of the color name at the end of the toColorObject() function.
```js
  //identify (and return) the index of the
  //current color in the reference arrays
  var index = getColorArr('hexs').indexOf(c);
```
The index is then added to the arguments passed to the ColorObject.
```js
  return colorObject(rgb, a, hue, sat, index);
```
### Adding the name property to the colorObject
Within the colorObject() function, tThe index value will be used in the colorObject function as "i", and will be used to set a new var -- "name".
```js
function colorObject(rgb, a, h, s, i) {
  var hsl, hwb, cmyk, ncol, color, hue, sat, name;
```
Lastly, the index is evaluated to set the name. Undefined colors will have an index of `-1` and will default to a value of `undefined`. Index values >= 0 will return the associated color name.
```js
  //retrieve the name of the current colorObject
  name = (i > -1 ? getColorArr('names')[i] : 'undefined');
  
  color = {
    red : rgb.r,
    green : rgb.g,
    blue : rgb.b,
    hue : hue,
    sat : sat,
    lightness : hsl.l,
    whiteness : hwb.w,
    blackness : hwb.b,
    cyan : cmyk.c,
    magenta : cmyk.m,
    yellow : cmyk.y,
    black : cmyk.k,
    ncol : ncol,
    opacity : a,
    valid : true,
    name : name                 //<-- set the name of the color object as a property
  };

```

### Resulting name retrieval
This allows for us to directly access the name of colors without having to call a getter function.
```js
  var color_ex = toColorObject('#B22222')
  console.log(color_ex.name)    //<-- will log "FireBrick" to the console
```
