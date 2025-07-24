# CS:APP Malloc Lab Repository Guide

## Overview

This repository contains an implementation of a dynamic memory allocator (malloc) as part of the CS:APP (Computer Systems: A Programmer's Perspective) Malloc Lab. The project implements a custom memory management system using explicit free lists with immediate coalescing.

## Repository Structure

### Core Implementation Files
- **`mm.c`** - Main implementation file containing the malloc, free, and realloc functions
- **`mm.h`** - Header file defining the malloc interface and team information structure
- **`memlib.h/c`** - Memory library that models the heap and provides sbrk functionality

### Testing and Evaluation
- **`mdriver.c`** - Test driver program that evaluates the malloc implementation
- **`mdriver`** - Compiled test driver executable
- **`short1-bal.rep`, `short2-bal.rep`** - Small trace files for initial testing
- **`Makefile`** - Build configuration for compiling the project

### Timing and Measurement
- **`clock.h/c`** - Low-level timing routines using cycle counters
- **`fcyc.h/c`** - Timer functions based on cycle counters
- **`ftimer.h/c`** - Timer functions using interval timers and gettimeofday()
- **`fsecs.h/c`** - Wrapper functions for different timer packages

### Configuration
- **`config.h`** - Configuration settings for the malloc lab driver
- **`README`** - Original lab instructions and file descriptions

## Implementation Details

### Memory Allocator Design

The current implementation uses an **explicit free list** approach with the following characteristics:

#### Data Structures
- **Block Structure**: Each memory block has a header and footer containing size and allocation status
- **Free List**: Free blocks are organized in a doubly-linked list for efficient management
- **Alignment**: 8-byte alignment is maintained for all allocated blocks

#### Key Components

1. **Block Layout**:
   ```
   Allocated Block:    Free Block:
   +--------+          +--------+
   | Header |          | Header |
   +--------+          +--------+
   | Data   |          | Next   |
   |   ...  |          +--------+
   |        |          | Prev   |
   +--------+          +--------+
   | Footer |          | Footer |
   +--------+          +--------+
   ```

2. **Macros and Constants**:
   - `WSIZE = 8`: Word size (8 bytes for 64-bit systems)
   - `DSIZE = 16`: Double word size
   - `CHUNKSIZE = 4096`: Default heap extension size

3. **Core Functions**:
   - `mm_init()`: Initialize the heap and free list
   - `mm_malloc(size)`: Allocate memory block
   - `mm_free(ptr)`: Free memory block
   - `mm_realloc(ptr, size)`: Reallocate memory block

#### Algorithms Used

1. **Placement Policy**: First-fit search through the explicit free list
2. **Splitting**: Large free blocks are split when possible to minimize internal fragmentation
3. **Coalescing**: Immediate coalescing of adjacent free blocks using boundary tags
4. **Free List Management**: LIFO (Last-In-First-Out) insertion policy

## How to Work with This Repository

### Building the Project

```bash
# Clean previous builds
make clean

# Build the malloc driver
make

# Or build specific components
make mdriver
```

### Testing Your Implementation

#### Basic Testing
```bash
# Test with small trace files (verbose output)
./mdriver -V -f short1-bal.rep
./mdriver -V -f short2-bal.rep

# Test without verbose output
./mdriver -f short1-bal.rep
```

#### Advanced Testing Options
```bash
# Print per-trace performance breakdowns
./mdriver -v

# Compare with libc malloc
./mdriver -l

# Generate summary for autograder
./mdriver -g

# Use specific trace directory
./mdriver -t /path/to/traces
```

### Understanding Test Output

The driver evaluates your implementation on two metrics:

1. **Utilization (util)**: How efficiently you use the allocated heap space
2. **Throughput (thru)**: How fast your allocator performs operations

Example output:
```
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   66%      12  0.000000 60000
Total          66%      12  0.000000 60000

Perf index = 40 (util) + 40 (thru) = 80/100
```

- **valid**: Whether the implementation passes correctness checks
- **util**: Utilization percentage (higher is better)
- **ops**: Number of operations performed
- **Kops**: Throughput in thousands of operations per second
- **Perf index**: Combined score out of 100

### Development Workflow

#### 1. Understand the Current Implementation
- Review the explicit free list implementation in `mm.c`
- Understand the coalescing strategy and block structure
- Analyze the first-fit placement policy

#### 2. Identify Improvement Areas
Common optimization strategies:
- **Better Placement Policies**: Implement best-fit or next-fit
- **Segregated Free Lists**: Use multiple free lists for different size classes
- **Advanced Coalescing**: Implement deferred coalescing strategies
- **Memory Layout Optimization**: Reduce overhead, improve cache locality

#### 3. Implement Changes
- Modify only the `mm.c` file (as per lab requirements)
- Test frequently with the provided trace files
- Maintain the required interface (`mm_init`, `mm_malloc`, `mm_free`, `mm_realloc`)

#### 4. Testing and Validation
```bash
# Always test after changes
make clean && make
./mdriver -V -f short1-bal.rep

# Test multiple traces if available
./mdriver -v
```

### Debugging Tips

#### 1. Use Verbose Mode
```bash
./mdriver -V -f short1-bal.rep
```
This provides detailed information about each malloc/free operation.

#### 2. Add Debug Prints
Insert debugging code in `mm.c`:
```c
#ifdef DEBUG
printf("malloc: size=%zu, returning %p\n", size, bp);
#endif
```

#### 3. Heap Consistency Checking
Implement a heap checker function to validate your data structures:
```c
static void heap_check(void) {
    // Check heap consistency
    // Validate free list integrity
    // Verify block headers/footers match
}
```

### Performance Optimization Guidelines

#### Utilization Improvements
- Minimize internal fragmentation through better splitting strategies
- Reduce external fragmentation with improved coalescing
- Optimize block sizes and alignment requirements

#### Throughput Improvements
- Use faster search algorithms (e.g., segregated lists)
- Reduce the number of heap traversals
- Optimize frequently-used operations

### Common Issues and Solutions

1. **Segmentation Faults**
   - Check pointer arithmetic in macros
   - Validate block boundaries
   - Ensure proper initialization

2. **Low Utilization**
   - Review splitting strategy
   - Check for excessive padding
   - Optimize minimum block sizes

3. **Poor Throughput**
   - Profile the find_fit function
   - Consider alternative data structures
   - Optimize coalescing operations

## Current Implementation Status

The repository contains a working explicit free list implementation with:
- ✅ Basic malloc/free/realloc functionality
- ✅ Immediate coalescing
- ✅ First-fit placement policy
- ✅ 8-byte alignment
- ✅ Basic performance (80/100 on test traces)

Recent changes (based on Git history):
- Implemented explicit free list with coalescing
- Added first-fit placement strategy
- Integrated with testing framework

## Next Steps for Development

1. **Analyze Performance**: Profile current implementation to identify bottlenecks
2. **Implement Optimizations**: Consider segregated free lists or better placement policies
3. **Expand Testing**: Add more comprehensive test cases
4. **Documentation**: Add inline comments explaining complex algorithms
5. **Validation**: Implement comprehensive heap checking functionality

This implementation provides a solid foundation for a dynamic memory allocator and serves as an excellent starting point for further optimizations and enhancements.