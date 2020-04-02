Introduction
============

What is this?
-------------

This is the next project in a series to teach electronics to members of 
[KwartzLab](https://www.kwartzlab.ca/)
(and the public, by association).

It is a power meter meant to exercise knowledge of Arduino and intermediate 
electronics, as voted by the membership.

What will it do?
----------------

This will measure current - and calculate power, and estimated cost - of 
electricity going through a household single-phase 120Vac mains plug. This is
useful for a number of reasons, not limited to:

- Knowing which appliances are more or less efficient
- Learning how to be more environmentally conscious
- Knowing if an appliance is really off when it says it's off

Design notes
============

Current transformer
-------------------

First, we choose a current-sensing strategy. Our strategy will be
"non-intrusive":

- Offer a female 120V polarized, grounded plug for the appliance to plug into
- On the inside of the meter, run live (possibly with multiple turns) through
  the air core of a current transformer coil, without a physical connection
- On the other side, connect the mains wires to a male 120V polarized plug

The current transformer coil will be physically connected to the rest of the
isolation and sensing circuit. Even though there is not a physical connection 
between the mains wire and the rest of the circuit, a small amount of energy
will be transferred through the principle of electromagnetic induction. This
is the same principle that allows electric motors and power transformers to
work.

Since it's easy, let's look in the
[Digikey Current Sense Transformers](https://www.digikey.ca/products/en/transformers/current-sense-transformers/163)
section, with these criteria:

- Part Status = Active
- In Stock
- View prices at quantity 10
- Type: Non-invasive (either split or solid core is fine)
- Frequency range - anything whose minimum frequency is at or below 60 Hz. This
  is the frequency of alternating current (AC) mains voltage in North America.

This narrows our options considerably. Let's choose the least expensive part
that meets these criteria, the
[ASM-010](https://talema.com/uploads/documents/product-datasheets/ASM%20Series.pdf).

It operates as the secondary coil of a transformer, where the primary "coil" is
the sensed wire. In a normal transformer, the primary and secondary coils have
a different number of windings and are not electrically connected, and are used
for the purposes of voltage step-up/down and current step-down/up. Here, this 
is technically a step-up transformer because the number of windings from the
primary to secondary coil goes way up (the primary only has one winding, or
maybe two if we can squeeze in another wire). But a voltage step-up transformer
is a current step-down transformer, and this is what we use it for. We use it
to step down the current from whatever is going through the mains line.

Other characteristics of this part:

- It is through-hole, so it is easy to prototype
- The primary-to-secondary isolation rating is at least 2500 volts. This is
  called
  [galvanic isolation](https://en.wikipedia.org/wiki/Galvanic_isolation) and is
  a critical safety measure in high-voltage circuits.
- It can fit a mains wire up to 5mm in diameter
- It can inductively couple to 50-60Hz mains
- It has an accuracy tolerance of ±10% in the output voltage
- It is rated to a maximum of 10 amps AC
- They recommend that, to convert the output current into voltage, a 50Ω load
  resistor be added

Looking at the spreadsheet, the most important graph is the Typical Response.
For the ASM-010, the sensitivity is:

    30mV
    ---- ~ 3.33 mΩ
    9A 

This sensitivity, 3.33 mΩ, can also be thought of as the
[transresistance](https://en.wikipedia.org/wiki/Transconductance#Transresistance)
of the transformer. Transresistance is a ratio of voltage output change to
current input change. It's more conveniently expressed as 1/3.33 mΩ ~ 300℧ (300
mho, the unit of conductance which is the reciprocal of resistance).

In the case where the primary has a single winding and the recommended 50Ω load
is used, if the input is 10A and the output is 33.3mV, then the output current is

    33.3mV
    ---- = 667μA
    50Ω

In this case, at the top of the range, only 667 micro-amps (0.000667 A) are
being output from the transformer. At the other end of the range, we would
expect to see devices like a phone charger at partial load draw much less than
this. The minimum resolvable current is dictated by the resolution of our
analog-to-digital converter (ADC) as well as our gain. If we use the
[Arduino Uno rev 3](https://store.arduino.cc/usa/arduino-uno-rev3) with an
[ATmega328P microcontroller](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf),
then it is a 10-bit ADC:

    Resolution = 2^10 = 1024
    Vout min = 33.3mV / 1024 ~ 32.6μV
    Vin min = 32.6μV * 300℧ ~ 9.77mA

In other words, the minimum resolvable current for the Arduino would be one
where the ADC reads "1" out of a possible 1023. With a rectifier/amplifier that
amplifies 33.3mVac to 5V peak, this would be read for a transformer output voltage of
32.6μV, which would be produced by a current of 9.77mA AC through the primary.

This in turn means that, if we only have a single gain, our minimum resolvable
power is

    P = 120V * 9.77mA = 1.17W

To find out how much this costs: of Mar 30, 2020, Waterloo North Hydro's
rates disregarding COVID-19 discounts are:

- $0.101/kWh off-peak, 12 hours/day weekdays, 24 hours/day weekends
- $0.144/kWh mid-peak, 6 hours/day weekdays
- $0.208/kWh on-peak, 6 hours/day weekdays

An amortized monthly rate is then

    4*[5*(12*0.101 + 6*0.144 + 6*0.208) + 2*24*0.101]/1000 = $0.085872/W/month

Over one month, the cost of 1.17W is then

    1.17W * $0.085872 = $0.10

So our single-gain system has a resolution of ten cents per month. We could add
a second amplifier to get better resolution for low-power devices but that does
not seem worth it.
