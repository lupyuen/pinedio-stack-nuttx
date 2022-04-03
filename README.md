# PineDio Stack BL604 RISC-V Board on Apache NuttX RTOS

[__Follow the updates on Twitter__](https://twitter.com/MisterTechBlog/status/1510406086326513668)

Apache NuttX RTOS, ST7789 Driver and LVGL Graphics Library all run OK on Pine64's PineDio Stack BL604!

Source code is here...

-   [lupyuen/incubator-nuttx (pinedio branch)](https://github.com/lupyuen/incubator-nuttx/tree/pinedio)

-   [lupyuen/incubator-nuttx-apps (pinedio branch)](https://github.com/lupyuen/incubator-nuttx-apps/tree/pinedio)

# Shared SPI Bus

Acording to the PineDio Stack Schematic...

-   [PineDio Stack Schematic (2021-09-15)](pinedio_stack_v1_0-2021_09_15-a.pdf)

-   [PineDio Stack Baseboard Schematic (2021-09-27)](PINEDIO_STACK_BASEBOARD_V1_0-SCH-2021-09-27.pdf)

The SPI Bus is shared by...

-   ST7789 Display Controller

-   Semtech SX1262 LoRa Transceiver

-   SPI Flash

![Shared SPI Bus on PineDio Stack BL604](https://lupyuen.github.io/images/pinedio-spi2.jpg)

Here are the BL604 GPIO Numbers for the shared SPI Bus...

| Function | GPIO |
| :------- | :---: |
| SPI MOSI | 13 |
| SPI MISO | 0  |
| SPI SCK  | 11 |
| SPI CS _(Unused)_ | 8 |

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/include/board.h#L99-L105)

To prevent crosstalk, we select each SPI Device by flipping its Chip Select Pin from High to Low...

| SPI Device | Device ID | Swap MISO/MOSI | Chip Select | 
| :--------- | :-------: | :------------: | :---------: |
| ST7789 Display     | 0x40000 | No  | 20
| SX1262 Transceiver | 1       | Yes | 15
| SPI Flash          | 2       | Yes | 14
| _(Default Device)_ | -1      | Yes | 8 _(Unused)_

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/include/board.h#L106-L127)

## SPI Device ID

NuttX auto-assigns 0x40000 as the SPI Device ID for ST7789 Display. [(See this)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/include/nuttx/spi/spi.h#L459)

We assigned the other SPI Device IDs ourselves.

Device ID -1 is meant as a fallthrough to catch all SPI Devices that don't match the Device IDs. This also works for simple SPI setups where the Device ID is not needed.

## Swap MISO / MOSI

According to [BL602 Reference Manual](https://github.com/bouffalolab/bl_docs/blob/main/BL602_RM/en/BL602_BL604_RM_1.2_en.pdf) (Table 3.1 "Pin Description", Page 26)...

-   GPIO 13 is MOSI

-   GPIO 0 is MISO

But due to a BL602 SPI quirk we need to Swap MISO and MOSI to get this behaviour. That's why the "Swap MISO / MOSI" column is marked "Yes" for SX1262 Transceiver and SPI Flash.

ST7789 Display Controller is wired differently...

-   ST7789 receives SPI Data on GPIO 0

-   ST7789 Data / Command Pin is connected on GPIO 13

    (High for ST7789 Data, Low for ST7789 Commands)

The direction of SPI Data is flipped for ST7789. That's why the "Swap MISO / MOSI" column is marked "No" for ST7789 Display Controller.

## SPI Device Table

We represent the above SPI Device Table in NuttX as a flat int array...

| SPI Device | Device ID | Swap MISO/MOSI | Chip Select | 
| :--------- | :-------: | :------------: | :---------: |
| _ST7789 Display_     | 0x40000 | 0 | 20
| _SX1262 Transceiver_ | 1       | 1 | 15
| _SPI Flash_          | 2       | 1 | 14
| _(Default Device)_   | -1      | 1 | 8

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L112-L133)

Here's the source code...

```c
#ifdef CONFIG_BL602_SPI0
/* SPI Device Table: SPI Device ID, Swap MISO/MOSI, Chip Select */

static const int32_t bl602_spi_device_table[] =
{
#ifdef BOARD_LCD_DEVID  /* ST7789 Display */
  BOARD_LCD_DEVID, BOARD_LCD_SWAP, BOARD_LCD_CS,
#endif  /* BOARD_LCD_DEVID */

#ifdef BOARD_SX1262_DEVID  /* LoRa SX1262 */
  BOARD_SX1262_DEVID, BOARD_SX1262_SWAP, BOARD_SX1262_CS,
#endif  /* BOARD_SX1262_DEVID */

#ifdef BOARD_FLASH_DEVID  /* SPI Flash */
  BOARD_FLASH_DEVID, BOARD_FLASH_SWAP, BOARD_FLASH_CS,
#endif  /* BOARD_FLASH_DEVID */

  /* Must end with Default SPI Device */

  -1, 1, BOARD_SPI_CS,  /* Swap MISO/MOSI */
};
#endif  /* CONFIG_BL602_SPI0 */
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L112-L133)

The columns of the SPI Device Table...

```c
/* Columns in the SPI Device Table */

#define DEVID_COL 0  /* SPI Device ID */
#define SWAP_COL  1  /* 1 if MISO/MOSI should be swapped, else 0 */
#define CS_COL    2  /* SPI Chip Select Pin */
#define NUM_COLS  3  /* Number of columns in SPI Device Table */
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/include/board.h#L36-L41)

Here are the functions for accessing the SPI Device Table...

-   [bl602_spi_get_device](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L210-L239): Lookup a device in the SPI Device Table

-   [bl602_spi_deselect_devices](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L178-L208): Deselect all devices in the SPI Device Table

-   [bl602_spi_validate_devices](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/src/bl602_bringup.c#L140-L176): Validate the devices in the SPI Device Table

## Pin Definitions

TODO: Pin Definitions for PineDio Stack

```c
/* SPI for PineDio Stack: Chip Select (unused), MOSI, MISO, SCK */

#define BOARD_SPI_CS   (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN8)  /* Unused */
#define BOARD_SPI_MOSI (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN13)
#define BOARD_SPI_MISO (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN0)
#define BOARD_SPI_CLK  (GPIO_INPUT | GPIO_PULLUP | GPIO_FUNC_SPI | GPIO_PIN11)

#ifdef CONFIG_LCD_ST7789
/* ST7789 for PineDio Stack: Chip Select, Reset and Backlight */

#define BOARD_LCD_DEVID SPIDEV_DISPLAY(0)  /* SPI Device ID: 0x40000 */
#define BOARD_LCD_SWAP  0    /* Don't swap MISO/MOSI */
#define BOARD_LCD_BL_INVERT  /* Backlight is active when Low */
#define BOARD_LCD_CS  (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN20)
#define BOARD_LCD_RST (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN3)
#define BOARD_LCD_BL  (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN21)
#endif  /* CONFIG_LCD_ST7789 */

/* SX1262 for PineDio Stack: Chip Select */

#define BOARD_SX1262_DEVID 1  /* SPI Device ID */
#define BOARD_SX1262_SWAP  1  /* Swap MISO/MOSI */
#define BOARD_SX1262_CS (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN15)

/* SPI Flash for PineDio Stack: Chip Select */

#define BOARD_FLASH_DEVID 2  /* SPI Device ID */
#define BOARD_FLASH_SWAP  1  /* Swap MISO/MOSI */
#define BOARD_FLASH_CS (GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO | GPIO_PIN14)
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/boards/risc-v/bl602/bl602evb/include/board.h#L99-L128)

## Select / Deselect SPI Device

TODO: This is how we select and deselect each SPI Device, by looking up the SPI Device Table...

```c
static void bl602_spi_select(struct spi_dev_s *dev, uint32_t devid,
                             bool selected)
{
  const int32_t *spidev;

  spiinfo("devid: %lu, CS: %s\n", devid, selected ? "select" : "free");

  /* get device from SPI Device Table */

  spidev = bl602_spi_get_device(devid);
  DEBUGASSERT(spidev != NULL);

  /* swap MISO and MOSI if needed */

  if (selected)
    {
      bl602_swap_spi_0_mosi_with_miso(spidev[SWAP_COL]);
    }

  /* set Chip Select */

  bl602_gpiowrite(spidev[CS_COL], !selected);

#ifdef CONFIG_SPI_CMDDATA
  /* revert MISO and MOSI from GPIO Pins to SPI Pins */

  if (!selected)
    {
      bl602_configgpio(BOARD_SPI_MISO);
      bl602_configgpio(BOARD_SPI_MOSI);
    }
#endif
}
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/arch/risc-v/src/bl602/bl602_spi.c#L439-L471)

`bl602_spi_select` is called after locking the SPI Bus.

## SPI Command / Data

TODO: We flip the ST7789 Data / Command pin depending on MISO / MOSI Swap...

```c
#ifdef CONFIG_SPI_CMDDATA
static int bl602_spi_cmddata(struct spi_dev_s *dev,
                              uint32_t devid, bool cmd)
{
  spiinfo("devid: %" PRIu32 " CMD: %s\n", devid, cmd ? "command" :
          "data");

  if (devid == SPIDEV_DISPLAY(0))
    {
      const int32_t *spidev;
      gpio_pinset_t dc;
      gpio_pinset_t gpio;
      int ret;

      /* get device from SPI Device Table */

      spidev = bl602_spi_get_device(devid);
      DEBUGASSERT(spidev != NULL);

      /* if MISO/MOSI are swapped, DC is MISO, else MOSI */

      dc = spidev[SWAP_COL] ? BOARD_SPI_MISO : BOARD_SPI_MOSI;

      /* reconfigure DC from SPI Pin to GPIO Pin */

      gpio = (dc & GPIO_PIN_MASK)
             | GPIO_OUTPUT | GPIO_PULLUP | GPIO_FUNC_SWGPIO;
      ret = bl602_configgpio(gpio);
      if (ret < 0)
        {
          spierr("Failed to configure MISO as GPIO\n");
          DEBUGPANIC();

          return ret;
        }

      /* set DC to high (data) or low (command) */

      bl602_gpiowrite(gpio, !cmd);

      return OK;
    }

  spierr("SPI cmddata not supported\n");
  DEBUGPANIC();

  return -ENODEV;
}
#endif
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/arch/risc-v/src/bl602/bl602_spi.c#L726-L774)

## Deselect All SPI Devices

TODO: We deselect all SPI Devices at startup...

```c
static void bl602_spi_init(struct spi_dev_s *dev)
{
  struct bl602_spi_priv_s *priv = (struct bl602_spi_priv_s *)dev;
  const struct bl602_spi_config_s *config = priv->config;

  /* Initialize the SPI semaphore that enforces mutually exclusive access */

  nxsem_init(&priv->exclsem, 0, 1);

  bl602_configgpio(BOARD_SPI_CS);
  bl602_configgpio(BOARD_SPI_MOSI);
  bl602_configgpio(BOARD_SPI_MISO);
  bl602_configgpio(BOARD_SPI_CLK);

  /* set master mode */

  bl602_set_spi_0_act_mode_sel(1);

  /* swap MOSI with MISO to be consistent with BL602 Reference Manual */

  bl602_swap_spi_0_mosi_with_miso(1);

  /* spi cfg  reg:
   * cr_spi_deg_en 1
   * cr_spi_m_cont_en 0
   * cr_spi_byte_inv 0
   * cr_spi_bit_inv 0
   */

  modifyreg32(BL602_SPI_CFG, SPI_CFG_CR_M_CONT_EN
              | SPI_CFG_CR_BYTE_INV | SPI_CFG_CR_BIT_INV,
              SPI_CFG_CR_DEG_EN);

  /* disable rx ignore */

  modifyreg32(BL602_SPI_CFG, SPI_CFG_CR_RXD_IGNR_EN, 0);

  bl602_spi_setfrequency(dev, config->clk_freq);
  bl602_spi_setbits(dev, 8);
  bl602_spi_setmode(dev, config->mode);

  /* spi fifo clear */

  modifyreg32(BL602_SPI_FIFO_CFG_0, SPI_FIFO_CFG_0_RX_CLR
              | SPI_FIFO_CFG_0_TX_CLR, 0);

  /* deselect all spi devices */

  bl602_spi_deselect_devices();
}
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/arch/risc-v/src/bl602/bl602_spi.c#L1191-L1240)

# ST7789 Display

TODO: SPI Mode depends on MISO / MOSI Swap

```c
#ifdef CONFIG_BL602_SPI0
#include "../boards/risc-v/bl602/bl602evb/include/board.h"
#endif  /* CONFIG_BL602_SPI0 */

#ifdef CONFIG_LCD_ST7789

/****************************************************************************
 * Pre-processor Definitions
 ****************************************************************************/

/* Verify that all configuration requirements have been met */

#ifdef CONFIG_BL602_SPI0
#  if defined(BOARD_LCD_SWAP) && BOARD_LCD_SWAP == 0  /* If MISO/MOSI not swapped... */
#    warning Using SPI Mode 1 for ST7789 on BL602 (MISO/MOSI not swapped)
#    define CONFIG_LCD_ST7789_SPIMODE SPIDEV_MODE1  /* SPI Mode 1: Workaround for BL602 */
#  else
#    warning Using SPI Mode 3 for ST7789 on BL602 (MISO/MOSI swapped)
#    define CONFIG_LCD_ST7789_SPIMODE SPIDEV_MODE3  /* SPI Mode 3: Workaround for BL602 */
#  endif /* BOARD_LCD_SWAP */
#else
#  ifndef CONFIG_LCD_ST7789_SPIMODE
#    define CONFIG_LCD_ST7789_SPIMODE SPIDEV_MODE0
#  endif /* CONFIG_LCD_ST7789_SPIMODE */
#endif   /* CONFIG_BL602_SPI0 */
```

[(Source)](https://github.com/lupyuen/incubator-nuttx/blob/pinedio/drivers/lcd/st7789.c#L42-L66)

# SX1262 LoRa Transceiver

TODO

# SPI Flash

TODO

# Touch Panel

TODO

# Push Button

TODO

# Accelerometer

TODO

# GPS

TODO

# Output Log

TODO

```text
gpio_pin_register: Registering /dev/gpio0
gpio_pin_register: Registering /dev/gpio1
gpint_enable: Disable the interrupt
gpio_pin_register: Registering /dev/gpio2
bl602_gpio_set_intmod: ****gpio_pin=115, int_ctlmod=1, int_trgmod=0
spi_test_driver_register: devpath=/dev/spitest0, spidev=0
st7789_sleep: sleep: 0
st7789_sendcmd: cmd: 0x11
st7789_sendcmd: OK
st7789_bpp: bpp: 16
st7789_sendcmd: cmd: 0x3a
st7789_sendcmd: OK
st7789_setorientation:
st7789_sendcmd: cmd: 0x36
st7789_sendcmd: OK
st7789_display: on: 1
st7789_sendcmd: cmd: 0x29
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x21
st7789_sendcmd: OK
st7789_fill: color: 0xaaaa
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
board_lcd_getdev: SPI port 0 bound to LCD 0
st7789_getplaneinfo: planeno: 0 bpp: 16

NuttShell (NSH) NuttX-10.2.0-RC0
nsh>
nsh> ?
help usage:  help [-v] [<cmd>]

  ?      cat    help   ls     uname

Builtin Apps:
  tinycbor_test            spi_test                 nsh
  lorawan_test             timer                    sensortest
  sx1262_test              bl602_adc_test           ikea_air_quality_sensor
  bas                      spi_test2                gpio
  sh                       getprime
  lvgldemo                 hello
nsh> spi_test2
spi_test_driver_open:
gpout_write: Writing 0
spi_test_driver_write: buflen=2
spi_test_driver_configspi:
spi_test_driver_read: buflen=256
gpout_write: Writing 1
Get Status: received
  a2 22
SX1262 Status is 2
gpout_write: Writing 0
spi_test_driver_write: buflen=5
spi_test_driver_configspi:
spi_test_driver_read: buflen=256
gpout_write: Writing 1
Read Register 8: received
  a2 a2 a2 a2 80
SX1262 Register 8 is 0x80
spi_test_driver_close:
nsh> lvgldemo
fbdev_init: Failed to open /dev/fb0: 2
st7789_getvideoinfo: fmt: 11 xres: 240 yres: 240 nplanes: 1
lcddev_init: VideoInfo:
        fmt: 11
        xres: 240
        yres: 240
        nplanes: 1
lcddev_init: PlaneInfo (plane 0):
        bpp: 16
st7789_putarea: row_start: 0 row_end: 19 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 20 row_end: 39 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 40 row_end:59 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 60 row_end: 79 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 80 row_end: 99 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 100 row_end: 119 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 120 row_end: 139 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 140 row_end: 159 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 160 row_end: 179 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 180 row_end: 199 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 200 row_end: 219 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
st7789_putarea: row_start: 220 row_end: 239 col_start: 0 col_end: 239
st7789_sendcmd: cmd: 0x2b
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2a
st7789_sendcmd: OK
st7789_sendcmd: cmd: 0x2c
st7789_sendcmd: OK
monitor_cb: 57600 px refreshed in 1110 ms
```
