This is the initial ndstrim announcement on GBAtemp ([thread](http://gbatemp.net/index.php?showtopic=59580)):

# History: #
I want to share with you all my own nds rom trimmer.
I got my [R4](https://code.google.com/p/ndstrim/source/detail?r=4) a while back and needed a trimmer, I tried a few, but most of them were slow and didn't run on linux.
Anyway, after I had done some research I found out that there were two ways rom trimmers worked, let us call them 1st generation and 2nd generation.
1st generation just trimmed the roms from the end of the file until it found something else than the hex data 00 or FF, this often caused problems with wifi and download play.
2nd generation read the size of the actual rom which were written down at 4 bytes in the nds header, then they trimmed the rom to that size+136 bytes (for wifi data).
I started to make my own trimmer and I coded it to be fast, simple and reliable.
I finally came up with a really good trimmer which I'm now deciding to release as open source.

My trimmer is a 2nd generation trimmer, so it should be able to handle all the commercial roms without problems with wifi or download play.

# Download: #
In the file [ndstrim-1.0.tar.bz2](http://code.google.com/p/ndstrim/downloads/detail?name=ndstrim-1.0.tar.bz2) (you can open it with winrar/7-zip) you'll find the source, binaries for windows and linux as well as bat/sh-files to make it easier to trim many roms at once.
If you are planning to use ndstrim on windows, you only need the files "ndstrim.exe" and "trim.bat".
If you are planning to use ndstrim on linux, you only need the files "ndstrim" and "trim.sh" (make sure they are both executable - chmod +x ndstrim trim.sh).
Feel free to host this on any site you want, it's open source after all!

# How to make it all work: #
To trim roms, put them in the same directory as the files you unpacked and run trim.bat (or trim.sh if you are on linux), a window will popup and it will go through all the files ending in .nds in the same directory, trimming each one (and also telling you how much space you "saved" by trimming the roms).
The trimmed roms will appear in a subdirectory called "trimmed" with their original filename.
**Windows users:** Do not drag roms on ndstrim.exe, it won't work!

I compiled everything on a 32-bit computer (Windows XP + [MinGW](http://www.mingw.org/) and Ubuntu Linux 7.04 + gcc)
Even though it was compiled on a 32-bit computer it should be able to run fine on 64-bit computer (but I have not tested this).

Here's a screenshot of how it could look like (I did this on linux so the window might not look the same for you):
![http://ndstrim.googlecode.com/svn/wiki/ndstrim-1.0.png](http://ndstrim.googlecode.com/svn/wiki/ndstrim-1.0.png)

Running ndstrim without specifying output will show you how much space you will save when trimming the file (without the file being trimmed).

# Mac users: #
**Update:** n45800 was so kind to compile ndstrim 1.0 for Mac (Universal binary). Thanks n45800!
I have not tested the program on a Mac since I don't own one (nor have any experience with them), so there are no Mac binaries available from me.
Even so, ndstrim should work perfectly on Macs, it should only need to be compiled to work (I can't help you with this).
Let me know how it works on your Mac.

# Source code: #
Here are the source for those who are interested, the code should be very easy to understand even to novice programmers (ndstrim.c):
```
/* ndstrim by recover89@gmail.com
   Trims NDS roms fast and reliable.
   ROM size is available in four bytes at 0x80-0x83.
   Wifi data is stored on 136 bytes after ROM data.
   Filesize is checked to be at least 0x200 bytes to make sure it contains a DS cartridge header.
   Filesize is then checked to be at least the rom size+wifi to avoid errors.
   Source code licensed under GNU GPL version 2 or later.
   
   Sources:
   http://nocash.emubase.de/gbatek.htm
   http://forums.ds-xtreme.com/showthread.php?t=1964
   http://gbatemp.net/index.php?showtopic=44022
*/

#include <stdio.h>
#include <stdlib.h>

//#define DEBUG
#define BUFFER 1000000 //1 MB buffer size

//The rom size is located in four bytes at 0x80-0x83
struct uint32 {
	unsigned data:32; //4*8
};

int main(int argc, char *argv[]) {
	if (argc < 2) {
		fprintf(stderr,"%s: Too few arguments.\n",argv[0]);
		fprintf(stderr,"%s: Usage: %s <input> [output]\n",argv[0],argv[0]);
		exit(1);
	}
	
	//Debug?
	char debug=0;
	#ifdef DEBUG
	debug=1;
	#endif
	
	//Open input
	#ifdef DEBUG
	printf("%s: Opening input file '%s'.\n",argv[0],argv[1]);
	#endif
	FILE *input;
	if ((input=fopen(argv[1],"rb")) == NULL) {
		fprintf(stderr,"%s: fopen() failed in file %s, line %d.\n",argv[0],__FILE__,__LINE__);
		exit(1);
	}
	
	//Get input filesize
	fseek(input,0,SEEK_END);
	unsigned int filesize=ftell(input);
	#ifdef DEBUG
	printf("%s: Filesize: %d bytes.\n",argv[0],filesize);
	#endif
	
	//Check if file is big enough to contain a DS cartridge header
	if (filesize <= 0x200) {
		fprintf(stderr,"%s: Error: '%s' is too small to contain a NDS cartridge header (corrupt rom?).\n",argv[0],argv[1]);
		exit(1);
	}
	
	//Read rom size
	fseek(input,0x80,SEEK_SET);
	struct uint32 romsize;
	fread(&romsize,sizeof(romsize),1,input);
	unsigned int newsize=romsize.data+136; //Wifi data is located on 136 bytes after rom data
	
	//Check if file is big enough to contain the rom+wifi
	if (filesize < newsize) {
		fprintf(stderr,"%s: Error: '%s' is too small to contain the whole rom+wifi (corrupt rom?).\n",argv[0],argv[1]);
		exit(1);
	}
	
	//Check if this file seems to have been trimmed before
	if (filesize == newsize) {
		fprintf(stderr,"%s: Warning: '%s' is the same size as the trimmed rom will be (already trimmed?).\n",argv[0],argv[1]);
	}
	
	//Print info
	if (argc < 3 || debug) {
		printf("%s: ROM size: %d bytes.\n",argv[0],romsize.data);
		printf("%s: ROM size + wifi: %d bytes.\n",argv[0],newsize);
		printf("%s: Can save: %d bytes.\n",argv[0],(filesize-newsize));
	}
	
	//Output trimmed rom
	if (argc >= 3) {
		//Open output
		#ifdef DEBUG
		printf("%s: Opening output file '%s'.\n",argv[0],argv[2]);
		#endif
		FILE *output;
		if ((output=fopen(argv[2],"wb")) == NULL) {
			fprintf(stderr,"%s: fopen() failed in file %s, line %d.\n",argv[0],__FILE__,__LINE__);
			exit(1);
		}
		
		//Reset input pos
		rewind(input);
		
		//Start copying
		#ifdef DEBUG
		printf("%s: Copying data.\n",argv[0]);
		#endif
		char buffer[BUFFER];
		unsigned int fpos=0;
		unsigned int tocopy=BUFFER;
		while (fpos < newsize) {
			if (fpos+BUFFER > newsize) {
				tocopy=newsize-fpos;
			}
			fread(&buffer,tocopy,1,input);
			fwrite(&buffer,tocopy,1,output);
			fpos+=tocopy;
		}
		
		//Done
		printf("%s: Trimmed '%s' to %d bytes (saved %.2f MB).\n",argv[0],argv[1],newsize,(filesize-newsize)/(float)1000000);
		
		//Close output
		#ifdef DEBUG
		printf("%s: Closing output.\n",argv[0]);
		#endif
		if (fclose(output) == EOF) {
			fprintf(stderr,"%s: fclose() failed in file %s, line %d.\n",argv[0],__FILE__,__LINE__);
			exit(1);
		}
	}
	
	//Close input
	#ifdef DEBUG
	printf("%s: Closing input.\n",argv[0]);
	#endif
	if (fclose(input) == EOF) {
		fprintf(stderr,"%s: fclose() failed in file %s, line %d.\n",argv[0],__FILE__,__LINE__);
		exit(1);
	}
	
	return 0;
}

```

# Help me help you: #
This is a very simple trimmer and whilst it is very fast compared to other trimmers I have tried, of course I want to make it better...
I'm not very skilled in coding GUI so I don't think I'll be doing that anytime soon.
I tried to optimize read/write as much as I could, but I'm sure it's possible to make it even faster, if anyone have any experience with this, I'll gladly listen (currently: the program uses a 1 MB buffer which means it reads and writes in 1 MB chunks).

Any suggestions are welcome!