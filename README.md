#!/usr/bin/env python3
"""Pure-stdlib PNG chroma key: remove green-screen background, despill edges."""
import struct, zlib, sys

def read_png(path):
    data = open(path, 'rb').read()
    assert data[:8] == b'\x89PNG\r\n\x1a\n'
    pos, idat, w, h = 8, b'', 0, 0
    while pos < len(data):
        length = struct.unpack('>I', data[pos:pos+4])[0]
        ctype = data[pos+4:pos+8]
        chunk = data[pos+8:pos+8+length]
        if ctype == b'IHDR':
            w, h, bitd, color, comp, filt, inter = struct.unpack('>IIBBBBB', chunk)
            assert bitd == 8 and color == 6 and inter == 0, (bitd, color, inter)
        elif ctype == b'IDAT':
            idat += chunk
        pos += 12 + length
    raw = zlib.decompress(idat)
    stride = w * 4
    out = bytearray(h * stride)
    prev = bytearray(stride)
    p = 0
    for y in range(h):
        f = raw[p]; p += 1
        line = bytearray(raw[p:p+stride]); p += stride
        if f == 1:
            for i in range(4, stride):
                line[i] = (line[i] + line[i-4]) & 0xff
        elif f == 2:
            for i in range(stride):
                line[i] = (line[i] + prev[i]) & 0xff
        elif f == 3:
            for i in range(stride):
                a = line[i-4] if i >= 4 else 0
                line[i] = (line[i] + ((a + prev[i]) >> 1)) & 0xff
        elif f == 4:
            for i in range(stride):
                a = line[i-4] if i >= 4 else 0
                b = prev[i]
                c = prev[i-4] if i >= 4 else 0
                pa, pb, pc = abs(b-c), abs(a-c), abs(a+b-2*c)
                pr = a if (pa <= pb and pa <= pc) else (b if pb <= pc else c)
                line[i] = (line[i] + pr) & 0xff
        out[y*stride:(y+1)*stride] = line
        prev = line
    return w, h, out

def write_png(path, w, h, px):
    stride = w * 4
    raw = bytearray()
    for y in range(h):
        raw.append(0)
        raw += px[y*stride:(y+1)*stride]
    comp = zlib.compress(bytes(raw), 9)
    def chunk(t, d):
        c = t + d
        return struct.pack('>I', len(d)) + c + struct.pack('>I', zlib.crc32(c) & 0xffffffff)
    with open(path, 'wb') as f:
        f.write(b'\x89PNG\r\n\x1a\n')
        f.write(chunk(b'IHDR', struct.pack('>IIBBBBB', w, h, 8, 6, 0, 0, 0)))
        f.write(chunk(b'IDAT', comp))
        f.write(chunk(b'IEND', b''))

def main(src, dst):
    w, h, px = read_png(src)
    n = 0
    for i in range(0, len(px), 4):
        r, g, b = px[i], px[i+1], px[i+2]
        # hard key: strongly green pixel -> fully transparent
        if g > 90 and g > r * 1.4 + 20 and g > b * 1.4 + 20:
            px[i+3] = 0
            n += 1
        # soft key / despill: greenish edge -> clamp green, fade alpha
        elif g > 80 and g > r * 1.15 and g > b * 1.15:
            m = max(r, b)
            excess = g - m
            px[i+1] = m  # despill
            a = px[i+3]
            px[i+3] = max(0, a - min(255, excess * 2))
    write_png(dst, w, h, px)
    print(f'{w}x{h}, keyed {n} px -> {dst}')

if __name__ == '__main__':
    main(sys.argv[1], sys.argv[2])
