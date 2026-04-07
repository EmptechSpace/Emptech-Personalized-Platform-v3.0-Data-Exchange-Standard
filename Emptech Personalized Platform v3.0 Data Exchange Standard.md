# Emptech Personalized Platform v3.0 Data Exchange Standard

**Version**: V1.0

**Release Date**: 2026-04-07

---

## 1. Overview

This standard defines the data exchange format for personalization between a Business Management System and the Card Personalization System. Before personalization, the business system generates XML data files according to this specification, which are then manually imported by operators into the personalization system.

---

## 2. Data Exchange Specification

### 2.1 File Naming

- **Format**: `<CardType>_<BatchNumber>.xml`
- **Length**: ≤30 characters (excluding extension)
- **Example**: `IDCard_202602120001.xml`

### 2.2 Exchange Method

- **File Format**: XML
- **Encoding**: UTF-8
- **Batch Capacity**: Each XML file contains one batch of personalization data, with a maximum of 200 records per file.

---

## 3. XML Data Format

### 3.1 Root Element Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<DataPreparation Lcid="String" MCESTransform="@Style_asby.xsl">
  <Job Name="BatchName">
    <JobConfigurationCustomerName/>
    <Cards>
      <Card>
        <TxtLines>
          <TxtLine Nr="FieldName" Format="Text">value</TxtLine>
        </TxtLines>
      </Card>
    </Cards>
  </Job>
</DataPreparation>
```

### 3.2 Field Definitions

|                  |                 |        |            |          |                                            |
| ---------------- | --------------- | ------ | ---------- | -------- | ------------------------------------------ |
| Field Name       | Nr Attribute    | Type   | Max Length | Required | Description                                |
| Card Type        | CardType        | String | 255        | Yes      | CITIZEN / LEGAL / REFUGEE                  |
| Card ID          | IDNumber        | String | 255        | Yes      | e.g., 19650608-91180-0001-29               |
| Surname          | Surname         | String | 255        | Yes      | Family name                                |
| Given Name       | GivenName       | String | 255        | Yes      | First name                                 |
| Date of Birth    | DateOfBirth     | String | 255        | Yes      | Format: DD.MMM.YYYY (e.g., 16.MAY.1964)    |
| Sex              | Sex             | String | 255        | Yes      | F / M                                      |
| Nationality      | Nationality     | String | 255        | Yes      | e.g., TANZANIAN                            |
| Resident Status  | ResidentStatus  | String | 255        | Yes      | e.g., Stateless Person (for refugee cards) |
| Holder Signature | HolderSignature | String | 255        | Yes      | Path to signature image                    |
| Portrait Photo   | HeadImage       | String | 255        | Yes      | Path to photo                              |
| Issue Date       | IssueDate       | String | 255        | Yes      | Format: DD.MMM.YYYY (e.g., 09.MAY.2024)    |
| Expiry Date      | ExpiryDate      | String | 255        | Yes      | Format: DD.MMM.YYYY (e.g., 09.MAY.2029)    |
| Issue Place      | IssuePlace      | String | 255        | Yes      | e.g., DAR ES SALAAM                        |
| Document Number  | DocumentNumber  | String | 255        | Yes      | e.g., 123456789                            |
| QR Code          | QRCode          | String | 255        | Yes      | QR code content                            |
| MRZ Line 1       | MRZ1            | String | 31         | Yes      | ICAO standard                              |
| MRZ Line 2       | MRZ2            | String | 31         | Yes      | ICAO standard                              |
| MRZ Line 3       | MRZ3            | String | 31         | Yes      | ICAO standard                              |
| Chip Config      | ChipConfig      | String | 255        | No       | Path to JSON config file                   |
| Chip File        | ChipFile        | String | 255        | No       | Path to DAT file                           |

### 3.3 Complete XML Example

```xml
<?xml version="1.0" encoding="utf-8"?>
<DataPreparation Lcid="String" MCESTransform="@Style_asby.xsl">
  <Job Name="IDCrad201601120001">
    <JobConfigurationCustomerName/>
    <Cards>
      <Card>
        <TxtLines>
          <TxtLine Nr="CardType">CITIZEN</TxtLine>
          <TxtLine Nr="IDNumber">19650608-91180-0001-29</TxtLine>
          <TxtLine Nr="Surname" Format="Text">KAJI</TxtLine>
          <TxtLine Nr="GivenName" Format="Text">ZHAN SAN</TxtLine>
          <TxtLine Nr="DateOfBirth" Format="Text">16.MAY.1964</TxtLine>
          <TxtLine Nr="Sex" Format="Text">F</TxtLine>
          <TxtLine Nr="Nationality" Format="Text">TANZANIAN</TxtLine>
          <TxtLine Nr="HolderSignature" Format="Text">D:/data/HolderSignature1.jpg</TxtLine>
          <TxtLine Nr="HeadImage" Format="Text">D:/data/HeadPortrait1.jpg</TxtLine>
          <TxtLine Nr="IssueDate" Format="Text">09.MAY.2024</TxtLine>
          <TxtLine Nr="ExpiryDate" Format="Text">09.MAY.2029</TxtLine>
          <TxtLine Nr="IssuePlace" Format="Text">DAR ES SALA</TxtLine>
          <TxtLine Nr="DocumentNumber" Format="Text">123456789</TxtLine>
          <TxtLine Nr="QRCode" Format="Text">1234567890000123414411</TxtLine>
          <TxtLine Nr="MRZ1" Format="Text">I<TZA20010627<<<<<<<<<<<<<<<<<</TxtLine>
          <TxtLine Nr="MRZ2" Format="Text">6506087M3505078TZA<<<<<<<<<<<<<</TxtLine>
          <TxtLine Nr="MRZ3" Format="Text">SAID<<LAILA<NUHU<<<<<<<<<<<<<<<<</TxtLine>
          <TxtLine Nr="ChipFile" Format="Text">D:/data/ChipFile.dat</TxtLine>
        </TxtLines>
      </Card>
    </Cards>
  </Job>
</DataPreparation>
```
