# pcx-js - loading and decoding pcx files using JavaScript

This is just a little attempt at writing a very simple [PCX](https://en.wikipedia.org/wiki/PCX) decoder for JavaScript.

This loader only supports 24bit (3 planes) PCX files.

See it in action: 

## How to use it?

Simply open the html file and drop a local PCX file onto the drop zone and it
will be decoded:

![page1](./img/inaction.png)

## PCX file format

PCX file format is quite simple.

Every PCX file starts with the magic word `10` (0xA) and is followed by a version number from 1 to 5.

The header is 128 bytes long and can contain an optional 16 colors palette:

| Offset        | Size (bytes)  | Description                 |
| ------------- |:-------------:| ---------------------------:|
| 0             | 1             | PCX magic word              |
| 1             | 1             | PCX version                 |
| 2             | 1             | Encoding (0x1 == RLE)       |
| 3             | 1             | bitsPerPlane                |
| 4             | 2             | Xmin                        |
| 6             | 2             | Ymin                        |
| 8             | 2             | Xmax                        |
| 10            | 2             | Ymax                        |
| 12            | 2             | Horizontal resolution (dpi) |
| 14            | 2             | Vertical resolution (dpi)   |
| 16            | 48            | 16 color palette            |
| 65            | 1             | bitplanes                   |
| 66            | 2             | bytesPerRow                 |

The width and height of the image is given by `Xmax - Xmin + 1` and the size of a scanline is give by `width * bytesPerRow`.

* Note * Since bytesPerRow is even, there may be an optional marker at the end of each plane.

## PCX RLE encoding

PCX uses a quite simple encoding called RLE (`Run-length encoding`) which is a lossless data compression in which several consecutive pixels of the same color are grouped in a sequence. Instead of saving each pixel, RLE saves the count and then the value that needs to get repeated.

Let's imagine a picture with the following colors:

```
0x5 0x10 0x10 0x10 0x10 ...
```

The encoded RLE data would look like this:

```
0x5 0xc4 0x10 ...
```

The first pixel is not repeated so appears decoded, for the next one, we generate the run length using 0xc0 | 0x4 and then append the pixel value 0x10.

## PCX 24 pixel format vs HTML Canvas

Each PCX scanline is divided into RGB `planes` (there may be an optional third play for luminance or alpha but this is not supported). For example, a 4x4 24bit file would look like this:

```
row 1
RRRR
GGGG
BBBB

row 2
RRRR
GGGG
BBBB

row 3
RRRR
GGGG
BBBB

row 4
RRRR
GGGG
BBBB
```

HTML Canvas uses the following `RGBA` pixel format and stores each pixel consecutively, so the previous 4x4 24bit file would be encoded like this in Canvas' pixel format:

```
row 1
RGBA RGBA RGBA RGBA
row 2
RGBA RGBA RGBA RGBA
row 3
RGBA RGBA RGBA RGBA
row 4
RGBA RGBA RGBA RGBA
```

It may be possible to decoded RLE pixel data and convert it to RGBA pixel format in a single pass, but it's done in two pass for now:

 - PCX.decode first decodes RLE pixel data into `RRRGGGBBB` format
 - PCX.toRGBA then converts PCX pixel data into Canvas `RGBA` format

 ## Rendering using Canvas

 Rendering pixel data using the Canvas object is quite simple using the `ImageData` object return by `CanvasContext.createImageData`.

 After having filling in each pixel RGBA's data, drawing pixel onto the screen is as simple as calling `CanvasContext.putImageData`.

 ## Working on binary data using JavaScript

 Since a few years, JavaScript provides an easy way to work with binary Data using the `ArrayBuffer` and `TypedArray` objects.

 And most filesystem APIs like the FileReader API have been updated to work with it:

 ```javascript
 var reader = new FileReader();
        
reader.onload = (e) => {
    // An array buffer is a simple binary buffer
    let buffer = e.target.result;
    
    // To access it we simple create a TypedArray view
    let byteView = new Uint8Array(e.target.result);
    
    // now we may read/write on the buffer using the typedArray:
    byteView[0] = 0x24;

    // The neat thing is that much like with C pointers, you may
    // have several views pointing to the same buffer:
    // create a Word view on the same buffer
    let wordView = new Uint16Array(e.target.result)
    wordView[0] = 0x2435;
}

reader.readAsArrayBuffer(file);
```

## What's missing

This was a simple experiment and does not support any paletted PCX files nor files using a transparency channel.