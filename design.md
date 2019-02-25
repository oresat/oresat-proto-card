# OreSat Prototype Card Design Notes

## TPS62111-based buck switching power supply

The TPS6211 is the Best Thing Ever because it's a high efficiency, high frequency (1 MHz) low Iq design that 
uses an internal P channel MOSFET (PFETs) for switching, not a boosted N channel. It's got a decent temp rating and
it's small. It can be frequency locked, which is awesome. And it requires very few external components.

TPS62111RSA

**TODO:** TPS6211x Driving EN and SYNC Pins (SLVA295)
**TODO:** Check the evaluation board?

### Calculations

- TPS6211X where X = {0 = adj, 1 = 3.3V, 2 = 5V, 3 = adj}
   - TPS62111 should be +/- 3%, so 3.3 +/- 0.1V
- Vin = 2S2P LiPo pack = (6.0, 7.2, 8.4) V @ some ungodly number of amps if we let it.
- Imax for the TPS62111 = 1.5 A.
- Temp range is -40 to 85 C, but Tshutdown is 145 C (with a hysteresis of 10 C)
- Startup
   - 250 us steps of Iswitch={300 mA, 600 mA, 1200} mA
   - Co = 22uF = ~ 1 ms (typical) of startup time
- Power Save Mode of Operation (
   - PFM mode is entered when `Iskip ~ Vi/25ohm = (250, 280, 280) mA` (note: Vi > 7 has no effect)
   - Iq = ~ 20 uA when in PFM and there's no output load: Already at 80% eff. when Io = 1 mA. 
   - Iq = ~ 10 mA (!!!!!) when in PWM mode
   - Iq = ~ 20 mA (!!!!!) when in synchronized clock mode
- Inductor current
   - Ripple inductor current: `del_I_L = Vo * (1 -Vi/Vo)/(L*f) = 3.3 * (1 - 3.3/8.4)/(.0000068*1000000) = 0.295 A`
   - Max Inductor current: `I_Lmax = I_max + del_I_L/2 = 1.5 + 0.295/2 = 1.65 A`
   - Conservative max inductor current: FET switch current of the TPS is 2.4 A
- Output voltage ripple
   - `I_RMS(Co) = Vo * (1-Vi/Vo)/(L*f) * 1/(2*sqrt(3) = 3.3 * (1 - 3.3/8.4)/(.0000068*1000000) * 0.289 = 85 mA`
   - `V_RMS(Co) = Vo * (1-Vi/Vo)/(L*f) * (1/(8*Co*f) + R_esr) = 3.3*(1-3.3/8.4)/(.0000068*1000000)*(1/(8*0.000022*1000000) + 0.050) = 0.016 V `
      - **FIXME:** Assumes R_esr of 50 mOhms, actual Resr = ?
- Input capacitor ripple
   - `I_rms_Ci = Io_max * SQRT(Vo/Vi*(1-Vo/Vi)) = 1.5*(3.3/8.4*(1-3.3/8.4)**0.5) = 0.459 A`
   - That's 33% of the current; that's a lot of ripple

### Musings

- Reverse current
   - NO PROTECTION. Vo > Vi otherwise current will flow the PFET body diode! 
- EN pin
   - We've got the external PFET to shut down, no reason to use EN.
- Sync pin
   - Maybe tie Sync low for an extremely low power startup condition.
   - uC can then pull sync high to reduce noise; better yet begin sync pulses
- LBI/LBO/PG
   - Could leave floating, but by connecting to ground, we make a much better ground connection to the top plane
   - No reason to use LBI/LBO, our battery protection circuit will handle low battery voltages.
   - PG does let us know if we're overcurrenting without tripping the breaker; this could even be tied into the
     MAX7310 as a status input. For now, it goes to a test point.

### Components

- Inductor: ECS-MPI4040R4-6R8-R 
   - Really only the inductor is special; everything else is just low Resr ceramic caps, where less R and more C is better.
   - Chose the smallest foot print inductor with height <= 2.0 mm that worked for the worst case TPS6211 switch current  of 2.4A
   - 6.8 uH +/- 20%, 4.7 x 4.31 x 2.0 mm, Irms = 2.4 A, Isat = 3.2 A, DCR = 91 mOhm, AEC-Q200 qualified
   - Don't love the 20%, but everything else, based on the size, is good. Don't forget: must be < 2.0 mm tall.
   

-----------------------------------------------------------------------------------------------------------------------

## MAX7310 2-Wire-Interfaced 8-Bit I/O Port Expander

This is an I2C-based GPIO expander that we're using as a on-off switch for each power domain. It kills us to use I2C across the
satellite as a safety critical bus.. but... we just can't come up with a better solution. CAN is too power consumptive, UARTs are
dumb and also power consumptive, and this particular IO expander seems pretty bad ass, including a timeout on the I2C bus that resets
the state machine when it times out, which is awesome. It'll take 400 KHz I2C but we'll run it much slower, maybe 100 KHz or even less.
Weirdly and delightfully, it has up to 56 addresses available through 3 address bits. But, we also hates this because it uses the SDA/SCL
lines as the extra line which seems weird (does this get probed? How does it know it's SDA or SCL? Part of the I2C statemachine? Weird).

When using this in a circuit, it'll be easy to forgot to protect this IC from the rest of the circuit. So be paranoid when attaching to things 
that have a > 5V voltage, like Vbus or Vbusp.

- https://datasheets.maximintegrated.com/en/ds/MAX7310.pdf
- 2.7 - 5V, 1.7 uA standby, -40°C to +125°C
- QFN package: MAX7310ATE

### Components

- Capacitor
   - 47 nF is spec'd, give it 100 nF because that's better. Some possibility of badness with dumping that 100n through a latched transistor, but meh.
   - R in series with power: something to prevent death and destruction if it latches up.
   - R in series with SDA/SCL: something to prevent D&D if those lines latch up. Would be great if we could have OC SDA latch up and still use 
     I2C bus... is that even possible?

### Musings

- What can we do to make this thing even more bulletproof? What about some kind of CAN transciever-like "disconnect if low too long"?
  Unclear how to do this since we don't have access to the driver internals :( - Maybe disconnect ALL drivers if low too long?

-----------------------------------------------------------------------------------------------------------------------

## MAX4211 High-Side Power and Current Monitors

We wanted a circuitbreaker that used a PFET for the switch. This is very, very hard to find. Very. Did we mention very? Instead of a decent
efuse, we found this IC instead. It's a "power and current monitor", and it's really just a circuit breaker IC with an external PFET.
Is it better than building it ourselves? Well, frankly, it's just about the same with some nice features built into it, like latching off, 
and less comparators, and decent voltage range. So OK, we'll use it. 

QFN package, largest current gain (reduces Rsense), internal voltage divider, MAX4211CETE
   - MAX4211FETE is OK, but the power output is lost.

### Components

- PFETs
   - Looking for a PFET in a SOT23-6 package that can handle bunches of amps, and with a pinout with multiple versions available.
   - Diodes Inv DMP3050LVT-7
      -  BVdss = -30V, Rdson < 75 mOhm, IdMax < -3.7 A, VGS(TH) = -2.0 V, -55 to +150 C
      - https://www.diodes.com/assets/Datasheets/DMP3050LVT.pdf
      - https://www.digikey.com/product-detail/en/diodes-incorporated/DMP3050LVT-7/DMP3050LVT-7DICT-ND/3720102
   - Alternative: Infineon BSL307SP
- NFETs
   - Meh, whatever, SOT-23 package and good temp range and didn't even bother to pick one out yet.
- LDO
   - Important to have very low Iq, great temp, and again a PGET pass transistor
   - Microchip MCP1703A 250 mA, 16V, Low Quiescent Current LDO Regulator
      - Iq = 2 uA, Imax = 250 mA, 1.0 uF or more caps, -40°C to +125°C
      - https://www.digikey.com/product-detail/en/microchip-technology/MCP1703AT-3302E-CB/MCP1703AT-3302E-CBCT-ND/3598398
      - http://ww1.microchip.com/downloads/en/DeviceDoc/20005122B.pdf
   - Alternative: Taiwan Semiconductor TS9011

### Calculating Rsense

Some protocards will take ~ 50 mA when on (magnetometers) could take up to 3 amps (DxWiFi PA) We'll need to be able to adjust this. Arbitrarily, 
it seems like 100 mA is a good lower bound on the circuit breaker, and 5 A (which is greater than the PFET max current of 3.7A) is a good 
maximum. 

Design example for a 250 mA trip

- Just assume a 25 mV drop across Rsense. This means @ 250 mA, a 25 mV drop implies Rsense = 0.1 ohm
- Ploss = .25^2*.1 = 6.25 mW
- Iout = 25 mV *  40.96 current gain = 1.024V
- Vref = 1.19 - 1.23 V

Huh! Let's assume a different way:

- Rsense = 0.1 ohms
- VIout = 40.96 * Rsense * I
- So to find I @ 1.19V minimum cutoff is 1.19 V /(40.96 * .1) = 290 mA
   - 1.23 V / (40.96 * .1) = 300 mA

So a 300 mA current draw with a 0.1 ohm Rsense and losing 9 mW will always cause a trip.

Looking at the comparator input errors:

- Comparator Input Offset = +/- 5 mV max
- Comparator Hysteresis = 5 mV

So worst case, 10 mV error. To keep this under 10%, make sure the trip voltage is > 100 mV.

What's the power use at 300 mA? That's (1.8, 2.2, 2.52) W. So 9 mW is max 0.05% power loss of the system. seems OK.










