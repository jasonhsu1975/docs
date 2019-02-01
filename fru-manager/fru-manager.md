
# Ampere FRU manager

Ampere FRU Manager package is targeted to replace OpenBMC [ipmi-fru-paser](https://github.com/openbmc/ipmi-fru-parser/) package. It assists in:
- Unifying the method to access the FRU EEPROM device
- Improving some limitation of ipmi-fru-parser package
- Removing the IPMI FRU command implementation

## Limitation of current FRU handling package (ipmi-fru-parser)

- Mix many functionality into single package, including:
   - FRU EEPROM handler
     - Handle the D-Bus interface for FRU EEPROM
     - Parse FRU EEPROM data
     - Read/Write FRU EEPROM device directly
   - IPMI FRU handler
     - Handle IPMI FRU write command
     - Read/Write the EERPOM device directly
- Cannot parse the FRU EEPROM device when the Multi Record Area exists
- By somehow, the data of FRU EEPROM device is not correct (wrong FRU format), but ipmi-fru-parser cannot provide any information from D-Bus interface to notify the status and require the recovery action

## Overview about the Implementation

FRU manager package contains two individual parts; each part has different functionality.
They are:
- FRU manager part
- FRU parser part.

### FRU manager part

The package will handle the following:
- Initializing the D-Bus interface for all FRU EEPROM areas
- Providing the D-Bus methods for other packages, in order to update the data on D-Bus interface
- Providing the D-Bus methods for other packages, in order to support the FRU EEPROM read/write requests

### FRU parser part

The package will handle the following:
- Parsing the supported information for all FRU EEPROM areas
- Providing the D-Bus methods for other packages, in order to support updating new data for all "Custom Info Fields" (**note #01**)

### Communication between FRU manager part and FRU parser part

```
    ---------------           ---------     --------------        --------------
    | FRU manager |           | D-Bus |     | FRU parser |        | FRU EEPROM |
    ---------------           ---------     --------------        --------------
          |                       |                |                     |
          |  init the D-Bus intf  |                |                     |
          |  with default values  |                |                     |
          o---------------------->x                |                     |
          |                       |                | parse EEPROM data   |
          |                       |                o-------------------->x
          |       request to update D-Bus data     |                     |
          x<----------------------|----------------o                     |
          |  update D-Bus data    |                |                     |
          o---------------------->x                |                     |
          |                       |                |                     |
          |         read/write FRU EERPOM device   |                     |
          x(1)- - - - - - - - - - | - - - - - - - -|- - - - - - - - - - >x
          |                       |                |                     |

(1): main loop of FRU manager to handle:
- Read/Write FRU EEPROM data from D-Bus interface requests
- Read/Write FRU EEPROM device requests
```

## Advantages of new FRU manager package

- Only one functionality is provided, include:
   - FRU EEPROM handler
     - Handle the D-Bus interface for FRU EEPROM
     - Parse FRU EEPROM data
     - Read/Write FRU EEPROM device directly
- Can parse the FRU EEPROM device when the Multi Record Area exists
- When the data of FRU EEPROM device is not correct (wrong FRU format), it will provide the information on D-Bus interface to notify the status and require the recovery action


## Limitations of new FRU manager package

- Have not supported the "Updating the MultiRecord Info - Record fields" functionality
- Just support single FRU device (FRU ID 0)
- The "updating FRU EEPROM device" functionality requires the "system reset" or "FRU service reset" action to take the effect

## Detail about the Implementation

### FRU manager part

FRU manager part is the main part of FRU manager package. The main responsibilities of FRU manager part are:
- Handle D-Bus interface
- Handle the FRU EEPROM device

When the FRU manager part is started, it will:
- Initialize the D-Bus interface for all FRU EEPROM areas with default value.
- Register the supported methods to D-Bus interface in order to support:
  - Update new data for D-Bus interface
  - Read/write the FRU EEPROM device
- Enter the main loop to handle the incoming requests

#### D-Bus interface implementation

The FRU manager part will create below D-Bus interface for all "FRU Info Areas" with the default value for each property is "**Error! Need to update the EEPROM**".
This value will be updated by **FRU parser part** later, after **FRU parser part** parses the FRU EEPROM data and request **FRU manager part** to update the D-Bus data via supported D-Bus methods.

In case of **bad FRU EEPROM image** (**note 2**), this value will **NOT** be updated. So we'll know when the FRU EEPROM device need to be recovered.

##### FRU Board Info Area
```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.Board
Object path: /xyz/openbmc_project/inventory/fru0/board
Properties:
- Size  (Read only)
- Name  (Read only)
- PartNumber  (Read only)
- SerialNumber  (Read only)
- MfgDate  (Read only)
- Manufacturer  (Read only)
- FRUFileID  (Read only)
- CusomField1  (Read/Write)
- CusomField2  (Read/Write)
- CusomField3  (Read/Write)
- CusomField4  (Read/Write)
- CusomField5  (Read/Write)
- CusomField6  (Read/Write)
- CusomField7  (Read/Write)
- CusomField8  (Read/Write)
```
##### FRU Chassis Info Area
```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.Chassis
Object path: /xyz/openbmc_project/inventory/fru0/chassis
Properties:
- Size  (Read only)
- Type  (Read only)
- PartNumber  (Read only)
- SerialNumber  (Read only)
- SKU  (Read only)
- AssetTag  (Read only)
- CusomField1  (Read/Write)
- CusomField2  (Read/Write)
- CusomField3  (Read/Write)
- CusomField4  (Read/Write)
- CusomField5  (Read/Write)
- CusomField6  (Read/Write)
- CusomField7  (Read/Write)
- CusomField8  (Read/Write)
```
##### FRU Product Info Area
```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.Product
Object path: /xyz/openbmc_project/inventory/fru0/product
Properties:
- Size  (Read only)
- Name  (Read only)
- AssetTag (Read only)
- Version (Read only)
- PartNumber  (Read only)
- SerialNumber  (Read only)
- ModelNumbner  (Read only)
- Manufacturer  (Read only)
- SKU (Read only)
- FRUFileID  (Read only)
- CusomField1  (Read/Write)
- CusomField2  (Read/Write)
- CusomField3  (Read/Write)
- CusomField4  (Read/Write)
- CusomField5  (Read/Write)
- CusomField6  (Read/Write)
- CusomField7  (Read/Write)
- CusomField8  (Read/Write)
```
##### FRU Multi Record Info Area
```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.MultiRecord
Object path: /xyz/openbmc_project/inventory/fru0/multirecord
Properties:
- Record1  (Read Only)
- Record2  (Read Only)
- Record3  (Read Only)
- Record4  (Read Only)
- Record5  (Read Only)
- Record6  (Read Only)
- Record7  (Read Only)
- Record8  (Read Only)
```
#### D-Bus supported methods implementation
```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.FRUManager
Object path: /xyz/openbmc_project/inventory/
Supported Methods:
- Init D-Bus interface with default value
  - InitBoardAreaData
  - InitChassisAreaData
  - InitMultiRecordAreaData
  - InitProductAreaData
- Update data for D-Bus interface
  - UpdateBoardAreaData
  - UpdateChassisAreaData
  - UpdateMultiRecordAreaData
  - UpdateProductAreaData
- Update data for FRU EEPROM device
  - ReadFRUImage
  - UpdateFRUImage
```

### FRU parser part

FRU parser part is the sub-part of FRU manager package. The main responsibilities of FRU parser part are:
- Reading the FRU EEPROM device to parse the FRU data
- Handling the D-Bus request for "Updating the MultiRecord Info - Record fields" request (**note 1**)

When the FRU parser part is started, it will:
- Read the FRU EEPROM device and parse the supported data.
- If the parser processing is:
  - Completed: Request **FRU manager part** to update the data for D-Bus interface
  - **NOT** completed (bad FRU EEPROM image (**note 2**)), return the **FAILED** status and stop handling any request.
- Enter the main loop to handle the income requests

#### D-Bus supported methods implementation

```
Service name: xyz.openbmc_project.Inventory.FRU
Interface: xyz.openbmc_project.Inventory.FRU.Updater
Object path: /xyz/openbmc_project/inventory/fru0
Supported Methods:
- Update data for D-Bus interface
  - UpdateBoardCustomField
  - UpdateChassisCustomField
  - UpdateProductCustomField
- Update data for FRU EEPROM device
  - UpdateEEPROMImage
```

## Notes

1. Currently this functionality is specific for Ampere OpenBMC framework only.
   In order to providing the D-Bus supported methods to update some specific info (not whole FRU EEPROM image), such as:
- BMC MAC Address
- UUID
- Etc
2. The root-cause of "bad FRU EEPROM image" can be:
- Wrong checksum
- Wrong header format
- Data format is not correct (follow the [Platform Management FRU Information Storage Definition](https://www.intel.com/content/dam/www/public/us/en/documents/product-briefs/platform-management-fru-document-rev-1-2-feb-2013.pdf) specification)
