# GPS + INS Project

I wanted to see if I could keep track of my position even when the GPS stops working, like it does in a tunnel. For this project I used a GPS module and a BNO055 motion sensor (the IMU, which is the "INS" part) with an Arduino, and I tried mixing the two together.

## Why mix them?

- **GPS** gives me a position about once every second. It's a bit noisy and sometimes it disappears, but it doesn't drift away over time.
- **The motion sensor** gives me acceleration about 50 times a second. If I add up the acceleration to get speed, and add up the speed to get position, I get a position that updates really fast. But small errors add up too, so the position slowly drifts further and further off.

Each one is good at what the other one is bad at, so I combined them.

## How it works

1. The BNO055 gives me acceleration (with gravity already taken out) and which way it's pointing (roll, pitch, yaw).
2. I use those angles to turn the acceleration into East and North directions. If the acceleration is really tiny (less than 0.05 m/s²), I ignore it since it's mostly noise.
3. I add up the acceleration to get velocity, then add up the velocity to get position. This is the "IMU only" position.
4. The first time the GPS gets a good fix, I call that spot (0, 0). After that, I change each GPS latitude/longitude into meters East and North from that spot.
5. Every time a new GPS fix comes in, I mix the two:
   - position = 90% IMU + 10% GPS
   - velocity = 80% IMU + 20% GPS

   So the IMU does most of the work, and the GPS slowly pulls the answer back so it doesn't drift away. This mix is called a complementary filter. I also slow down the velocity a little (multiply by 0.9) whenever the acceleration reads zero, so it doesn't keep creeping.
6. To test a GPS outage, I type `b` in the Serial Monitor. The code then ignores the GPS for 10 seconds and only uses the motion sensor. After 10 seconds it goes back to using GPS.

The Arduino prints one line of numbers about 50 times a second:

```
GPS_X, GPS_Y, INS_X, INS_Y, Complementary_X, Complementary_Y, Kalman_X, Kalman_Y, Blackout(0 or 1)
```

Everything is in meters (X = East, Y = North). I copy the numbers into MATLAB to make my plots.

## What I found

**It fills in the gaps in the GPS.** In this plot I walked a loop. The blue X's are the GPS positions, and the arrows show where the GPS had gaps. The light blue line (my mix) keeps going smoothly through the gaps. The red line, which is just the motion sensor, wanders off the loop.

![GPS gaps filled by the complementary filter](images/gpsins_07_gap_fill_complementary.png)

**The motion sensor alone drifts a LOT.** When I zoom out, the red line (motion sensor only) ends up more than 1,000 m away from where I started, even though my loop is only about 400 m wide. Adding up acceleration twice makes errors grow really fast. The mixed version stays with the loop.

![Motion sensor only drifts away](images/gpsins_06_ins_drift_zoomed_out.png)

## What still doesn't work well

- The 90/10 and 80/20 mix is fixed, so it can't change if the GPS gets noisier or I move around a lot. The mixed path is still a little off from the GPS path.
- I don't fix the accelerometer's built-in offset, so errors still build up when the GPS is out.
- I use the BNO055's angles to figure out which way is East and North. If those are a bit off, the acceleration gets pushed in the wrong direction.
- Changing GPS to meters the way I did only works for small areas, because it pretends the Earth is flat.

## Kalman filter (still experimenting)

The file `MyInsGpsKalman.h` is my first try at a Kalman filter, one for each direction (East and North). It keeps track of position and velocity. It uses the acceleration to guess where I'll be next, then fixes that guess every time a GPS fix comes in. I set the acceleration noise to 0.5 and the GPS noise to 2.5 m. It prints as `Kalman_X, Kalman_Y`. I haven't tuned it or compared it carefully to my complementary filter yet, so the plots above only show the complementary filter.

## What I want to try next

- Tune the Kalman filter and compare it to the complementary filter using the same walk
- Make the filter trust the GPS more or less depending on how noisy it is and how I'm moving
- Figure out the accelerometer offset and remove it
- Measure how far off I really am compared to a known path, so I have actual error numbers

## What I used

- Arduino [board]
- Adafruit BNO055 motion sensor (I2C)
- GPS module [model], connected to pins 2 (RX) and 3 (TX), 9600 baud
- Arduino libraries: Adafruit BNO055, Adafruit Unified Sensor, TinyGPSPlus, SoftwareSerial
- MATLAB for the plots

## How to run it

1. Install the libraries above in the Arduino IDE.
2. Connect the BNO055 to the I2C pins and the GPS to pins 2 and 3.
3. Open `gpsPlusIns/gpsPlusIns.ino` and upload it.
4. Open the Serial Monitor (115200 baud). Take it outside so the GPS can find satellites, and walk around.
5. Type `b` any time to pretend the GPS is out for 10 seconds.
6. Copy the numbers it prints into MATLAB or Python to make plots.

## Files

| File | What it is |
|---|---|
| `gpsPlusIns/gpsPlusIns.ino` | The main Arduino code |
| `gpsPlusIns/MyInsGpsKalman.h` | My experimental Kalman filter |
| `images/` | Plots from my test runs |

## Credits

[Libraries, tutorials, and anything I used to learn this or write the Kalman filter.]
