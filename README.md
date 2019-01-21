# inertial_sense


A ROS wrapper for the InertialSense uINS3 RTK-GPS-INS and Dual GPS (GPS Compassing) sensor.

## NOTICE:

To use this node, you will need to update firmware on your uINS to v1.7.3 [release page](https://github.com/inertialsense/InertialSenseSDK/releases). Download the appropriate `.hex` file and use the `firmware_update` ROS service to update the firmware
``` 
rosservice call /firmware_update /home/superjax/Download/IS_uINS-3_v1.7.3<...>.hex
```

## Installation
This is a ROS package, with the InertialSenseSDK as a submodule, so just create a catkin workspace, clone this into the `src` folder, pull down the submodule and build

``` bash
mkdir -p catkin_ws/src
cd catkin_ws/src
catkin_init_workspace
git clone https://inertialsense/inertial_sense_ros
cd inertial_sense
git submodule update --init --recursive
cd ../..
catkin_make
```

## Running the Node

```bash
rosrun inertial_sense inertial_sense_node
```

Make sure that you are a member of the `dailout` group, or you won't have access to the serial port.

For changing parameter values and topic remapping from the command line using `rosrun` refer to the [Remapping Arguments](http://wiki.ros.org/Remapping%20Arguments) page. For setting vector parameters, use the following syntax:

``` bash
rosparam set /inertial_sense_node/GPS_ref_lla "[40.25, -111.67, 1556.59]"
rosrun inertial_sense inertial_sense_node
```

For setting parameters and topic remappings from a launch file, refer to the [Roslaunch for Larger Projects](http://wiki.ros.org/roslaunch/Tutorials/Roslaunch%20tips%20for%20larger%20projects) page, or the sample `launch/test.launch` file in this repository.

## RTK
RTK (Realtime Kinematic) GPS requires two gps receivers, a _base_ and a _rover_.  The GPS observations from the base GPS are sent to the rover and the rover is able to calculate a much more accurate (+/- 3cm) relative position to the base.  This requires a surveyed base position and a relatively high-bandwidth connection to the rover.  If using a uINS with two GPS receviers, GPS 1 is used for base corrections.  The RTK functionality in this node is performed by setting parameters shown below.

It is important that the base position be accurate.  There are two primary methods for getting a surveyed base position.

  1. Find the location of the base on Google Maps (quick and easy, not as accurate)
  2. Put the base into rover mode with a 3rd-party base station such as a NTRIP caster.  Once the base has RTK fix, the absolute position of the base is accurate to within 3 cm.  Averaging this position over time is usually the most accurate way to get a base position, but takes more effort.


Once the base position has been identified, set the `refLLA` of the base uINS to your surveyed position to indicate a surveyed base position.

## Dual GNSS (GPS Compassing)
GPS Compassing is supported on units with two GPS receivers.  It also requires a very precise measurement of the locations of both GPS antennas relative to the uINS (+/- 1cm).  If you want to use the Dual GNSS mode, you must set the `dual_GNSS` parameter, and specify both the `GPS_ant1_xyz` and `GPS_ant2_xyz` vector parameters.

Dual GNSS uses the onboard RTK engine, so it is currently impossible for a uINS to be configured as both an RTK rover and for dual GNSS compassing simultaneously.  It is possible to provide base corrections while also acting in dual GNSS mode.

## Time Stamps

If GPS is available, all header timestamps are calculated with respect to the GPS clock but are translated into UNIX time to be consistent with the other topics in a ROS network.  If GPS is unvailable, then a constant offset between uINS time and system time is estimated during operation  and is applied to IMU and INS message timestamps as they arrive.  There is often a small drift in these timestamps (on the order of a microsecond per second), due to variance in measurement streams and difference between uINS and system clocks, however this is more accurate than stamping the measurements with ROS time as they arrive.  

In an ideal setting, there should be no jump in timestamps when GPS is first acquired, because the timestamps should be identical, however, due to inaccuracies in system time, there will likely be a small jump in message timestamps after the first GPS fix.

## Topics

Topics are enabled and disabled using parameters.  By default, only the `ins` topic is published to save processor time in serializing unecessary messages.
- `ins`(nav_msgs/Odometry)
    - full 12-DOF measurements from onboard estimator (pose portion is from inertial to body, twist portion is in body frame)
- `imu`(sensor_msgs/Imu)
    - Raw Imu measurements from IMU1 (NED frame)
- `gps`(inertial_sense/GPS)
    - unfiltered GPS measurements from onboard GPS unit
- `gps/info`(inertial_sense/GPSInfo)
    - sattelite information and carrier noise ratio array for each sattelite
- `mag` (sensor_msgs/MagneticField)
    - Raw magnetic field measurement from magnetometer 1
- `baro` (sensor_msgs/FluidPressure)
    - Raw barometer measurements in kPa
- `preint_imu` (inertial_sense/DThetaVel)
    - preintegrated coning and sculling integrals of IMU measurements
- `RTK/info` (inertial_sense/RTKInfo)
    - information about RTK status
- `RTK/rel` (inertial_sense/RTKRel)
    * Relative measurement between RTK base and rover
- `gps/obs` (inertial_sense/GNSSObservation)
    * Raw satellite observation (psuedorange and carrier phase)
- `gps/eph` (inertial_sense/GNSSEphemeris)
    * Satellite Ephemeris for GPS and Galileo GNSS constellations
- `gps/geph`
    * Satellite Ephemeris for Glonass GNSS constellation

## Parameters

* `~port` (string, default: "/dev/ttyUSB0")
  - Serial port to connect to
* `~baudrate` (int, default: 921600)
  - baudrate of serial communication
* `~frame_id` (string, default "body")
  - frame id of all measurements

**Topic Configuration**
* `~navigation_dt_ms` (int, default: Value retrieved from device flash configuration)
   - milliseconds between internal navigation filter updates (min=2ms/500Hz).  This is also determines the rate at which the topics are published.
* `~stream_INS` (bool, default: true)
   - Flag to stream navigation solution or not
* `~stream_IMU` (bool, default: false)
   - Flag to stream IMU measurements or not
* `~stream_baro` (bool, default: false)
   - Flag to stream baro or not
* `~stream_mag` (bool, default: false)
   - Flag to stream magnetometer or not
* `~stream_preint_IMU` (bool, default: false)
   - Flag to stream preintegrated IMU or not
* `~stream_GPS`(bool, default: false)
   - Flag to stream GPS
* `~stream_GPS_info`(bool, default: false)
   - Flag to stream GPS info messages
- `stream_GPS_raw` (bool, default: false)
   - Flag to stream GPS raw messages

**RTK Configuration**
* `~RTK_rover` (bool, default: false)
  - Enables RTK rover mode (requires base corrections from an RTK base)
* `~RTK_base` (bool, default: false)
  - Makes the connected uINS a RTK base station and enables the publishing of corrections
* `~dual_GNSS` (bool, default: false)
  - Uses both GPS antennas in a dual-GNSS configuration
* `~RTK_server_IP` (string, default: 172.0.0.1)
  - If operating as base, attempts to create a TCP port on this IP for base corrections, if rover, connects to this IP for corrections.
* `~RTK_server_port` (int, default: 7777)
  - If operating as base, creates a TCP connection at this port for base corrections, if rover, connects to this port for corrections.
* `~RTK_correction_type` (string, default: UBLOX)
  - If operating with limited bandwidth, choose RTCM3 for a lower bandwidth, but less accurate base corrections,  rover and base must match

**Sensor Configuration**
* `~INS_rpy` (vector(3), default: {0, 0, 0})
    - The roll, pitch, yaw rotation from the INS frame to the output frame
* `~INS_xyz` (vector(3), default: {0, 0, 0})
    - The NED translation vector between the INS frame and the output frame (wrt output frame)
* `~GPS_ant1_xyz` (vector(3), default: {0, 0, 0})
    - The NED translation vector between the INS frame and the GPS 1 antenna (wrt INS frame)
* `~GPS_ant2_xyz` (vector(3), default: {0, 0, 0})
    - The NED translation vector between the INS frame and the GPS 2 antenna (wrt INS frame)
* `~GPS_ref_lla` (vector(3), default: {0, 0, 0})
    - The Reference longitude, latitude and altitude for NED calculation in degrees, degrees and meters (use the `set_refLLA` service to update this automatically)
* `~inclination` (float, default: 1.14878541071)
    - The inclination of earth's magnetic field (radians)
* `~declination` (float, default: 0.20007290992)
    - The declination of earth's magnetic field (radians)
* `~dynamic_model` (int, default: 8)
    - Dynamic model used in internal filter of uINS.
       - 0 = portable
       - 2 = stationary
       - 3 = pedestrian
       - 4 = automotive
       - 5 = sea
       - 6 = airborne 1G
       - 7 = airborne 2G
       - 8 = airborne 4G
       - 9 = wrist

**ASCII Output Configuration**
* `~ser1_baud_rate` (int, default: 921600)
    - baud rate for serial1 port used for external NMEA messaging (located on H6-5) [serial port hardware connections](http://docs.inertialsense.com/user-manual/Setup_Integration/hardware_integration/#pin-definition)
* `~NMEA_rate` (int, default: 0)
    - Rate to publish NMEA messages
* `~NMEA_configuration` (int, default: 0x00)
    - bitmask to enable NMEA messages (bitwise OR to enable multiple message streams).
      - GPGGA = 0x01
      - GPGLL = 0x02
      - GPGSA = 0x04
      - GPRMC = 0x08
* `~NMEA_ports` (int, default: 0x00)
    - bitmask to enable NMEA message on serial ports (bitwise OR to enable both ports) 
      - Ser0 (USB/H4-4)  = 0x01 
      - Ser1 (H6-5) = 0x02 

## Services
- `single_axis_mag_cal` (std_srvs/Trigger)
  - Put INS into single axis magnetometer calibration mode.  This is typically used if the uINS is rigidly mounted to a heavy vehicle that will not undergo large roll or pitch motions, such as a car. After this call, the uINS must perform a single orbit around one axis (i.g. drive in a circle) to calibrate the magnetometer [more info](http://docs.inertialsense.com/user-manual/Setup_Integration/magnetometer_calibration/)
- `multi_axis_mag_cal` (std_srvs/Trigger)
  - Put INS into multi axis magnetometer calibration mode.  This is typically used if the uINS is not mounted to a vehicle, or a lightweight vehicle such as a drone.  Simply rotate the uINS around all axes until the light on the uINS turns blue [more info](http://docs.inertialsense.com/user-manual/Setup_Integration/magnetometer_calibration/)
- `firmware_update` (inertial_sense/FirmwareUpdate)
  - Updates firmware to the `.hex` file supplied (use absolute filenames)
* `set_refLLA` (std_srvs/Trigger)
  - Takes the current estimated position and sets it as the `refLLA`.  Use this to set a base position after a survey, or to zero out the `ins` topic.1
