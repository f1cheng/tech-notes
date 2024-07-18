
- Read From One Register in a Device  
-- i2c master send read command with slave register addr to ask slave, and following bits get the data from slave register.(etc power value, temp value)
-- how slave register value stored, it's slave device which writes the info(temp value) into corresponding register based on request.
-- how i2c master get slave's register addr, it is got via device tree of slave devices.  
-- https://www.ti.com/lit/an/slva704/slva704.pdf?ts=1721224584453&ref_url=https%253A%252F%252Fwww.google.com%252F  
![image](https://github.com/user-attachments/assets/e00b0e1f-44df-41c9-88b7-be42a4814dca)  


```

I2C master 读写都是master_xfer，readwrite bit来标识是读还是写，写0，读1：
比如axia---fpga i2c master---temperature slave.
==temperature unit write the tem value into own register.
https://www.ti.com/lit/an/slva704/slva704.pdf?ts=1721224584453&ref_url=https%253A%252F%252Fwww.google.com%252F


one byte data to idev->base+MST_DATA

304  static int axxia_i2c_fill_tx_fifo(struct axxia_i2c_dev *idev)
305  {
306  	struct i2c_msg *msg = idev->msg;
307  	size_t tx_fifo_avail = FIFO_SIZE - readl(idev->base + MST_TX_FIFO);
308  	int bytes_to_transfer = min(tx_fifo_avail, msg->len - idev->msg_xfrd);
309  	int ret = msg->len - idev->msg_xfrd - bytes_to_transfer;
310  
311  	while (bytes_to_transfer-- > 0)
312  		writel(msg->buf[idev->msg_xfrd++], idev->base + MST_DATA);
313  
314  	return ret;
315  }


axxia_i2c_xfer(struct i2c_adapter *adap, struct i2c_msg msgs[], int num)

632  	for (i = 0; ret == 0 && i < num; ++i)
633  		ret = axxia_i2c_xfer_msg(idev, &msgs[i], i == (num - 1));
634  


725  static const struct i2c_algorithm axxia_i2c_algo = {
726  	.master_xfer = axxia_i2c_xfer,-------------------------
727  	.functionality = axxia_i2c_func,
728  	.reg_slave = axxia_i2c_reg_slave,
729  	.unreg_slave = axxia_i2c_unreg_slave,
730  };




slave:

1  // SPDX-License-Identifier: GPL-2.0-or-later
2  /*
3   * Linux I2C core SMBus and SMBus emulation code
4   *
5   * This file contains the SMBus functions which are always included in the I2C
6   * core because they can be emulated via I2C. SMBus specific extensions
7   * (e.g. smbalert) are handled in a separate i2c-smbus module.
8   *
9   * All SMBus-related things are written by Frodo Looijaard <frodol@dds.nl>
10   * SMBus 2.0 support by Mark Studebaker <mdsxyz123@yahoo.com> and
11   * Jean Delvare <jdelvare@suse.de>
12   */
13  #include <linux/device.h>
14  #include <linux/err.h>
15  #include <linux/i2c.h>
16  #include <linux/i2c-smbus.h>
17  #include <linux/slab.h>
18  
19  #include "i2c-core.h"
20  
21  #define CREATE_TRACE_POINTS
22  #include <trace/events/smbus.h>
23  
24  
25  /* The SMBus parts */
26  
27  #define POLY    (0x1070U << 3)
28  static u8 crc8(u16 data)
29  {
30  	int i;
31  
32  	for (i = 0; i < 8; i++) {
33  		if (data & 0x8000)
34  			data = data ^ POLY;
35  		data = data << 1;
36  	}
37  	return (u8)(data >> 8);
38  }
39  
40  /**
41   * i2c_smbus_pec - Incremental CRC8 over the given input data array
42   * @crc: previous return crc8 value
43   * @p: pointer to data buffer.
44   * @count: number of bytes in data buffer.
45   *
46   * Incremental CRC8 over count bytes in the array pointed to by p
47   */
48  u8 i2c_smbus_pec(u8 crc, u8 *p, size_t count)
49  {
50  	int i;
51  
52  	for (i = 0; i < count; i++)
53  		crc = crc8((crc ^ p[i]) << 8);
54  	return crc;
55  }
56  EXPORT_SYMBOL(i2c_smbus_pec);
57  
58  /* Assume a 7-bit address, which is reasonable for SMBus */
59  static u8 i2c_smbus_msg_pec(u8 pec, struct i2c_msg *msg)
60  {
61  	/* The address will be sent first */
62  	u8 addr = i2c_8bit_addr_from_msg(msg);
63  	pec = i2c_smbus_pec(pec, &addr, 1);
64  
65  	/* The data buffer follows */
66  	return i2c_smbus_pec(pec, msg->buf, msg->len);
67  }
68  
69  /* Used for write only transactions */
70  static inline void i2c_smbus_add_pec(struct i2c_msg *msg)
71  {
72  	msg->buf[msg->len] = i2c_smbus_msg_pec(0, msg);
73  	msg->len++;
74  }
75  
76  /* Return <0 on CRC error
77     If there was a write before this read (most cases) we need to take the
78     partial CRC from the write part into account.
79     Note that this function does modify the message (we need to decrease the
80     message length to hide the CRC byte from the caller). */
81  static int i2c_smbus_check_pec(u8 cpec, struct i2c_msg *msg)
82  {
83  	u8 rpec = msg->buf[--msg->len];
84  	cpec = i2c_smbus_msg_pec(cpec, msg);
85  
86  	if (rpec != cpec) {
87  		pr_debug("Bad PEC 0x%02x vs. 0x%02x\n",
88  			rpec, cpec);
89  		return -EBADMSG;
90  	}
91  	return 0;
92  }
93  
94  /**
95   * i2c_smbus_read_byte - SMBus "receive byte" protocol
96   * @client: Handle to slave device
97   *
98   * This executes the SMBus "receive byte" protocol, returning negative errno
99   * else the byte received from the device.
100   */
101  s32 i2c_smbus_read_byte(const struct i2c_client *client)
102  {
103  	union i2c_smbus_data data;
104  	int status;
105  
106  	status = i2c_smbus_xfer(client->adapter, client->addr, client->flags,
107  				I2C_SMBUS_READ, 0,
108  				I2C_SMBUS_BYTE, &data);
109  	return (status < 0) ? status : data.byte;
110  }
111  EXPORT_SYMBOL(i2c_smbus_read_byte);
112  
113  /**
114   * i2c_smbus_write_byte - SMBus "send byte" protocol
115   * @client: Handle to slave device
116   * @value: Byte to be sent
117   *
118   * This executes the SMBus "send byte" protocol, returning negative errno
119   * else zero on success.
120   */
121  s32 i2c_smbus_write_byte(const struct i2c_client *client, u8 value)
122  {
123  	return i2c_smbus_xfer(client->adapter, client->addr, client->flags,
124  	                      I2C_SMBUS_WRITE, value, I2C_SMBUS_BYTE, NULL);
125  }
126  EXPORT_SYMBOL(i2c_smbus_write_byte);
127  
128  /**
129   * i2c_smbus_read_byte_data - SMBus "read byte" protocol
130   * @client: Handle to slave device
131   * @command: Byte interpreted by slave
132   *
133   * This executes the SMBus "read byte" protocol, returning negative errno
134   * else a data byte received from the device.
135   */
136  s32 i2c_smbus_read_byte_data(const struct i2c_client *client, u8 command)
137  {
138  	union i2c_smbus_data data;
139  	int status;
140  
141  	status = i2c_smbus_xfer(client->adapter, client->addr, client->flags,
142  				I2C_SMBUS_READ, command,
143  				I2C_SMBUS_BYTE_DATA, &data);
144  	return (status < 0) ? status : data.byte;
145  }
146  EXPORT_SYMBOL(i2c_smbus_read_byte_data);
147  


544  	res = __i2c_smbus_xfer(adapter, addr, flags, read_write,
545  			       command, protocol, data);



577  	xfer_func = adapter->algo->smbus_xfer;
578  	if (i2c_in_atomic_xfer_mode()) {
579  		if (adapter->algo->smbus_xfer_atomic)
580  			xfer_func = adapter->algo->smbus_xfer_atomic;
581  		else if (adapter->algo->master_xfer_atomic)
582  			xfer_func = NULL; /* fallback to I2C emulation */
583  	}
584  
585  	if (xfer_func) {
586  		/* Retry automatically on arbitration loss */
587  		orig_jiffies = jiffies;
588  		for (res = 0, try = 0; try <= adapter->retries; try++) {
589  			res = xfer_func(adapter, addr, flags, read_write,
590  					command, protocol, data);
591  			if (res != -EAGAIN)


321  static s32 i2c_smbus_xfer_emulated(struct i2c_adapter *adapter, u16 addr,
322  				   unsigned short flags,
323  				   char read_write, u8 command, int size,
324  				   union i2c_smbus_data *data)
325  {

467  
468  	status = __i2c_transfer(adapter, msg, nmsgs);



2106  int __i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
2107  {
2108  	unsigned long orig_jiffies;
2109  	int ret, try;
2110  
2111  	if (WARN_ON(!msgs || num < 1))
2112  		return -EINVAL;
2113  
2114  	ret = __i2c_check_suspended(adap);
2115  	if (ret)
2116  		return ret;
2117  
2118  	if (adap->quirks && i2c_check_for_quirks(adap, msgs, num))
2119  		return -EOPNOTSUPP;
2120  
2121  	/*
2122  	 * i2c_trace_msg_key gets enabled when tracepoint i2c_transfer gets
2123  	 * enabled.  This is an efficient way of keeping the for-loop from
2124  	 * being executed when not needed.
2125  	 */
2126  	if (static_branch_unlikely(&i2c_trace_msg_key)) {
2127  		int i;
2128  		for (i = 0; i < num; i++)
2129  			if (msgs[i].flags & I2C_M_RD)
2130  				trace_i2c_read(adap, &msgs[i], i);
2131  			else
2132  				trace_i2c_write(adap, &msgs[i], i);
2133  	}
2134  
2135  	/* Retry automatically on arbitration loss */
2136  	orig_jiffies = jiffies;
2137  	for (ret = 0, try = 0; try <= adap->retries; try++) {
2138  		if (i2c_in_atomic_xfer_mode() && adap->algo->master_xfer_atomic)
2139  			ret = adap->algo->master_xfer_atomic(adap, msgs, num);
2140  		else
2141  			ret = adap->algo->master_xfer(adap, msgs, num);
2142  



542   */
543  struct i2c_algorithm {
544  	/*
545  	 * If an adapter algorithm can't do I2C-level access, set master_xfer
546  	 * to NULL. If an adapter algorithm can do SMBus access, set
547  	 * smbus_xfer. If set to NULL, the SMBus protocol is simulated
548  	 * using common I2C messages.
549  	 *
550  	 * master_xfer should return the number of messages successfully
551  	 * processed, or a negative value on error
552  	 */
553  	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
554  			   int num);
555  	int (*master_xfer_atomic)(struct i2c_adapter *adap,
556  				   struct i2c_msg *msgs, int num);
557  	int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
558  			  unsigned short flags, char read_write,
559  			  u8 command, int size, union i2c_smbus_data *data);
560  	int (*smbus_xfer_atomic)(struct i2c_adapter *adap, u16 addr,
561  				 unsigned short flags, char read_write,
562  				 u8 command, int size, union i2c_smbus_data *data);
563  
564  	/* To determine what the adapter supports */
565  	u32 (*functionality)(struct i2c_adapter *adap);
566  
567  #if IS_ENABLED(CONFIG_I2C_SLAVE)
568  	int (*reg_slave)(struct i2c_client *client);
569  	int (*unreg_slave)(struct i2c_client *client);
570  #endif
571  };

```
