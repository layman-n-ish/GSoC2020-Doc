# IIO Driver for ADXRS290 Gyroscope

## Project Overview
Industrial I/O subsystem, the home of Linux sensors, as Daniel Baluta rightly calls it, provides software support for sensors, ADCs, DACs, clock generators, etc. under the Linux kernel tree. The increasing demand for consolidation due to the bloom in fields of Internet of Things (IoT) and the Automotive industry requires increased efforts to provide kernel support for novel devices being manufactured at pace; this project, a small step in the similar direction, aims to implement an IIO driver for Analog Devices, Inc.’s (ADI) ADXRS290 gyroscope with support of IIO channels, buffers and triggers for both its pitch and roll axes.

The ultimate goal is to:

- merge ADXRS290's compatible driver upstream to ADI's kernel tree
- deliver corresponding ABI documentation
- implement device-tree bindings for ADXRS290 in YAML
- exhibit a working example with the evaluation board on Raspberry Pi 3B, utilizing the SPI protocol.

## Accomplishments

lorem ipsum

## Demo

lorem ipsum

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

lorem ipsum

## Acknowledgement

lorem ipsum
