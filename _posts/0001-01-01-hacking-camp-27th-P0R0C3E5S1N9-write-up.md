---
layout: post
title:  "[HackingCamp 27th CTF] PR0C3E5S1N9 Write-up(for English)"
author: "SeungYong Lee"
categories: [CTF, Write-up]
tags: [HackingCamp, CTF, Write-up, Exploit, Reverse Engineering]
image: assets/images/hacking-camp-27th-P0R0C3E5S1N9-write-up/thumbnail.gif
---

Hello readers!

In the second half of 2023, I participated in the 27th Hacking Camp held in Korea.

Hacking Camp is a non-profit organization so by holding Hacking Camp, security dreamers can get great opportunities in Korea every year.

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

While I was thinking about how to analyze the code, I came up a website called beautifier.io. It improves the readability of input JavaScript code.
I inputted JavaScript code on the site. The website improved the code's readability, and I enhanced it further based on my own preferences. The code of below is the result from that

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
        },
    
    encrypt = (a) => {
        let arr1 = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
        let arr1_8 = new Uint8Array(arr1); // 1byte
        let arr1_32 = new Uint32Array(arr1); // 4byte

        let arr2 = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
        let arr2_8 = new Uint8Array(arr2); // 1byte
        let arr2_32 = new Uint32Array(arr2); // 4 byte

        let arr3 = new ArrayBuffer(a.length); // The buffer has space equal to a.length: 1byte * a.length
        let arr3_32 = new Uint32Array(arr3); // 4 byte
        
        arr1_8[0] = 0x37;

        for (let i = 0; i < a.length; i++) {
            arr2_8[i] = a.charCodeAt(i);
        };
        
        arr1_8.forEach((_, index, array) => {
            if (index == 0) return;
            array[index] = ((((array[index - 1] ^ 0x96) + 0xDD) ^ 0xA4) + 0x96) ^ 0xC8;
        });
        
        arr2_32.forEach((_, index) => {
            arr3_32[index] = (((_ + arr1_32[index]) ^ reverseBits(arr1_32[index]) ) +  reverseBits(arr1_32[index])) ^ arr1_32[index];
        });
        return new Uint8Array(arr3);
    };

flag_enc = [164, 72, 70, 191, 200, 156, 172, 79, 52, 69, 146, 160, 106, 90, 169, 94, 108, 204, 156, 47, 106, 122, 198, 5, 206, 52, 249, 21, 70, 125, 172, 196, 96, 156, 186, 190, 81, 97, 105, 119];
text = "HCAMP{AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}";

console.log(encrypt(text).map((element, index) => {
    console.log(element);
    if (flag_enc[index] != element) return false;
    return true;
}).reduce((accumulator, current_value) => accumulator * current_value, flag_enc['length'] == text['length']));
</pre>
</div>

Thereafter, I understood it step by step and then I realized that the "encrypt" function is core of this logic after that.

I realized that I  must needed to understand about ArrayBuffer of JavaScript and Uint(n)Array concepts in order to solving this challenge.

Actually, when I opened Chrome’s Developer Tools and entered new ArrayBuffer(8) in the console, a transistor-shaped icon appeared next to the output. click on that icon, it opens the Memory Inspector panel at the bottom, where you can view a memory dump of the object.

<p align="center">
    <img src="assets/images/hacking-camp-27th-P0R0C3E5S1N9-write-up/arraybuffer_dump.png">
</p>
