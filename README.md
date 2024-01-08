# HDF5 policy for BioC file formats

## General

HDF5-based representations should aim to be self-contained, i.e., all parameters required for interpretation of the file contents should be stored in the file.
This is usually done via descriptive attributes on the groups or datasets, depending on the Bioconductor object being described.
The aim is to minimize the amount of information that needs to be drawn from external metadata,
reduce the amount of communication between the loading/validating code (typically in C++) and wrapper languages (e.g., R or Python),
and thus decrease the risk of errors in language-specific implementations when reading or loading the file.

## Datatype constraints

### Overview

The HDF5 specification supports many different datatypes... so many, in fact, it can be difficult for readers to know what they should expect.
This document aims to place some constraints on the datatypes that can be used to represent Bioconductor objects,
to simplify the process of validation and/or reading objects into memory.

### Integers 

Representations of integers should specify the largest integer datatype that can be used in the HDF5 dataset,
i.e., the integer type that is guaranteed to be capable of representing all values of the dataset.
This eliminates the need for readers to dispatch on different datatypes, though they may still choose to do so if a smaller datatype provides more efficient downstream processing.

For example, most objects containing integers will specify that the corresponding HDF5 dataset should contain values that can be exactly represented in memory by a 32-bit signed integer.
Readers can thus safely assume that an `int32_t` is sufficient to represent all possible values.
Writers may create HDF5 datasets with any integer datatype that can be fully represented by an `int32_t`, e.g., `H5T_NATIVE_INT8`, `H5T_NATIVE_UINT16`.

### Floats

Representations of floating-point numbers should specify the largest floating-point datatype that can be used in the HDF5 dataset,
i.e., the floating-point type that is guaranteed to be capable of representing all values of the dataset.
This eliminates the need for readers to dispatch on different datatypes, though they may still choose to do so if a smaller datatype provides more efficient downstream processing.

For example, most objects containing floats will specify that the HDF5 dataset should contain values that can be exactly represented in memory by 64-bit floationg-point numbers.
Readers can safely assume that an `double` is sufficient to represent all values in memory.
Writers may create HDF5 datasets with any floating-point or integer datatype that can be exactly represented by a `double`, e.g., `H5T_NATIVE_FLOAT`, `H5T_NATIVE_UINT32`.

It should be assumed that all representations are IEEE754-compliant, to ensure the faithful propagation of special values like infinity and NaN.

### Strings

Representations of strings should specify the encoding(s) that can be used in the HDF5 dataset.
From a specification perspective, there is no difference between variable and fixed length string datatypes -
in the latter, the length of the string is either equal to the fixed length or is defined by null termination.

Most objects containing strings will specify that the HDF5 dataset should contain values that can be exactly represented by UTF-8 encoded strings.
This supports both ASCII and UTF-8 encodings as the former is a subset of the latter anyway. 

## Missing value placeholders

### Overview

To support missing values, writers should define a placeholder attribute on the dataset.
(The exact name of this placeholder depends on the specification for each object, but is generally something like `missing-value-placeholder` or `missing_placeholder`.)
This attribute should be a scalar that holds a "missing value placeholder",
where all values in the dataset that are equal to this placeholder should be considered as missing.
If no placeholder is present, readers may assume that no values are missing.

Our placeholder approach is a variation of the sentinel method used by R for its 32-bit integers.
Here, the lowest value (-2147483648) is considered to be missing, which works well but has a few pitfalls.
Most obviously, it excludes -2147483648 as a valid integer.
It also prohibits the use of a smaller integer datatype for efficient storage of a dataset with missing values in a HDF5 file.
Both problems can be avoided by allowing writers to customize the placeholder according to the contents of the dataset.

It is worth mentioning that, in R, a NaN with a special payload is used to mark missing floating-point values.
This is ingenious but has potential problems with stability and portability, given that preservation of NaN payloads is not mandated by the IEEE754 specification.
In such cases, a writer-defined placeholder can more reliably identify missing values.

For completeness, we note that NumPy uses masking to represent missing values.
While this is the most explicit approach, we chose not to use it as the burden on the reader is too high.
Readers must scan both the actual and masking datasets for full interpretation, which complicates the implementation and increases memory usage and disk I/O.
Writers also need to carefully coordinate the chunk dimensions in both datasets to enable efficient simultaneous iteration.

### For integers

This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset comparing equal to the placeholder value should be treated as missing.

### For floating-point 

This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset comparing equal to the placeholder value should be treated as missing.
Note that the equality comparison has some implications:

- If the dataset is stored in double precision, readers should represent the data in memory with at least as much precision during the placeholder comparison.
  Otherwise, a cast to lower precision may cause non-missing values to be (incorrectly) equal to the placeholder.
- If the placeholder is an NaN value, all NaNs in the dataset should be treated as missing, regardless of the specific type of NaN.
  We do not consider the NaN payload during placeholder comparisons as this may not be handled reliably across machines.

### For strings

This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset with the same sequence of bytes (as defined by, e.g., `strcmp`) as the placeholder value should be treated as missing.
