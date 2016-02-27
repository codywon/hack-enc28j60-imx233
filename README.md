This is a guide on how to make ENC28J60 SPI ethernet controller module work
together with imx233 Olinuxino Mini board. 

![Photo](/photo.jpg)

Drop patches into target/linux/mxs/patches-X.X/ folder to have them applied to
linux kernel for this target. You do not need to add any code as described
below if you use the patches for OpenWRT. Below is a description of what the
patches do. 

Do `make menuconfig` and select your board. I choose imx23 mini in this case. 

	CONFIG_ENC28J60=m
	CONFIG_ENC28J60_WRITEVERIFY=m
	CONFIG_SPI_MXS=m

To instruct the kernel to load correct driver for module you connect to the spi
port on imx233, add device tree entry for the new spi device: 

ssp1: ssp@80034000 {
	#address-cells = <1>;
	#size-cells = <0>;
	compatible = "fsl,imx23-spi";
	pinctrl-names = "default";
	pinctrl-0 = <&spi2_pins_a>;
	status = "okay";

	eth1: enc28j60@0 {
		reg = <0>;
		compatible = "microchip,enc28j60";
		spi-max-frequency = <1000000>;
		interrupt-parent = <&gpio0>;
		interrupts = <2 0x02>;
	};
}

We set the ethernet interface to be on gpio2 which resides on chip0. You should
set this to whatever pin you plan to use for this interrupt. The driver will
use this interrupt for both transmission and reception so if you don't get this
part working then you will no be able to neither send nor receive packets (and
your interface will always appear in NO-CARRIER state). 

Note: 0x02 after the GPIO pin in ineterrupt definition means falling edge trigger. 

If your DTS changes worked and you can see the dts entry somewhere under
/proc/device-tree then everything should work now. 

Check for enc28j60 being loaded using lsmod too.  

```bash
# dmesg | grep enc
[    1.370000] enc28j60 spi1.0: enc28j60 Ethernet driver 1.01 loaded
[    1.390000] net eth0: enc28j60 driver registered
# uci set network.lan.ipaddr=192.168.2.1
# uci commit 
# ubus call network reload 
...

$ ping 192.168.2.1
PING 192.168.2.1 (192.168.2.1) 56(84) bytes of data.
64 bytes from 192.168.2.1: icmp_seq=1 ttl=64 time=7.69 ms
64 bytes from 192.168.2.1: icmp_seq=2 ttl=64 time=4.25 ms
^C

$ iperf3 -c 192.168.2.1
Connecting to host 192.168.2.1, port 5201
[  4] local 192.168.2.100 port 54947 connected to 192.168.2.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   150 KBytes  1.23 Mbits/sec   22   9.90 KBytes       
[  4]   1.00-2.00   sec  87.7 KBytes   718 Kbits/sec    9   2.83 KBytes       
[  4]   2.00-3.00   sec  63.6 KBytes   521 Kbits/sec    3   8.48 KBytes       
[  4]   3.00-4.00   sec  67.9 KBytes   556 Kbits/sec    8   5.66 KBytes       
[  4]   4.00-5.00   sec  63.6 KBytes   521 Kbits/sec    5   2.83 KBytes       
[  4]   5.00-6.00   sec  63.6 KBytes   521 Kbits/sec    7   2.83 KBytes       
[  4]   6.00-7.00   sec  63.6 KBytes   521 Kbits/sec    8   4.24 KBytes       
[  4]   7.00-8.00   sec  67.9 KBytes   556 Kbits/sec    4   7.07 KBytes       
[  4]   8.00-9.00   sec  66.5 KBytes   544 Kbits/sec    8   2.83 KBytes       
[  4]   9.00-10.00  sec  63.6 KBytes   521 Kbits/sec    4   2.83 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec   758 KBytes   621 Kbits/sec   78             sender
[  4]   0.00-10.00  sec   631 KBytes   517 Kbits/sec                  receiver

```

Modifications done to the original driver: 

This is necessary to get the driver to work with imx SPI controller (half
duplex). Original function does not work. 

```c
static int
spi_read_buf(struct enc28j60_net *priv, int len, u8 *data)
{
  static int nbmsg=0;
  nbmsg++;
    u8 *rx_buf = priv->spi_transfer_buf + 4;
    u8 *tx2_buf=priv->spi_transfer_buf+2;
    u8 *tx_buf = priv->spi_transfer_buf;
    struct spi_transfer t = {
        .tx_buf = tx_buf,
        .rx_buf = NULL,
        .len = SPI_OPLEN,
        .cs_change = 0,
    };

    struct spi_transfer r = {
        .tx_buf = NULL,
        .rx_buf = rx_buf,
        .len = len,
        .cs_change = 1,
    };

    struct spi_transfer t2 = {
        .tx_buf = tx2_buf,
        .rx_buf = NULL,
        .len = SPI_OPLEN,
        .cs_change = 0,
    };

    struct spi_message msg;
    int ret;

    tx_buf[0] = ENC28J60_READ_BUF_MEM;
    tx_buf[1] = tx_buf[2] = tx_buf[3] = 0;  // don't care
    tx2_buf[0]= ENC28J60_MSG_DEFAULT;
    spi_message_init(&msg);
    spi_message_add_tail(&t, &msg);
    spi_message_add_tail(&r, &msg);
    spi_message_add_tail(&t2, &msg);
    ret = spi_sync(priv->spi, &msg);
    if (ret == 0) {
        memcpy(data, rx_buf, len);
        ret = msg.status;
    }
    if (ret && netif_msg_drv(priv))
    {
        printk(DRV_NAME ": %s() failed: ret = %d\n",
            __func__, ret);
    }

    return ret;
}
```


