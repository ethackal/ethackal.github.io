---
layout: post
title: Derbycon CTF - WAV Steganography
---

I recently attended DerbyCon in Louisville, Kentucky, teaming up with several co-workers to participate in the Capture the Flag competition as [Paid2Penetrate](https://www.youtube.com/watch?v=JGhoLcsr8GA).  After 48 hours of hacking, and a near photo finish, we walked out of the CTF room in 3rd place. I am particularly proud of having knocked out two relatively high-value challenges in the early hours of the competition. The first was an encryption challenge in the form of three documents: a plaintext clue, the ciphertext of the clue, and an encrypted message containing the flag. The second challenge was presented as a WAV file and the directory naming where the file was found hinted that we'd have to extract the flag hidden within using steganography.

There have been a few write-ups of the encryption challenge, but I have yet to see any on the WAV steganography. While a little more involved, all that is required to recover the hidden flag is a basic understanding of steganographic techniques and knowledge of the WAV format.

> **ste·ga·no·graph·y**  
> /ˌsteɡəˈnäɡrəfi/ _noun_  
> the practice of concealing messages or information within other nonsecret text or data.

One of the most rudimentary digital steganography techniques is called _least significant bit (LSB) insertion_. This is often used with carrier file formats that involve lossless compression, such as is found in bitmap (BMP) images and WAV audio files. BMP is a little more straight forward to understand, so let's explore the technique in terms of digital images and then apply that to the WAV format used in the CTF.

Depending on the color depth used for an image, pixels may be composed of many bits that describe their color. One of the most common color depths uses 24 bits for each pixel, 8 bits to determine the intensity of each primary color (Red, Green and Blue). With a color space covering over 16 million possible variations, minor alterations are not visible to a human observer. A message can be inserted into a cover image by  adjusting the LSB of each channel to match a corresponding bit in the secret. There are a lot of problems with such a simplistic technique, but this is a CTF, not super spy-level stuff, so it's a good bet for the type of steganography used.

With a little mental stretching, WAV can be thought of as the audio version of a BMP image. Instead of the pixels that make up digital pictures, the file contains a linear pulse-code modulation (LPCM) bitstream that encodes samples of an analog audio waveform at uniform intervals. The bitstream is usually uncompressed, though lossless compression is supported. Much like a BMP's pixels, adjustments to the LSB of each sample are inaudible and can be used to embed a hidden message one bit at a time.

With that background, we can start to look at extracting LSB inserted messages from the challenge file. The WAV format has been around a long time and most commonly used programming languages have libraries for processing them. Ruby is my go-to scripting language, so I immediately began by looking for a suitable gem, landing on [wav-file](https://github.com/shokai/ruby-wav-file). The README for wav-file contains an example that gets us a long way toward extracting the target data and has been adapted below. 

{% highlight ruby %}
require 'wav-file'

wav = open("Assignment1.wav")
format = WavFile::readFormat(wav)
# <WavFile::Format:0x007f872a034860 @id=1, @channel=2, @hz=44100, @bytePerSec=176400, @blockSize=4, @bitPerSample=16> 
chunk = WavFile::readDataChunk(wav)

wav.close
{% endhighlight %}

At this point, we've read in two chunks from the WAV file. The first provides format information and lets us know the bit-depth, or size of each sample. The second chunk is LPCM data; the samples that make up the encoded waveform. Inspecting the format chunk, we can see that the file is using 16-bit encoding, meaning each sample will be stored in a 16-bit signed integer. We can [#unpack](http://www.rubydoc.info/stdlib/core/String:unpack) the binary data to expand each sample into an array of the 16-bit integers.

{% highlight ruby %}
wavs = chunk.data.unpack('s*')
{% endhighlight %}

Now, to get the LSB for each sample. Ruby provides bit reference access via [Fixnum#[]](http://www.rubydoc.info/stdlib/core/Fixnum:%5B%5D), where index 0 represents the LSB. Easy enough to [#map](http://www.rubydoc.info/stdlib/core/Array:map) the array of unpacked values and [#join](http://www.rubydoc.info/stdlib/core/Array:join) that result into a string of binary digits while we're at it.

{% highlight ruby %}
lsb = wavs.map{|sample| sample[0]}.join
# 00000000000000000000000000000000000000<snip a lot of zeros>...
# 1000110001100110100101101100111000010110010011000110001010010110110011100001011011001100011001101001011011001110000101100100001000110110101011101010011001100010100101101100111000010110
{% endhighlight %}

This is unexpected... There are a whole lot of zeros there. It looks like every sample has a LSB of 0 except for the those at the very end of the chunk, starting at index 1146416. Let's grab the binary starting at the first occurence of '1' and see what we can make of it by [#pack](http://www.rubydoc.info/stdlib/core/Array:pack)ing it back into a string.

{% highlight ruby %}
flag = lsb[(lsb.index('1'))..-1]
# "1000110001100110100101101100111000010110010011000110001010010110110011100001011011001100011001101001011011001110000101100100001000110110101011101010011001100010100101101100111000010110"
puts [flag].pack('b*')
# 1fish2Fish3fishBlueFish
{% endhighlight %}

That looks like a flag! The only thing left to do is submit it for 400 points!

[Challenge WAV]({{ site.url }}/resources/derbycon-ctf-wav-steganography/Assignment1.wav)

Complete code for extracting the flag:

{% highlight ruby %}
require 'wav-file'

wav = open("Assignment1.wav")
format = WavFile::readFormat(wav)
chunk = WavFile::readDataChunk(wav)

wav.close

wavs = chunk.data.unpack('s\*')
lsb = wavs.map{|sample| sample[0]}.join
flag = lsb[(lsb.index('1'))..-1]
puts [flag].pack('b*')
{% endhighlight %}
