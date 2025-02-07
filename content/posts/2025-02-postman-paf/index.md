---
author: "Nathan Morris"
title: "Postman-Paf-js"
description: "Open Source library to apply Royal Mail Rules & Exceptions to PAF (Postcode Address File) addresses when converting to a printable format."
draft: false
date: 2025-02-06
tags: ["JavaScript"]
categories: ["JavaScript"]
ShowToc: true
TocOpen: true
---

## Postman-Paf-js

We've recently released a library called [postman-paf-js](https://github.com/dvla/postman-paf-js) that we use to apply Royal Mail rules & exceptions to PAF (Postcode Address File) addresses when converting to a printable format. We also have a [ruby implementation](https://github.com/dvla/postman-paf).

## Why is Postman Paf required?

To put it simply, UK addresses are more complicated than they might seem. But aren't addresses just a few set lines that appear on your letters? Unfortunately, it's not that simple! Most addresses in the UK are stored in a database called the Postal Address File (PAF), which is owned and managed by the 
Royal Mail. This database includes over 30 million UK postal delivery addresses and 1.8 million postcodes. However, these addresses are not stored in the familiar format you 
see on your envelope. Instead, they are stored in a specific data structure, as shown
[here](https://www.poweredbypaf.com/wp-content/uploads/2017/07/Latest-Programmers_guide_Edition-7-Version-6-1.pdf#page=12)


Example of a PAF address
 ```JSON
 {
            "thoroughfareName": "NORTH EAST SECTOR",
            "doubleDependentLocality": "BOURNEMOUTH INTERNATIONAL AIRPORT",
            "postcode": "BH23 6NE",
            "dependentLocality": "HURN",
            "postTown": "CHRISTCHURCH",
            "buildingName": "A B P 21",
            "organisationName": "BREATHE SAFETY LTD"
 }
```

This data is structured very differently from the address format that appears on your envelope. Before these PAF addresses (which we at the DVLA call "Structured Addresses")
can be used for postal mail, they must be converted into a format recognized by the Royal Mail. We refer to addresses in this format as "Unstructured Addresses."


## Rules are rules

To convert addresses from a Structured format to an Unstructured format, certain rules must be applied to each element of the address to ensure it is ordered correctly.
The Royal Mail has provided a developer guide detailing how to correctly perform this conversion, which includes several rules and exceptions that need to be followed 
in the correct order.

There are 4 ordering notes, 7 rules, and 4 exceptions that must be applied to a PAF address, all in the correct order, to ensure it is converted according to the Royal 
Mail's guidelines.

## Conversion example step by step

An example of the process followed on converting a Structured address to an unstructured address can be found below:
 ```JSON
    {
        "structuredAddress" : {
                    "buildingName": "ENDEAVOUR QUAY",
                    "subBuildingName": "THE DESIGN OFFICE",
                    "thoroughfareName": "MUMBY ROAD",
                    "postcode": "PO12 1AH",
                    "postTown": "GOSPORT",
            }
        }
```

### Step 1 - Order premises elements

Rule 6 applies Address has a sub building name and building name.

Check the format of Sub Building Name. If an Exception Rule applies, the Sub Building Name should appear on the same line as, and before, the Building Name
Check format of Building Name, If an Exception Rule applies, the Building Name should appear at the beginning of the first Thoroughfare line, or the first Locality
line if there is no Thoroughfare information.

In this case none of these exceptions apply.

Otherwise, the Sub Building Name should appear on a line preceding the Building Name, Thoroughfare and Locality information.
 
 ```JSON
    {
        "line1": "THE DESIGN OFFICE",
        "line2": "ENDEAVOUR QUAY"
    }
```

### Step 2 - Order Thoroughfare/locality

```JSON
    {
        "line1": "THE DESIGN OFFICE",
        "line2": "ENDEAVOUR QUAY",
        "line3": "MUMBY ROAD"
    }
```

### Step 3 - Add in post town and postcode

```JSON
    {
        "line1": "THE DESIGN OFFICE",
        "line2": "ENDEAVOUR QUAY",
        "line3": "MUMBY ROAD",
        "line5": "GOSPORT",
        "postcode": "PO12 1AH"
    }
```


## How we did it

Originally, the address converter library was written in Java and forked from an existing library: [paf-address-format](https://github.com/steinfletcher/paf-address-format). However, this library was relatively old (last updated 5 years ago) and did not conform to all the up-to-date rules and exceptions outlined in the latest Royal Mail Developers Guide.

To ensure the library could correctly handle every rule and exception, the DVLA team painstakingly tested each rule and exception individually. This process involved using the notes and examples from the Royal Mail Developer's Guide, testing sets of PAF addresses from a PAF dataset, and comparing results against the Royal Mail Postcode Finder. After the conversion library was manually tested, a comprehensive test pack was created consisting of 92 tests and 1,380 assertions to thoroughly verify each rule and exception.

Since JavaScript is widely used across the DVLA in applications requiring structured addresses to be converted, the team decided to undertake the task of creating a JavaScript version of the original Java library. The code was ported from Java to JavaScript and then thoroughly tested again using the same tests and tools developed during the improvements made to the Java library. The JavaScript library has since been posted publicly on GitHub to provide access to the developer community.

We currently use the library within our internal Addressing services where it is helping us handle millions of addresses per year UK driver and vehicle transactions.

## How to use the library

The library is simple to use, the convertStructuredToUnstructured function needs to be imported from the library then simply pass in a structured address.

```javascript
const { convertStructuredToUnstructured } = require('postman-paf-js');

const structuredAddressToConvert = {
    buildingNumber: '1',
    thoroughfareName: 'Example Street',
    postTown: 'City',
    postcode: 'AA1 1AA',
};

const convertedUnstructuredAddress = convertStructuredToUnstructured(structuredAddressToConvert);

```

The function will then return an unstructured (Address with up to 5 lines and a postcode ) in the royal mail format

```javascript
convertedUnstructuredAddress = {
    line1: '1 Example Street',
    line5: 'City',
    postcode: 'AA1 1AA',
};
```

