# MiniVSFS-A-C-based-VSFS-Image-Generator
This is a project where we will build a C program that will create a raw disk image and another program that will take a MiniVSFS raw file system image and a file to be added to the file system.


## MiniVSFS
MiniVSFS, based on VSFS, is fairly simple – a block-based file system structure with a superblock, inode and data bitmaps, inode tables, and data blocks. Compared to the regular VSFS, MiniVSFS cuts a few corners:

Indirect pointer mechanism is not implemented
Only supported directory is the root (/) directory
Only one block each for the inode and data bitmap
Limited size and inode counts

## Program Workflow
1. MKFS_BUILDER
It should perform the following tasks in order:
Parse the command line inputs
Create the file system according to the provided specifications
Save the file system as a binary file with the name specified by the --image flag
2. MKFS_ADDER
It should perform the following tasks in order:
Parse the command line inputs
Open the input image as a binary file
Search the file in the present working directory, and add the file to the file system.
Update the file system binary image


## MiniVSFS Specifications
Block Size = 4096 Bytes
Inode Size = 128 Bytes
Total Blocks = size_kib * 1024 / 4096




## Project COntribution: 
# Yeasin FIyaz 
&
# Abrar Shahrier Arnob
