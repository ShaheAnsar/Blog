---
title: "Using Transistors With Motors"
date: 2021-04-13T07:30:39+05:30
draft: false
tags : ["transistor", "basics", "fundamentals", "MOSFET", "BJT"]
---
## Using transistors with motors (Or Any other Load)

It's very common to want to interface with a motor in a 
simple manner (No direction control) or to switch relays
with a GPIO pin. Unfortunately, they don't provide enough current,
so don't use them for these applications - you'll end up damaging
the device to which the GPIO pin is connected. You instead want to
use a transistor controlled by the GPIO pin.
Both BJTs and FETs are suitable, although you may want
to choose a logic level input FET for high current applications.
Logic level input simply means that the FET can be easily switched
by our weak GPIO pins. Use a resistor in series otherwise (This will increase the
switching time so if you want to use a PWM signal, you'll want a logic
level input FET or a FET driver IC and bypass the resistor).
Also remember to use a resistor in series for the BJT.
Keep in mind, the input to
a BJT is essentially just a diode, so set the input current by subtracting 
0.7V (or whatever the forward voltage of the transistor is. Datasheets are 
your friend) and then dividing by the desired input current to find the
resistor value.  
Anyway, let's talk about selecting a suitable transistor for driving a 12V 100ma
input relay.

### Selecting a BJT
BJTs are pretty simple for the most part. We'll talk about NPN transistors
mainly. Let's look at a datasheet (The BC337. Good enough for a relay) to make things
more understandable.
![DataSheet](/using-transistors-with-motors-bc337datasheet1.png)

Make sure that all the maximums are not exceeded. For our application,
this transistor cuts the butter just fine.

The first thing to care about is the $h_{ fe }$ (Also denoted by $\beta$ or 
current gain) of the transistor. It's a ratio
of the output current to the input current. For a small relay, we want
an output current of about 100mA. A GPIO pin can often only supply a few
mA, so we want an hfe that's at least 100 at our current.
The $h_{ fe }$ is generally given as a number as well as a graph. 
We almost always want to go
with the graph. Let's take a look at it.
![Current Gain Vs Output Current](/using-transistors-with-motors-bc337datasheet2.png) 
At about 100ma, the $h_{fe}$ is well past 100, so this transistor will do
just fine for us. You also want to look at the minimum value of the current
gain that's given in the table
![Current Gain Table](/using-transistors-with-motors-bc337datasheet3.png) 
Again, for our model (The vanilla 337), the minimum $h_{fe}$ at our 
desired current is more than good enough for our application.  

Before conquering the thermal calculations, we first want to know
the power dissipated. The power dissipated in a BJT is given by-
$$ P_{diss} = V_{CE}  I_C $$
Now, for our purposes we operate the BJT as a switch, so it operates
in the saturation region, so we want to know the saturation voltage
of the transistor (This is the voltage between the collector and emitter
when the transistor is saturated. It increases with the output current).
Again, let's look at the graph from the datasheet-
![Vce sat graph](/using-transistors-with-motors-bc337datasheet4.png) 
We know that we want an input current of about 1mA 
(100mA(output current)/100(minimum required $h_{fe}$)), so let's select the
line closest to what we want but gives us worse values if the 
exact curve isn't present. Conveniently, there's an $I_c = 100mA$ curve
we can look at. We know that our input current (Also called base current) is
1mA, and from the graph, we know that $V_{CEsat}$ at that current is about
0.15V. We're gonna call that 0.2V for convenience in calculations and
as a safety margin. Now we know all that we need to calculate $P_{diss}$,
which is 20mW for us, this is an insignificant amount of heat, so we
don't even need to calculate anything to know if we're using the transistor
safely.
But if we were dissipating more, we'd pay attention to more things.
In particular, the junction temperature (the internal temperature of the
transistor) and the Safe Operating Area. Let's have a look at the 
Safe Operating Area first.
![SOA graph](/using-transistors-with-motors-bc337datasheet5.png) 
We always want to stay in the part of the graph that is below all of these
lines. Note that two of these lines represent the same thing (The thermal
limit) but with two different temperatures (One is for the case held at
25C and the other for no heatsink and ambient at 25C). You'll choose the no
heatsink one if you're designing without a heatsink. Otherwise, choose the
other one. Either way you'll need more detailed calculations for thermals,
but staying below the appropriate lines is a must as long as conventional
cooling is used.

For thermal calculations, we need to look at the $R_{\theta jc}$
and $R_{\theta ja}$ values. These are the thermal 
resistances from the junction to the package, and from the junction to the
surrounding air (without a heatsink). Now, as for how to use thermal 
resistances, well that's pretty simple - we use the thermal Ohm's Law - 
$$ \Delta T = R_\theta \cdot P_{diss} $$
This just means that the power dissipated times the thermal resistance
equals the increase in temperature to some reference (Ambient for our use
cases).  
We know the ambient temperature (Make a conservative guess. If the peak at
summer is about 40 C, then set it to 50C). We know the max power we will
dissipate. We know the net thermal resistance.
We can use all of this information to calculate the junction temperature.
$$ \Delta T = T_j - T_{amb} $$
$$ T_j = R_\theta \cdot P_{diss} + T_{amb} $$
In fact, more than the thermal Ohm's Law, you'll be using the above equation
a lot more.  
Let's use this equation for our case, with and without a heatsink.
#### Without a heatsink
Here we will use $R_{\theta ja}$ as our thermal resistance. We know we'll be
dissipating 20mW, and the ambient temperature where I live is 50 C. So let's
plug this stuff into our equation. We find that $T_j$, the junction
temperature is about 54 C (Keep in mind, this is not the temperature of the
package, it's the temperature of the junction). This is well below the
maximum of 150C, so we can rest easy.
For this particular situation, there's a more convenient way of calculating
whether the transistor junction temperature is under control. Refer to the
first image of the datasheet, you'll see the the maximum power dissipation
at ambient = 25 C. There's also a derating value. We simply take our max power
dissipation and subtract the derating value times the change in ambient from
25 C (In our case, ambient is 50C, so the net change is 50 - 25 = 25C).
If we calculate the max power dissipation at 50C, we get 500mW. Since our
transistor only dissipates 20mW, we're safe. We never want to get too close
to our max dissipation value though, it means our junction temperature will
be near it's maximum. We want to avoid that, so always have a reasonable
margin from the maximum.
#### With a heatsink
We really don't need a heatsink for this application, so let's deal
with something more beastly that does - a BJT that's used to drive 2A through
a motor. We'll use the TIP122 for this. Keep in mind that power
transistors have a lower current gain, so we use a 2 in 1 package (This
configuration is called a darlington). Power transistors also have a higher
forward voltage as well as higher saturation voltage, and this worsens
significantly with a darlington. For now, just take my word that the power
dissipation we deal with in the transistor is about 2.5W 
and that the SOA is satisfied. Now, let's take a
look at the thermal resistance for this transistor -
![TIP122 thermal resistance](/using-transistors-with-motors-bc337datasheet6.png) 
We want $R_{\theta jc}$ here, but it alone isn't the entire story. The
heatsink we use will also have its own thermal resistance, and the thermal
grease/pad we use to attach the heatsink to the transistor will also have
a thermal resistance. The net resistance is the sum of these resistances.
Let's say we use a heatsink with an $R_\theta$ of 12 C/W and a thermal
pad with an $R_\theta$ of 1 C/W. We're talking about a total $R_\theta$
of about 15 C/W. Plug in the numbers to our favorite formula and you'll find
that $T_j$ is about 87.5 C. Again, much lower than the 150 maximum for
our transistor so it's fine.

#### Dealing with inductive spiking
Now, an issue you'll have to face when dealing with any inductive load is
inductive spiking caused when the transistor turns off. The inductor has
very high voltages across it, since inductors will have a voltage
proportional to the change of current (Which is ideally instant here).
This voltage can very easily destroy the transistor, so it's important
that we clamp the inductive spike.
![TIP122 thermal resistance](/using-transistors-with-motors-bc337datasheet7.png) 
We just use the circuit above. Just put a diode across the load in the way
shown, and you're transistor won't give out the magic smoke. A very simple way to find the
direction the diode should be facing is to look at the current when the transistor is on.
Add a wire around the load's terminals and imagine that current looping through it. Your
diode should be added to the load's terminals so that this current is not blocked.

### Selecting a MOSFET
Much of what we discussed applies directly to MOSFETs as well. There are
some differences, and we'll only go through that.
1. MOSFETs have no input current at DC. That doesn't mean we don't need to
care about the input current at all as
2. MOSFETs, especially power MOSFETs have significant input capactiance (The IRFz44n has
a typical input capacitance of 1.5nF or so. Keep in mind that with FETs, 
variations are quite huge, so the input capacitance can be fairly big in
the worst case). This
results in a lot of IO current and can damage your GPIO. MOSFETs with a logic
level inputs should be used instead. Otherwise use a resistor to limit the
current. You can get the resistance required by dividing your logic
HIGH voltage by your desired input current (So for 5V and 1mA we'll use 
a 5K resistor). This does mean that the switching time is increased. If
switching time matters, and you can't go with a logic level input MOSFET, use
a MOSFET driver IC. Also keep in mind, the MOSFET threshold voltage should
be quite a bit lower than the IO voltage, otherwise you won't be able to
turn the MOSFET on (unless you're using a MOSFET driver IC, in which case
you'll need a separate power rail for your FET if your FET doesn't have a
threshold voltage low enough)
3. Power dissipation calculations with MOSFETs are different. Instead of the
$V_{CEsat}$ with BJTs, we only need to worry about the on resistance denoted
by $R_{DS(ON)}$. You can calculate power dissipated the same way you would
with a resistor. This makes MOSFETs very efficient at high currents as even
at those currents the drop across a good power MOSFET is only 100mV or so.
To give an example, $R_{DS(ON)}$ for an IRFZ44N is 17.5m$\Omega$ in the worst case scenario.
4. Depending on how things are set up, it is ideal for your MOSFET gate
to have a protection zener to make sure ESD doesn't puncture the gate insulation.
Most modern MOSFETs will come with protection diodes, so it isn't a huge deal.
My tip - better be safe than sorry.

Well that's it for today folks.
