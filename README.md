Battery Sense [![Build Status][travis-status]][travis]
=============
[travis]: https://travis-ci.org/rlogiacco/BatterySense
[travis-status]: https://travis-ci.org/rlogiacco/BatterySense.svg?branch=master


This is a simple Arduino library to monitor battery consumption of your battery powered projects, being LiPo, LiIon, NiCd or any other battery type, single or multiple cells: if it can power your Arduino you can monitor it!

The principle is simple: we are going to measure our battery capacity by measuring the voltage across the battery terminals.

The big assumption here is that battery capacity is linearly correlated to its voltage: the assumption itself is wrong, but in most cases it's *close enough* to reality, especially when it comes to the battery higher capacity side.

In reality, the relation between battery capacity and its voltage is better represented by a curve and there are many factors affecting it: current drawn, temperature, age, etc...

![Battery discharge curves at 750 mA](http://www.philohome.com/batteries/discharge-750.gif)

<!-- toc -->

- [How to](#how-to)
  * [Lesser than 5V, with voltage booster](#lesser-than-5v-with-voltage-booster)
  * [Higher than 5V, with internal voltage regulator](#higher-than-5v-with-internal-voltage-regulator)
  * [Higher than 5V, with external voltage regulator](#higher-than-5v-with-external-voltage-regulator)
  * [Higher than 5V, activated on demand](#higher-than-5v-activated-on-demand)
- [Voltage divider ratio](#voltage-divider-ratio)
- [Examples](#examples)
  * [Single cell Li-Ion on 3.3V MCU](#single-cell-li-ion-on-33v-mcu)
  * [Double cell Li-Ion (2S) on 5V MCU](#double-cell-li-ion-2s-on-5v-mcu)
  * [9V Alkaline on 5V MCU](#9v-alkaline-on-5v-mcu)

<!-- tocstop -->

## How to
The library requires at least 1 analog pin (we will call this the `sense pin`) and no less than 2 bits of info on your battery: the voltage you will consider the minimum acceptable level, below which your project/product becomes unreliable and should be shut down, and the maximum voltage you can expect when the battery is fully charged.

Additionally, you can provide a second pin (either analog or digital) to activate the battery measurement circuit (we call it the `activation pin`), useful in all those situations where you can sacrifice a pin to further increase your battery duration.

If you want your readings to be more accurate we *strongly suggest* to calibrate the library by providing your board reference voltage: most of the times you assume your board has exactly 5V between `Vcc` and `GND`, but this is rarely the case. To improve this we suggest using the [VoltageReference](https://github.com/rlogiacco/VoltageReference) library to obtain a better calibration value for all analog readings.

The `sense pin` wiring can vary depending on your battery configuration, but here are a few examples based on the assumption you are using a 5V board: in case of a 3.3V board you should be performing the necessary adjustments.

### Lesser than 5V, with voltage booster
Voltage sources made of single cell LiPo or LiIon, along with some single or multi-cell NiCd configurations (like up to 3 AA or AAA), are not able to provide the *suggested* `5.0 volts input` to your board and a voltage booster can solve your problem.
What does that mean when it comes to measuring your battery level? We need to measure the battery voltage *before* it gets boosted, which means your `sense pin` must be connected between the battery positive terminal and the booster positive input and we don't need any additional components as the voltage is already in the acceptable range:

```
                                               +---------+
          +----------------------------- SENSE |         |
          |        +---------+                 |         |
  BAT+ ---+--- IN+ |   5V    | OUT+ ------- 5V | Arduino |
                   | BOOSTER |                 |         |
  BAT- ------- IN- |         | OUT- ------ GND |         |
                   +---------+                 +---------+
```

### Higher than 5V, with internal voltage regulator
Voltage sources made of multiple cells LiPo or LiIon, along with some single or multi-cell NiCd configurations (like up the classic 9V battery or 4+ AA or AAA), provide voltages above the `5.0 volts input`: most of the Arduino boards are equipped with voltage regulators able to dissipate into heat all the excess.
To measure such batteries we need to hook our `sense pin` *before* it gets regulated, between the battery positive terminal and the Arduino unregulated input `VIN` or `RAW`, but we require two resistors to reduce the voltage to acceptable values:

```
                         +---------+
  BAT+ ---+--------- VIN |         |
          |              |         |
         R1              |         |
          |              |         |
          +------- SENSE | Arduino |
          |              |         |
         R2              |         |
          |              |         |
  BAT- ------------- GND |         |
                         +---------+
```

The values of R1 and R2 determine the `voltage ratio` parameter for this library: for information about this value refer to the section below.

Because the resistors in this configuration will constantly draw power out of your battery, you shouldn't pick values under `1k Ohm`, or you'll deplete your batteries much faster than normal. On the other end, going too high on the resistor values will impede the library from getting accurate readings.

### Higher than 5V, with external voltage regulator
Whenever your battery maximum voltage exceeds the onboard regulator (if there is any) an external voltage regulator is required.
Once again, to measure such batteries we need to hook our `sense pin` *before* it gets regulated, between the battery positive terminal and the voltage regulator positive input `VIN` or `RAW` and, as before, we require two resistors to reduce the voltage to acceptable values:

```
+---------------------------------+
|                  +---------+    |         +---------+
| BAT+ ---+--- IN+ |         |    +-- SENSE |         |
|         |        |         |              |         |
|        R1        |         |              |         |
|         |        |         |              |         |
+---------+        |         | OUT+ ---- 5V | Arduino |
          |        |         |              |         |
         R2        |         |              |         |
          |        |         |              |         |
  BAT- ---+--- IN- |         | OUT- --- GND |         |
                   +---------+              +---------+
```

The values of R1 and R2 determine the `voltage ratio` parameter for this library: for information about this value refer to the section below.

### Higher than 5V, activated on demand
Batteries are a precious resource and you want to prolong their life as much as you can so, deplete your battery to determine its capacity is not desirable.

As a consequence of connecting the battery terminals through two resistors we are drawing some energy out of the battery: for a 9V battery and 1k Ohm for R1 and R2, you will be adding a constant 4.5mA current consumption to your circuit. Not a huge amount, but definitely not desirable.

If you have an unused pin on your Arduino it will be easy to limit this additional current consumption to be drawn only when needed: during battery measurement. We will be turning the `activation pin` HIGH during battery measurement so that the voltage divider will be disconnected most of the time:

```
                         +---------+
  BAT+ ---+--------- VIN |         |
          |              |         |
         SW--------- ACT |         |
          |              |         |
         R1              |         |
          |              | Arduino |
          +------- SENSE |         |
          |              |         |
         R2              |         |
          |              |         |
  BAT- ------------- GND |         |
                         +---------+
```

In the above schematics **SW** is a circuit which can connect or disconnect the sensing circuit depending on the voltage on `ACT`: the most common and cheap circuit is made of a p-channel MOSFET *Q1*, a 100-500 Ohm resistor *R3* and a 5k-20k Ohm resistor *R4*:

```
  VDIV
    |
    +----+
    |    |
    |   R4
    |    |
    Q1 --+-- R3 --- ACT
    |
  BAT- 
```

## Voltage divider ratio
Whenever your battery voltage is above your board voltage you need a voltage divider to constraint your readings within the 0-5V range allowed by your Arduino and you will have to provide this library with its *ratio*.

```

  BAT+ ---+
          |
         R1
          |
          +------- SENSE
          |
         R2
          |
  BAT- ----
```

The `voltage divider ratio` is determined by the formula `(R1 + R2) / R2`: if you use two resistors of the same value the ratio will be **2**, which can be interpreted as *whatever value we read it will be **half** of the actual value*. This allows us to sense batteries up to 10V. 
If you use a 22k Ohm resistor for R1 and a 10k Ohm for R2 than your `voltage ratio` will be **3.2** and you will be able to safely monitor a 12-15V battery.

You **must** select the resistors in order to get a ratio which will produce values between the 0-5V range (or 0-3.3V for 3.3V devices) **at all the times** and to obtain that the process is quite simple: divide your battery maximum voltage by 5V and you'll get the *absolute minimum value* for the `voltage ratio`, then pick any two resistors values whose combination produce a ratio *equal or higher* than the absolute minimum. For a 12V battery the *absolute minimum voltage ratio* is **12/5=2.4**, meaning you can't use a *split supply divider* made of two equal resistors: you need R1 to be a higher value than R2! Get this wrong and you will probably burn your `sense pin`.

The *voltage divider total resistance*, made of `R1 + R2`, will determine the current drawn from your battery by the sensing circuit: lower is the total resistance and more accurate are your readings, higher the resistance and less current is drawn from your battery. My suggestion is to keep this value within 20k-22k Ohm when using an *always-connected* circuit and under 10k Ohm if you use an *on-demand* configuration.

When determining the *ratio* don't stick with the resistors nominal values, instead, if possible, use a multimeter to actually measure their resistance so to improve your results: a `4.7kΩ` resistor could easily be a `4.75kΩ` in reality!

## Examples
Here follow a few real case scenarios which can guide you in using this library.

### Single-cell Li-Ion on 3.3V MCU
As an example, for a single cell Li-Ion battery (4.2V - 3.7V) powering a `3.3V MCU`, you'll need to use a voltage divider with a ratio no less than `1.3`. Considering only E6 resistors, you can use a `4.7kΩ` (R1) and a `10kΩ` (R2) to set a ratio of `1.47`: this allows to measure batteries with a maximum voltage of `4.85V`, well within the swing of a Li-Ion. It's a little too current hungry for my tastes in an *always-connected* configuration, but still ok. The minimum voltage to set is clearly the lowest safe value: if a Li-Ion is drained below `3.7V` the risk of permanent damage is high, so your code should look like:

```cpp
Battery batt = Battery(3700, 4200, SENSE_PIN); // also specify an activationPin for on-demand configurations

void setup() {
  batt.begin(3300, 1.47);
}
```

### Double cell Li-Ion (2S) on 5V MCU
For a double cell Li-Ion battery (8.4V - 7.4V) powering a `5V MCU`, you'll need to use a voltage divider with a ratio no less than `1.68`: you can use a `6.8kΩ` (R1) and a `10kΩ` (R2) to set the ratio *precisely* at `1.68`, perfect for our `8.4V` battery pack. The circuit will continuously draw 0.5mA in an *always-connected* configuration, if you can live with that. As we don't want to ruin our battery pack we'll have to set the minimum voltage to `7.4V` to avoid risking of permanent damages, meaning your code should look like:

```cpp
Battery batt = Battery(7400, 8400, SENSE_PIN); // also specify an activationPin for on-demand configurations

void setup() {
  batt.begin(5000, 1.68);
}
```

### 9V Alkaline on 5V MCU
Another classic example might be a single 9V Alkaline battery (9V - 6V) powering a `5V MCU`. In this case, you'll need to use a voltage divider with a ratio no less than `1.8` and, for sake of simplicity, we'll go for a nice round `2` ratio. Using a nice `10kΩ` both for R1 and R2 we'll be able to measure batteries with a maximum voltage of `10V` consuming only 0.45mA. The trick here is to determine when our battery should be considered empty: a 9V Alkaline, being a non-rechargeable one, can potentially go down to 0V, but it's hard our board can still be alive when this occurs. Assuming we are using a linear regulator to step down the battery voltage to power our board we'll have to account for the regulator voltage drop: assuming it's a `1.2V` drop, we might safely consider our battery empty when it reaches `6.2V` (5V + 1.2V), leading to the following code:

```cpp
Battery batt = Battery(6200, 9000, SENSE_PIN); // also specify an activationPin for on-demand configurations

void setup() {
  batt.begin(5000, 2.0);
}
```
