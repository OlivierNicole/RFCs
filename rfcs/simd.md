# SIMD

This proposes to provide access to amd64 and ARM64 SIMD instructions to OCaml programmers.

The API would make use of the following 128 and 256-bit vector types:

```
int8x16         int8x32
int16x8         int16x16
int32x4         int32x8
int64x2         int64x4
float32x4       float32x8
float64x2       float64x4
```


