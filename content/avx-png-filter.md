+++
title = "Researching Novel Algorithms to Decode PNG Filters"
date = "2023-01-25"
+++
<!-- title = "Researching the Use of Arbitrarily Wide SIMD Lanes to decode the PNG sub filter" -->
<!-- title = "Decoding PNG Filters Using AVX2" -->

PNG compression involves two schemes -- filters and DEFLATE. Filters are a pre-processing step that operate row-by-row and are used to decrease entropy in the data. They work off the assumption that pixels and their neighbors are usually similar, but not necessarily the exact same. DEFLATE is a common lossless compression format combining LZ77 and Huffman coding.

The part that we're interested in right now are the filters. You can find a pretty good explanation of them in the [official PNG specification](https://www.w3.org/TR/PNG-Filters.html), but we'll walk through a quick summary of the parts that are relevant to us here.

Before we talk about what each filter does, I want introduce the concept of a pixel and "bpp" or bits per pixel. PNGs support a number of different color formats, and those formats can affect how we encode and decode pixels. There are two fields we care about -- bit depth and color type.

Color type defines the channels that make up a pixel. In the RGBA color type, pixels consist of 4 channels -- red, green, blue, and alpha. PNGs support simple grayscale, grayscale with alpha, RGB, RGBA, and an "indexed" color type that can be thought of as string interning but for colors, with each color getting an 8 bit number assigned to it.

Bit depth defines the number of bits per channel. Certain color types only permit certain bit depths. If you're curious, the list of permitted combinations can be found in [the spec](http://www.libpng.org/pub/png/spec/1.2/PNG-Chunks.html), but this isn't too important to us right now.

By combining the color type, which defines the number of channels, and the bit depth, which defines the number of bits per channel, we can find the number of bits per pixel. We refer to this value as "bpp." Although `bpp` typically refers to bits per pixel, for the rest of this post, the "b" in "bpp" will refer to "bytes."

Let's look at a simple example:

If we have an RGB color type with a bit depth of 8, our bits per pixel = 3 * 8 = 24, and our bytes per pixel = (bits per pixel) / 8 = 3.

When applying filters, the minimum bytes per pixel used is 1, even if the number of bits per pixel is less than a full byte.

Filters are applied for every byte, regardless of bit depth. This means that if the number of bits per channel is greater than a full byte, we operate on the bytes of that channel separately.

Now we can look at what the filters actually do:

There are a total of 5 filters -- none, up, sub, average, and paeth. Each filter applies a certain operation to a row of bytes. 

The `none` filter, as the name suggests, does not alter the bytes and just copies them as-is. 

The `up` filter takes the pixel at position `n` and subtracts it by the pixel at position `n` in the row above it. For example, if we have two rows that looks like this:

```python
[1, 2, 3, 4, 5]
[1, 2, 3, 4, 5]
```

After applying the `up` filter, we get this compressed result:

```python
[1, 2, 3, 4, 5]
[0, 0, 0, 0, 0]
```

We consider row before the first row contain only zeros, so the first row is unchanged.

The `sub` filter takes the pixel at position `n` and subtracts it by the pixel in the same row at position `n - 1`. For example, if we have a row that looks like this:

```python
[1, 2, 3, 4, 5]
```

If we apply the `sub` filter, we get this result:

```python
# [1 - 0, 2 - 1, 3 - 2, 4 - 3, 5 - 4]
[1, 1, 1, 1, 1]
```

The `sub` filter operates on individual channels. That is, the red channel of pixel `n` is subtracted by the red channel of pixel `n - 1`, the blue channel subtracted by the prior pixel's blue channel, and so on. The calculation for finding the corresponding channel involves the pixel's `bpp`.

If we look at the `sub` filter as operating on individual bytes, we say that the algorithm is `filtered[n] = unfiltered[n] - unfiltered[n - bpp]`. Where `bpp` is calculated based on the color type and bit depth.

If this sounds a bit confusing, it should make a lot more sense when we start looking at the implementation in code. For now, let's keep looking at the other filters.

The `average` filter subtracts the pixel at position `n` by the average of the pixel above and to the left of it.

```python
[1, 2, 3, 4, 5]
[1, 2, 3, 4, 5]
```

turns into 

```python
[1, 2, 3, 4, 5]
# [
#   1 - (1 + 0) // 2,
#   2 - (2 + 1) // 2,
#   3 - (3 + 2) // 2,
#   4 - (4 + 3) // 2,
#   5 - (5 + 4) // 2,
# ]
[1, 1, 1, 1, 1]
```

Here the `//` is python's [floor division operator](https://en.wikibooks.org/wiki/Python_Programming/Operators#Floor_Division_and_True_Division). Because the filters operate on bytes, the resulting values are integers and will wrap if they go outside the range `[0, 255]`. An interesting part of the `average` filter is that the average is computed using _9_ bits of precision, rather than 8. This means that if you're storing bytes as fixed width 8-bit integers, you will need to handle the case in which addition causes rounding.
<!-- todo: ugly last sentence -->

The last and most complex filter is the `paeth` filter. The name comes from the inventor, Alan Paeth. This filter uses a special predictor function that looks at the pixel to the left, the pixel immediately above, and the pixel up and to the left. 

If our image looks like this,

```python
[a, b, c, d]
[e, X, f, g]
```

If we want to find the filtered value of `X`, we call the predictor function with `a`, `b`, and `e` as inputs. Then we take the result of this predictor function and subtract `X` by it.

Based on my non-scientific look at a number of images on Wikimedia Commons, the `paeth` filter appears to be the most common filter in well-compressed files, though this is currently just my speculation. At some point I may write a post investigating this claim.

To decode any of these filters, you just have to add to the filtered value, rather than subtract from the raw value.

#### Revisiting the `sub` Filter

This is all just required background reading to understand what we're really interested in: optimizing the `sub` filter for 8-bit RGBA pixels. Although we introduced the filters by discussing how they're encoded, for the rest of this post we'll only be talking about how they're decoded.

Before moving on, however, I do want to note that the performance characteristics of filters inside the context of PNG decoders and PNG encoders are very different. This is because PNG decoders only have to apply a filter once per row, while a good encoder will likely try all filters for all rows. This can make PNG encoding somewhat slow, and also makes optimizations to individual filters more impactful. If we optimize a filter for PNG decoding, we only see wins for images that use that filter heavily. This may be an area I explore in the future, but for now my focus is primarily on decoding, as that's the operation most commonly performed on PNG files.

As promised, let's look at a simple code implementation of decoding the `sub` filter. All code examples going forward will be in rust.

```rust
pub fn sub(raw_row: &[u8], decoded_row: &mut [u8]) {
    for i in 0..BYTES_PER_PIXEL.min(decoded_row.len()) {
        decoded_row[i] = raw_row[i];
    }

    for i in BYTES_PER_PIXEL..decoded_row.len() {
        let left = decoded_row[i - BYTES_PER_PIXEL];

        decoded_row[i] = raw_row[i].wrapping_add(left)
    }
}
```

Bytes before the start of the row are 0, so we can just copy the first `bpp` bytes into the decoded row without doing any operations. For the next bytes, we add `decoded_row[i - bpp]` to the filtered byte.

Let's look at how LLVM does with this:

```asm
example::sub:
        push    rax
        test    rsi, rsi
        je      .LBB2_1
        test    rcx, rcx
        je      .LBB2_11
        movzx   eax, byte ptr [rdi]
        mov     byte ptr [rdx], al
        cmp     rsi, 1
        je      .LBB2_13
        cmp     rcx, 1
        je      .LBB2_15
        movzx   eax, byte ptr [rdi + 1]
        mov     byte ptr [rdx + 1], al
        cmp     rsi, 2
        je      .LBB2_17
        cmp     rcx, 2
        je      .LBB2_19
        movzx   eax, byte ptr [rdi + 2]
        mov     byte ptr [rdx + 2], al
        cmp     rsi, 3
        je      .LBB2_21
        cmp     rcx, 3
        je      .LBB2_23
        movzx   eax, byte ptr [rdi + 3]
        mov     byte ptr [rdx + 3], al
        cmp     rcx, 4
        jbe     .LBB2_27
        lea     r8, [rsi - 4]
        cmp     rcx, r8
        cmovb   r8, rcx
        lea     rax, [rcx - 5]
        cmp     r8, rax
        cmovae  r8, rax
        inc     r8
        mov     r10d, 4
        cmp     r8, 4
        jbe     .LBB2_4
        mov     r9d, r8d
        and     r9d, 3
        mov     eax, 4
        cmovne  rax, r9
        mov     r9, r8
        sub     r9, rax
        neg     rax
        lea     r10, [r8 + rax]
        add     r10, 4
        xor     r8d, r8d
.LBB2_9:
        vmovd   xmm0, dword ptr [rdi + r8 + 4]
        vmovd   xmm1, dword ptr [rdx + r8]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + r8 + 4], xmm0
        add     r8, 4
        cmp     r9, r8
        jne     .LBB2_9
.LBB2_4:
        mov     r8, rsi
        neg     r8
        add     r10, -4
        mov     r9, rcx
        neg     r9
.LBB2_5:
        cmp     rcx, r10
        je      .LBB2_28
        lea     rax, [r8 + r10]
        cmp     rax, -4
        je      .LBB2_7
        movzx   eax, byte ptr [rdx + r10]
        add     al, byte ptr [rdi + r10 + 4]
        mov     byte ptr [rdx + r10 + 4], al
        lea     rax, [r9 + r10]
        inc     rax
        inc     r10
        cmp     rax, -4
        jne     .LBB2_5
.LBB2_27:
        pop     rax
        ret

; ... panic handling code omitted for brevity
```

It does surprisingly well -- though not perfect. It looks like our first loop is performed in serial and has bounds checks on every iteration. This is because we don't actually know that our input or output slice has at least `BYTES_PER_PIXEL` elements. If this were in the context of a real PNG decoder and this function got inlined, LLVM may be able to do a better job eliding bounds checks. In a real PNG decoder, a row length of 0 would imply the image data is empty and defiltering can be skipped altogether. The row length is guaranteed to be a multiple of `BYTES_PER_PIXEL`, and if it isn't, due to the image being malformed or truncated, we'd expect the PNG decoder to have errored out by this point. Going forward, our implementations of the `sub` filter will rely on these two assumptions.

The second loop is pretty interesting. It looks like LLVM is doing some magic to be able to vectorize it and perform the loads and additions `BYTES_PER_PIXEL` elements at a time. As we'll see later, this actually comes surprisingly close to our handwritten SIMD implementation.

Though, it does look like there's still a lot of code dedicated to handling bounds checks. Let's see the impact of removing them. For simplicity, we'll drop into unsafe to remove them. This code is just for experimenting, so we won't include any debug assertions that you'd expect in proper code using unsafe in this way.

```rust
pub unsafe fn sub_no_bound_checks(raw_row: &[u8], decoded_row: &mut [u8]) {
    for i in 0..BYTES_PER_PIXEL {
        *decoded_row.get_unchecked_mut(i) = *raw_row.get_unchecked(i);
    }

    for i in BYTES_PER_PIXEL..decoded_row.len() {
        let left = *decoded_row.get_unchecked(i - BYTES_PER_PIXEL);

        *decoded_row.get_unchecked_mut(i) = raw_row.get_unchecked(i).wrapping_add(left)
    }
}
```

```asm
example::sub_no_bound_checks:
        push    rbx
        mov     eax, dword ptr [rdi]
        mov     dword ptr [rdx], eax
        cmp     rcx, 5
        jb      .LBB2_12
        lea     r8, [rcx - 4]
        mov     ebx, 4
        cmp     r8, 4
        jb      .LBB2_11
        mov     rbx, r8
        vmovd   xmm0, dword ptr [rdx]
        and     rbx, -4
        lea     rsi, [rbx - 4]
        mov     r10, rsi
        shr     r10, 2
        inc     r10
        mov     r9d, r10d
        and     r9d, 7
        cmp     rsi, 28
        jae     .LBB2_4
        xor     esi, esi
        jmp     .LBB2_6
.LBB2_4:
        and     r10, -8
        xor     esi, esi
.LBB2_5:
        vmovd   xmm1, dword ptr [rdi + rsi + 4]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 4], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 8]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 8], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 12]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 12], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 16]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 16], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 20]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 20], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 24]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 24], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 28]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 28], xmm0
        vmovd   xmm1, dword ptr [rdi + rsi + 32]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [rdx + rsi + 32], xmm0
        add     rsi, 32
        add     r10, -8
        jne     .LBB2_5
.LBB2_6:
        test    r9, r9
        je      .LBB2_9
        lea     r10, [rdx + rsi]
        add     r10, 4
        lea     r11, [rdi + rsi]
        add     r11, 4
        shl     r9, 2
        xor     esi, esi
.LBB2_8:
        vmovd   xmm1, dword ptr [r11 + rsi]
        vpaddb  xmm0, xmm1, xmm0
        vmovd   dword ptr [r10 + rsi], xmm0
        add     rsi, 4
        cmp     r9, rsi
        jne     .LBB2_8
.LBB2_9:
        cmp     r8, rbx
        je      .LBB2_12
        add     rbx, 4
.LBB2_11:
        movzx   eax, byte ptr [rdi + rbx]
        add     al, byte ptr [rdx + rbx - 4]
        mov     byte ptr [rdx + rbx], al
        lea     rax, [rbx + 1]
        mov     rbx, rax
        cmp     rcx, rax
        jne     .LBB2_11
.LBB2_12:
        pop     rbx
        ret
```

This does look a lot nicer. LLVM was able to autovectorize even more and it unrolled the second loop. Let's set up some benchmarks to see how big of an impact we had here. To benchmark, we'll use rust's stdlib benchmarking tools. We could use something like [criterion](https://github.com/bheisler/criterion.rs), but that's not necessary for what we're doing here. We _do_ want good and scientific benchmarks as we start to iterate and explore new algorithms, but the stdlib solution is sufficient for that.

```rust
#![feature(test)]

// ... our `sub` and `sub_no_bound_checks` implementations

#[cfg(test)]
mod bench {
    use super::*;
    use test::Bencher;

    const BUFFER_SIZE: usize = 2_usize.pow(20);

    #[bench]
    fn bench_sub_naive_scalar(b: &mut Bencher) {
        let raw_row = std::hint::black_box([10; BUFFER_SIZE]);
        let mut decoded_row = std::hint::black_box([0; BUFFER_SIZE]);

        b.iter(|| sub(&raw_row, &mut decoded_row));

        std::hint::black_box(decoded_row);
    }

    #[bench]
    fn bench_sub_no_bound_checks(b: &mut Bencher) {
        let raw_row = std::hint::black_box([10; BUFFER_SIZE]);
        let mut decoded_row = std::hint::black_box([0; BUFFER_SIZE]);

        b.iter(|| unsafe { sub_no_bound_checks(&raw_row, &mut decoded_row) });

        std::hint::black_box(decoded_row);
    }
}
```

And now if we run our benchmarks using `cargo bench`, we get

```sh
running 2 tests
test tests::bench_sub_naive_scalar    ... bench:      86,305 ns/iter (+/- 1,123)
test tests::bench_sub_no_bound_checks ... bench:      86,289 ns/iter (+/- 2,324)
```

Although removing the bounds checks made our codegen look a lot nicer, it doesn't seem to actually improve our performance above noise. That's unfortunate. Let's see if we can do better.

First let's try to get an idea of how close to the optimal solution we are. We can try comparing our filters to `memcpy`. We want to see how much overhead our subtraction is adding to the copying of bytes from `raw_row` to `decoded_row`.

We can add a benchmark that looks like this,

```rust
pub unsafe fn baseline_memcpy(raw_row: &[u8], decoded_row: &mut [u8]) {
    decoded_row
        .get_unchecked_mut(0..raw_row.len())
        .copy_from_slice(&*raw_row);
}

fn bench_baseline_memcpy(b: &mut Bencher) {
    let raw_row = std::hint::black_box([10; BUFFER_SIZE]);
    let mut decoded_row = std::hint::black_box([0; BUFFER_SIZE]);

    b.iter(|| unsafe { baseline_memcpy(&raw_row, &mut decoded_row) });

    std::hint::black_box(decoded_row);
}
```

And to double check that `baseline_memcpy` gets optimized how we expect,

```asm
example::baseline_memcpy:
        mov     rax, rdx
        mov     rdx, rsi
        mov     rsi, rdi
        mov     rdi, rax
        jmp     qword ptr [rip + memcpy@GOTPCREL]
```

Great. Now let's compare it to our `sub` implementations.

```sh
running 3 tests
test tests::bench_baseline_memcpy     ... bench:      61,875 ns/iter (+/- 3,949)
test tests::bench_sub_naive_scalar    ... bench:      86,731 ns/iter (+/- 1,833)
test tests::bench_sub_no_bound_checks ... bench:      86,584 ns/iter (+/- 1,374)
```

Our initial implementations don't actually seem to be _that_ bad. But there's probably a lot of room for improvement here.

`libpng` has had optimized filter implementations using explicit SIMD for [close to a decade](https://github.com/glennrp/libpng/commit/9c946e22fcad10c2a44c0380c0909da6732097ce).

The optimization that they make is based around `bpp`. 

It's difficult to decode the PNG filters in parallel. At first glance, it looks like there's a pretty strict data dependency on previous iterations. If we look at an example calculation, this becomes pretty clear. For simplicity in our example, we'll say `bpp` is 1. When we implement things in code, we'll usually default to a `bpp` of 4.

We'll start with the filtered array `[1, 2, 3]` and walk through defiltering it.
```python
filtered = [1, 2, 3]
defiltered = [0, 0, 0]

# defiltered = [1, 0, 0]
defiltered[0] = filtered[0]

# defiltered = [1, 3, 0]
defiltered[1] = defiltered[0] + filtered[1]

# defiltered = [1, 3, 6]
defiltered[2] = defiltered[1] + filtered[2]
```

The last calculation depends on the results of the second-to-last calculation. How can we work around this?

`libpng`'s optimization doesn't really try to -- it still does a lot of things in serial. But when the `bpp` is greater than 1, it can operate on `bpp` bytes per iteration rather than going byte-by-byte. This ends up being pretty fast -- for a `bpp` of 4, you're operating on 4x the number of bytes.

Let's look at how this implementation works in practice. We're going to port [`libpng`'s 4 `bpp` implementation](https://github.com/glennrp/libpng/blob/libpng16/intel/filter_sse2_intrinsics.c#L85) to rust:

```rust
unsafe fn load4(x: [u8; 4]) -> __m128i {
    let tmp = i32::from_le_bytes(*x);
    _mm_cvtsi32_si128(tmp)
}

unsafe fn store4(x: &mut [u8; 4], v: __m128i) {
    let tmp = _mm_cvtsi128_si32(v);
    x.get_unchecked_mut(..4).copy_from_slice(&tmp.to_le_bytes());
}

pub unsafe fn sub_sse2(raw: &[u8], current: &mut [u8]) {
    let mut a: __m128i;
    let mut d = _mm_setzero_si128();

    let mut rb = raw_row.len() + 4;
    let mut idx = 0;

    while rb > 4 {
        a = d;
        d = load4([
            *raw_row.get_unchecked(idx),
            *raw_row.get_unchecked(idx + 1),
            *raw_row.get_unchecked(idx + 2),
            *raw_row.get_unchecked(idx + 3),
        ]);
        d = _mm_add_epi8(d, a);
        store4(&mut decoded_row.get_unchecked_mut(idx..), d);

        idx += 4;
        rb -= 4;
    }
}
```

So instead of loading, adding, and storing one byte at a time, we can operate on 4 bytes at a time. Let's benchmark this implementation to see how much better it is.

```rust
#[bench]
fn bench_sub_sse2(b: &mut Bencher) {
    let raw_row = std::hint::black_box([10; BUFFER_SIZE]);
    let mut decoded_row = std::hint::black_box([0; BUFFER_SIZE]);

    b.iter(|| unsafe { sub_sse2(&*raw_row, &mut *decoded_row) });

    std::hint::black_box(decoded_row);
}
```

When we benchmark this time, we want to make use of SIMD intrinsics. To force LLVM to compile them optimally we'll have to configure `target-cpu=native`. If we don't do this, our intrinsics will be compiled suboptimally and we actually tend to get slower code than the scalar version.

```sh
RUSTFLAGS='-Ctarget-cpu=native' cargo bench
```

```sh
running 4 tests
test tests::bench_baseline_memcpy     ... bench:      62,844 ns/iter (+/- 3,258)
test tests::bench_sub_naive_scalar    ... bench:      86,798 ns/iter (+/- 1,502)
test tests::bench_sub_no_bound_checks ... bench:      86,719 ns/iter (+/- 2,057)
test tests::bench_sub_sse2            ... bench:      86,573 ns/iter (+/- 1,004)
```

Pretty much no improvement. We more or less wrote by hand what LLVM already optimized our naive implementation to. It could be that we missed something in porting the C code, but that seems unlikely. In general, it doesn't seem that we can get a massive win here.

About a year ago, I had the idea to try solving the PNG filters using AVX and AVX2. AVX enables us to operate on 32 bytes at a time, compared to our current implementation that operates on at most 4 bytes at a time. If we're able to use AVX registers and instructions, we'd be able to operate on 8x the number of bytes as existing implementations of the filters.

After playing around with the problem for a while, I realized that decoding the `sub` filter can be pretty trivially reduced down to a pretty well-studied problem called [prefix sum](https://en.wikipedia.org/wiki/Prefix_sum). Prefix sum happens to be extremely easy to compute in parallel, which makes our problem a lot simpler.

The idea behind parallel prefix sum is that you can trivially subdivide the problem and then combine the results of the separate executions. Let's take a simple example:

Given an array `[1, 2, 3, 4]`, the serial solution would be to just loop over the entire array. In a parallel version, we can split this array into `[1, 2]` and `[3, 4]` and compute the prefix sums separately. Then, we can take the last element of the first array and add it to each element in the second array. We'll call this the accumulate step. 

So after executing the prefix sum step, we end up with the two arrays `[1, 3]` and `[3, 7]`. Then we apply the accumulate step and end up with two arrays `[1, 3]` and `[6, 10]`. Combining them get `[1, 3, 6, 10]` which is the correct prefix sum result we're looking for.

This sounds like we're doing more work -- and we are. But these kinds of operations are really fast in SIMD, so the actual number of instructions per byte is significantly less than in the scalar solution. 

[Algorithmica has a pretty good explanation of vectorized prefix sum](https://en.algorithmica.org/hpc/algorithms/prefix/) that goes quite a bit deeper than we need to for this problem, but is a great read if you're interested in learning more.

We'll actually end up with something that looks very similar to their first vectorized example. The only difference is that the algorithm presented by algorithmica operates on 32-bit integers in sequence, while we want to operate on 8-bit bytes offset by `bpp`. When `bpp` is 4, this looks strikingly similar to just adding 32 bit integers.

For the accumulate step, we're going to use `_mm256_extract_epi32`. I wasn't able to port their accumulate implementation very well, but I'd assume it compiles down to roughly the same thing.

Here's our implementation of the `sub` filter using the full width of AVX registers:

```rust
pub unsafe fn sub_avx(raw_row: &[u8], decoded_row: &mut [u8]) {
    let mut last = 0;
    let mut x: __m256i;

    let len = raw_row.len();
    let mut i = 0;

    // we ensure the length in our SIMD loop is divisible by 32
    let offset = len % 32;
    if offset != 0 {
        sub_sse2(raw_row.get_unchecked(..offset), decoded_row.get_unchecked_mut(..offset));
        last = i32::from_be_bytes([
            *decoded_row.get_unchecked(offset - 1),
            *decoded_row.get_unchecked(offset - 2),
            *decoded_row.get_unchecked(offset - 3),
            *decoded_row.get_unchecked(offset - 4),
        ]);
        i = offset;
    }
    
    while len != i {
        // load 32 bytes from input array
        x = _mm256_loadu_si256(raw_row.get_unchecked(i) as *const _ as *const __m256i);

        // do prefix sum
        x = _mm256_add_epi8(_mm256_slli_si256::<4>(x), x);
        x = _mm256_add_epi8(_mm256_slli_si256::<{ 2 * 4 }>(x), x);

        // accumulate for first 16 bytes
        let b = _mm256_extract_epi32::<3>(x);
        x = _mm256_add_epi8(_mm256_set_epi32(b, b, b, b, 0, 0, 0, 0), x);

        // accumulate for previous chunk of 16 bytes
        x = _mm256_add_epi8(_mm256_set1_epi32(last), x);

        // extract last 4 bytes to be used in next iteration
        last = _mm256_extract_epi32::<7>(x);
        
        // write 32 bytes to out array
        _mm256_storeu_si256(decoded_row.get_unchecked_mut(i) as *mut _ as *mut __m256i, x);

        i += 32;
    }
}
```

We can reuse our SSE implementation for our remainder loop at the start. How does this perform?

```sh
RUSTFLAGS='-Ctarget-cpu=native' cargo bench
```

```sh
running 5 tests
test tests::bench_baseline_memcpy     ... bench:      61,624 ns/iter (+/- 3,384)
test tests::bench_sub_avx             ... bench:      82,168 ns/iter (+/- 3,754)
test tests::bench_sub_naive_scalar    ... bench:      86,713 ns/iter (+/- 3,411)
test tests::bench_sub_no_bound_checks ... bench:      86,567 ns/iter (+/- 1,432)
test tests::bench_sub_sse2            ... bench:      86,422 ns/iter (+/- 3,934)
```

We actually get something that's a bit faster. It isn't too much above noise, but it does appear to run consistently ~5% faster. That's not nothing, but it's definitely a much smaller win than we'd expect from operating on 8x the number of bytes at a time.

This is roughly where I left things for about a year. I came back to this problem every once in a while after being inspired by blog posts, reading code, or learning about interesting applications of x86 SIMD intrinsics, but in general I wasn't able to improve on this problem too much.

The particularly slow part is `_mm256_extract_epi32`, especially the second call with a value of `7`. For values above 3, this intrinsic will compile down to multiple expensive instructions. If we remove this intrinsic, we approach the speed of `memcpy`. However, we can't really remove this intrinsic, since it's necessary for the accumulate step.

Last week, as part of a larger blog post investigating the performance of PNG decoders, I revisited this problem. For the sake of completion, I was interested to see how a similar algorithm would perform if we used SSE registers instead. 

The initial implementation looks like this:

```rust

pub unsafe fn sub_sse_prefix_sum(raw_row: &[u8], decoded_row: &mut [u8]) {
    let mut last = 0;
    let mut x: __m128i;

    let len = raw_row.len();
    let mut i = 0;

    let offset = len % 16;
    if offset != 0 {
        sub_sse2(raw_row.get_unchecked(..offset), decoded_row.get_unchecked_mut(..offset));
        last = i32::from_be_bytes([
            *decoded_row.get_unchecked(offset - 1),
            *decoded_row.get_unchecked(offset - 2),
            *decoded_row.get_unchecked(offset - 3),
            *decoded_row.get_unchecked(offset - 4),
        ]);
        i = offset;
    }

    while len != i {
        // load 16 bytes from array
        x = _mm_loadu_si128(raw_row.get_unchecked(i) as *const _ as *const __m128i);

        // do prefix sum
        x = _mm_add_epi8(_mm_slli_si128::<4>(x), x);
        x = _mm_add_epi8(_mm_slli_si128::<{ 2 * 4 }>(x), x);

        // accumulate for previous chunk of 16 bytes
        x = _mm_add_epi8(x, _mm_set1_epi32(last));

        last = _mm_extract_epi32::<3>(x);

        // write 16 bytes to out array
        _mm_storeu_si128(decoded_row.get_unchecked_mut(i) as *mut _ as *mut __m128i, x);

        i += 16;
    }
}
```

This is pretty much the same as our AVX implementation, except now we operate on 16 bytes at a time. Let's see how this performs:

```sh
running 6 tests
test tests::bench_baseline_memcpy     ... bench:      61,482 ns/iter (+/- 10,144)
test tests::bench_sub_avx             ... bench:      82,137 ns/iter (+/- 623)
test tests::bench_sub_naive_scalar    ... bench:      86,471 ns/iter (+/- 2,308)
test tests::bench_sub_no_bound_checks ... bench:      86,571 ns/iter (+/- 9,310)
test tests::bench_sub_sse2            ... bench:      86,146 ns/iter (+/- 2,569)
test tests::bench_sub_sse_prefix_sum  ... bench:     113,024 ns/iter (+/- 1,683)
```

It's quite a bit slower. I guess that's to be expected. The AVX implementation operates on 2x the number of bytes and only gets a 5% speedup. If we only use SSE registers, we don't see any gains.

But, something interesting about using SSE registers is that we can actually avoid the extract by doing a bitshift and a broadcast. Let's look at an implementation using this,

```rust
pub unsafe fn sub_sse_prefix_sum_no_extract(raw_row: &[u8], decoded_row: &mut [u8]) {
    let mut last = _mm_setzero_si128();
    let mut x: __m128i;

    let len = raw_row.len();
    let mut i = 0;

    let offset = len % 16;
    if offset != 0 {
        sub_sse2(raw_row.get_unchecked(..offset), decoded_row.get_unchecked_mut(..offset));
        last = _mm_castps_si128(_mm_broadcast_ss(&*(decoded_row.get_unchecked(offset - 4) as *const _ as *const f32)));
        i = offset;
    }
    
    while len != i {
        // load 16 bytes from array
        x = _mm_loadu_si128(raw_row.get_unchecked(i) as *const _ as *const __m128i);

        // do prefix sum
        x = _mm_add_epi8(_mm_slli_si128::<4>(x), x);
        x = _mm_add_epi8(_mm_slli_si128::<{ 2 * 4 }>(x), x);

        // accumulate for previous chunk of 16 bytes
        x = _mm_add_epi8(x, last);

        // shift right by 12 bytes and then broadcast the lower 4 bytes
        // to the rest of the register
        last = _mm_srli_si128::<12>(x);
        last = _mm_broadcastd_epi32(last);

        _mm_storeu_si128(decoded_row.get_unchecked_mut(i) as *mut _ as *mut __m128i, x);

        i += 16;
    }
}
```

Running the benchmarks:

```sh
running 7 tests
test tests::bench_baseline_memcpy               ... bench:      62,683 ns/iter (+/- 18,025)
test tests::bench_sub_avx                       ... bench:      82,232 ns/iter (+/- 9,210)
test tests::bench_sub_naive_scalar              ... bench:      86,680 ns/iter (+/- 2,137)
test tests::bench_sub_no_bound_checks           ... bench:      86,756 ns/iter (+/- 1,310)
test tests::bench_sub_sse2                      ... bench:      86,519 ns/iter (+/- 2,527)
test tests::bench_sub_sse_prefix_sum            ... bench:     112,770 ns/iter (+/- 4,366)
test tests::bench_sub_sse_prefix_sum_no_extract ... bench:      69,864 ns/iter (+/- 1,696)
```
<!-- todo: also add benchmarks for different input sizes -->
It's a lot faster! We're approaching the speed of `memcpy`. With this new algorithm, we can go ~25% faster than a naive approach operating on only 4 bytes at a time. It seems hard to improve on this further -- at some point we'll be bound by memory. As a proof of concept for this algorithm, I think this works quite well. It may be possible to improve on this by making better use of AVX intrinsics, but for right now it's likely not worth the effort to optimize this further.

#### Impact of this research

The goal up to this point has largely been to demonstrate that this algorithm can improve the performance of PNG decoding. The work demonstrated here is a proof-of-concept and doesn't contain a production-ready implementation.

The `sub` filter is just a small part of PNG decoding. Although we managed to speed it up by 25% for inputs of this size, this doesn't correlate to a 25% improvement of PNG decoding. The exact improvement here is a bit hard to calculate as it depends heavily on the input -- the dimensions of the PNG, DEFLATE's compression level, the distribution of filters, etc. In general I wouldn't expect this to be too large of a win in total decode time, but it may become meaningful if you're trying to write the fastest theoretical PNG decoder (this is foreshadowing).

I think it may be possible to apply similar ideas to the `avg` and `paeth` filters, but I haven't yet come up with a performant solution for them. One particularly painful issue with the `avg` filter is that the average is taken with 9 bits of precision, rather than 8, and then truncated -- so it's not sufficient to just do a bitshift in order to divide by two. Future research here may involve the `VPAVGB` instruction.

I haven't yet investigated the `paeth` filter, so I'm not sure how difficult such a solution for this filter would be. At first glance it appears quite a bit more challenging than the other filters, but there may be a clever solution hiding somewhere.

<!-- Even if someone wanted to write the fastest theoretical PNG decoder (foreshadowing), this algorithm is sufficiently fast to achieve this. -->

<!-- _mm_shuffle_epi32  -->

<!-- https://github.com/etemesi254/zune-image/blob/fc5c78593906f18722969de60af1ca9b6b99b7f7/zune-png/src/filters.rs -->

<!-- https://www.agner.org/optimize/instruction_tables.pdf -->
<!-- https://en.algorithmica.org/hpc/algorithms/prefix/ -->