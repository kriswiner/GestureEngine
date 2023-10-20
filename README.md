# GestureEngine
Arduino sketches for high-accuracy accel/gyro and barometer configured as gesture detector/tracker

The idea here is to make use of two new ST sensors (LSM6DSV and LPS22DF) which, in combination, should allow ultra-low-power gesture recognition and detection for applications such as head or limb tracking, ergonometrics, UAVs and general navigation.

I designed a breakout board for the [LSM6DSV](https://www.st.com/resource/en/datasheet/lsm6dsv.pdf) combination accel/gyro with embedded finite-state machine and sensor fusion which I hope will allow six DoF orientation estimation without having to run a fusion filter on the host MCU. I added the [LPS22DF](https://www.st.com/resource/en/datasheet/lps22df.pdf) barometer which allows altitude estimation with ~10 cm accuracy. Both of these sensors can perform these feats while consuming just a few uA of power.

![GestureEngine](https://user-images.githubusercontent.com/6698410/270500414-0c61126a-d074-4be5-9fb3-47e5183ae3b9.jpg)

I am using a hand-built [Ladybug](https://www.tindie.com/products/tleracorp/ladybug-stm32l432-development-board/?pt=ac_prod_search) (STM32L432KC) development board (see [here](https://oshpark.com/shared_projects/Yi34KlP5) for pcb design) which has an excellent [Arduino core](https://github.com/GrumpyOldPizza/arduino-STM32L4) making it easy to get the most out of these sensors. The Ladybug uses only 2.4 uA in stop mode so is an excellent ultra-low-power development board for this application. The resulting sketches can be adapted to other Arduino-based MCUs like the ESP32 or Teensy, etc with a bit of effort and a lot more power usage.

The plan is to create standalone Arduino sketches for each sensor separately showing basic sensor operations and then combine into a single set of "firmware" that can track gestures. My target is hand gestures and a slightly different approach might be required for head tracking or vehicle tracking, etc. But the basic building blocks should provide a good place to start for other applications.

**First up is the LSP22DF barometer.** The basic sketch shows how to configure the sensor data rate, averaging, and low pass filter, set up the interrupt, set up and configure the FIFO, read the barometer raw data and convert to properly scaled pressure and temperature. The scaled pressure is used to convert to an altitude estimate. 

The sketch does not show how to use the barometer in differential mode although this should be straightforward. One particularly interesting use case for differential pressure would be to set the current pressure as a reference, then set up an interrupt for when the change in pressure exceeds some value (like 3 or 5 mBar) and trigger the FIFO to capture data on either side of this transition. This would be useful to monitor "man-down" applications for firefighters, etc. As you can see in the following demo, this can also be done in the main loop just by selecting the proper amount of data averaging and detecting meaningful changes in altitude.

Here are typical results running the sensor at 1 Hz with low pass filter of ODR/4 and two settings for the averaging, either 4-sample or 64-sample averaging. The difference in power usage (according to the data sheet) is 2.5 uA vs 6.3 uA, respectively. It costs more power to average more data, of course. But for this little bit of extra power, the data jitter drops by a factor of ~3:

![Altitude test](https://user-images.githubusercontent.com/6698410/270502358-2e9ddef7-a1be-41c0-8efd-57bd10e0dd88.jpg)

The blue triangles are altitude estimated from scaled pressure data with averaging at 4x with the breadboard flat on a table. At 55 seconds, I lifted the breadboard over my head and held it. The ~three feet of elevation change is easy to detect despite the data jitter. With averaging set to 64x the data jitter is dramatically reduced and the discrimination (or signal-to-noise) in elevation change is significantly improved. This improvement comes at a cost of ~4 uA, so there is a simple trade between elevation jitter and power consumption. In either case, the power usage is well within the ultra-low power regime. Further reductions in pressure (and therefore, altitude) jitter are possible as are higher data rates to track dynamic motion all at increased power usage.

**Next is the LSM6DSV combination accel/gyro.** This is a rather complicated sensor with a lot of features; the **LSM6DSV_Basic_Ladybug** sketch demonstrates only a few of them. The basic sketch shows how to configure the accel/gyro full-scale, odr, and bandwidth settings as well as set up the data ready interrupts, perform the self tests, calibrate the accel and gyro for offset biases, read the raw data, properly scale the data and display this on the serial monitor. The 6 DoF Madgwick sensor fusion algorithm running on the STM32L4 MCU is used to generate yaw, pitch, and roll estimates from the scaled data. The sketch demonstrates operation of the FIFO to collect uncompressed accel and gyro data, read the raw data from the FIFO, and reconstruct properly-scaled data therefrom. Lastly, three of the simpler hardware functions of the LSM6DSV are implemented: activity detection, rotation change detection and single tap detection.

Running both the gyro and accel at 240 Hz produces a steady yaw estimate with surprisingly low drift:

![Madgwick yaw test](https://user-images.githubusercontent.com/6698410/271437726-b179273b-bcc6-4766-9d93-19d1ea4e99de.jpg)

In this test,  the yaw, pitch and roll are plotted vs time every second while the breadboard containing the LSM6DSV sensor (see above) is manually held flat and stationary along a table edge. Every minute or so the orientation was changed by 90 degrees (i.e., the 90-degree table turning test), completing a 360-degree turn back to the start. This is a manual test with a USB cable so not super high precision nor high accuracy. It is good enough to detect jitter and drift, especially yaw drift, which is expected for any 6DoF AHRS solution. The absolute accuracy of the measured turning angle is poor; mostly +/- 10 degrees of what should be 90 degrees (due to an error in the gyro scale factor, since corrected). While the breadboard is stationary, there is very little (~1 degree or less over ~1 minute) yaw drift. Remarkable. Absolute orientation requires use of a magnetometer with its more complicated calibration requirements, larger suceptibility to environmental interference, and higher BOM cost and complexity. A super-low-jitter 6DoF solution (sans mag) is very attractive for a lot of use cases (especially dynamic gesturing and limb tracking, etc). The next step is to move the sensor fusion processing from the MCU to the LSM6DSV itself and see what the tradeoff is between accuracy and power usage....

It was surprisingly easy to make the SFLP (**sensor fusion low power**) embedded function work (**LSM6DSV_SFLP_Ladybug** sketch). I did have to make some changes from the basic sketch. Firstly, I changed the FIFO handling to be interrupt based and moved FIFO handling out of the RTC Alarm handler as a separate (and only) INT1 activity; no need to read the individual sensors via data ready since all data collection can and should be managed by FIFO reads for highest efficiency. Next, I discovered that the LSM6DSV uses the ENU orientation convention rather than the NED I usually use, so I changed the Madgwick fusion filter accordingly to allow comparison, but this doesn't matter since I dropped the Madgwick fusion filter from this sketch anyway. The FIFO data comes tagged to allow identifying the type of data and also the order (time association) of the data in the FIFO. This is typical serial monitor output including accel, gyro, gravity, and quaternions as well as yaw, pitch and roll which is derived from the quaternions.

![typicaloutput](https://user-images.githubusercontent.com/6698410/271735999-61fbd0fe-bedb-439f-8859-13c893a3495e.jpg)

I am plotting just the first two data sets to make for easier reading. Since I am using 15 Hz data rate for both the accel/gyro ODR and SFLP ODR and a FIFO watermark of 60, I expect to get a FIFO full interrupt every second with 15 sets of accel data, gyro data, gravity data and quaternion data each for a total of 60 seven-byte (tag + six data bytes) FIFO entries.  

In an application, the righter way to manage this is to configure the 256-entry (256 entries x 7 bytes/entry = 1792 total bytes uncompressed) FIFO such that the interrupt triggers when this is full, use the MCU to burst read the FIFO, and then immediately log the FIFO data via seven page writes (256 bytes x 7 = 1792 bytes total) to SPI NOR flash, for example. All the processing would be done off-board, later. This would be the most power-efficient way to collect gesture data. The time between FIFO reading/data logging events would depend on the SFLP ODR.

I haven't taken a look at power usage yet, but I did look at (static) orientation accuracy, which was quite surprising.

![90degreeturntestcomparison](https://user-images.githubusercontent.com/6698410/271735946-8e0f5686-421e-46dc-b806-4ea425c4f02b.jpg)

I am comparing the yaw from the Madgwick sensor fusion solution with the yaw from the SFLP solution again using the simple 90-degree table turning test. Both solutions show excellent stability (not surprising since they are using the same data after all!) but the Madgwick solution is systematically too low with typical turn angles of ~80 degrees (due to incorrect scaling of the gyro, since fixed) whereas the SFLP is showing 90 degrees within a degree or two. In fact, the SFLP is probably more accurate than I am in placing the breadboard against the table edge. Remember, these are not absolute orientation estimates wrt True North like one might get by using a magnetometer and 9 DoF sensor fusion. Both  6DoF solutions here start off at or near 0 degrees from whatever orientation they happen to be in upon power on. So they provide a relative orientation estimation. But for a lot of applications, even relative orientation estimation with this kind of low jitter and high relative accuracy is plenty good enough.

The **LSM6DSV_Embedded_Ladybug** sketch shows how to configure the finite-state machine (FSM) to execute four programs: 1) wrist-tilt detect, 2) shake detect, 3) any-motion detect, and 4) glance detect each on four of the eight available FSMs. These are taken from examples found in [AN5273]((https://www.st.com/content/ccc/resource/technical/document/application_note/group1/6f/b8/c2/59/7e/00/43/c6/DM00572971/files/DM00572971.pdf/jcr:content/translations/en.DM00572971.pdf)) that I previously got to work using the LSM6DSO; they required some small changes since the register names and bit positions are different between the two sensors although the programming method is the same. These functions duplicate the embedded functions somewhat such as tilt detect and activity detect so not all of these need to be included in an application program. Also, it is likely the FSM programs will need to be tuned for the specific application requirements, etc. The four programs combined use of 106 of the 256 programming bytes per page, and there are at least two pages available for programming, so many more programs could be added. Every program line in the sketch is commented and using the LSM6DSV FSM application note ([AN5907]((https://www.st.com/content/ccc/resource/technical/document/application_note/group1/6f/b8/c2/59/7e/00/43/c6/DM00572971/files/DM00572971.pdf/jcr:content/translations/en.DM00572971.pdf))) it should be possible to understand what each program does. However, it will take a bit of work to fully master FSM programming; it ain't Arduino!

So there are enough building blocks here to create a basic gesture detection and logging application. Left to decide is whether to have the LSM6DSV manage the barometer as a slave or keep these two separate to be managed by the MCU host. There might be some value in having a FSM program detect altitude change events using data input from the barometer. Also, I have been having some trouble routing embedded interrupts to INT2, not sure why these sometimes seem not to work as expected. With the LPS22DF as LSM6DSV slave, I would be using INT2 for the barometer data ready interrupt anyway. I will probably make use of the [Katydid](https://www.tindie.com/products/tleracorp/katydid-wearable-ble-sensor-board/) wearable device for application development and testing, so I would need to design a module to plug into the Katydid connectors. The BLE will help in application debugging. The main gesture info is the 6 bytes of orientation quaternions. At a rate of 15 Hz, the 16 MByte SPI flash on board the Katydid should allow more than 24 hours of continuous orientation plus gesture + timestamp data to be logged. Even at 60 Hz there is likely enough flash capacity to capture gesture data for an entire day's (8 hours) worth of activity. Especially when making good use of the activity/inactivity detector to meter the data logging, etc. The ultra-low-power of the Katydid, LSM6DSV, and LPS22DF devices should allow this with a very small (< 100 mAH) LiPo battery to minimize weight.  

**Some preliminary power results.** Running the Embedded Sketch with the Ladybug CPU at 80 MHz, AODR and GODR at 30 Hz in high-performance mode with STM32.stop; at the end of the main loop (which is designed such that the STM32L4 is in STOP mode unless handling an interrupt) I measured (using Nordic's Power Profiler II) 752 uA average current usage after the initial configuration in setup and while the main loop was running. Once the inactivity interrupt fires, the FIFO is disabled, the gyro is powered down and the accel is placed in low-power mode LP1 at 1.875Hz ODR. This is all configurable, of course, and different choices can be made, but these seem reasonable to me for a low-power device application. In the inactive state, average current usage is 14.8 uA. Let's see if these numbers make sense.

*The STM32L4 uses 2.4 uA in STOP mode. The LSM6DSV data sheet says that the accel uses 4.0 uA in LP1 mode at 1.875Hz, and the gyro in power down mode uses 2.6 uA, the LPS22DF uses 0.9 uA in power down mode. Total accounting for known sources in the inactive state is 10.1 uA. The remaining ~4.7 uA is due to the led blinking for 1 ms every second and the STM32 waking up to do so. STM32L4 at 80 mHz uses ~5 mA, so led blinking for 1 ms should use ~5000 uA * 1 ms/1 s ~5 uA. All accounted for!*

*The LSM6DSM accel and gyro use 650 uA in high-performance mode for all ODRs. The cost of running the SLFP (3.5 uA at 15 Hz) and FSMs (~(n + 1) * 2 uA ~ 10 uA for n = 4) inside the LSM6DSV is another ~14 uA. STM32 STOP current is 2.4 uA. The extra ~75 uA is likely due to the STM32 waking up to service the FIFO and RTC interrupts and flash the led; at 80 MHz CPU speed being awake even a small fraction of the time can easily use up ~75 uA. In any case, 752 uA average current usage means about 130 hours (> 5 days) on a small 100 mAH LiPo battery, as expected.*

I would normally run an application at 4 MHz CPU speed to minimize power usage from the MCU (14.8 uA goes down to 11.5 uA when inactive). As long as the "inactive" accel ODR stays above those specified for the SFLP and FSM these will continue to function properly even when the gyro is powered down.

The **integration of the LPS22DF barometer as a slave to the LSM6DSV** using the latter's embedded Sensor Hub (Mode 2) worked as expected (**LSM6DSV_LPS22DF_Mode2_Ladybug**). The benefit of this arrangement is that the baro data gets into the LSM6DSV FIFO and is available for use in an FSM program, if desired. The board I designed to test this exposes only power, GND, LSM6DSV slave I2C bus (SDA/SCL), and an interrupt (LSM6DSV INT1). The baro is connected to the LSM6DSV master I2C bus and the data ready output of the baro is connected to INT2 as a master data ready input. This allows reading of the slave (baro) data at the natural rate of the baro. Otherwise, the baro data can be read at the rate of the accel. The downside of using the baro data ready is that the accel/gyro and baro ODRs are not commensurate conmplicating temporal alignment, if required.

![Mode 2 image](https://user-images.githubusercontent.com/6698410/277066660-8d390b9d-7b60-46fb-a7e1-322bee31f446.jpg)

There is one quirk (isn't there always!) I found with Mode 2 as connected. When I route the activity/inactivity detector interrupt to INT1 (INT2 is taken up with the baro data ready) I couldn't get any interrupts at all. When routing the activity/inactivity interrupt to INT2 or not specifying an interrupt everything works as expected, except that there is no interrupt indication of sleep/wake events even though the LSM6DSV indeed toggles between a "sleep" state and "active" state as requested. It's just that the interrupt cannot work on INT1. Why? TBD....

