JGIFTI: A JSON/Binary JSON Extension to the GIFTI Surface Format
================================================================

- **Copyright**: (C) 2026 Qianqian Fang <q.fang at neu.edu>
- **License**: Apache License, Version 2.0
- **Version**: V1 (Draft-1)
- **URL**: https://neurojson.org/jgifti/draft1
- **Status**: This document is currently a work-in-progress
- **Development**: https://github.com/NeuroJSON/jgifti
- **Abstract**:

> This specification defines the JGIFTI standard format. The JGIFTI format
allows one to store and extend the widely used GIFTI format (.gii) using JavaScript
Object Notation (JSON) [RFC4627] and binary JSON serialization methods.
It losslessly maps all GIFTI XML elements and data structures to a human-readable 
JSON-based wrapper, leveraging the JMesh specification for mesh data representation.


## Table of Content

- [Introduction](#introduction)
- [Grammar](#grammar)
- [JGIFTI Keywords](#jgifti-keywords)
  * [GIFTIHeader](#giftiheader)
  * [GIFTIData](#giftidata)
- [JMesh Object Structure](#jmesh-object-structure)
- [Property Name Mapping](#property-name-mapping)
- [Storage Scenarios](#storage-scenarios)
  * [Scenario 1: Compatibility Mode](#scenario-1-compatibility-mode-separate-files)
  * [Scenario 2: Merged File](#scenario-2-merged-file)
  * [Scenario 3: Multiple Surfaces](#scenario-3-multiple-surfaces)
- [Standard MetaData Keywords](#standard-metadata-keywords)
- [Complete Examples](#complete-examples)
- [Recommended File Specifiers](#recommended-file-specifiers)
- [Summary](#summary)


Introduction
------------

JGIFTI is an extensible framework for storing surface-based neuroimaging data using 
the JData and JMesh representations, with a syntax compatible with widely-used JSON 
and Binary JData/BJData file formats.

Unlike GIFTI, which stores a single data construct per file, JGIFTI/JMesh is much 
more flexible - it can combine multiple data types in a single file or store them 
separately. A JGIFTI file must contain at least one `GIFTIData` object, but can 
include any other JMesh/JData constructs alongside GIFTI objects.

Related specifications:
- [JData specification](https://github.com/NeuroJSON/jdata/blob/master/JData_specification.md)
- [JNIfTI specification](https://github.com/NeuroJSON/jnifti/blob/master/JNIfTI_specification.md)
- [JMesh specification](https://github.com/NeuroJSON/jmesh/blob/master/JMesh_specification.md)


Grammar
------------------------

All JGIFTI files are JData specification compliant. JGIFTI provides both a text-based 
format (JSON) and a binary format (BJData). Please refer to the JData specification 
for definitions.


JGIFTI Keywords
------------------------

JGIFTI uses two primary containers:

* **`GIFTIHeader`**: file-level metadata, labels, and default coordinate system
* **`GIFTIData`**: JMesh objects containing mesh geometry and associated properties


### GIFTIHeader

The `"GIFTIHeader"` structure stores file-level information:

```json
"GIFTIHeader": {
    "Version": "1.0",
    "MetaData": {
        "Date": "Thu Nov 15 09:05:22 2007",
        "UserName": "researcher",
        "Description": "Example surface file",
        "LengthUnit": "mm",
        "TimeUnit": "s"
    },
    "LabelTable": {
        "0": {"Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
        "1": {"Label": "Positive", "RGBA": [1.0, 1.0, 0.0, 1.0]}
    },
    "CoordSystem": {
        "DataSpace": "scanner_anat",
        "TransformedSpace": "talairach",
        "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]
    }
}
```

#### Version

GIFTI format version string (currently "1.0").

#### MetaData

File-level name-value pairs corresponding to GIFTI's file-level `<MetaData>`.

#### LabelTable

Label definitions for label data. Maps integer keys to label names and RGBA colors 
(float values 0.0-1.0). Unassigned nodes should have `Alpha=0.0` (transparent).

```json
"LabelTable": {
    "0": {"Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
    "1": {"Label": "V1", "RGBA": [1.0, 0.0, 0.0, 1.0]}
}
```

Or array form:
```json
"LabelTable": [
    {"Key": 0, "Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
    {"Key": 1, "Label": "V1", "RGBA": [1.0, 0.0, 0.0, 1.0]}
]
```

#### CoordSystem

Default coordinate system transformation. Can be a single object or array for 
multiple transforms.

```json
"CoordSystem": {
    "DataSpace": "scanner_anat",
    "TransformedSpace": "talairach",
    "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]
}
```

For `DataSpace` and `TransformedSpace` values, see the **QForm/SForm** section in the 
[JNIfTI specification](https://github.com/NeuroJSON/jnifti/blob/master/JNIfTI_specification.md#qformsform-nifti-1-header-qform_codesform_code).


### GIFTIData

The `"GIFTIData"` container holds JMesh objects representing mesh geometry and data.

```json
"GIFTIData": {
    "MeshVertex3": { "_DataInfo_": {...}, "Data": [...], "Properties": {...} },
    "MeshTri3": { "_DataInfo_": {...}, "Data": [...], "Properties": {...} }
}
```

**JMesh keyword mapping:**

| GIFTI Intent | JMesh Keyword | Data Shape |
|--------------|---------------|------------|
| `NIFTI_INTENT_POINTSET` | `MeshVertex3` | N×3 (coordinates) |
| `NIFTI_INTENT_TRIANGLE` | `MeshTri3` | M×3 (indices) |
| `NIFTI_INTENT_VECTOR` | `MeshVertex3` | N×3 (vectors) |

**Index convention:** JMesh uses 1-based indices; GIFTI uses 0-based. Converters 
must adjust triangle indices accordingly.


JMesh Object Structure
------------------------

All JMesh objects in `GIFTIData` **must** use the annotated structure form:

```json
"MeshVertex3": {
    "_DataInfo_": {
        "MetaData": {...},
        "CoordSystem": {...}
    },
    "Data": [...],
    "Properties": {
        "Thickness": [...],
        "Curvature": [...],
        "Label": [...],
        "TimeSeries": [...],
        "HbO": [...],
        "HbR": [...],
        "HbT": [...],
        "Normal": [...],
        "Color": [...]
    }
}
```

### _DataInfo_

Contains per-DataArray metadata:

| Field | Description |
|-------|-------------|
| `MetaData` | Per-DataArray name-value pairs |
| `CoordSystem` | Coordinate system(s) - can be array for multiple transforms |

Example:
```json
"_DataInfo_": {
    "MetaData": {
        "AnatomicalStructurePrimary": "CortexLeft",
        "AnatomicalStructureSecondary": "Pial",
        "GeometricType": "Anatomical",
        "UniqueID": "{565f3bd1-c0b1-49da-af3d-707cd6a8ccd0}"
    },
    "CoordSystem": [
        {"DataSpace": "scanner_anat", "TransformedSpace": "talairach", "MatrixData": [...]},
        {"DataSpace": "scanner_anat", "TransformedSpace": "mni_152", "MatrixData": [...]}
    ]
}
```

### Data

For `MeshVertex3`: N×3 array of vertex coordinates.
For `MeshTri3`: M×3 array of triangle indices (1-based).

The `Data` field can be:

**Direct array:**
```json
"Data": [[-16.07, -66.19, 21.27], [-16.71, -66.05, 21.23], ...]
```

**Annotated array (typed/compressed):**
```json
"Data": {
    "_ArrayType_": "float32",
    "_ArraySize_": [143479, 3],
    "_ArrayZipType_": "zlib",
    "_ArrayZipSize_": [143479, 3],
    "_ArrayZipData_": "<base64-encoded data>"
}
```

**Reference to external file (for compatibility mode):**
```json
"Data": {
    "_DataLink_": "file://./subj01.coord.jgii"
}
```

The `_DataLink_` value can be a URL or JSONPath per the 
[JData specification](https://github.com/NeuroJSON/jdata/blob/master/JData_specification.md).

**GIFTI binary attribute mapping:**

| GIFTI Attribute | JData Annotation |
|-----------------|------------------|
| `DataType` | `_ArrayType_` (`"uint8"`, `"int32"`, `"float32"`) |
| `ArrayIndexingOrder` | `_ArrayOrder_` (`"c"`, `"f"`) |
| `Endian` | `_ArrayZipEndian_` (`"little"`, `"big"`) |
| `Encoding=GZipBase64Binary` | `_ArrayZipType_: "zlib"` |

### Properties

All per-node values (shape, labels, colors, time series, etc.) are stored in 
`Properties`. For `MeshVertex3`, each property array must have length N (number of nodes).

```json
"Properties": {
    "Thickness": [2.5, 2.3, 2.8, ...],
    "Curvature": [0.1, -0.2, 0.05, ...],
    "Label": [0, 1, 1, 2, ...],
    "TimeSeries": [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], ...],
    "HbO": [1.5, 1.2, 1.8, ...],
    "HbR": [0.5, 0.4, 0.6, ...],
    "Normal": [[0.1, 0.2, 0.97], [0.1, 0.2, 0.97], ...],
    "Color": [[1.0, 0.8, 0.6, 1.0], [0.9, 0.7, 0.5, 1.0], ...]
}
```


Property Name Mapping
------------------------

GIFTI per-node data intents map to `Properties` field names:

| GIFTI Intent | Property Name | Data Shape | Description |
|--------------|---------------|------------|-------------|
| `NIFTI_INTENT_SHAPE` | `Shape` or specific name | N×1 | Shape measurements |
| - | `Thickness` | N×1 | Cortical thickness |
| - | `Curvature` | N×1 | Surface curvature |
| - | `SulcalDepth` | N×1 | Sulcal depth |
| `NIFTI_INTENT_LABEL` | `Label` | N×1 (int) | Label indices into LabelTable |
| `NIFTI_INTENT_TIME_SERIES` | `TimeSeries` | N×T | Time series (T timepoints) |
| `NIFTI_INTENT_RGB_VECTOR` | `Color` | N×3 | RGB colors |
| `NIFTI_INTENT_RGBA_VECTOR` | `Color` | N×4 | RGBA colors |
| `NIFTI_INTENT_VECTOR` | `Vector` | N×3 | 3D vectors |
| `NIFTI_INTENT_GENMATRIX` | `Tensor` | N×M | Tensor data |
| `NIFTI_INTENT_NODE_INDEX` | `NodeIndex` | K×1 | Sparse node indices |
| Statistical intents | `Functional` or specific name | N×1 | Statistical values |
| fNIRS: HbO | `HbO` | N×1 or N×T | Oxy-hemoglobin (µM) |
| fNIRS: HbR | `HbR` | N×1 or N×T | Deoxy-hemoglobin (µM) |
| fNIRS: HbT | `HbT` | N×1 or N×T | Total hemoglobin (µM) |
| fNIRS: µa | `AbsorptionCoefficient` | N×1 or N×T | Absorption coefficient (1/mm) |
| - | `Normal` | N×3 | Vertex normals |
| - | `Color` | N×3 or N×4 | Vertex colors |

**Custom property names:** For specific measurements, use descriptive names 
(e.g., `Thickness`, `Curvature`, `T1w_T2w_Ratio`) stored in `_DataInfo_.MetaData.Name`.


Storage Scenarios
------------------------

### Scenario 1: Compatibility Mode (Separate Files)

For compatibility with GIFTI workflows, mesh components can be stored in separate 
files with the same extensions as GIFTI. In this mode:

- Each `.jgii` file stores one type of data
- Per-node value files reference the coordinate file via `_DataLink_`
- The `Data` field is either empty `[]` or contains a `_DataLink_` reference

**Coordinate file (`subj01.coord.jgii`):**
```json
{
    "GIFTIHeader": {
        "Version": "1.0",
        "MetaData": {"LengthUnit": "mm"}
    },
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {
                "MetaData": {
                    "AnatomicalStructurePrimary": "CortexLeft",
                    "GeometricType": "Anatomical"
                },
                "CoordSystem": {"DataSpace": "talairach", "TransformedSpace": "talairach",
                             "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]}
            },
            "Data": [[-16.07, -66.19, 21.27], [-16.71, -66.05, 21.23], ...]
        }
    }
}
```

**Topology file (`subj01.topo.jgii`):**
```json
{
    "GIFTIHeader": {"Version": "1.0"},
    "GIFTIData": {
        "MeshTri3": {
            "_DataInfo_": {
                "MetaData": {"TopologicalType": "Closed"}
            },
            "Data": [[1, 2, 4], [5, 4, 2], ...]
        }
    }
}
```

**Shape file (`subj01.thickness.shape.jgii`):**
```json
{
    "GIFTIHeader": {"Version": "1.0"},
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {
                "MetaData": {"Name": "thickness"}
            },
            "Data": {"_DataLink_": "file://./subj01.coord.jgii"},
            "Properties": {
                "Thickness": [2.5, 2.3, 2.8, ...]
            }
        }
    }
}
```

**Label file (`subj01.label.jgii`):**
```json
{
    "GIFTIHeader": {
        "Version": "1.0",
        "LabelTable": {
            "0": {"Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
            "1": {"Label": "V1", "RGBA": [1.0, 0.0, 0.0, 1.0]}
        }
    },
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {
                "MetaData": {"Name": "parcellation"}
            },
            "Data": {"_DataLink_": "file://./subj01.coord.jgii"},
            "Properties": {
                "Label": [0, 1, 1, 2, 2, 0, ...]
            }
        }
    }
}
```

**Functional file (`subj01.func.jgii`):**
```json
{
    "GIFTIHeader": {"Version": "1.0"},
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {
                "MetaData": {
                    "Name": "task_contrast",
                    "intent_p1": "8.0"
                }
            },
            "Data": {"_DataLink_": "file://./subj01.coord.jgii"},
            "Properties": {
                "Functional": [0.5, -0.3, 1.2, ...]
            }
        }
    }
}
```

**Time series file (`subj01.time.jgii`):**
```json
{
    "GIFTIHeader": {
        "Version": "1.0",
        "MetaData": {"TimeStep": "2.0", "TimeUnit": "s"}
    },
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {},
            "Data": {"_DataLink_": "file://./subj01.coord.jgii"},
            "Properties": {
                "TimeSeries": [[0.1, 0.2, 0.3, 0.4], [0.5, 0.6, 0.7, 0.8], ...]
            }
        }
    }
}
```


### Scenario 2: Merged File

All mesh data and properties can be stored in a single file. Each GIFTI per-node 
data type becomes a named property in `MeshVertex3.Properties`:

```json
{
    "GIFTIHeader": {
        "Version": "1.0",
        "MetaData": {
            "Date": "2024-01-15",
            "SubjectID": "subj01",
            "LengthUnit": "mm",
            "TimeUnit": "s"
        },
        "LabelTable": {
            "0": {"Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
            "1": {"Label": "V1", "RGBA": [1.0, 0.0, 0.0, 1.0]},
            "2": {"Label": "V2", "RGBA": [0.0, 1.0, 0.0, 1.0]}
        }
    },
    "GIFTIData": {
        "MeshVertex3": {
            "_DataInfo_": {
                "MetaData": {
                    "AnatomicalStructurePrimary": "CortexLeft",
                    "AnatomicalStructureSecondary": "Pial",
                    "GeometricType": "Anatomical"
                },
                "CoordSystem": {
                    "DataSpace": "scanner_anat",
                    "TransformedSpace": "talairach",
                    "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]
                }
            },
            "Data": [
                [-16.072, -66.188, 21.267],
                [-16.706, -66.054, 21.233],
                [-17.614, -65.402, 21.071]
            ],
            "Properties": {
                "Thickness": [2.5, 2.3, 2.8],
                "Curvature": [0.1, -0.2, 0.05],
                "SulcalDepth": [1.2, 0.8, 1.5],
                "Label": [0, 1, 2],
                "Functional": [0.5, -0.3, 1.2],
                "TimeSeries": [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], [0.7, 0.8, 0.9]],
                "HbO": [1.5, 1.2, 1.8],
                "HbR": [0.5, 0.4, 0.6],
                "Normal": [[0.1, 0.2, 0.97], [0.1, 0.2, 0.97], [0.1, 0.2, 0.97]],
                "Color": [[1.0, 0.8, 0.6, 1.0], [0.9, 0.7, 0.5, 1.0], [0.8, 0.6, 0.4, 1.0]]
            }
        },
        "MeshTri3": {
            "_DataInfo_": {
                "MetaData": {"TopologicalType": "Closed"}
            },
            "Data": [[1, 2, 3]]
        }
    }
}
```


### Scenario 3: Multiple Surfaces

Multiple anatomical surfaces (e.g., pial, white, inflated, sphere) can be stored 
in a single file. The recommended naming convention for anatomy ID follows 
`<subject>_<hemisphere>_<structure>` (e.g., `P001_L_pial`).

Each surface can be stored directly under `GIFTIData` using the anatomy ID as the key,
or optionally wrapped inside `MeshObject(anatomyID)` to explicitly indicate it is a 
mesh object:

```json
{
    "GIFTIHeader": {
        "Version": "1.0",
        "MetaData": {
            "SubjectID": "P001",
            "Date": "2024-01-15",
            "LengthUnit": "mm"
        },
        "LabelTable": {
            "0": {"Label": "???", "RGBA": [0.667, 0.667, 0.667, 1.0]},
            "1": {"Label": "V1", "RGBA": [1.0, 0.0, 0.0, 1.0]}
        }
    },
    "GIFTIData": {
        "P001_L_pial": {
            "MeshVertex3": {
                "_DataInfo_": {
                    "MetaData": {
                        "AnatomicalStructurePrimary": "CortexLeft",
                        "AnatomicalStructureSecondary": "Pial",
                        "GeometricType": "Anatomical"
                    },
                    "CoordSystem": {"DataSpace": "scanner_anat", "TransformedSpace": "talairach",
                                 "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]}
                },
                "Data": [[-16.07, -66.19, 21.27], ...],
                "Properties": {
                    "Thickness": [2.5, 2.3, ...],
                    "Curvature": [0.1, -0.2, ...],
                    "Label": [0, 1, ...]
                }
            },
            "MeshTri3": {
                "_DataInfo_": {"MetaData": {"TopologicalType": "Closed"}},
                "Data": [[1, 2, 3], ...]
            }
        },
        "P001_L_white": {
            "MeshVertex3": {
                "_DataInfo_": {
                    "MetaData": {
                        "AnatomicalStructurePrimary": "CortexLeft",
                        "AnatomicalStructureSecondary": "GrayWhite",
                        "GeometricType": "Anatomical"
                    },
                    "CoordSystem": {"DataSpace": "scanner_anat", "TransformedSpace": "talairach",
                                 "MatrixData": [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]]}
                },
                "Data": [[-15.5, -65.8, 20.9], ...],
                "Properties": {}
            },
            "MeshTri3": {
                "_DataInfo_": {},
                "Data": {"_DataLink_": "$.GIFTIData.P001_L_pial.MeshTri3.Data"}
            }
        },
        "P001_L_inflated": {
            "MeshVertex3": {
                "_DataInfo_": {
                    "MetaData": {
                        "AnatomicalStructurePrimary": "CortexLeft",
                        "GeometricType": "Inflated"
                    }
                },
                "Data": [[-20.0, -70.0, 25.0], ...],
                "Properties": {}
            },
            "MeshTri3": {
                "_DataInfo_": {},
                "Data": {"_DataLink_": "$.GIFTIData.P001_L_pial.MeshTri3.Data"}
            }
        },
        "P001_L_sphere": {
            "MeshVertex3": {
                "_DataInfo_": {
                    "MetaData": {
                        "AnatomicalStructurePrimary": "CortexLeft",
                        "GeometricType": "Spherical"
                    }
                },
                "Data": [[100.0, 0.0, 0.0], ...],
                "Properties": {}
            },
            "MeshTri3": {
                "_DataInfo_": {},
                "Data": {"_DataLink_": "$.GIFTIData.P001_L_pial.MeshTri3.Data"}
            }
        }
    }
}
```

**Note:** Surfaces sharing the same topology can reference a common `MeshTri3` using 
`_DataLink_` with JSONPath (e.g., `$.GIFTIData.P001_L_pial.MeshTri3.Data`).


Standard MetaData Keywords
------------------------

### File-Level (GIFTIHeader.MetaData)

| Keyword | Description |
|---------|-------------|
| `Date` | Date/time file was written |
| `UserName` | User who wrote the file |
| `Description` | Text description |
| `SubjectID` | Subject identifier |
| `TimeStep` | TR for time series data |
| `LengthUnit` | Unit for spatial data (default: `"mm"`) |
| `TimeUnit` | Unit for time data (default: `"s"`) |
| `FrequencyUnit` | Unit for frequency data (default: `"Hz"`) |

### MeshVertex3 (_DataInfo_.MetaData)

| Keyword | Values |
|---------|--------|
| `AnatomicalStructurePrimary` | `CortexLeft`, `CortexRight`, `CortexRightAndLeft`, `Cerebellum`, `Head`, `HippocampusLeft`, `HippocampusRight` |
| `AnatomicalStructureSecondary` | `GrayWhite`, `Pial`, `MidThickness` |
| `GeometricType` | `Reconstruction`, `Anatomical`, `Inflated`, `VeryInflated`, `Spherical`, `SemiSpherical`, `Ellipsoid`, `Flat`, `Hull` |
| `UniqueID` | UUID string (optional) |
| `SurfaceID` | Surface identifier (optional) |
| `Name` | Short name for property data |

### MeshTri3 (_DataInfo_.MetaData)

| Keyword | Values |
|---------|--------|
| `TopologicalType` | `Closed`, `Open`, `Cut` |
| `UniqueID` | UUID string (optional) |

### Property-Specific (_DataInfo_.MetaData)

| Keyword | Description |
|---------|-------------|
| `Name` | Property name for display |
| `intent_p1`, `intent_p2`, `intent_p3` | Statistical parameters |


GIFTI to JGIFTI Conversion
------------------------

| GIFTI XML | JGIFTI |
|-----------|--------|
| `<GIFTI Version>` | `GIFTIHeader.Version` |
| `<MetaData>` (file) | `GIFTIHeader.MetaData` |
| `<LabelTable>` | `GIFTIHeader.LabelTable` |
| `<Label Key R G B A>text</Label>` | `"Key": {"Label": "text", "RGBA": [R,G,B,A]}` |
| `<DataArray Intent=POINTSET>` | `MeshVertex3` |
| `<DataArray Intent=TRIANGLE>` | `MeshTri3` |
| `<DataArray Intent=SHAPE/LABEL/...>` | `MeshVertex3.Properties.<Name>` |
| `<MetaData>` (DataArray) | `_DataInfo_.MetaData` |
| `<CoordinateSystemTransformMatrix>` | `_DataInfo_.CoordSystem` |
| `<Data>` (coordinates) | `MeshVertex3.Data` |
| `<Data>` (triangles) | `MeshTri3.Data` |
| `<Data>` (per-node values) | `MeshVertex3.Properties.<Name>` |


Recommended File Specifiers
------------------------------

| Content | GIFTI | JGIFTI Text | JGIFTI Binary |
|---------|-------|-------------|---------------|
| Generic | `.gii` | `.jgii` | `.bgii` |
| Coordinates | `.coord.gii` | `.coord.jgii` | `.coord.bgii` |
| Functional | `.func.gii` | `.func.jgii` | `.func.bgii` |
| Labels | `.label.gii` | `.label.jgii` | `.label.bgii` |
| RGB/RGBA | `.rgba.gii` | `.rgba.jgii` | `.rgba.bgii` |
| Shape | `.shape.gii` | `.shape.jgii` | `.shape.bgii` |
| Surface | `.surf.gii` | `.surf.jgii` | `.surf.bgii` |
| Tensors | `.tensor.gii` | `.tensor.jgii` | `.tensor.bgii` |
| Time Series | `.time.gii` | `.time.jgii` | `.time.bgii` |
| Topology | `.topo.gii` | `.topo.jgii` | `.topo.bgii` |
| Vector | `.vector.gii` | `.vector.jgii` | `.vector.bgii` |

MIME types: **`application/jgifti-text`** and **`application/jgifti-binary`**


Summary
----------

JGIFTI provides a flexible JSON/BJData format for GIFTI surface data:

1. **GIFTIHeader**: File-level Version, MetaData, LabelTable, and default CoordSystem

2. **GIFTIData**: JMesh objects using structure form `{_DataInfo_, Data, Properties}`

3. **Properties**: All per-node data (shape, labels, colors, time series, functional, fNIRS) 
   stored in `MeshVertex3.Properties` with consistent naming

4. **Three storage scenarios**:
   - **Compatibility mode**: Separate files with `_DataLink_` references
   - **Merged file**: All data in one file with properties
   - **Multiple surfaces**: Anatomy ID keys (e.g., `P001_L_pial`) with shared topology via JSONPath

5. **JData annotations**: Binary attributes map to `_ArrayType_`, `_ArrayOrder_`, 
   `_ArrayZipType_`, `_ArrayZipEndian_`

6. **fNIRS support**: Properties include `HbO`, `HbR`, `HbT`, `AbsorptionCoefficient`

7. **Unit specification**: `LengthUnit`, `TimeUnit`, `FrequencyUnit` in MetaData

8. **Extensibility**: Can include any JMesh/JData constructs alongside GIFTI objects
