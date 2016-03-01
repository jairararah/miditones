# MIDITONES #

MIDITONES reads a standard MIDI file and outputs a simplified sequence of commands that can be played on a synthesizer having only tone generators without any volume or timbre controls.

This was written for the "Playtune" Arduino library (http://code.google.com/p/arduino-playtune/), which plays polyphonic music using up to 6 tone generators run by the timers on the processor.  Perhaps MIDITONES can prove useful for other tone generating systems too.

## **_THIS HAS MIGRATED TO https://www.github.com/lenshustek_** ##

---


**Update on 28 Feb 2011**: I fixed a bug that caused it to stop some notes too soon.

I also wrote a "MIDITONES\_SCROLL" program that displays a miditones bytestream as a time-ordered scroll, sort of like a piano roll but with non-uniform time.This is primarily useful to debug programming errors that cause some MIDI scripts to sound strange.  It reads the .bin file created from a .mid file by MIDITONES using the -b option.

**Update on 25 Apr 2011**: Scott Stickeler pointed out that it doesn't work if compiled for a 64-bit environment.  I'll put fixing that on my to-do list, but in the meantime the workaround is just to compile for 32 bits.  (Thanks, Scott.)

**Update on 20 Nov 2011, V1.4**: Added options to mask which channels (tracks) to process,
and to change key by any chromatic distance.  These are in support of music-playing on
my Tesla Coil.

**Update on 25 Aug 2013, V1.6**: I finally fixed it to compile in 64-bit environments. I didn't have a way to test that, so thanks to David Azar for doing so.

**Update on 30 Dec 2013**: I added version 1.1 of MINITONES\_SCROLL with a "-c" option
to create annotated C source code initialization of the music bytestream.
This makes it easier to manually edit the bytestream. See the beginning of the
MINITONES\_SCROLL source code for more details.

**Update on 7 Mar 2013**: I compiled 32-bit versions for people running Windows XP and Vista.  Unfortunately code.google.com no longer allows downloaded files!  So I put them in a Google Drive folder here:<br>
<a href='https://drive.google.com/folderview?id=0B1ZOnb_w5lfBQkNPeFpvRHdQNnc'>https://drive.google.com/folderview?id=0B1ZOnb_w5lfBQkNPeFpvRHdQNnc</a>

<b>Update on 5 April 2015: Now code.google.code is disappearing altogether. I have migrated this code to <a href='https://www.github.com/lenshustek'>https://www.github.com/lenshustek</a>. I will put new versions and deal with issues there.</b>
<pre><code>/*--------------------------------------------------------------------------------<br>
*<br>
*<br>
*                               About MIDITONES<br>
*<br>
*<br>
*  MIDITONES converts a MIDI music file into a much simplified stream of commands,<br>
*  so that a version of the music can be played on a synthesizer having only<br>
*  tone generators without any volume or tone controls.<br>
*<br>
*  Volume ("velocity") and instrument specifications in the MIDI files are discarded.<br>
*  All the tracks are prcoessed and merged into a single time-ordered stream of<br>
*  "note on", "note off", and "delay" commands.<br>
*<br>
*  This was written for the "Playtune" Arduino library, which plays polyphonic music<br>
*  using up to 6 tone generators run by the timers on the processor.  See the separate<br>
*  documentation for Playtune.  But MIDITONES may prove useful for other tone<br>
*  generating systems.<br>
*<br>
*  The output can be either a C-language source code fragment that initializes an<br>
*  array with the command bytestream, or a binary file with the bytestream itself.<br>
*<br>
*  MIDITONES is written in standard ANSI C (plus strlcpy and strlcat functions), and<br>
*  is meant to be executed from the command line.  There is no GUI interface.<br>
*<br>
*  The MIDI file format is complicated, and this has not been tested on a very<br>
*  wide variety of file types.  In particular, we have tested only format type "1",<br>
*  which seems to be what most of them are.  Let me know if you find MIDI files<br>
*  that it won't digest and I'll see if I can fix it.<br>
<br>
*  This has been tested only on a little-endian PC, but I think it should work on<br>
*  big-endian processors too.  Note that the MIDI file format is inherently<br>
*  big-endian.<br>
*<br>
*<br>
*  *****  The command line  *****<br>
*<br>
*  To convert a MIDI file called "chopin.mid" into a command bytestream, execute<br>
*<br>
*     miditones chopin<br>
*<br>
*  It will create a file in the same directory called "chopin.c" which contains<br>
*  the C-language statement to intiialize an array called "score" with the bytestream.<br>
*<br>
*<br>
*  The general form for command line execution is this:<br>
*<br>
*     miditones [-p] [-lg] [-lp] [-s1] [-tn] [-b] [-cn] [-kn] &lt;basefilename&gt;<br>
*<br>
*  The &lt;basefilename&gt; is the base name, without an extension, for the input and<br>
*  output files.  It can contain directory path information, or not.<br>
*<br>
*  The input file is the base name with the extension ".mid".  The output filename(s)<br>
*  are the base name with ".c", ".bin", and/or ".log" extensions.<br>
*<br>
*<br>
*  The following command-line options can be specified:<br>
*<br>
*  -p   Only parse the MIDI file;  don't generate an output file.<br>
*       Tracks are processed sequentially instead of being merged into chronological order.<br>
*       This is mostly useful when generating a log to debug MIDI file parsing problems.<br>
*<br>
*  -lp  Log input file parsing information to the &lt;basefilename&gt;.log file<br>
*<br>
*  -lg  Log output bytestream generation information to the &lt;basefilename&gt;.log file<br>
*<br>
*  -sn  Use bytestream generation strategy "n".<br>
*       Two strategies are currently implemented:<br>
*          1: favor track 1 notes instead of all tracks equally<br>
*          2: try to keep each track to its own tone generator<br>
*<br>
*  -tn  Generate the bytestream so that at most n tone generators are used.<br>
*       The default is 6 tone generators, and the maximum is 16.<br>
*       The program will report how many notes had to be discarded because there<br>
*       weren't enough tone generators.  Note that for the Arduino Playtunes<br>
*       library, it's ok to have the bytestream use more tone genreators than<br>
*       exist on your processor because any extra notes will be ignored, although<br>
*       it does make the file bigger than necessary . Of course, too many ignored<br>
*       notes will make the music sound really strange!<br>
*<br>
*  -b   Generate a binary file with the name &lt;basefilename&gt;.bin, instead of a<br>
*       C-language source file with the name &lt;basefilename&gt;.c.<br>
*<br>
*  -cn  Only process the channel numbers whose bits are on in the number "n".<br>
*       For example, -c3 means "only process channels 0 and 1"<br>
*<br>
*  -kn  Change the musical key of the output by n chromatic notes.<br>
*       -k-12 goes one octave down, -k12 goes one octave up, etc.<br>
*<br>
*<br>
*  *****  The score bytestream  *****<br>
*<br>
*  The generated bytestream is a series of commands that turn notes on and off, and<br>
*  start delays until the next note change.  Here are the details, with numbers<br>
*  shown in hexadecimal.<br>
*<br>
*  If the high-order bit of the byte is 1, then it is one of the following commands:<br>
*<br>
*    9t nn  Start playing note nn on tone generator t.  Generators are numbered<br>
*           starting with 0.  The notes numbers are the MIDI numbers for the chromatic<br>
*           scale, with decimal 60 being Middle C, and decimal 69 being Middle A (440 Hz).<br>
*<br>
*    8t     Stop playing the note on tone generator t.<br>
*<br>
*    F0     End of score: stop playing.<br>
*<br>
*    E0     End of score: start playing again from the beginning.<br>
*           (Shown for completeness; MIDITONES won't generate this.)<br>
*<br>
*  If the high-order bit of the byte is 0, it is a command to delay for a while until<br>
*  the next note change..  The other 7 bits and the 8 bits of the following byte are<br>
*  interpreted as a 15-bit big-endian integer that is the number of milliseconds to<br>
*  wait before processing the next command.  For example,<br>
*<br>
*     07 D0<br>
*<br>
*  would cause a delay of 0x07d0 = 2000 decimal millisconds, or 2 seconds.  Any tones<br>
*  that were playing before the delay command will continue to play.<br>
*<br>
*<br>
*  Len Shustek, 4 Feb 2011, and later<br>
*<br>
*----------------------------------------------------------------------------------*/<br>
</code></pre>