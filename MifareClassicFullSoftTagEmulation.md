# Introduction #

**“Mifare Classic Full SoftTag Emulation”** = Software emulation (either in a PC, or a custom hardware programmable controller) of a physical Mifare Classic tag/device based on a dumped tag data, including:
  * UID/Manufacturer Block information
  * All sectors data, keys, AC bits, etc.
  * Authentication and encryption

# Mifare Classic  100% SoftTag Emulation – using ACR122U #

  * Seems like impossible because of not very accurate timings, USB timing delays, slow Crapto1 implementation
  * Check this link for a small discussion: http://www.libnfc.org/community/topic/113/mifare-classic-softtag-emulation/

# Mifare Classic  100% SoftTag Emulation – using Proxmark3 #

  * Seems possible, but not too promising as of now
  * Needs special firmware version to have proper timings
  * Not very sure how to optimally stick Crapto1 implementation into Proxmark3 and making sure ISO 14443 timings are preserved

# Mifare Classic  100% SoftTag Emulation – using Nokia 6131 or Nokia 6212 #

  * Some Mifare Classic with Nokia 6212 demo video:
<a href='http://www.youtube.com/watch?feature=player_embedded&v=SDQSRpS46Fo' target='_blank'><img src='http://img.youtube.com/vi/SDQSRpS46Fo/0.jpg' width='425' height=344 /></a>
  * As of now, the **most promising direction** for Mifare Classic 100% SoftTag Emulation
  * Exploit vectors:
    1. Getting around the software checks using the holes in Nokia 6131/6212 SDKs
    1. Patching the cldc11.jar in the SDK and test the emulator
    1. Patching the cldc11.jar in the Nokia 6131/6212 device and test

## Getting around the software checks using the holes in Nokia 6131/6212 SDKs ##

  * Block 0 operations and results:
```
MFStandardConnection conn = null;
String internalUrl = System.getProperty("internal.mf.url");
conn = (MFStandardConnection) Connector.open(internalUrl);

MFBlock block;
byte KAbytes[] = { (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF}; 
MFKey.KeyA KA = new MFKey.KeyA(KAbytes);

block = conn.getBlock(0);

byte block_FF[] = {
    (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, 
    (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF, (byte) 0xFF
};
block.getBlockType(); // returns  com.nokia.nfc.nxp.mfstd.MFBlock.BLOCKTYPE_MANUFACTURER == 2
if (block instanceof MFBlock) // returns true
{
}
if (block instanceof MFManufacturerBlock) // returns true
{
}
block.write(KA, block_FF, 0);

com.nokia.nfc.nxp.mfstd.MFStandardException
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFManufacturerBlockImpl.write(+8)
	at nokiatest.startApp(+249)
	at javax.microedition.midlet.MIDletProxy.startApp(+7)
	at com.nokia.mid.impl.isa.ui.MIDletManager.callStartApp(+4)
	at com.nokia.mid.impl.isa.ui.MIDletManager.activateMIDlet(+10)
	at com.nokia.mid.impl.isa.ui.MIDletManager.run(+15)
```

  * getBlock(-1) or getBlock(255) – operations and results:
```
MFStandardConnection conn = null;
String internalUrl = System.getProperty("internal.mf.url");
conn = (MFStandardConnection) Connector.open(internalUrl);

MFBlock block;
byte KAbytes[] = { (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF}; 
MFKey.KeyA KA = new MFKey.KeyA(KAbytes);

block = conn.getBlock(-1); // getBlock(256);

java.lang.IllegalArgumentException: Invalid block index
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFStandardConnectionImpl.getBlock(+21)
	at nokiatest.startApp(+157)
	at javax.microedition.midlet.MIDletProxy.startApp(+7)
	at com.nokia.mid.impl.isa.ui.MIDletManager.callStartApp(+4)
	at com.nokia.mid.impl.isa.ui.MIDletManager.activateMIDlet(+10)
	at com.nokia.mid.impl.isa.ui.MIDletManager.run(+15)
```

  * iii.	getSector(0), read(), write() offset 0/16/32 etc – **no exception**, however **no change in block0** of the “Virtual 4K” “Embedded Tag” occurs, but block1, block2 and block3 changes the data. **Is there a physical/hardware check on this case?**
```
MFStandardConnection conn = null;
String internalUrl = System.getProperty("internal.mf.url");
conn = (MFStandardConnection) Connector.open(internalUrl);

byte KAbytes[] = { (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF}; 
MFKey.KeyA KA = new MFKey.KeyA(KAbytes);

MFSector sector = conn.getSector(0);

byte offset = 0;
byte sector_bytes[] = new byte[64-offset];

sector.read(KA, sector_bytes, 0, 0, 64-offset);

// Overwrite with 0xFF all 4 block of sector0/sector1
for (int i=0; i < (com.nokia.nfc.nxp.mfstd.MFBlock.BLOCK_LEN * 4) - offset; i++)
{
    sector_bytes[i] = (byte) 0xFF;
}

sector.write(KA, sector_bytes, offset);
```

  * getSector(0) – write offset other than 0/16/32 (i.e. multiples of BLOCK\_LEN==16) etc – **exception**:
```
MFStandardConnection conn = null;
String internalUrl = System.getProperty("internal.mf.url");
conn = (MFStandardConnection) Connector.open(internalUrl);

byte KAbytes[] = { (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF}; 
MFKey.KeyA KA = new MFKey.KeyA(KAbytes);

MFSector sector = conn.getSector(0);

byte offset = 4; // skip the UID, maybe we can overwrite some other parts of the manufacturer block?
byte sector_bytes[] = new byte[64-offset];

sector.read(KA, sector_bytes, 0, 0, 64-offset);

// Overwrite with 0xFF all 4 block of sector0/sector1
for (int i=0; i < (com.nokia.nfc.nxp.mfstd.MFBlock.BLOCK_LEN * 4) - offset; i++)
{
    sector_bytes[i] = (byte) 0xFF;
}

sector.write(KA, sector_bytes, offset);

com.nokia.nfc.nxp.mfstd.MFStandardException
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFStandardConnectionImpl.write(+183)
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFSectorImpl.write(+24)
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFSectorImpl.write(+10)
	at nokiatest.startApp(+448)
	at javax.microedition.midlet.MIDletProxy.startApp(+7)
	at com.nokia.mid.impl.isa.ui.MIDletManager.callStartApp(+4)
	at com.nokia.mid.impl.isa.ui.MIDletManager.activateMIDlet(+10)
	at com.nokia.mid.impl.isa.ui.MIDletManager.run(+15)
```

  * getSector (1) – write offset 0 – **no exception**, all data written
```
MFStandardConnection conn = null;
String internalUrl = System.getProperty("internal.mf.url");
conn = (MFStandardConnection) Connector.open(internalUrl);

byte KAbytes[] = { (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF, (byte)0xFF}; 
MFKey.KeyA KA = new MFKey.KeyA(KAbytes);

MFSector sector = conn.getSector(0);

byte offset = 0;
byte sector_bytes[] = new byte[64-offset];

sector.read(KA, sector_bytes, 0, 0, 64-offset);

// Overwrite with 0xFF all 4 block of sector0/sector1
for (int i=0; i < (com.nokia.nfc.nxp.mfstd.MFBlock.BLOCK_LEN * 4) - offset; i++)
{
    sector_bytes[i] = (byte) 0xFF;
}

sector.write(KA, sector_bytes, offset);
```

  * getSector(-1) - operations and results:
```
java.lang.IllegalArgumentException: Invalid sector index
	at com.nokia.mid.impl.isa.io.protocol.external.nfc.MFStandardConnectionImpl.getSector(+21)
	at nokiatest.startApp(+347)
	at javax.microedition.midlet.MIDletProxy.startApp(+7)
	at com.nokia.mid.impl.isa.ui.MIDletManager.callStartApp(+4)
	at com.nokia.mid.impl.isa.ui.MIDletManager.activateMIDlet(+10)
	at com.nokia.mid.impl.isa.ui.MIDletManager.run(+15)
```


## Conclusions about code implementations and ways to patch/get around ##

  * _cldc11.jar/com/nokia/mid/impl/isa/io/protocol/external/nfc/MFManufacturerBlockImpl.class_ **possibly looks like**:
```
public void write(MFKey key, byte src[], int dstOffset) 
{
    throw new MFStandardException(0);
}

public void write(MFKey key, byte src[], int srcOffset, int length, int dstOffset) 
{
    throw new MFStandardException(0);
}

public void writeValue(MFKey key, MFValue newValue) 
{
    throw new MFStandardException(0);
}
```

  * _cldc11.jar/com/nokia/mid/impl/isa/io/protocol/external/nfc/MFStandardConnectionImpl.class_ **possibly looks like**:
```
public MFBlock getBlock(int blockIndex)
{
    // Some code

    if(blockIndex < 0 || blockIndex >= getBlockCount())
        throw new IllegalArgumentException("Invalid block index");

    // Some code

    /* Need the below patched out. JVM opcode equivalent is:
     * 
     * 22. iload_1
     * 23. ifne 31 (+8)
     * 26. aload_0
     * 27. invokevirtual #17 <com/nokia/mid/impl/isa/io/protocol/external/nfc/MFStandardConnectionImpl.getManufacturerBlock>
     * 30. areturn
     * 
     * Use BCEL from Apache to change bytecode of the class file.
     */
    if(blockIndex == 0)
        return manufacturerBlock; // implements the above restricted MfManufacturerBlock interface

    // Some code

    return normalBlock; // Implements the normal data block, non-restrictive MFBlock
}

public MFSector getSector(int sectorIndex)
{
    if(sectorIndex < 0 || sectorIndex >= getSectorCount())
        throw new IllegalArgumentException("Invalid sector index");
    else
        return new someMFSectorConstructor();
}

public void write(MFKey key, byte src[], int srcOffset, int length, int dstOffset)
{
    // Some code
    
    if(dstOffset % 16 != 0 || length % 16 != 0)
    {
        // Some codef
        while(someCondition)
        {
            currentBlockIndex = getCurrentBlockToWrite();
            
            if (currentBlockIndex == 0)
                throw new MFStandardException(0);
        }
    }
    
    // Some other code, where no check for block index 0 is made, so sector0's write(keya, bytes, offset_0) goes thru
}
```


## Patching the cldc11.jar in the SDK and test the emulator ##

  * Check out this book for techniques _"Covert Java: Techniques for Decompiling, Patching, and Reverse Engineering"_ (http://www.amazon.com/Covert-Java-Techniques-Decompiling-Engineering/dp/0672326388)

  * Possible approaches:
    1. Directly patch the needed class files and update them in the cldc11.jar of the SDK/emulator – not easy, but preferable
    1. Recompile needed classes and update them in cldc11.jar – seems easier, but not preferable since the decompiled sources are not 100% trustworthy, it is not very easy to compile them back and/or preserving unpatched functionality intact

  * TODO list of patching/recompilation for the SDK:
    1. _cldc11.jar/com/nokia/mid/impl/isa/io/protocol/external/nfc/MFManufacturerBlockImpl.class_ - instead of throwing an exception on write(), to call super.write()
    1. _cldc11.jar/com/nokia/mid/impl/isa/io/protocol/external/nfc/ MFStandardConnectionImpl.class_ - in getBlock() method instead of returning a MFManufacturerBlock class in case of block 0, just remove that “if() return;” statement
    1. _Unlock\_Midlet.jar_ - Is obfuscated, maybe some juicy things are there which might give some insight for exploitation, since this midlet somehow accesses at low-level the Mifare Classic and Secure Smart Card elements, resetting the keys for Mifare Classic and doing some other certificates/keys nasty things


## Patching the cldc11.jar in the Nokia 6131/6212 device and test ##

  * As far as I know from a GSM tech guy, getting to filesystem and files is not very easy – please let me know if you know how to access and/or patch/replace cdlc11.jar on a physical device (JTAG, other techniques)
  * As far as I know from a GSM tech guy, the flash is encrypted and possibly signed by Nokia, so preparing an already-patched flash (i.e. a flash memory dump with an already patched cldc11.jar and/or other files as required) is a bit of a problem (not to say pain in the arse) – please let me know if you know an exploit in the chain of trust of software booting and loading of these (wish these 6131 and 6212 had the same hype as iPhone :) )


## Open items ##

  * In case of physical unlock with Unlock\_Midlet.jar, does the UID of Mifare Classic element changes?
  * In case of software upgrade/downgrade using “PC Nokia Suite” or JTAG reflash in GSM service centers, does the UID of Mifare Classic element changes?
  * What is the hardware chip (and it’s specifications) that emulates/implements Mifare Classic and Secure Smart Card elements in Nokia 6131/Nokia 6212?


# Mifare Classic  100% SoftTag Emulation – using iCarte adapter with an iPhone #

  * Possibly is achievable using low-level UNIX programming and given iCarte is “exploitable” for this
  * TODO
    1. Need some detailed specs for iCarte hardware design, API/SDK design, etc.
    1. Need some dissected iCarte (pictures from someone else also might give some insights on implementation)
  * Exploit vectors
    1. Unknown. Anyone?

# Mifare Classic  100% SoftTag Emulation – using MiKeyCard #

  * Main project page: http://www.mikeycard.org/
  * Another **promising direction**
  * It's good that project is open-source/open-hardware project
  * It's good it has smart guys' support
  * Need a testing version of hardware to play with
  * Sort of resembling Proxmark3, though have specific usage direction

# Mifare Classic  100% SoftTag Emulation – using OpenPCD #

  * Also a promising direction
  * Not much information gathered on the progress of emulation though
  * Some demo video:
<a href='http://www.youtube.com/watch?feature=player_embedded&v=Srzf2MSCO6Y' target='_blank'><img src='http://img.youtube.com/vi/Srzf2MSCO6Y/0.jpg' width='425' height=344 /></a>