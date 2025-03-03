---
title: Drawing package requirements in Microsoft Azure Maps Creator
titleSuffix: Microsoft Azure Maps Creator
description: Learn about the drawing package requirements to convert your facility design files to map data
author: eriklindeman
ms.author: eriklind
ms.date: 02/17/2023
ms.topic: conceptual
ms.service: azure-maps
services: azure-maps

---

# Drawing package requirements

You can convert uploaded drawing packages into map data by using the Azure Maps [Conversion service]. This article describes the drawing package requirements for the Conversion API. To view a sample package, you can download the sample [Drawing package].

For a guide on how to prepare your drawing package, see [Conversion Drawing Package Guide].

## Prerequisites

The drawing package includes drawings saved in DWG format, which is the native file format for Autodesk's AutoCAD® software.

You can choose any CAD software to produce the drawings in the drawing package.  

The [Conversion service] converts the drawing package into map data. The Conversion service works with the AutoCAD DWG file format `AC1032`.

## Glossary of terms

For easy reference, here are some terms and definitions that are important as you read this article.

| Term  | Definition                                   |
|:------|:---------------------------------------------|
| Layer | An AutoCAD DWG layer from the drawing file.  |
| Entity| An AutoCAD DWG entity from the drawing file. |
| Xref  | A file in AutoCAD DWG file format, attached to the primary drawing as an external reference.  |
| Level | An area of a building at a set elevation. For example, the floor of a building. |
|Feature| An instance of an object produced from the Conversion service that combines a geometry with metadata information. |
|Feature classes| A common blueprint for features. For example, a *unit* is a feature class, and an *office* is a feature. |

## Drawing package structure

A drawing package is a .zip archive that contains the following files:

- DWG files in AutoCAD DWG file format.
- A _manifest.json_ file that describes the DWG files in the drawing package.

The drawing package must be zipped into a single archive file, with the .zip extension. The DWG files can be organized in any way inside the package, but the manifest file must live at the root directory of the zipped package. The next sections detail the requirements for the DWG files, manifest file, and the content of these files. To view a sample package, you can download the [sample drawing package].

## DWG file conversion process

The [Conversion service] does the following on each DWG file:

- Extracts feature classes:
  - Levels
  - Units
  - Zones
  - Openings
  - Walls
  - Vertical penetrations
- Produces a *Facility* feature.  
- Produces a minimal set of default Category features to be referenced by other features:
  - room
  - structure
  - wall
  - opening.door
  - zone
  - facility

## DWG file requirements

A single DWG file is required for each level of the facility. All data of a single level must be contained in a single DWG file.  Any external references (_xrefs_) must be bound to the parent drawing. For example, a facility with three levels will have three DWG files in the drawing package.

Each DWG file must adhere to the following requirements:

- The DWG file must define the _Exterior_ and _Unit_ layers. It can optionally define the following layers: _Wall_, _Door_, _UnitLabel_, _Zone_, and _ZoneLabel_.
- The DWG file can't contain features from multiple levels.
- The DWG file can't contain features from multiple facilities.
- The DWG must reference the same measurement system and unit of measurement as other DWG files in the drawing package.

## DWG layer requirements

Each DWG layer must adhere to the following rules:

- A layer must exclusively contain features of a single class. For example, units and walls can’t be in the same layer.
- A single class of features can be represented by multiple layers.
- Self-intersecting polygons are permitted, but are automatically repaired. When they repaired, the [Conversion service] raises a warning. It's advisable to manually inspect the repaired results, because they might not match the expected results.
- Each layer has a supported list of entity types. Any other entity types in a layer will be ignored. For example, text entities aren't supported on the wall layer.

The table below outlines the supported entity types and converted map features for each layer. If a layer contains unsupported entity types, then the [Conversion service] ignores those entities.  

| Layer | Entity types | Converted Features |
| :----- | :-------------------| :-------
| [Exterior](#exterior-layer) | Polygon, PolyLine (closed), Circle, Ellipse (closed) | Levels
| [Unit](#unit-layer) |  Polygon, PolyLine (closed), Circle, Ellipse (closed) |  Units and Vertical penetrations
| [Wall](#wall-layer)  | Polygon, PolyLine (closed), Circle, Ellipse (closed), Structures |
| [Door](#door-layer) | Polygon, PolyLine, Line, CircularArc, Circle | Openings
| [Zone](#zone-layer) | Polygon, PolyLine (closed), Circle, Ellipse (closed) | Zones
| [UnitLabel](#unitlabel-layer) | Text (single line) | Not applicable. This layer can only add properties to the unit features from the Units layer. For more information, see the [UnitLabel layer](#unitlabel-layer).
| [ZoneLabel](#zonelabel-layer) | Text (single line) | Not applicable. This layer can only add properties to zone features from the ZonesLayer. For more information, see the [ZoneLabel layer](#zonelabel-layer).

The sections below describe the requirements for each layer.

### Exterior layer

The DWG file for each level must contain a layer to define that level's perimeter. This layer is referred to as the *exterior* layer. For example, if a facility contains two levels, then it needs to have two DWG files, with an exterior layer for each file.

No matter how many entity drawings are in the exterior layer, the [resulting facility dataset](tutorial-creator-feature-stateset.md) will contain only one level feature for each DWG file. Additionally:

- Exteriors must be drawn as Polygon, PolyLine (closed), Circle, or Ellipse (closed).
- Exteriors may overlap, but are dissolved into one geometry.
- Resulting level feature must be at least 4 square meters.
- Resulting level feature must not be greater 400,000 square meters.

If the layer contains multiple overlapping PolyLines, the PolyLines are dissolved into a single Level feature. Instead, if the layer contains multiple non-overlapping PolyLines, the resulting Level feature has a multi-polygonal representation.

You can see an example of the Exterior layer as the outline layer in the [sample drawing package].

### Unit layer

The DWG file for each level defines a layer containing units. Units are navigable spaces in the building, such as offices, hallways, stairs, and elevators. If the `VerticalPenetrationCategory` property is defined, navigable units that span multiple levels, such as elevators and stairs, are converted to Vertical Penetration features. Vertical penetration features that overlap each other are assigned one `setid`.

The Units layer should adhere to the following requirements:

- Units must be drawn as Polygon, PolyLine (closed), Circle, or Ellipse (closed).
- Units must fall inside the bounds of the facility exterior perimeter.
- Units must not partially overlap.
- Units must not contain any self-intersecting geometry.

Name a unit by creating a text object in the UnitLabel layer, and then place the object inside the bounds of the unit. For more information, see the [UnitLabel layer](#unitlabel-layer).

You can see an example of the Units layer in the [sample drawing package].

### Wall layer

The DWG file for each level can contain a layer that defines the physical extents of walls, columns, and other building structure.

- Walls must be drawn as Polygon, PolyLine (closed), Circle, or Ellipse (closed).
- The wall layer or layers should only contain geometry that's interpreted as building structure.

You can see an example of the Walls layer in the [sample drawing package].

### Door layer

You can include a DWG layer that contains doors. Each door must overlap the edge of a unit from the Unit layer.

Door openings in an Azure Maps dataset are represented as a single-line segment that overlaps multiple unit boundaries. The following images show how Azure Maps converts door layer geometry into opening features in a dataset..

![Four graphics that show the steps to generate openings](./media/drawing-requirements/opening-steps.png)

### Zone layer

The DWG file for each level can contain a Zone layer that defines the physical extents of zones. A zone is a non-navigable space that can be named and rendered. Zones can span multiple levels and are grouped together using the zoneSetId property.

- Zones must be drawn as Polygon, PolyLine (closed), or Ellipse (closed).
- Zones can overlap.
- Zones can fall inside or outside the facility's exterior perimeter.

Name a zone by creating a text object in the ZoneLabel layer, and placing the text object inside the bounds of the zone. For more information, see [ZoneLabel layer](#zonelabel-layer).

You can see an example of the Zone layer in the [sample drawing package].

### UnitLabel layer

The DWG file for each level can contain a UnitLabel layer. The UnitLabel layer adds a name property to units extracted from the Unit layer. Units with a name property can have more details specified in the manifest file.

- Unit labels must be single-line text entities.
- Unit labels must fall entirely inside the bounds of their unit.
- Units must not contain multiple text entities in the UnitLabel layer.

You can see an example of the UnitLabel layer in the [sample drawing package].

### ZoneLabel layer

The DWG file for each level can contain a ZoneLabel layer. This layer adds a name property to zones extracted from the Zone layer. Zones with a name property can have more details specified in the manifest file.

- Zones labels must be single-line text entities.
- Zones labels must fall inside the bounds of their zone.
- Zones must not contain multiple text entities in the ZoneLabel layer.

You can see an example of the ZoneLabel layer in the [sample drawing package].

## Manifest file requirements

The zip folder must contain a manifest file at the root level of the directory, and the file must be named **manifest.json**. It describes the DWG files to allow the [Conversion service] to parse their content. Only the files identified by the manifest are ingested. Files that are in the zip folder, but aren't properly listed in the manifest, are ignored.

The file paths in the `buildingLevels` object of the manifest file must be relative to the root of the zip folder. The DWG file name must exactly match the name of the facility level. For example, a DWG file for the "Basement" level is "Basement.dwg." A DWG file for level 2 is named as "level_2.dwg." Use an underscore, if your level name has a space.

Although there are requirements when you use the manifest objects, not all objects are required. The following table shows the required and optional objects for version 1.1 of the [Conversion service].

>[!NOTE]
> Unless otherwise specified, all properties with a string property type allow for one thousand characters.

| Object    | Required | Description                                                  |
| :-------- | :------- | :----------------------------------------------------------- |
| `version` | true |Manifest schema version. Currently, only version 1.1 is supported.|
| `directoryInfo` | true | Outlines the facility geographic and contact information. It can also be used to outline an occupant geographic and contact information. |
| `buildingLevels` | true | Specifies the levels of the buildings and the files containing the design of the levels. |
| `georeference` | true | Contains numerical geographic information for the facility drawing. |
| `dwgLayers` | true | Lists the names of the layers, and each layer lists the names of its own features. |
| `unitProperties` | false | Can be used to insert more metadata for the unit features. |
| `zoneProperties` | false | Can be used to insert more metadata for the zone features. |

The next sections detail the requirements for each object.

### `directoryInfo`

| Property  | Type | Required | Description |
|-----------|------|----------|-------------|
| `name`    |string| true     | Name of building. |
| `streetAddress`|string|false| Address of building. |
|`unit`     |string| false    | Unit in building. |
|`locality` |string | false   | Name of a city, town, area, neighborhood, or region.|
|`adminDivisions`|JSON array of strings | false| An array containing address designations. For example: (Country, State) Use ISO 3166 country codes and ISO 3166-2 state/territory codes. |
|`postalCode`|string| false   | The mail sorting code. |
|`hoursOfOperation` |string|false| Adheres to the [OSM Opening Hours] format. |
|`phone`    |string| false   |    Phone number associated with the building. |
|`website`  |string| false   | Website associated with the building. |
|`nonPublic`|bool| false     | Flag specifying if the building is open to the public. |
|`anchorLatitude`|numeric|false | Latitude of a facility anchor (pushpin). |
|`anchorLongitude`|numeric |false | Longitude of a facility anchor (pushpin). |
|`anchorHeightAboveSeaLevel`| numeric | false | Height of the facility's ground floor above sea level, in meters. |
|`defaultLevelVerticalExtent` numeric | false | Default height (thickness) of a level of this facility to use when a level's `verticalExtent` is undefined. |

### `buildingLevels`

The `buildingLevels` object contains a JSON array of buildings levels.

| Property  | Type  | Required | Description |
|-----------|-------|----------|-------------|
|`levelName`|string |true      |Descriptive level name. For example: Floor 1, Lobby, Blue Parking, or Basement.|
|`ordinal`  |integer|true      | Determines the vertical order of levels. Every facility must have a level with ordinal 0. |
|`heightAboveFacilityAnchor`| numeric | false |Level height above the anchor in meters. |
|`verticalExtent`|numeric|false| Floor-to-ceiling height (thickness) of the level in meters. |
|`filename` |string |true |File system path of the CAD drawing for a building level. It must be relative to the root of the building's zip file. |

### `georeference`

| Property  | Type | Required | Description |
|-----------|------|----------|-------------|
|`lat`|numeric|true|Decimal representation of degrees latitude at the facility drawing's origin. The origin coordinates must be in WGS84 Web Mercator (`EPSG:3857`).|
|`lon`|numeric|true|Decimal representation of degrees longitude at the facility drawing's origin. The origin coordinates must be in WGS84 Web Mercator (`EPSG:3857`).|
|`angle`|numeric|true|The clockwise angle, in degrees, between true north and the drawing's vertical (Y) axis.|

### `dwgLayers`

| Property  | Type | Required | Description                                                   |
|-----------|------|----------|---------------------------------------------------------------|
|`exterior` |array of strings|true |Names of layers that define the exterior building profile.|
|`unit`     |array of strings|false|Names of layers that define units.                        |
|`wall`     |array of strings|false|Names of layers that define walls.                        |
|`door`     |array of strings|false|Names of layers that define doors.                        |
|`unitLabel`|array of strings|false|Names of layers that define names of units.               |
|`zone`     |array of strings|false|Names of layers that define zones.                        |
|`zoneLabel`|array of strings|false|Names of layers that define names of zones.               |

### `unitProperties`

The `unitProperties` object contains a JSON array of unit properties.

| Property  | Type | Required | Description |
|-----------|------|----------|-------------|
|`unitName`|string|true|Name of unit to associate with this `unitProperty` record. This record is only valid when a label matching `unitName` is found in the `unitLabel` layers. |
|`categoryName`|string|false|Purpose of the unit. A list of values that the provided rendering styles can make use of is available [here](https://atlas.microsoft.com/sdk/javascript/indoor/0.1/categories.json).|
|`occupants`|array of directoryInfo objects |false    |List of occupants for the unit. |
|`nameAlt`|string|false|Alternate name of the unit. |
|`nameSubtitle`|string|false|Subtitle of the unit. |
|`addressRoomNumber`|string|false|Room, unit, apartment, or suite number of the unit.|
|`verticalPenetrationCategory`|string|false| When this property is defined, the resulting feature is a vertical penetration (VRT) rather than a unit. You can use vertical penetrations to go to other vertical penetration features in the levels above or below it. Vertical penetration is a [Category](https://aka.ms/pa-indoor-spacecategories) name. If this property is defined, the `categoryName` property is overridden with `verticalPenetrationCategory`. |
|`verticalPenetrationDirection`|    string|    false    |If `verticalPenetrationCategory` is defined, optionally define the valid direction of travel. The permitted values are: `lowToHigh`, `highToLow`, `both`, and `closed`. The default value is `both`. The value is case-sensitive.|
| `nonPublic` | bool | false | Indicates if the unit is open to the public. |
| `isRoutable` | bool | false | When this property is set to `false`, you can't go to or through the unit. The default value is `true`. |
| `isOpenArea` | bool | false | Allows the navigating agent to enter the unit without the need for an opening attached to the unit. By default, this value is set to `true` for units with no openings, and `false` for units with openings. Manually setting `isOpenArea` to `false` on a unit with no openings results in a warning, because the resulting unit won't be reachable by a navigating agent.|

### `zoneProperties`

The `zoneProperties` object contains a JSON array of zone properties.

| Property  | Type | Required | Description |
|-----------|------|----------|-------------|
|zoneName |string |true |Name of zone to associate with `zoneProperty` record. This record is only valid when a label matching `zoneName` is found in the `zoneLabel` layer of the zone.  |
|categoryName| string| false |Purpose of the zone. A list of values that the provided rendering styles can make use of is available [here](https://atlas.microsoft.com/sdk/javascript/indoor/0.1/categories.json).|
|zoneNameAlt| string| false |Alternate name of the zone.  |
|zoneNameSubtitle| string | false |Subtitle of the zone. |
|zoneSetId| string | false | Set ID to establish a relationship among multiple zones so that they can be queried or selected as a group. For example, zones that span multiple levels. |

### Sample drawing package manifest

Below is the manifest file for the sample drawing package. Go to the [Sample drawing package] for Azure Maps Creator on GitHub to download the entire package.

#### Manifest file

```JSON
{
    "version": "1.1", 
    "directoryInfo": { 
        "name": "Contoso Building", 
        "streetAddress": "Contoso Way", 
        "unit": "1", 
        "locality": "Contoso eastside", 
        "postalCode": "98052", 
        "adminDivisions": [ 
            "Contoso city", 
            "Contoso state", 
            "Contoso country" 
        ], 
        "hoursOfOperation": "Mo-Fr 08:00-17:00 open", 
        "phone": "1 (425) 555-1234", 
        "website": "www.contoso.com", 
        "nonPublic": false, 
        "anchorLatitude": 47.636152, 
        "anchorLongitude": -122.132600, 
        "anchorHeightAboveSeaLevel": 1000, 
        "defaultLevelVerticalExtent": 3  
    }, 
    "buildingLevels": { 
        "levels": [ 
            { 
                "levelName": "Basement", 
                "ordinal": -1, 
                "filename": "./Basement.dwg" 
            }, { 
                "levelName": "Ground", 
                "ordinal": 0, 
                "verticalExtent": 5, 
                "filename": "./Ground.dwg" 
            }, { 
                "levelName": "Level 2", 
                "ordinal": 1, 
                "heightAboveFacilityAnchor": 3.5, 
                "filename": "./Level_2.dwg" 
            } 
        ] 
    }, 
    "georeference": { 
        "lat": 47.636152, 
        "lon": -122.132600, 
        "angle": 0 
    }, 
    "dwgLayers": { 
        "exterior": [ 
            "OUTLINE", "WINDOWS" 
        ], 
        "unit": [ 
            "UNITS" 
        ], 
        "wall": [ 
            "WALLS" 
        ], 
        "door": [ 
            "DOORS" 
        ], 
        "unitLabel": [ 
            "UNITLABELS" 
        ], 
        "zone": [ 
            "ZONES" 
        ], 
        "zoneLabel": [ 
            "ZONELABELS" 
        ] 
    }, 
    "unitProperties": [ 
        { 
            "unitName": "B01", 
            "categoryName": "room.office", 
            "occupants": [ 
                { 
                    "name": "Joe's Office", 
                    "phone": "1 (425) 555-1234" 
                } 
            ], 
            "nameAlt": "Basement01", 
            "nameSubtitle": "01", 
            "addressRoomNumber": "B01", 
            "nonPublic": true, 
            "isRoutable": true, 
            "isOpenArea": true 
        }, 
        { 
            "unitName": "B02" 
        }, 
        { 
            "unitName": "B05", 
            "categoryName": "room.office" 
        }, 
        { 
            "unitName": "STRB01", 
            "verticalPenetrationCategory": "verticalPenetration.stairs", 
            "verticalPenetrationDirection": "both" 
        }, 
        { 
            "unitName": "ELVB01", 
            "verticalPenetrationCategory": "verticalPenetration.elevator", 
            "verticalPenetrationDirection": "high_to_low" 
        } 
    ], 
    "zoneProperties": 
    [ 
        { 
            "zoneName": "WifiB01", 
            "categoryName": "Zone", 
            "zoneNameAlt": "MyZone", 
            "zoneNameSubtitle": "Wifi", 
            "zoneSetId": "1234" 
        }, 
        { 
            "zoneName": "Wifi101",
            "categoryName": "Zone",
            "zoneNameAlt": "MyZone",
            "zoneNameSubtitle": "Wifi",
            "zoneSetId": "1234"
        }
    ]
}
```

## Next steps

> [!div class="nextstepaction"]
> [Tutorial: Creating a Creator indoor map](tutorial-creator-indoor-maps.md)

Learn more by reading:

> [!div class="nextstepaction"]
> [Creator for indoor maps](creator-indoor-maps.md)

[Conversion service]: /rest/api/maps/v2/conversion
[Drawing package]: https://github.com/Azure-Samples/am-creator-indoor-data-examples
[Conversion Drawing Package Guide]: drawing-package-guide.md
[sample drawing package]: https://github.com/Azure-Samples/am-creator-indoor-data-examples
[OSM Opening Hours]: https://wiki.openstreetmap.org/wiki/Key:opening_hours/specification
