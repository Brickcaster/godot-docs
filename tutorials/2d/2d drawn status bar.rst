.. _doc_custom_drawing_in_2d:

Custom drawn status bar
=======================

Do you want an awesome status bar that shows off a ton of information in a quick and easy manner?  Check this out, we have level, exp, current health (relative), absolute health, and temporary health all displayed in a convenient and slim package with a fancy non-flat look!  

.. image:: /img/2d_status_bar.png

The sliders are there to change the values super easily for testing. This builds upon :ref:`custom_drawing_in_2d` so be sure to be familiar with that before continuing.

Our setup
---------

To start we need a simple testing platform to feed our values into the drawing script.  I cheated a little and used a radial progress bar instead of drawing for a smoother look.  Later versions of this tutorial may include anti-aliasing using draw commands.  We’re going to need a slider for each of our values to display.  In this example I centered the level and exp progress indicator on the left side of the sprite and the bar will extend across the width of the sprite.

.. image:: /img/2d_status_bar_dev.png

Now we need to tie our progress bars in with code.  We’re going to go into our sprite script and add: 

::

extends Sprite

var TNL = 100
var Experience = 90.0

var MaxHealth = 100
var Health = 60.0
var Shield = 0.0

func _ready():

	set_process(true)

func _process(delta):
	Health = float(get_node("HSlider").get_value())
	Shield = float(get_node("HSlider2").get_value())
	get_node("TextureProgress").set_value(get_node("HSlider1").get_value())
	update() // Necessary for _draw() to run

Our max experience (TNL), and MaxHealth match the maximum value of our slider.  Be sure to rescale the slider to account for different maximums.  For shield it doesn’t matter because there is no max.

The core drawing function
-------------------------

::

	var label = get_node("Label")
	var labelcenter = label.get_pos() + (label.get_size()/2)
	var outerrad = 18
	var TotalHealth = MaxHealth+Shield
	var HealthPer = Health/TotalHealth
	var HealthCol = Health/MaxHealth
	var EffectiveHealth = Health + Shield
	var ShieldPer = Shield/TotalHealth
	var poly = []
	var polyfill = []
	var polyshield = []
	var polycolors = []

There’s a lot going on to make it look this good.  We use the center of the level label as a reference point and the top of the radial progress object (labelcenter – Vector2(0,outerrad)) as our second reference point.  This will be the 2 anchor points of our health bar and defines its height.  EffectiveHealth is used to help rescale the proportions of our health bar so we still have some indication that the unit is below max health if it has a large shield.  HealthCol is a ratio we use to define the color of our health bar so it changes from green to orange to red.

::

	# Health bar back
	poly.append(labelcenter + Vector2(0,-outerrad))
	poly.append(poly[0]+Vector2(get_texture().get_width(), 0))
	poly.append(poly[1]+Vector2(outerrad/tan(PI*7/12), outerrad))
	poly.append(labelcenter)

Here’s the back of the health bar.  poly[0] and poly[4] are those anchor points we talked about.  We use a little math to add a 240 degree sweep from the topright corner.

::

	# The green portion of the Heatlh bar
	polyfill.append(labelcenter + Vector2(0,-outerrad))
	#If health == 0 then we'll get an error in the draw poly.
		polyfill.append(polyfill[0]+Vector2(get_texture().get_width()*clamp(HealthPer, 0.01, 1), 0) )
	if HealthPer >= 0.999: # a tiny cheat to skip
		polyfill.append(poly[2])
	# Are we past the angled point on the bottom?
	elif polyfill[1].x > poly[2].x: #Draw another point straight down.
		polyfill.append(polyfill[1] - Vector2(0, (1-HealthPer)*get_texture().get_width()*tan(PI*7/12)) ) #tan of pi*7/12 is negative, but we need it to point down.
		polyfill.append(poly[2])
	else: #Else go straight down
		polyfill.append(polyfill[1] + Vector2(0,outerrad))
	polyfill.append(labelcenter)


We have a few reference points from our back we can use.  We need to do some math if health exceeds a certain amount to draw that extra vertex along the slope.  Our tiny cheat is there to skip the extra vertex.  Also we don’t want our polygon lines crossing each other so we ensure polyfill[1].x is at least greater than polyfill[0].x regardless of actual health.

Now for the shield:

::

	# Shield portion of Health Bar
	polyshield.append(polyfill[1])
	if ShieldPer >= 0.005: #Skip if there is no meaningful amount of shield
		polyshield.append(polyshield[0] + Vector2(get_texture().get_width() * ShieldPer, 0))
		if polyshield[1].x > poly[2].x:
			if EffectiveHealth/TotalHealth < 0.995: #Need to draw to the slope
				polyshield.append(polyshield[1] - Vector2(0, (1-HealthPer-					ShieldPer)*get_texture().get_width()*tan(PI*7/12)) ) #tan of pi 7/12 is negative, but we need it to point down.
			if polyshield[0].x < poly[2].x:
				polyshield.append(poly[2])
				polyshield.append(polyshield[0]+Vector2(0, outerrad))
		 	else: #Trapezoid case
				polyshield.append(polyfill[3])
		else: #Dealing with just a normal rectangle.
			polyshield.append(polyshield[1] + Vector2(0,outerrad))
			polyshield.append(polyshield[0]+Vector2(0, outerrad))

Our shield polygon starts where our health stops (polyfill[1]) so we need to account for the possibility that the shield starts and ends inside of the sloped portion of our health bar.

The next step is to make it look good.  We want to add color to give it a bit of depth.

::

	#Let's build the shader.
	polycolors.append(Color(0,0,0,0.6))
	polycolors.append(Color(0,0,0,0.6))
	polycolors.append(Color(0,0,0,0))
	polycolors.append(Color(0,0,0,0))


We’ve done a lot of math but we don’t actually have drawable polygons yet.  This step will convert our array of vectors into actual polygons.  polycolors uses a different function because it’s not a polygon but the color we use for our polygon.

::

	#Need to convert these before drawing.
	poly = Vector2Array(poly)
	polyfill = Vector2Array(polyfill)
	polyshield = Vector2Array(polyshield)
	polycolors = ColorArray(polycolors)

Now for the part we’ve all been waiting for, draw time!

::

	draw_colored_polygon(poly, Color(0,0,0,0.7)) # Health bar back
	draw_colored_polygon(polyfill, Color(1-HealthCol,HealthCol,0)) # Current health, changes from green to red.
	if ShieldPer >= 0.01:
		draw_colored_polygon(polyshield, Color(1,1,1)) # Shield Health
	var ticks = 10 #Tick size should be a global variable roughly based on average health to scale as the game progresses.
	while ticks < EffectiveHealth:
		var linex = polyfill[0].x + get_texture().get_width()*ticks/TotalHealth
		var liney = polyfill[0].y
		if linex < poly[2].x:
			draw_line(Vector2(linex,liney), Vector2(linex, liney+outerrad/2), Color(0,0,0,0.8)) #This doesn't do subpixels
		else:
			draw_line(Vector2(linex,liney), Vector2(linex, liney+outerrad/4), Color(0,0,0,0.8)) #This needs to be shorter
		ticks += 10
	draw_polygon(poly, polycolors) #Draw the shader to color the health bar
	draw_circle(labelcenter, label.get_size().x/2, Color(0.4, 0.4, 0.4)) #Label background.


We waited until the last moment to draw our gauge ticks because they need to be on top.  We use a bit of transparency so the ticks are slightly less obvious and take the overall hue of the color of the healthbar.  But there we go, one fancy status bar!

From here you can make all sorts of fun modifications like changing the shape, adding an additional bar underneath (limit break!), adding additional vertices to polycolor to give it more texture, or putting in a counter that modifies the color so bar pulsates.  The sky is the limit!
