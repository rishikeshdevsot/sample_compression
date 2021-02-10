# sample_compression

## Versions
1. Version1: Using C++
2. Version2: Using python2


## Version 1
### Dependencies
1. libarchive-dev
2. openmp
3. g++

### Compile Instructions
1. `cd src`
2. `make` \
will output binaries `program_1` and `program_2`

### Runnning the decompressor
`cat <archive-location>.tar.gz | ./program_1 | ./program_2`

### Architecture
program_1 creates a struct archive object `ina` which access the `stdin` stream using the `archive_read_open_fd()` function by passing in 0(fd for stdin is 0) as the fd. Another archive object `outa` accesses the `stdout` stream by passing in `fd = 1`. The ina reads each archive entry in 256 byte chunks using the `archive_read_data()` function and outa writes this data in uncompressed form to `stdout`. program_2 then reads this uncompressed data and writes to disk using the `archive_read_extract()` function. 

### Testing
#### Edge cases : result
1. Empty archives : extracts to an empty folder
2. Archives with archives: Keeps the inner arhives i.e. doesn't perform recursive decompression
3. Archives with symlinks: Extracts the symlinks which still contain the same data(for both hard and soft links)
4. Archives with names pipes/fifos: Extracts the fifo without problems
5. No input/invalid input to program_1: program_1 would detect this, give error `Couldn't open archive fd to stdin` and kill itself, program_2 follows suit
6. Archives with different formats(zip, tar, etc): libarchive supports many compression formats: [found here](https://www.freebsd.org/cgi/man.cgi?query=archive_read_support_format_all&apropos=0&sektion=3&manpath=FreeBSD+12.2-RELEASE+and+Ports&arch=default&format=html )
#### Verification
Verification was perfomed using the `diff -ur <dir1> <dir2>` which recursively compares whether two directories are the same.

### Optimization
#### Benchmarks used
1. Sparse directory archive: Used the gnome [Papirus](https://www.gnome-look.org/s/Gnome/p/1166289) icone theme which contains the icons for each type of gnome application for many different sizes and different themes. As such it is very sparse and has several files with deep directory entries.
2. Large file arhive: Used `/dev/random` to generate a 1GB file with random data and archived it. 

#### Benchmark results
The main parameter that allowed for optimization is the size of the character buffer to copy data from `ina` to `outa`. If it is kept too small then it uses less memory but takes a long time to copy each entry, if it is too large it copies quickly but uses a large block of memory. After testing with several values 256 byte buffer seemed to give the best results for both benchmarks. Interestingly when using a buffer size which is too large(around 8192 bytes) was used, the copying speed degrades again, a possible cause could be the time taken to write the buffer to the stdout stream. 

### Previous Implementations
#### Blocking Implementation
The initial implementation was performed by reading from the `stdin` using `std::cin` with `getlines()`. Storing this input into a character buffer and passing it to `archive_read_open_memory()` function. This method was functional but time consuming. The input stream was blocked until entirely read and only then did decompression start. Upon realizing that `stdin` and `stdout` can be represented with fd 0 and fd 1. I switched the implementation to use `archive_read_open_fd()` which allowed streaming reads and writes giving better performance. The code structure was also neater when using this function

#### Multi-threaded implementation
This is a failed attempt at multi-threading the decompression and writing to `stdout` process using OpenMP. In this implementation threads start performing reads and writes to different archive entries in parallel. The pathnames of the entries which were already processed were added to a vector `addedEntries` which is checked before a new entry is processed to avoid conflicts. The implementation was working for an archive with few entries but upon testing it with the sparse directory benchmark, several races/conflicts were discovered and adding the synchornization primitives made the code comples. Also for sparse entries as the program processed more entries the vector `addedEntries` kept getting larger. So before processing an entry an `O(n)` string comparison operation takes place. When combined with the overarching loop which iterates over all the archives, the algorithm becomes `O(n^2)` thereby overshadowing the benefits of multithreading. Due to these reasons, this implementation was given up on.

