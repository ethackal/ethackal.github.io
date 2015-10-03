---
layout: post
title: Derbycon CTF - WAV Steganography
---

I recently attended Kentucky for DerbyCon, teaming up with several coworkers to participate in the Capture the Flag competition as Paid2Penetrate.  After 48 hours of hacking, and a near photo finish, we walked out of the CTF room in 3rd place. I knocked out two relatively high-value challenges in the first hours of the competition, setting us up for an early lead. The first was an encryption challenge in the form of three documents: a plaintext clue, the ciphertext of the clue, and an encrypted message containing the flag. The second challenge was presented as a WAV file and the directory structure of the server indicated that we'd have to extract the flag hidden within using steganography.

There have been a few write-ups of the encryption challenge, but I haven't seen any on the WAV steganography. While a little more involved, all that's required to recover the hidden flag is a basic understanding of steganographic techniques and knowledge of the WAV format.

> **ste·ga·no·graph·y**  
> /ˌsteɡəˈnäɡrəfi/ _noun_  
> the practice of concealing messages or information within other nonsecret text or data.

One of the most rudimentary digital steganography techniques is called _least significant bit (LSB) insertion_. This is often used with carrier file formats that involve lossless compression, such as is found in bitmap (BMP) images and WAV audio files. BMP is a little more straight forward to understand, so let's explore the technique in terms of digital images and then apply that to the WAV format used in the CTF.

Depending on the color depth used for an image, pixels may be composed of many bits that describe their color. One of the most common color depths uses 24 bits for each pixel, 3 bytes to determine the intensity of each primary color (Red, Green and Blue) and the fourth byte to encode an Alpha channel that describes a level of transparency. With a color space covering over 16 million possible variations, minor alterations are not visible to a human observer. A message can be inserted into a cover image by  adjusting the LSB of each channel to match a corresponding bit in the secret. There are a lot of problems with such a simplistic technique, but this is a CTF, not super spy-level stuff, so it's a good bet it's the type of steganography used.

With a little mental stretching, WAV can be thought of as the audio version of a BMP image. Instead of the pixels that make up digital pictures, the file contains a linear pulse-code modulation (LPCM) bitstream that encodes samples of an analog audio waveform at uniform intervals. The bitstream is usually uncompressed, though lossless compression is supported. Much like a BMP's pixels, adjustments to the LSB of each sample are inaudible and can be used to embed a hidden message one bit at a time.

With that background, we can start to look at extracting LSB inserted messages from the challenge file. The WAV format has been around a long time and most commonly used programming languages have libraries for processing them. Ruby is my go-to scripting language, so I immediately began by looking for a suitable gem, landing on [wav-file](https://github.com/shokai/ruby-wav-file). The README for wav-file contains an example that gets us a long way toward extracting the target data and has been adapted below. 

{% highlight ruby %}
require 'wav-file'

wav = open("Assignment1.wav")
format = WavFile::readFormat(wav)
chunk = WavFile::readDataChunk(wav)
wav.close
{% endhighlight %}

At this point, we've read in two chunks from the WAV file. The first provides format information and lets us know the bit-depth, or size of each sample. The second chunk is LPCM data; the samples that make up the encoded waveform. Examining the format chunk, we can see that the file is using 16-bit encoding, meaning each sample will be stored in a 16-bit signed integer. The gem example goes on to unpack the binary bitstream into an array of integers, but we don't really need to go through all of that. Instead, we can just read off 16-bits at a time and capture the LSB.

{% highlight ruby %}

{% endhighlight %}
