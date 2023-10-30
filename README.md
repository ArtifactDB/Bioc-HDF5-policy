# HDF5 policy for BioC file formats

## Motivation

The HDF5 specification supports many different datatypes.
This document aims to place some constraints on the types that can be used to represent Bioconductor objects,
to simplify the process of validation and/or reading objects into memory.

## Integer types

Object representations containing integer datasets should specify the largest integer type that can be used to represent the data.
For example, most representations will specify that any integer array/vector should contain values that can be exactly represented in memory by a 32-bit signed integer.
Writers may choose to create HDF5 datasets with any integer datatype, but readers can still safely assume that an `int32_t` is sufficient to represent all values.
This eliminates the need for readers to dispatch on different types, though they may still choose to do so if a smaller type provides more efficient downstream processing.

To support missing values, writers should define a placeholder attribute on the dataset.
(The exact name of this placeholder depends on the specification for each object, but is generally something like `missing-value-placeholder` or `missing_placeholder`.)
This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset comparing equal to the placeholder value should be treated as missing.
If no placeholder is present, readers may assume that no values are missing.

## Floating-point types

All floating-point datasets should contain values that can be exactly represented in memory by IEEE 754 double-precision floats.
This is necessary for faithful propagation of special values like infinity and NaN.
Readers can safely assume that an IEEE-compliant `double` is sufficient to represent all values in memory.

To support missing values, writers should define a placeholder attribute on the dataset.
This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset comparing equal to the placeholder value should be treated as missing.
If no placeholder is present, readers may assume that no values are missing.

Note that the equality comparison has some implications:

- If the dataset is stored in double precision, readers should represent the data in memory with at least as much precision during the placeholder comparison.
  Otherwise, a cast to lower precision may cause non-missing values to be (incorrectly) equal to the placeholder.
- If the placeholder is an NaN value, all NaNs in the dataset should be treated as missing, regardless of the specific type of NaN.
  We do not consider the NaN payload during placeholder comparisons as this may not be handled reliably.

## String types

String datasets may be represented by any HDF5 string type, i.e., variable or fixed length, ASCII or UTF-8 encoding.
Readers may assume UTF-8 encoding for all strings as this is a superset of ASCII anyway.

To support missing values, writers should define a placeholder attribute on the dataset.
This attribute should be a scalar of the same exact datatype as the dataset.
All values in the dataset with the same sequence of bytes (as defined by, e.g., `strcmp`) as the placeholder value should be treated as missing.
If no placeholder is present, readers may assume that no values are missing.
