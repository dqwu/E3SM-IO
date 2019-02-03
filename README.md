## Parallel I/O Kernel Case Study -- E3SM

This repository contains a case study of parallel I/O kernel from the
[E3SM](https://github.com/E3SM-Project/E3SM) climate simulation model. The
benchmark program, e3sm_io.c, can be used to evaluate
[PnetCDF](https://github.com/Parallel-NetCDF/PnetCDF) library for its
performance on the I/O patterns used by E3SM. One of the most challenging I/O
patterns used in E3SM is the cubed sphere variables whose I/O pattern consists
of a long list of small and noncontiguous requests on every MPI process.

The benchmark program evaluates two implementations of using PnetCDF blocking
vard and nonblocking varn APIs separately. Both APIs can aggregate multiple
requests across variables. In addition to timings and I/O bandwidths, the
benchmark reports the PnetCDF internal memory footprint (high water mark).

The I/O patterns (data decompositions among MPI processes) used in this case
study were captured by the [PIO](https://github.com/NCAR/ParallelIO) library.
A data decomposition file records the data access patterns at the array element
level for each MPI process. The patterns are stored in a text file, referred by
PIO as the `decomposition file`. Note E3SM uses three unique data decomposition
patterns shared by its 381 2D and 3D variables.

* Prepare the data decomposition file
  * The three data decomposition files generated by PIO library are in text
    format with file extension name `.dat`. The decomposition files must first
    be combined and converted into a NetCDF file, to be read in parallel as the
    input file by this benchmark program.
  * A utility program, `dat2nc.c`, is provided to combine the three text files.
    To build this utility program, run command `make dat2nc`.
  * The command to combine the three dat files to a NetCDF file is:
    ```
      % ./dat2nc -o outputfile.nc -1 decomp_1.dat -2 decomp_2.dat -3 decomp_3.dat
    ```
  * Command-line options of `./dat2nc`:
    ```
      % ./dat2nc -h
      Usage: ./dat2nc [OPTION]... [FILE]...
            -h               Print help
            -v               Verbose mode
            -l num           Max number of characters per line in input file
            -o out_file      Name of output netCDF file
            -1 input_file    Name of 1st 1D decomposition file
            -2 input_file    Name of 2nd 1D decomposition file
            -3 input_file    Name of     2D decomposition file
    ```
  * Three small input decomposition files are provide in directory `datasets/`.
    * `piodecomp16tasks16io01dims_ioid_514.dat`  (decomposition along the fastest dimensions)
    * `piodecomp16tasks16io01dims_ioid_516.dat`  (decomposition along the fastest dimensions)
    * `piodecomp16tasks16io02dims_ioid_548.dat`  (decomposition along the fastest two dimensions)
  * The NetCDF file combined from these 3 decomposition dat files is provided
    in folder `datasets` named `866x72_16p.nc`. Its metadata is shown below.
    ```
      % cd ./datasets
      % ncmpidump -h 866x72_16p.nc
      netcdf 866x72_16p {
      // file format: CDF-1
      dimensions:
          num_procs = 16 ;
          D3.max_nreqs = 4032 ;
          D2.max_nreqs = 56 ;
          D1.max_nreqs = 70 ;
      variables:
          int D3.nreqs(num_procs) ;
                D3.nreqs:description = "Number of noncontiguous subarray requests by each MPI process" ;
          int D3.offsets(num_procs, D3.max_nreqs) ;
                D3.offsets:description = "Flattened starting indices of noncontiguous requests. Each row corresponds to requests by an MPI process." ;
          int D2.nreqs(num_procs) ;
                D2.nreqs:description = "Number of noncontiguous subarray requests by each MPI process" ;
          int D2.offsets(num_procs, D2.max_nreqs) ;
                D2.offsets:description = "Flattened starting indices of noncontiguous requests. Each row corresponds to requests by an MPI process." ;
          int D1.nreqs(num_procs) ;
                D1.nreqs:description = "Number of noncontiguous subarray requests by each MPI process" ;
          int D1.offsets(num_procs, D1.max_nreqs) ;
                D1.offsets:description = "Flattened starting indices of noncontiguous requests. Each row corresponds to requests by an MPI process." ;

      // global attributes:
          :dim_len_Y = 72 ;
          :dim_len_X = 866 ;
          :D3.max_nreqs = 4032 ;
          :D3.min_nreqs = 3744 ;
          :D2.max_nreqs = 56 ;
          :D2.min_nreqs = 52 ;
          :D1.max_nreqs = 70 ;
          :D1.min_nreqs = 40 ;
      }
    ```

* Compile command to build the executable file, `e3sm_io`:
  * Edit `Makefile` to customize the MPI compiler, compile options, location of
    PnetCDF library, etc.
  * The minimum required PnetCDF version is 1.11.0.
  * Run command `make e3sm_io` to generate the executable program named
    `e3sm_io`.

* Run command:
  * An example run command using `mpiexec` and 16 MPI processes is:
    `mpiexec -n 16 ./e3sm_io datasets/866x72_16p.nc`
  * The number of MPI processes can be different from the value of the variable
    `num_procs` stored in the decomposition NetCDF file. For example, in file
    `866x72_16p.nc`, `num_procs` is 16, the number of MPI processes originally
    used to produce the decomposition dat files. When using less number of MPI
    processes to run this benchmark, the workload will be divided among all
    the allocated MPI processes. When using more processes, those processes
    with MPI ranks greater than or equal to 16 will have no data to write but
    still participate the collective I/O in the benchmark.
  * Command-line options:
    ```
      % ./e3sm_io -h
      Usage: ./e3sm_io [OPTION]... FILE
             [-h] Print help
             [-v] Verbose mode
             [-k] Keep the output files when program exits
             [-d] Run test that uses PnetCDF vard API
             [-n] Run test that uses PnetCDF varn API
             [-m] Run test using noncontiguous write buffer
             [-t] Write 2D variables followed by 3D variables
             [-r num] Number of records (default 1)
             [-o output_dir] Output directory name (default ./)
             FILE: Name of input netCDF file describing data decompositions
    ```
  * An example batch script file for running a job on Cori @NERSC with 8 KNL
    nodes, 64 MPI processes per node is provided in `./slurm.knl`.
  * A median-size decomposition file `datasets/48602x72_512p.nc.gz` contains
    the I/O pattern from a bigger problem size from an E3SM experiment ran on
    512 MPI processes.

* Example outputs shown on screen
  ```
    % mpiexec -n 512 ./e3sm_io -k -r 3  -o $SCRATCH/FS_1M_64 datasets/48602x72_512p.nc

    Total number of MPI processes      = 512
    Input decomposition file           = $SCRATCH/FS_1M_64/48602x72_512p.nc
    Output file directory              = $SCRATCH/FS_1M_64
    Variable dimensions (C order)      = 72 x 48602
    Write number of records (time dim) = 3
    Using noncontiguous write buffer   = no

    ==== benchmarking vard API ================================
    Variable written order: same as variables are defined

    History output file                = testfile_h0_vard.nc
    MAX heap memory allocated by PnetCDF internally is 2.22 MiB
    Total number of variables          = 408
    Total write amount                 = 2699.36 MiB = 2.64 GiB
    Max number of requests             = 310598
    Max Time of open + metadata define = 0.0533 sec
    Max Time of I/O preparing          = 0.1156 sec
    Max Time of ncmpi_put_vard         = 5.4311 sec
    Max Time of close                  = 0.0306 sec
    Max Time of TOTAL                  = 5.6385 sec
    I/O bandwidth (open-to-close)      = 478.7341 MiB/sec
    I/O bandwidth (write-only)         = 496.9981 MiB/sec
    -----------------------------------------------------------
    History output file                = testfile_h1_vard.nc
    MAX heap memory allocated by PnetCDF internally is 2.22 MiB
    Total number of variables          = 51
    Total write amount                 = 52.30 MiB = 0.05 GiB
    Max number of requests             = 6022
    Max Time of open + metadata define = 0.0338 sec
    Max Time of I/O preparing          = 0.0014 sec
    Max Time of ncmpi_put_vard         = 0.2489 sec
    Max Time of close                  = 0.0055 sec
    Max Time of TOTAL                  = 0.2907 sec
    I/O bandwidth (open-to-close)      = 179.8902 MiB/sec
    I/O bandwidth (write-only)         = 210.1002 MiB/sec
    -----------------------------------------------------------

    ==== benchmarking varn API ================================
    Variable written order: same as variables are defined

    History output file                = testfile_h0_varn.nc
    MAX heap memory allocated by PnetCDF internally is 35.07 MiB
    Total number of variables          = 408
    Total write amount                 = 2699.36 MiB = 2.64 GiB
    Max number of requests             = 310464
    Max Time of open + metadata define = 0.0635 sec
    Max Time of I/O preparing          = 0.0018 sec
    Max Time of ncmpi_iput_varn        = 0.2468 sec
    Max Time of ncmpi_wait_all         = 5.8602 sec
    Max Time of close                  = 0.0190 sec
    Max Time of TOTAL                  = 6.2001 sec
    I/O bandwidth (open-to-close)      = 435.3753 MiB/sec
    I/O bandwidth (write-only)         = 460.6144 MiB/sec
    -----------------------------------------------------------
    History output file                = testfile_h1_varn.nc
    MAX heap memory allocated by PnetCDF internally is 35.07 MiB
    Total number of variables          = 51
    Total write amount                 = 52.30 MiB = 0.05 GiB
    Max number of requests             = 5888
    Max Time of open + metadata define = 0.0370 sec
    Max Time of I/O preparing          = 0.0005 sec
    Max Time of ncmpi_iput_varn        = 0.0048 sec
    Max Time of ncmpi_wait_all         = 0.2423 sec
    Max Time of close                  = 0.0058 sec
    Max Time of TOTAL                  = 0.2925 sec
    I/O bandwidth (open-to-close)      = 178.7747 MiB/sec
    I/O bandwidth (write-only)         = 215.7512 MiB/sec
    -----------------------------------------------------------
  ```
* Output files
  * The above example command uses command-line option `-k` to keep the output
    files (otherwise the default is to delete them when program exits.) Each
    run of `e3sm_io` produces four output netCDF files named
    `testfile_h0_vard.nc`, `testfile_h1_vard.nc`, `testfile_h0_varn.nc`, and
    `testfile_h1_varn.nc`. The contents of h0 and h1 files should be the same
    between vard and varn APIs. Their file header (metadata) obtainable by
    command `ncdump -h` from running the provided decomposition file
    `866x72_16p.nc` is available in
    [datasets/outputfile_h0.txt](datasets/outputfile_h0.txt),
    and
    [datasets/outputfile_h1.txt](datasets/outputfile_h1.txt).

## Questions/Comments:
email: wkliao@eecs.northwestern.edu

Copyright (C) 2018, Northwestern University.

See [COPYRIGHT](COPYRIGHT) notice in top-level directory.

