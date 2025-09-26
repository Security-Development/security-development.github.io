---
layout: post
title: "[HackingCamp 27th CTF] PR0C3E5S1N9 Write-up(for English)"
author: "SeungYong Lee"
categories: [CTF, Write-up]
tags: [HackingCamp, CTF, Write-up, Exploit, Reverse Engineering]
image: assets/images/hacking-camp-27th-P0R0C3E5S1N9-write-up/thumbnail.gif
---

Hello readers!

In the second half of 2023, I participated in the 27th Hacking Camp held in Korea.

Hacking Camp is a non-profit organization and by holding Hacking Camp, people dreaming of working in cyber security can get great opportunities in Korea every year.

I was the leader of team "어디에도(Everywhere)" so, I took it seriously during Hacking Camp.

Through this camp, I learned how to respect and rely on my teammates. In return, my teammates trusted me and participated with unwavering determination.

Now, let me explain how I approached and solved the reversing challenge called PR0C3E5S1N9.

### Understanding the Problem
The challenge provided encrypted JavaScript code. So, my first question was "How can I decrypt this?"

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
// [ PROB.JS ]
var _cs=["\x42\x79","\x70\x6f\x70","\x6d\x61\x70","\x6f\x64","\x65\x41","\x64\x75","\x72\x65","\x61\x63","\x63\x68","\x61\x72\x43","\x67\x74","\x6e\x67\x74","\x72\x45","\x64\x6f\x77","\x61\x62\x73","\x62\x36\x34",'\x49\x64',"\x6e\x67",'\x31\x30\x32\x34','\x6c\x6f\x63',"\x74","\x72\x45\x61",'\x2d',"\x63\x65","\x66\x75\x6e\x63","\x68","\x6c\x65","\x30","\x6e\x61\x76","\x6c\x65\x6e","\x6c\x6f\x67","\x74\x68","\x7a\x6f\x6e\x65","\x66\x6f","\x74\x69\x6d\x65","\x6d\x65\x6e\x74",'\x45\x6c\x65']; const _g0 = (a) => { let _g1 = 0; while(a){ _g1 <<= 1; _g1 |= a & 1; a >>>= 1; } return _g1; }, _g2 = (a) => { let _g3 = new ArrayBuffer(a[_cs[26]+_cs[17]+_cs[31]]); let _g4 = new Uint8Array(_g3); let _g5 = new Uint32Array(_g3); let _g6 = new ArrayBuffer(a[_cs[26]+_cs[11]+_cs[25]]); let _g7 = new Uint8Array(_g6); let _g8 = new Uint32Array(_g6); let _g9 = new ArrayBuffer(a[_cs[29]+_cs[10]+_cs[25]]); let _g1 = new Uint32Array(_g9); _g4[0] = 0x37; for(let _ga = 0; _ga < a[_cs[26]+_cs[17]+_cs[31]]; _ga++){ _g7[_ga] = a[_cs[8]+_cs[9]+_cs[3]+_cs[4]+_cs[20]](_ga); }; _g4[_cs[33]+_cs[12]+_cs[7]+_cs[25]]((_, _ga, _g4) => { if(_ga == 0) return; _g4[_ga] = ((((_g4[_ga - 1] ^ 0x96) + 0xDD) ^ 0xA4) + 0x96) ^ 0xC8; }); _g8[_cs[33]+_cs[21]+_cs[8]]((_, _ga) => { _g1[_ga] = _; _g1[_ga] += _g5[_ga]; _g1[_ga] ^= _g0(_g5[_ga]); _g1[_ga] += _g0(_g5[_ga]); _g1[_ga] ^= _g5[_ga]; }); return new Uint8Array(_g9); }; _g7 = [164,72,70,191,200,156,172,79,52,69,146,160,106,90,169,94,108,204,156,47,106,122,198,5,206,52,249,21,70,125,172,196,96,156,186,190,81,97,105,119]; _gb = ""; console[_cs[30]](_g2(_gb)[_cs[2]]((_g5, _ga) => {if (_g7[_ga] != _g5) return false; return true;})[_cs[6]+_cs[5]+_cs[23]]((a,_g3)=>a*_g3,_g7[_cs[29]+_cs[10]+_cs[25]] == _gb[_cs[26]+_cs[17]+_cs[31]]));
</pre>
</div>

While I was thinking about how to analyze the code, I came up with a website called "beautifier.io". It improves the readability of input JavaScript code.
I inputted JavaScript code on the site. The website improved the code's readability, and I enhanced it further based on my own preferences. The code below is the result from that

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
const reverseBits = (a) => {
    let _g1 = 0;

    while (a) {
        _g1 <<= 1;
        _g1 |= a & 1;
        a >>>= 1; 
    }

    return _g1;
}
    
const encrypt = (a) => {
    let key_stream = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let key_stream_8 = new Uint8Array(key_stream); // 1byte
    let key_stream_32 = new Uint32Array(key_stream); // 4byte

    let input_text = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let input_text_8 = new Uint8Array(input_text); // 1byte
    let input_text_32 = new Uint32Array(input_text); // 4 byte

    let result = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let result_32 = new Uint32Array(result); // 4 byte
    
    key_stream_8[0] = 0x37;

    for (let i = 0; i < a.length; i++) {
        input_text_8[i] = a.charCodeAt(i);
    };
    
    key_stream_8.forEach((_, index, array) => {
        if (index == 0) return;
        array[index] = ((((array[index - 1] ^ 0x96) + 0xDD) ^ 0xA4) + 0x96) ^ 0xC8;
    });
    
    input_text_32.forEach((_, index) => {
        result_32[index] = (((_ + key_stream_32[index]) ^ reverseBits(key_stream_32[index]) ) +  reverseBits(key_stream_32[index])) ^ key_stream_32[index];
    });

    return new Uint8Array(result);
};

flag_enc = [164, 72, 70, 191, 200, 156, 172, 79, 52, 69, 146, 160, 106, 90, 169, 94, 108, 204, 156, 47, 106, 122, 198, 5, 206, 52, 249, 21, 70, 125, 172, 196, 96, 156, 186, 190, 81, 97, 105, 119];
text = "HCAMP{AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}";

console.log(encrypt(text).map((element, index) => {
    console.log(element);
    
    if (flag_enc[index] != element) 
        return false;
    
    return true;
}).reduce(
    (accumulator, current_value) => accumulator * current_value, flag_enc['length'] == text['length'])
);
</pre>
</div>

Thereafter, I understood it step by step and then after that I realized that the "encrypt" function is the core of this program

### What is ArrayBuffer in JavaScript?
I realized that I had needed to understand ArrayBuffer in JavaScript and Uint(n)Array concepts in order to solve this challenge.

JavaScript provides methods for directly manipulating ArrayBuffer. These methods are mostly used in WebAssembly and Binary Data Processing.

ArrayBuffer is a low-level binary data buffer in JavaScript. It provides a fixed-length raw memory space that you can use to store and manipulate binary data directly.

Actually, when I opened Chrome’s Developer Tools and entered new ArrayBuffer(8) in the console, a transistor-shaped icon appeared next to the output. If you click on that icon, it opens the Memory Inspector panel at the bottom, where you can view a memory dump of the object.

<p align="center">
    <img src="/assets/images/hacking-camp-27th-P0R0C3E5S1N9-write-up/arraybuffer_dump.png">
</p>

1byte is a unit made up of 8bits. In the code, Uint8Array, Uint32Array appear. The number n in Uint(n)Array refers to the number of bits.

<div align="center" style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em;">
  8BIT = 1BYTE          32BIT = 4BYTE
|                       |
 2^8                    2^32
</pre>
</div>

That is, 1byte has a buffer size of 256(2^8) and 4bytes has a buffer size of 4,294,967,296(2^32).

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
var buf = new ArrayBuffer(8); // Creates a buffer with a size of 8 bytes: [00, 00, 00, 00, 00, 00, 00, 00]
var buf_8 = new Uint8Array(buf); // Interprets the buffer as an array of unsigned 8bit integers (1 byte each)
var buf_32 = new Uint32Array(buf); // Interprets the buffer as an array of unsigned 32bit integers (4 bytes each)

buf_8[0] = 256; // A single byte can represent 256 values: 0 to 255. Since 256 is out of range, it wraps around and stores 0 in the first byte.
buf_32[0] = 256; // A 4byte integer can represent 4,294,967,296 values: 0 to 4,294,967,295. Storing 256 results in the buffer holding 00 01 at the corresponding position (little endian).
</pre>
</div>

### Explaining my Improved Code

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
const reverseBits = (a) => {
    let result = 0;
    while (a) {
        result <<= 1;
        result |= a & 1;
        a >>>= 1; 
    }
    return result;
}
</pre>
</div>
The reverseBits function takes an input number a and reverses its bits as if it were a 32bit unsigned integer.<br>
For example, reverseBits(0b00000000000000000000000000001101) returns 0b10110000000000000000000000000000.<br><br>
It works by shifting the result to the left one bit at a time while copying the least significant bit of a into it.<br>
At the same time, a is shifted to the right using the zero-fill right shift (>>>) operator until all bits have been processed.

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
const encrypt = (a) => {
    let keystream = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let keystream_8 = new Uint8Array(keystream); // 1byte
    let keystream_32 = new Uint32Array(keystream); // 4byte

    let input_text = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let input_text_8 = new Uint8Array(input_text); // 1byte
    let input_text_32 = new Uint32Array(input_text); // 4 byte

    let result = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
    let result_32 = new Uint32Array(result); // 4 byte

    keystream_8[0] = 0x37;

    for (let i = 0; i < a.length; i++) {
        input_text_8[i] = a.charCodeAt(i);
    };
    
    keystream_8.forEach((_, index, array) => {
        if (index == 0) return;
        array[index] = ((((array[index - 1] ^ 0x96) + 0xDD) ^ 0xA4) + 0x96) ^ 0xC8;
    });
    
    input_text_32.forEach((_, index) => {
        result_32[index] = (((_ + keystream_32[index]) ^ reverseBits(keystream_32[index]) ) +  reverseBits(keystream_32[index])) ^ keystream_32[index];
    });

    return new Uint8Array(result);
};
</pre>
</div>

The encrypt function is the most important function. Actually I think if you understand all the logic of this function you have basically solved 80% of the problem.

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
let keystream = new ArrayBuffer(a.length);
let keystream_8 = new Uint8Array(keystream);
let keystream_32 = new Uint32Array(keystream);
</pre>
</div>

- keystream: Allocate a buffer of "a.length" bytes for the keystream
- keystream_8: Create a view for accessing the buffer in 1byte units
- keystream_32: Create a view for accessing the same buffer in 4byte units

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
for (let i = 0; i < a.length; i++) {
    input_text_8[i] = a.charCodeAt(i);
}
</pre>
</div>

The input string a is converted character by character into 1byte values and stored in input_text_8.

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
keystream_8[0] = 0x37;

keystream_8.forEach((_, index, array) => {
    if (index == 0) return;
    array[index] = ((((array[index - 1] ^ 0x96) + 0xDD) ^ 0xA4) + 0x96) ^ 0xC8;
});
</pre>
</div>

The first byte is initialized to 0x37. Each subsequent byte is then generated using XOR and addition, based on the value of the previous byte.

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
input_text_32.forEach((_, index) => {
    result_32[index] = (((_ + keystream_32[index]) ^ reverseBits(keystream_32[index]) ) +  reverseBits(keystream_32[index])) ^ keystream_32[index];
});
</pre>
</div>

For each 32bit input block, it adds the corresponding keystream value, XORs the result with the reversed bits of the keystream, adds the reversed bits again, and finally XORs once more with the original keystream. The final encrypted value is stored in result_32.

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
return new Uint8Array(result);
</pre>
</div>

Finally, the result is returned as a Uint8Array.

### My Solution
<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
input_text_32.forEach((_, index) => {
    result_32[index] = (((_ + keystream_32[index]) ^ reverseBits(keystream_32[index]) ) +  reverseBits(keystream_32[index])) ^ keystream_32[index];
});
</pre>
</div>

I wrote the inverse operation here and applied a bitwise AND with 0xffffffff to each value to properly handle unsigned integers.

To understand how I reversed the encryption logic, let’s denote the variables as follow

- A = the original 32-bit plaintext block
- B = the corresponding keystream_32[index]
- C = the reversed bits of B = reverseBits(B)
- D = the encrypted result = result_32[index]

From the encryption code

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
D = (((A + B) ^ C) + C) ^ B
</pre>
</div>

I will solve this equation to recover A given B, C, D

<b>#1. XOR both sides with B</b>

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
D ^ B = ((A + B) ^ C) + C
</pre>
</div>

<b>#2. Subtract C from both sides</b>

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
(D ^ B) - C = (A + B) ^ C
</pre>
</div>

<b>#3. XOR with C again</b>

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
((D ^ B) - C) ^ C = A + B
</pre>
</div>

<b>#4. Finally, subtract B</b>

<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
A = (((D ^ B) - C) ^ C) - B
</pre>
</div>

### Solution Code
<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
result_32 = []

def byte2dword(a1):
&nbsp;&nbsp;&nbsp;&nbsp;ret = []
&nbsp;&nbsp;&nbsp;&nbsp;data = [a1[i:i + 4] for i in range(0, len(a1), 4)]<br>
&nbsp;&nbsp;&nbsp;&nbsp;for dword in data:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;rev_dword = dword[::-1]
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;hex_strings = '0x'+''.join([hex(byte)[2:] for byte in rev_dword])<br>
&nbsp;&nbsp;&nbsp;&nbsp;ret.append(int(hex_strings, 16))<br>
&nbsp;&nbsp;&nbsp;&nbsp;return ret

def dword2byte(a1):
&nbsp;&nbsp;&nbsp;&nbsp;ret = []<br>
&nbsp;&nbsp;&nbsp;&nbsp;for data in a1:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;word = [data[i:i + 2] for i in range(0, len(data), 2)][1:][::-1]
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ret += [chr(int('0x' + byte, 16)) for byte in word]<br>
&nbsp;&nbsp;&nbsp;&nbsp;return ''.join(ret)

enc_data = [164, 72, 70, 191, 200, 156, 172, 79, 52, 69, 146, 160, 106, 90, 169, 94, 108, 204, 156, 47, 106, 122, 198, 5, 206, 52, 249, 21, 70, 125, 172, 196, 96, 156, 186, 190, 81, 97, 105, 119]
enc_data = byte2dword(enc_data)
result_32 = [982366263,3203513355,3002446079,1983493363,1258145895,1323420795,1122353263,1189725283,2595035223,512826411]
enc1_result_32 = [990342231, -801271939, -15880371, 1737936439, 1930026921, 1863959481, 2063717281, 1662110641, -367479463, 445059375]

&#35; given values of B, C, D
B = 10
C = 20
D = 30

&#35; Calculation to find A
A = (((D ^ B) - C) ^ C) - B
D = (((A + B) ^ C) + C) ^ B

&#35; print A
print(A)
print(D)

&#35; result_32[index] = (((_ + key_stream_32[index]) ^ enc1(key_stream_32[index]) ) +  enc1(key_stream_32[index])) ^ key_stream_32[index];
flag_32 = []

for i in range(len(result_32)):
&nbsp;&nbsp;&nbsp;&nbsp;rev = (((enc_data[i] ^ result_32[i]) - enc1_result_32[i]) ^ enc1_result_32[i]) - result_32[i]<br>
&nbsp;&nbsp;&nbsp;&nbsp;if rev < 0:<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&#35; Convert to 32bit unsigned if negative
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;rev &= 0xffffffff<br>
&nbsp;&nbsp;&nbsp;&nbsp;flag_32.append(hex(rev))

print(f"FLAG: HCAMP{{{dword2byte(flag_32)}}}")
</pre>
</div>

### Decryption Completed and Flag Obtained
<div style="background-color: rgb(33, 37, 41); padding: 1em;">
<pre style="color: rgb(255, 255, 255); font-size: 1em; text-align: left;">
heapx@Mac tools % python3 solve.py
10
30
FLAG: HCAMP{40670b0248b9b931d3a6fe2d225dbb850c999ae7}
</pre>
</div>

### In Conclusion
When thinking back to the Hacking Camp, I'm really gratefull for the truly valuable experience for me upon reflection. Fortunately, our team won first place in the Hacking Camp CTF. I'm deeply grateful to my teammates from everywhere(어디에도)

As I close out this post, I want to mention that this was my first time writing an article in English. It was a new and meaningful experience for me.

So, I may have made some mistakes along the way. If you notice anything wrong or have any feedback about this post, please feel free to let me know!
