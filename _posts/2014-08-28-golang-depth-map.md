---
layout: post
title: In Too Deep with Google Depth Maps and Golang
thumb: 2014-08-28-golang-depth-map1.png
---

*Modern art or half-assed depth map visualization? You decide.*

![A not-quite-there-pointcloud](/assets/2014-08-28-golang-depth-map1.png)

I had an idea. This is usually a bad idea; when I think I have a good idea, I dedicated myself to sorting it out until I discover it was, in fact, a bad idea. I know I could break this loop, but I've only seen Primer twice and still don't understand recursion.

Anyway, back to the idea. Have you tried the Lens Blur option in the latest version of Android's camera software? [It's pretty neat](http://googleresearch.blogspot.com/2014/04/lens-blur-in-new-google-camera-app.html). Even neater is the depth map it generates, which led to all sorts of fun [web apps](http://depthy.me/).

So these depth maps got me to thinking...if the depth map is simply a representation of z-coordinate space for pixels in a photo, what if you had multiple angles of the same subject, with known distances for each origin point? With multiple lens blur images, couldn't you recreate a full pointcloud of the object you're photographing with a few vector transforms? That's just math. And I knew math once! Based on [Google's Depth Map metadata standard](https://developers.google.com/depthmap-metadata/) I should have everything I need to reconstruct a full pointcloud with multiple angles. Minus the actual skill to do what I wanted to do, but I figured I'd learn that along the way. Oh, the folly of youthful innocence.

## If You Stare Into the Abyss, XML Stares Back Into You

Idea formed, I decided to limit my scope to the reconstruction of a pointcloud in PLY format, which I could eventually use with [Potree](http://potree.org/) if I felt like getting WebGL-fancy. I decided to use only two angles (front and back) because the math is waaaay easier that way. I also opted to use Golang, because I'm most familiar with it and it's fun...

...except when it doesn't have the libraries you need. Turns out the depth field is embedded in what's called XMP, which is "Adobe" for "Fuck You." There were plenty of Javascript tools for extracting the depth field as an image, but I wanted to be "closer to the metal and keep it Golang-native," which is "Josh" for "I guess we're reinventing the wheel again."

After some rudimentary Googling, I realized that Adobe's XMP Specification documents were both useless and probably riddled with PDF-related security holes. Eventually, I discovered that there was a kind of "marker language" in JPEGs, and that XMP data is stored in "segments" denoted by markers that are just pairs of bytes. I spent the first few hours relearning Golang I'd forgotten, and deciding that the FF/APP1 markers OxFFFF/OxFFE1 meant a location in the byte slice instead of an actual byte pair (d'oh). This led to...gibberish. But still, I managed to figure out how to import an entire JPEG as a slice of bytes (in Golang it's the same as a slice of unsigned 8-bit integers) which was a minor victory. Here's how that code began to form:

	func gDepthReader(path string) {
		//jpeg markers for segments
		FF := byte(255)
		APP1 := byte(225)
		//import jpeg as slice of bytes
		data, err := ioutil.ReadFile(path)
		if err != nil {
			panic(err)
		}
		//OMG SO MUCH MORE TO DO
	}

In celebration of my minor victory, I had a drink and fell asleep, promptly forgetting everything I learned about JPEG markers the following morning. I call this phenomenon "alcohol-related technical debt." Expect to see it mentioned in the literature behind the "Drunk Startup Movement," which is poised to usurp the Lean Startup throne (I'm available for speaking/slurring engagements, by the way, and prices start at 4 beers per hour).

Long story marginally shorter, I finally found the XMP segment after discovering that the next two byte pairs after the FF/APP1 markers actually prescribe the length (in bytes) of the entire XMP segment...and after determining that I needed the SECOND APP1 marker for Android Lens Blur images. Huzzah! My code below:

	//initialize marker index, length
	var begin, leng uint
	var no2 bool

	//there must be a better way to do this, but i'm an idiot, thus, brute-force FTW

	for ; ; begin++ {
		if data[begin] == FF {
			if data[begin+1] == APP1 {
				//if this isn't the second marker, keep going
				if !no2 {
					no2 = true
					continue
				} else {
					//Herein lies dragons. The length of the APP1 segment, in bytes, 
					//equals the next two bytes, minus the namespace bytes for XMP.
					leng = 256*uint(data[begin+2]) + uint(data[begin+3]) - 31
					//Actual content for APP1 segment begins 33 bytes later
					begin += 33
					//check is a helper function for err-panic, and meta is a struct with Google Depth Map compliant XMP-XML
					err = xml.Unmarshal(data[begin:begin+leng], &meta)
					check(err)
					break
				}
			}
		}
	}

If there are holes here, don't worry, you can always skip ahead and check the [github repo](https://github.com/ACityInOhio/google-depth-to-pointcloud).  Anyway, I thankfully managed to extract the proper XML! Only to discover it wasn't all there...apparently JPEG only allows 64KB for each APP1 segment, which means if you want to embed something big like, oh, say, a giant fucking depth map, you need to use something called "Extensible XMP," which is "Adobe" for "Fuck You More." Basically, the rest of the depth map was hidden in successive APP1 markers of 64KB length. Figured out how to retrieve the full depth map through Extensible XMP with this awful loop:

	for ; begin < uint(len(data)); begin++ {
		if data[begin] == FF {
			if data[begin+1] == APP1 {
				//Herein lies bigger dragons.
				//The length of the ExtendedXMP APP1 segment, in bytes, equals the
				//next two bytes, minus namespace, GUID, length and offset bytes
				leng = 256*uint(data[begin+2]) + uint(data[begin+3]) - 77
				//Actual content for ExtendedXMP APP1 segment begins 79 bytes later
				begin += 79
				//Append the data to bite, which would later be unmarshalled to &meta
				bite = append(bite, data[begin:begin+leng]...)
				//Move loop forward
				begin = begin + leng - 1
			}
		}
	}

This all seemed to work, and I tested by outputting the XML and the included PNG into files (the PNG had to be base64-decoded, but luckily that isn't too hard with Golang's standard libraries). And it worked! I'm going to skip the hours I spent trying to make the XML properly unmarshal itself into a struct, and just say: Huzzah 2.0!

## The Curious Case of Missing XML Fields

Unfortunately, I learned that Android Lens Blur didn't actually use [all the fields in their supposed Depth Map metadata standard](https://developers.google.com/depthmap-metadata/reference), which makes complete sense...since some of the fields seem rather impossible to determine with typical phone sensors (like "depth units" for example). Here's part of my XML struct with the missing elements commented out:

	type GDepthMeta struct {
		XMLName xml.Name `xml:"Description"`
		Format  string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Format,attr"`
		Near    float64  `xml:"http://ns.google.com/photos/1.0/depthmap/ Near,attr"`
		Far     float64  `xml:"http://ns.google.com/photos/1.0/depthmap/ Far,attr"`
		Mime    string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Mime,attr"`
		Data    string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Data,attr"`
		//These attributes are standard but don't appear in Google Camera lensblur images
		/*Units          string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Units,attr"`
		MeasureType    string   `xml:"http://ns.google.com/photos/1.0/depthmap/ MeasureType,attr"`
		ConfidenceMime string   `xml:"http://ns.google.com/photos/1.0/depthmap/ ConfidenceMime,attr"`
		Confidence     string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Confidence,attr"`
		Manufacturer   string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Manufacturer,attr"`
		Model          string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Model,attr"`
		Software       string   `xml:"http://ns.google.com/photos/1.0/depthmap/ Software,attr"`
		ImageWidth     string   `xml:"http://ns.google.com/photos/1.0/depthmap/ ImageWidth,attr"`
		ImageHeight    string   `xml:"http://ns.google.com/photos/1.0/depthmap/ ImageHeight,attr"`*/
	}

Two of these fields (Format, Mime) appear to remain static with Android Lens Blur (RangeInverse, Image/PNG, respectively) so I don't really need to bother with them...but Near, Far and Data are quite important. Data is the actual base64-encoded PNG depth map, while Near and Far are the nearest and further points in "depth units." Ideally, these would have actual spatial information, but unfortunately, it appears to be a relative measure for each image. The metadata is missing the critical "units" measure, which would make manipulation across many images more manageable, but we'll make some fun assumptions in the meantime: namely, that the near/far units are relatively similar and that the two images were taken as symmetrically as possible.

## To the (Point)Cloud!

Now came the fun part. With the original JPEG, the XML properly unmarshaled into a Golang-y struct, and the PNG of the depth map, I could finally create X,Y,Z point values and do fun transforms. First, I made a "Point" type to hold each point and color value, and I needed a proper template that could interpret a slice of Points into a properly formed PLY file.

	//Pointcloud Data Structure
	type Point struct {
		X, Y    uint
		Z       float64
		R, G, B uint
	}

	//Template for PLY files
	var ply_template = template.Must(template.ParseFiles("ply_template.txt"))

And now my naive function for finding a pointcloud with "front" and "back" Lens Blur images, explained. Luckily didn't have as much trouble coding this, except for some annoying issues with the Golang color package...so I'll just go ahead and explain the function step by step.

	func MakePointCloud(front_path, back_path string) {
		//get front and back images, depths, near/far points
		front, front_depth, front_near, front_far := gDepthReader(front_path)
		back, back_depth, back_near, back_far := gDepthReader(back_path)

		//convert images and depth maps to PointCloud
		bounds := front.Bounds()
		point_num := (bounds.Max.Y - bounds.Min.Y) * (bounds.Max.X - bounds.Min.X) * 2
		the_cloud := make([]Point, point_num)
		//Index for pointcloud
		i := 0
		//normalize GDepth near/far by averaging
		near := (front_near + back_near) / 2.0
		far := (front_far + back_far) / 2.0

Okay, so first thing you'll note is that I modified my gDepthReader function with a different return signature...just so you have the data types straight, here it is:

	func gDepthReader(path string) (pic image.Image, depth image.Image, near float64, far float64)

So front, front\_depth are image.Images (part of the Golang standard library) while front\_near and front\_far are just float64s. Ditto for s/front/back/g. Bounds() is a rather droll image.Image method to determine the maximum and minimum resolution. I then allocate a slice of Points equivalent to the number of pixels in both images (yes, I'm assuming both images are the same size here). i is the running index for my slice of Points, the\_cloud. And in absence of fancier math and/or actual units for depth, I'm naively normalizing the near/far points in the depth maps. 

Now this is the part of Sprockets where we loop. Front, then back.

	for y := bounds.Min.Y; y < bounds.Max.Y; y++ {
		for x := bounds.Min.X; x < bounds.Max.X; x++ {
			z := findDepth(near, far, front_depth.At(x, y))
			r, g, b, _ := front.At(x, y).RGBA()
			the_cloud[i] = Point{uint(x), uint(y), z, uint(r >> 8), uint(g >> 8), uint(b >> 8)}
			i++
		}
	}

	for y := bounds.Min.Y; y < bounds.Max.Y; y++ {
		for x := bounds.Min.X; x < bounds.Max.X; x++ {
			z := findDepth(near, far, back_depth.At(x, y))
			r, g, b, _ := back.At(x, y).RGBA()
			the_cloud[i] = Point{uint(bounds.Max.X - x), uint(y), far - z, uint(r >> 8), uint(g >> 8), uint(b >> 8)}
			i++
		}
	}

Let's set aside the findDepth() function for a moment, and just assume it gives us a reliable z-value for a point. This is pretty much where the magic is happening. We find the z value of a particular pixel using the depth map, find the colors of the pixel, then do a little bit of type coercion and bit-shifting to get everything into our pointcloud. Notice anything different about the back loop? We're pulling a little Missy Elliot here and flipping and reversing the x and z directions (bounds.Max.X - x and far-z, respectively). The math would be a bit trickier with multiple angles (and would yield a higher quality pointcloud) but front/back images keep it simple. 

But how does the findDepth() function work? Well, in accordance with the [RangeInverse Method as explained by Google](https://developers.google.com/depthmap-metadata/encoding). And with some annoying color-conversion. Here's the function in all its glory.

	func findDepth(near float64, far float64, d color.Color) (z float64) {
		//For whatever reason this doesn't type assert to color.Gray, so I have to do some stupid math
		r, g, b, _ := d.RGBA()
		//converts to grayscale
		dn := float64((r + g + b) / 3)
		//normalizes once it's a float
		dn /= (255 * 256)
		//use Google Depth map's RangeInverse method to find z-value
		z = (far * near) / (far - dn*(far-near))
		return
	}

And that's it! Just need to output the\_cloud through our PLY template, and bob's your uncle/pointcloud. Here's the final part of the MakePointCloud function.

	file, err := os.Create("./the_cloud.ply")
	check(err)
	defer file.Close()

	writer := bufio.NewWriter(file)
	err = ply_template.Execute(writer, the_cloud)
	check(err)
	writer.Flush()

	return
	}

## So Baby, Do We Have a Pointcloud Stew Going?

Short answer...no. The results are lackluster, but frankly I'm just amazed that it worked. I would have attached a full PLY file, but it's 50 MB and I wouldn't do that to Github. Instead, here's an image of my tea thermos front-back-afied from Lens Blur images. I used [Meshlab](http://meshlab.sourceforge.net/) to view it...as you can see, it's a bit garbage.

![Not quite 3d pointcloud view 2](/assets/2014-08-28-golang-depth-map2.png)

I was hoping this idea would morph into an "Instagram of 3d imaging" product that could steal enough precious user attention away from Facebook that they'd be forced to buy me out, but alas, it's not even close in its current form. Back to the drawing board with user-attention-ransom-based-startups I suppose.

## Lessons Learned and Speculation Time

So there are two major things I took away from this experience:

* I should stop having rubbish ideas, or figure out they're rubbish a bit sooner. 
* Google's Depth Map metadata standard wasn't made for Lens Blur. It uses less than half the "standard" fields. It was really concocted for [Project Tango](https://www.google.com/atap/projecttango/#project).

Which makes me wonder (usually a bad idea, as you've seen). There's going to be a very cool space where VR and mobile imaging collide, and it seems like Project Tango is probably at the forefront of that evolution...so perhaps my idea is rubbish now, but if everyone is rocking portable depth sensing equipment and better sensor for 3d position tracking? Baby, we could have a stew going.

## Helpful Links

Check out the [code on github](https://github.com/ACityInOhio/google-depth-to-pointcloud). And if you feel like discussing, check the HN link.
