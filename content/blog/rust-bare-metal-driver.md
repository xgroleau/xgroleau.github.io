+++
title = "A Rust bare metal driver example"
date = 2023-01-16
+++
A good chunk of my job as a firmware dev is writing drivers and then integrate them. With such a good chunk of my time spent writing drivers and making sure that the code correspond to the data sheets specifications, I've develop a pattern when writing drivers. This is by no mean a complete guide on Rust embedded development or a guide to write your own macros.

<!-- more -->

This is by no mean a complete guide on Rust embedded development or a guide to write your own macros, nor is it a guide on **THE WAY** to write drivers. It's really just a short intro on what I personally do and it's more of a writing exercise for myself.


# The basics
Generally, 90% of device drivers are just reading/writing a register on a device via a bus like i2c/twi, spi or whatever. Then couldn't we just specify how to read and how to write to a register and just get the layout and address of each register from the data sheet? That is pretty much what we do, but most of it is manual. For the examples, we will use the [tmp117](https://www.ti.com/product/TMP117) as reference since it's simple and I know it pretty well, you can get the data sheet [here](https://www.ti.com/lit/ds/symlink/tmp117.pdf). 

So generally for a basic and low level driver would be something similar to this. Note that some unwrap and error handling is abstracted for keeping the code simple and we assume the device is configured for use.

```rs
pub struct Tmp117<B> {
   bus: B
}


impl Tmp117<B> where B: I2c<SevenBitAddress> {
    pub fn new(bus: B) -> Self {
        Self {
            bus
        }
    }
    
    /// We read the temperature register at address 0x00
    pub fn read_temp(&mut self) -> Result<TemperatureRegister, CustomError> {
        let mut buff = [0; 2];
        self.i2c.write_read(ADDRESS, &[0x00], &mut buff)?;
        let reg = u16::from_be_bytes(buff[0..2]);
        Ok(TemperatureRegister(reg))
    }

    /// We read the config register at address 0x01
    pub fn read_config(&mut self) -> Result<ConfigRegister, CustomError> {
        let mut buff = [0; 2];
        self.i2c.write_read(ADDRESS, &[0x01], &mut buff)?;
        let reg = u16::from_be_bytes(buff[0..2]);
        Ok(ConfigRegister(reg))
    }
}

/// Define a custom temperature register for easier access to fields
pub struct TemperatureRegister(u16);
impl TemperatureRegister {
    // Get the float value of the temperature
    pub fn temp(&self) -> f32 {
        self.0 as f32 * CELCIUS_CONVERSION
    }
}

/// Same but for the configuration register
pub struct ConfigRegister(u16);
impl ConfigRegister {
    pub fn high_alert(&self) -> bool {
        self.0 & 0x8000
    }

    pub fn low_alert(&self) -> bool {
        self.0 & 0x4000
    }
    
    // More methods to access each fields....
}
```


Here we see that the reading is always the same, only the conversion to the newtype and the address changes. To abstract this, we simply declare a `Register` interface that specifies the address of the register and how to convert it from an `u16`.

```rs
pub struct Tmp117<B> {
   bus: B
}


impl Tmp117<B> where B: I2c<SevenBitAddress> {
    pub fn new(bus: B) -> Self {
        Self {
            bus
        }
    }
    
    /// We read the register
    pub fn read_register<R: Register>(&mut self) -> Result<R, CustomError> {
        let mut buff = [0; 2];
        self.i2c.write_read(ADDRESS, &[R::Address], &mut buff)?;
        let reg = u16::from_be_bytes(buff[0..2]);
        Ok(R::from_u16(reg))
    }
}

pub trait Register {
    const Address: u8;

    fn from_u16(reg: u16) -> Self;
}

/// Define a custom temperature register for easier access to fields
pub struct TemperatureRegister(u16);
impl TemperatureRegister {
    // Get the float value of the temperature
    pub fn temp(&self) -> f32 {
        self.0 as f32 * CELCIUS_CONVERSION
    }
}

// Implement the Register interface
impl Register for TemperatureRegister {
    const Address: u16 = 0x00
    
    fn from_u16(reg: u16) -> Self {
        Self(u16)
    }
}


// Same here for the config register
// ... 

// This would allow use to use the interface like so:
let temp: TemperatureRegister = tmp117.read()?;
let config: ConfigurationRegister = tmp117.read()?;
```

# Macros by a thousand cuts
Now the issue here is that our implementation is pretty basic.
- Some of our registers are read only, others a write only and some are read/write
- It's cumbersome to always implement the `Register` interface
- It's **VERY** cumbersome to write all the accessors for the newtype

For the first two points, we could use [device-register](https://crates.io/crates/device-register) (which I wrote). For the last point, we could use [bilge](https://github.com/hecatia-elegua/bilge) or [modular-bitfield](https://docs.rs/modular-bitfield/latest/modular_bitfield/).
