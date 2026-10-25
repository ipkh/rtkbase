# LG290K on Olimex i.MX8MPlus: Serial and Timing Wiring

This note records the current USB setup and the planned transitions to PPS timing, direct UART, or both. The EXT header pin names and numbers below are supplied board information; they are not a substitute for confirming the board revision schematic and pin multiplexing before wiring.

## Current software profile

The proven connection is the LG290K USB serial interface at `/dev/ttyACM0`, 460800 baud. RTKBase streams receiver data; GPSD reads the mixed NMEA/RTCM stream from the local RTKBase TCP feed. Chrony receives GNSS time but should not be considered PPS-disciplined until the PPS path is wired, enabled, and verified.

Keep this USB configuration as the recovery baseline when adding header signals. Make one hardware/software change at a time and preserve a known-good receiver configuration.

## Olimex EXT header pins recorded

### EXT1 (26 pins)

| Pin | Signal |
| ---: | --- |
| 1 | +5 V |
| 2 | GND |
| 3 | VDD_3V3 |
| 4 | GND |
| 5 | VDD_1V8 |
| 20 | UART1_RXD |
| 22 | UART_TXD |
| 24 | UART1_CTS |
| 26 | UART1_RTS |

### EXT2

| Pin | Signal |
| ---: | --- |
| 1 | VDD_3V3 |
| 2, 4 | GND |
| 3 | VDD_1V8 |
| 5 | GPIO_1000 |
| 6 | GPIO_1001 |
| 39 | UART2_RTS |
| 40 | UART2_CTS |

## Electrical and pin-mux checks before connecting anything

- Verify the exact SOM/carrier revision and the pin functions against its schematic and current device tree.
- Confirm LG290K I/O voltage, UART and TIMEPULSE output levels, polarity, pulse width, and drive capability from the receiver documentation. The listed 1.8 V and 3.3 V header supplies do not by themselves establish the logic level of every signal.
- Never connect a receiver output to a board supply pin. Do not connect 5 V logic to a GPIO/UART. Use an appropriate level translator where required; share signal ground.
- Establish whether UART1 is currently assigned to the kernel console or a systemd getty. The current system has a serial console parameter (`console=ttymxc1`); identify its mapping before repurposing pins. Disable/move the console and getty only after a recovery console is available.
- Identify the SoC GPIO controller/line and supported pinctrl function for GPIO_1000/GPIO_1001 in the schematic and device tree. Do not infer a Linux GPIO line number or PPS-capable input from the connector label alone.
- Do not assume UART flow control is needed. If enabled, cross CTS/RTS according to the signal direction and verify both receiver and SoC configuration; otherwise leave CTS/RTS disconnected.

## Profile A: USB serial, no PPS (current baseline)

- Connect through the LG290K USB interface.
- RTKBase serial input: USB device (currently `ttyACM0`) at `460800:8:n:1`, receiver `quectel_LG290P03`, format `rtcm3`.
- Keep the configured NMEA output available in the RTKBase stream for GPSD and Chrony.
- Leave the Chrony PPS refclock disabled. NMEA time can provide a coarse GNSS clock source, but it is not equivalent to a PPS-disciplined clock.

## Profile B: USB serial plus PPS

Keep USB as the receiver's data/configuration interface and connect the receiver's documented TIMEPULSE/PPS output to a verified, electrically compatible SoC GPIO input. EXT2 pins 5 or 6 are possible GPIO candidates only after the schematic, pinmux, voltage, and kernel driver mapping have been confirmed; no PPS pin assignment is established by this note.

Software work after electrical validation:

1. Enable the chosen pin's input function in the board device tree/pinctrl and attach the Linux PPS client/overlay supported by this kernel. Confirm a stable `/dev/ppsN` node and timestamp events using PPS tools.
2. Keep the receiver's NMEA time feed active: PPS provides the precise second edge but needs a valid time-of-day source for association.
3. Configure GPSD/Chrony to use the actual PPS device and GNSS time source. For Chrony, use the PPS refclock locked to the GNSS source (for example `refclock PPS /dev/pps0 refid PPS lock GNSS`) only after the device name and edge are verified.
4. Check `chronyc sources -v`, `chronyc sourcestats`, and tracking. Require PPS to be reachable and selected/combined as intended; confirm UTC against a trusted reference and observe stable sub-millisecond or better offsets before treating this profile as operational.

## Profile C: direct UART serial, no PPS

Use this profile to move receiver data/control from USB to the Olimex header. It requires device-tree pinmux and serial-console review first.

- Wire receiver TX to EXT1 pin 20 (`UART1_RXD`) and receiver RX to EXT1 pin 22 (`UART_TXD`), with a common ground. Confirm the carrier's actual UART1 assignment and LG290 logic voltage before connection.
- Keep UART1 CTS/RTS (EXT1 pins 24/26) unused unless hardware flow control is deliberately enabled and cross-wired correctly.
- Enable the UART1 pinmux and kernel serial driver, remove any console/getty ownership on that UART, and reboot only with serial-console recovery available.
- Determine the resulting Linux device path from the booted system (for example, an SoC UART device). Do not hard-code an assumed `ttyS*` or `ttymxc*` name.
- Configure RTKBase to use that serial device at the receiver's configured baud rate, initially 460800 8-N-1, with receiver `quectel_LG290P03` and format `rtcm3`. Use a stable udev/by-path name where possible.
- The current Quectel auto-detection path probes USB ACM/USB serial devices; for direct UART, set the confirmed port, baud, and Quectel receiver model in RTKBase settings before running receiver configuration. Do not add blind probing of a UART that may still be a console.
- Avoid having RTKBase streaming, GPSD, and the configuration utility open the same UART concurrently. Stop the stream before receiver reconfiguration, then restart and verify it.
- Confirm the LG290 UART output includes the expected RTCM and NMEA data. If it does not, configure per-port message output/rates using the receiver protocol, save settings, and test after reboot.

With this profile, GPSD should consume the stream through the configured RTKBase TCP output rather than opening the serial device itself. Keep PPS disabled until separately wired and validated.

## Profile D: direct UART serial plus PPS

Combine Profile C's UART wiring, pinmux, console, and RTKBase port setup with Profile B's verified PPS input and Chrony/GPSD configuration. Test UART data and PPS independently first, then together. Keep NMEA available on the serial stream for time-of-day association; PPS alone does not identify the UTC second.

## Acceptance checks for every profile

- Reboot and confirm receiver port identity, baud rate, receiver mode, survey state, message output, and RTKBase service recovery.
- Confirm GPSD reports valid NMEA time and fix/satellite information from the intended feed.
- Validate RTCM frame CRCs and expected observation/station message types.
- For PPS profiles, confirm regular kernel PPS events, correct edge/polarity, PPS/GNSS lock, and Chrony reachability/source selection.
- Check that network NTP fallback policy is explicit. A visible GNSS/PPS refclock is not proof it is selected or that the system clock is using it.
- Keep the USB profile and a bootable backup until the new profile passes a sustained soak test.
