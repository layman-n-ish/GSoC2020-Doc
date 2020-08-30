# IIO Driver for ADXRS290 Gyroscope

## Project Overview
Industrial I/O subsystem, the home of Linux sensors, as Daniel Baluta rightly calls it, provides software support for sensors, ADCs, DACs, clock generators, etc. under the Linux kernel tree. The increasing demand for consolidation due to the bloom in fields of Internet of Things (IoT) and the Automotive industry requires increased efforts to provide kernel support for novel devices being manufactured at pace; this project, a small step in the similar direction, aims to implement an IIO driver for Analog Devices, Inc.’s (ADI) ADXRS290 gyroscope with support of IIO channels, buffers and triggers for both its pitch and roll axes.

The ultimate goal is to:

- merge ADXRS290's compatible driver upstream to ADI's kernel tree
- deliver corresponding ABI documentation
- implement device-tree bindings for ADXRS290 in YAML
- exhibit a working example with the evaluation board on Raspberry Pi 3B, utilizing the SPI protocol.

## Accomplishments

- Effective direct-mode single-shot data access of **raw** angular rate data (both roll and pitch axes) and temperature data from userspace by exposing attribute files under the `sysfs` filesystem of the Linux kernel. This was achieved by shaping up features of IIO channels (i/p or o/p type, raw or processed data, scale and offset to convert to processed data, etc.) and borrowing APIs constructed in the SPI subsystem to initiate the interface between RPi's SPI master and ADXRS290.
- Disclosed device attributes to hint conversion scheme of the raw data to their appropriate units as defined in the kernel's sysfs ABI (Application Binary Interface), for both angular rate and temperature data. For angular rate, the desired unit is *radians/sec* and *milli degree Celsius* for temperature data.
- The band-pass filter in ADXRS290 is made configurable from the userspace by writing to certain `sysfs` attribute files which introduce the interface to program the 3db cut-off frequencies for both the low-pass and high-pass filters in ADXRS290.
- Implemented kernel (software) buffer support based on `kfifo` which was configured to store data for x-axis angular rate, y-axis angular rate, temperature and time-stamp.  The buffer is available in the userspace by a character device interface in the `devfs` filesystem which uses the said configuration (storage size, endianness, shift) to present the data capture appropriately. These buffers can also be "exported" to other kernel drivers which are consumers of this device's IIO channels, however a strong case for the consumers of this gyroscope data is not made yet.
- The IIO buffers' data capture was tetsed by software-based triggers which signal when to push data into the buffers. Both such software-based triggers were tested for continuous data capture - `sysfs-trig` (triggers a data poll from userspace when signaled by the user by writing to a attribute file) and `hrtimer` (polls data into the buffer by facilitating a high resolution timer with configurable sampling frequency).
- The `SYNC` pin exposed by ADXRS290 was used as a DATA_READY interrupt hereby manifesting as a hardware-based trigger for the IIO buffers. A GPIO irq line was phased out in the devicetree overlay to register the IIO trigger device and attach it to the IIO buffer-trigger mechanism acting as a DATA_READY interrupt. When the interrupt is generated on the rising edge, a **bulk read** of the registers of ADXRS290 is exercised by the lower-half of the interrupt handler by a kernel thread and then pushed to the IIO buffer along with the time-stamp. [**tl;dr** *Challenges were faced when setting up the SYNC pin to activate by writing to the `DATA_RDY` register; the write to the register was not reflected back. With the help of my mentors, upon analyzing the SPI signals using an oscilloscope, we concluded with no fault of our written driver and hence made efforts to debug by conversing with the hardware folks at ADI. Unfortunately, a conclusion on the issue hasn't been drawn yet; it **could** arise due to the hardware ([EVAL-ADXRS290Z evaluation board](https://www.analog.com/en/design-center/evaluation-hardware-and-software/evaluation-boards-kits/eval-adxrs290.html)) or due to the underlying SPI peripherals in RPi. With the advice from my mentors, I had resorted to the use of [bit-banging](https://en.wikipedia.org/wiki/Bit_banging) to drive the SPI lines which worked like a charm!*]
- Interface in the `debugfs` filesystem was also implemented to read/write byte data from/to the device using SPI transactions. This proved to be extremely useful during the testing of the state of `DATA_RDY` register for the issue faced (as described above).
- [**Under development**] Started an [incremental blog series](https://layman-n-ish.github.io/tag/gsoc-2020/) to assist kernel newbies in constructing a device driver in the Linux kernel's IIO subsystem from scratch.

## Demo

Checking whether the `adxrs290` kernel module is loaded, viewing devices registered with the IIO core under `/sys/bus/iio/devices/` (`iio:device0` - our ADXRS290 gyroscope, `trigger0` - our DATA_READY hardware trigger), listing various attributes, IIO channels, etc. of ADXRS290 IIO device, polling raw temperature data and temperature scale value (raw value * scale = temperature in *milli degree Celsius*) every second:

![lsmod_temp_channel](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/lsmod_temp_channel.png)

Similarly, polling each axes angular velocity every second and reading the angular velocity scale value which when multiplied with the raw data gives angular velocity in *radians/sec*:

![anglvel_channel_poll](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/anglvel_channel_poll.png)

Configuring the 3db cut-off frequencies of both low-pass and high-pass filters (note that the desired frequency must be from the list of available frequencies):

![filter_conf](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/filter_conf.png)

Displaying the characteristics of each scan element (the `kfifo` buffer is made up of each of these scan elements), i.e. its index (where in the buffer is that element located), type (little or big endian, number of bits, signed or unsigned, shift, etc.) and if its enabled or not (scan element pushed to the buffer only of enabled, else that region is zero initialized). Towards the end, I enable data capture for all the channels - x-axis angular velocity, y-axis angular velocity, temperature and timestamp:

![buffer_scan_elem](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/buffer_scan_elem.png)

Once scan elements are enabled, set buffer length (optional) and enable buffer capture from the userspace by reading associated `devfs` character device (Note that this buffer capture is hardware triggered by the SYNC pin manifesting as a GPIO irq line):

![buf_capture_w_datardy_trig](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/buf_capture_w_datardy_trig.png)

Similarly, software-triggered buffer capture with `hrtimer` (after making an instance of hrtimer in `configfs` by `mkdir /sys/kernel/config/iio/triggers/hrtimer/test_timer` - this automatically registers itself with the IIO core); notice how the `current_trigger` of `adxrs290` IIO device is modified to use the `test_timer`; also, note that how the `sampling_frequency` can be modified with this interface for a bulk read of all data channels:

![buf_capture_w_hrtimer](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/buf_capture_w_hrtimer.png)

Similary, software-triggered buffer capture with `sysfs-trig`; with this interface, one can exercise a bulk read of all the data channels from the userspace whenever the user requires:

![buf_capture_w_sysfs_trig](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/buf_capture_w_sysfs_trig.png)

Continuous hardware-triggered (since using `adxrs290-dev0` trigger) buffer capture from userspace using the cross-compiled `iio_generic_buffer` tool; this tool helps in auto-enabling of data channels, enables the buffer capture by itself and displays **processed** data in appropriate units:

![util_iio_generic_buffer](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/util_iio_generic_buffer.png)

`debugfs` interface with direct register access to read/write byte data:

![debugfs_iio_test](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/debugfs_iio_test.png) 

Continuous buffer capture and plotting of each axes angular velocity, using [ADI's Oscilloscope](https://github.com/analogdevicesinc/iio-oscilloscope); I swing the gyroscope along each axis alternatively while capturing:

![adi_oscilloscope_buf_capture](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/adi_oscilloscope_buf_capture.gif)

Frequency domain plot (4096-length FFT), using ADI Oscilloscope, for:

1. LPF 3db freq = 480 Hz (default), HPF 3db freq = 0 Hz (default):

![freq_domain_l_480_h_0](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/freq_domain_l_480_h_0.png)

2.  LPF 3db freq = 20 Hz (min), HPF 3db freq = 0 Hz (default):

![freq_domain_l_20_h_0](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/freq_domain_l_20_h_0.png)

3. LPF 3db freq = 480 Hz (default), HPF 3db freq = 11.3 Hz (max):

![freq_domain_l_480_h_11-3](https://github.com/layman-n-ish/GSoC2020-Doc/blob/master/demo/freq_domain_l_480_h_11-3.png)

## Patches

Listed below in chronological order...  

### ADI's Linux Tree

- `gsoc2020-gyro` branch: [Skeleton driver for ADXRS290](https://github.com/analogdevicesinc/linux/pull/1046)
  - Adds basic driver support for ADXRS290
  - Enables ADXRS290 driver support by adding CONFIG symbol in RPi's defconfig
  - Includes minimal devicetree overlay for ADXRS290
  - Adds corresponding devicetree bindings doc

- `gsoc2020-gyro` branch: [Update ADXRS290's driver for channels support](https://github.com/analogdevicesinc/linux/pull/1059)
  - Write support for reading angular rate data - x-axis (roll) & y-axis (pitch) - through exposed sysfs channels. Also, adds a channel to read temperature data. sysfs attributes to program the 3db cut-off frequencies of the band-pass filter internal to the ADXRS290 are introduced.
  - Update devicetree overlay to configure the device in SPI mode 3 & declare a maximum frequency for SPI transactions 
  - Rewrite devicetree bindings doc in accordance with the overlay file

- `gsoc2020-gyro` branch: [Sync with upstream version](https://github.com/analogdevicesinc/linux/pull/1102)
  - sync driver and devicetree bindings with reviews from upstream 

- `master` branch: [Add driver support for ADXRS290](https://github.com/analogdevicesinc/linux/pull/1103)
  - merge ADXRS290 direct-mode channels support

- `gsoc2020-gyro` branch: [Add triggered buffer & debugfs support for ADXRS290](https://github.com/analogdevicesinc/linux/pull/1131)
  - Fix a mutex initialization bug
  - Provide a way for continuous data capture by setting up buffer support. The data ready signal exposed at the SYNC pin of the ADXRS290 is exploited as a hardware interrupt which triggers to fill the buffer.
  - Add a GPIO irq line to act as a DATA_RDY trigger in the devicetree overlay file
  - Update devicetree bindings with examples on using the GPIO irq line
  - Update driver code to enable direct access to the device's registers with debugfs
  

### IIO Maintainer's Tree [Upstream]

- [Add driver support for ADXRS290](https://patchwork.kernel.org/project/linux-iio/list/?series=&submitter=&state=&q=Add+driver+support+for+ADXRS290&archive=&delegate=) and [devicetree bindings doc](https://patchwork.kernel.org/project/linux-iio/list/?series=&submitter=&state=&q=Add+DT+binding+doc+for+ADXRS290&archive=&delegate=)
  - same as [channels support patch](https://github.com/analogdevicesinc/linux/pull/1059) for ADI's linux tree

- [**Under Review**] [Add triggered buffer & debugfs support for ADXRS290](https://patchwork.kernel.org/project/linux-iio/list/?series=338197)
  - same as [triggered buffer support patch](https://github.com/analogdevicesinc/linux/pull/1131) for ADI's linux tree

(Upstreamed commits in jic23/iio.git can be seen [here](https://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git/commit/?h=testing&id=2c8920fff1457a41912e8d3e3b9eafb582656440) & [here](https://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git/commit/?h=testing&id=107ce2e3dccceefd91a2af3069de63774cbaccf9))

## Future Work

- Support basic power management duties by setting up hooks that drive the device to `MEASUREMENT` mode only when data is being captured and is kept in `STANDBY` mode otherwise.
- Merge triggered-buffer & debugfs patches upstream (to Jonathan Cameron's (IIO Maintainer) tree - [jic23/iio.git](https://git.kernel.org/pub/scm/linux/kernel/git/jic23/iio.git/)).

## Acknowledgement

I'm extremely thankful to...

...my mentors - [Darius Berghe](https://github.com/buha) and [Dragos Bogdan](https://github.com/dbogdan) - for the constant support, guidance, fun weekly audio chats and funding the hardware.

...the folks at Analog Devices, Inc. for casually parting knowledge, from programming tips to internals of the kernel, and reviewing my code.

...the kernel hackers who are a part of the IIO mailing list and also, the maintainer of the IIO subsystem of the Linux kernel, Jonathan Cameron, who all assisted me a great deal to get my patches accepted.

...The Linux Foundation for organizing this learning-heavy IIO project.

...and finally, Google Open Source for orchestrating the GSoC program and ensuring a fruitful, fun-packed summer.
