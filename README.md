# GPS + INS Navigation on an Arduino

I wanted a position estimate that stays smooth and keeps working when GPS drops out, like in a tunnel. This project combines a GPS module with a BNO055 motion sensor (an IMU, which is the "INS" part) and tests what happens during a GPS outage.

## The problem

- **GPS** gives a position about once per second. It's noisy and sometimes goes missing, but it never drifts.
- **The IMU** gives acceleration about 50 times per second. If you add up acceleration twice you get position. It's smooth and fast, but small errors add up, so the position drifts away over time.

Each has a weakness the other doesn't, so I combined them.

## How it works

1. **Get acceleration.** The BNO055 gives linear acceleration (gravity already removed) and orientation (roll, pitch, yaw).
2. **Rotate into East/North.** I use all three angles to turn the sensor's x/y/z acceleration into East and North acceleration. A deadband of 0.05 m/s² ignores tiny readings.
3. **Dead reckoning.** Integrate acceleration into velocity, then velocity into position, at 50 Hz. This gives a position estimate from the IMU alone (`posX_INS`, `posY_INS`).
4. **Convert GPS to meters.** The first valid GPS fix becomes the starting point (0, 0). Later fixes are converted from latitude/longitude into meters East and North using a flat-earth approximation.
5. **Complementary filter.** Every time a new GPS fix arrives, I blend:
   - position = 90% from the IMU estimate + 10% from GPS
   - velocity = 80% from the IMU estimate + 20% from GPS speed and heading

   So I trust the IMU over short times, and GPS slowly pulls the estimate back. When acceleration reads zero, I also damp the velocity by 10% each step, so the estimate doesn't keep drifting while the sensor is still.
6. **Test an outage.** Typing `b` in the Serial Monitor ignores GPS for 10 seconds. The code keeps running on the IMU alone, then goes back to using GPS.

### What the sketch prints
One line per update (about 50 per second):

```
GPS_X, GPS_Y, INS_X, INS_Y, Complementary_X, Complementary_Y, Kalman_X, Kalman_Y, Blackout(0/1)
```

All positions are in meters (X = East, Y = North). I copy this into MATLAB to make the plots.

## Results

**Filling GPS gaps.** On a loop, the GPS fixes (blue) have gaps, marked by the arrows. The complementary filter (light blue) stays continuous through the gaps. The IMU alone (red) wanders away from the loop.

![GPS gaps filled by the complementary filter](images/gpsins_07_gap_fill_complementary.png)

**IMU drift over a longer time.** Zoomed out, the IMU-only path ends up more than 1,000 m from the start, even though the loop I walked is only about 400 m wide. Adding up acceleration twice makes errors grow very quickly. The fused estimate stays with the loop.

![IMU-only drift, zoomed out](images/gpsins_06_ins_drift_zoomed_out.png)

## What doesn't work well yet

- The 90/10 and 80/20 weights are fixed, so the filter can't adjust when GPS is noisy or the sensor is moving a lot. The fused path still sits a bit off the GPS path.
- I don't remove the accelerometer's own offset (bias), so errors build up during outages.
- The East/North conversion trusts the BNO055's orientation. If that is off, acceleration gets pushed in the wrong direction.
- The flat-earth conversion only works for small areas.

## Experimental: Kalman filter

`MyInsGpsKalman.h` is an early Kalman filter I'm trying alongside the complementary filter, one per axis. Its state is position and velocity. It predicts using the IMU acceleration, and corrects with each GPS fix. Its settings are an acceleration noise of 0.5 (m/s²)² and a GPS noise of 2.5 m. It prints out as `Kalman_X, Kalman_Y`. I haven't tuned it or compared it carefully with the complementary filter yet, so the results above only use the complementary filter.

## What I want to do next

- Tune the Kalman filter and compare it with the complementary filter on the same runs
- Make it adjust its trust in GPS depending on noise and how the sensor is moving
- Estimate and remove the accelerometer offset
- Measure the actual error compared to a known path, and report numbers

## Hardware and software

- Arduino [board]
- Adafruit BNO055 IMU (I2C)
- GPS module [model], 9600 baud, on pins 2 (RX) and 3 (TX)
- Arduino libraries: Adafruit BNO055, Adafruit Unified Sensor, TinyGPSPlus, SoftwareSerial
- MATLAB for the plots

## How to run it

1. Install the libraries above in the Arduino IDE.
2. Wire the BNO055 to the I2C pins and the GPS to pins 2 and 3.
3. Open `gpsPlusIns/gpsPlusIns.ino` and upload it.
4. Open the Serial Monitor at 115200 baud, take the sensor outside so the GPS gets a fix, and walk around.
5. Type `b` to simulate a 10-second GPS outage.
6. Copy the printed numbers into MATLAB or Python to plot them.

## Files

| File | What it is |
|---|---|
| `gpsPlusIns/gpsPlusIns.ino` | The main sketch |
| `gpsPlusIns/MyInsGpsKalman.h` | Experimental Kalman filter class |
| `images/` | Plots from my test runs |

## Credits

[Libraries, tutorials, and anything the Kalman filter code was based on.]
