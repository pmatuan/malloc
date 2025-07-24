# Memory Allocator Implementation Document

## Overview

This document provides a detailed implementation analysis of the dynamic memory allocator in `mm.c`. The implementation uses an **explicit free list** with **immediate coalescing** and **boundary tag** technique for efficient memory management.

## Overall Design

### Memory Management Strategy
- **Organization**: Explicit free list using doubly-linked list of free blocks
- **Placement Policy**: First-fit allocation strategy
- **Coalescing**: Immediate coalescing upon freeing blocks
- **Block Structure**: Boundary tags (headers and footers) for all blocks
- **Alignment**: 8-byte alignment for all allocated blocks

### Key Design Decisions
1. **Explicit Free List**: Maintains a doubly-linked list of free blocks for O(free blocks) search time instead of O(total blocks)
2. **LIFO Free List Management**: New free blocks are added to the beginning of the free list
3. **Immediate Coalescing**: Free blocks are immediately coalesced with adjacent free blocks
4. **Boundary Tags**: Both headers and footers store size and allocation status for constant-time coalescing

## Block Layout and Data Structures

### Block Structure
```
Allocated Block:
+--------+--------+--------+--------+
| Header |      Payload     | Footer |
+--------+--------+--------+--------+
^                            ^
bp-8                        bp+size-8

Free Block:
+--------+--------+--------+--------+--------+--------+
| Header | Next* | Prev* | ... Unused ... | Footer |
+--------+--------+--------+--------+--------+--------+
^        ^
bp-8     bp (points to next free block pointer)
```

### Header/Footer Format
- **Size**: 29 bits storing block size (including header and footer)
- **Allocation bit**: 1 bit indicating if block is allocated (1) or free (0)
- **Padding**: 2 bits unused (for 8-byte alignment)

### Key Constants and Macros

```c
#define WSIZE       8       /* Word size (8 bytes for 64-bit) */
#define DSIZE       16      /* Double word size */
#define CHUNKSIZE  (1<<12)  /* Initial heap extension (4KB) */
#define ALIGNMENT  8        /* 8-byte alignment requirement */
```

## Function Implementations

### `mm_init()` - Heap Initialization

**Purpose**: Initializes the heap and sets up initial data structures.

**Implementation Details**:
1. **Initial Heap Layout**: Creates a 4-word initial heap:
   ```
   +--------+--------+--------+--------+
   |  Pad   |Prologue|Prologue|Epilogue|
   |   0    | Header | Footer | Header |
   +--------+--------+--------+--------+
   ```

2. **Setup Process**:
   - Allocates 4 words (32 bytes) using `mem_sbrk()`
   - Sets alignment padding (word 0)
   - Creates prologue block (words 1-2): prevents coalescing with heap start
   - Creates epilogue header (word 3): marks heap end
   - Initializes `heap_listp` to point to prologue footer
   - Initializes `free_listp` to NULL (empty free list)

3. **Initial Extension**: Extends heap with `CHUNKSIZE` bytes to create first free block

**Return Value**: 0 on success, -1 on failure

### `mm_malloc(size_t size)` - Block Allocation

**Purpose**: Allocates a block of at least `size` bytes with proper alignment.

**Implementation Steps**:

1. **Size Validation**: Returns NULL for size 0 requests

2. **Size Adjustment**: 
   ```c
   if (size <= DSIZE)                                          
       asize = 2*DSIZE;  // Minimum block size (32 bytes)                                    
   else
       asize = DSIZE * ((size + DSIZE + (DSIZE-1)) / DSIZE);  // Align to 16-byte boundary
   ```
   - Ensures minimum block size of 32 bytes (header + footer + 16 bytes payload)
   - Rounds up to nearest 16-byte boundary for alignment

3. **Block Search**: Uses `find_fit()` to search free list with first-fit policy

4. **Block Placement**: If fit found, uses `place()` to allocate and potentially split block

5. **Heap Extension**: If no fit found:
   - Extends heap by `MAX(asize, CHUNKSIZE)` bytes
   - Places block in newly created space

**Alignment Strategy**: Ensures all blocks start on 8-byte boundaries and have sizes that are multiples of 8.

### `mm_free(void *ptr)` - Block Deallocation

**Purpose**: Marks block as free and coalesces with adjacent free blocks.

**Implementation Steps**:

1. **Null Pointer Check**: Returns immediately if `ptr` is NULL

2. **Block Marking**: 
   - Retrieves block size from header
   - Sets allocation bit to 0 in both header and footer
   ```c
   PUT(HDRP(bp), PACK(size, 0));
   PUT(FTRP(bp), PACK(size, 0));
   ```

3. **Immediate Coalescing**: Calls `coalesce()` to merge with adjacent free blocks

**Key Feature**: Uses immediate coalescing strategy to reduce external fragmentation.

### `mm_realloc(void *ptr, size_t size)` - Block Resizing

**Purpose**: Changes the size of previously allocated block.

**Implementation Strategy**: Simple approach using malloc/copy/free pattern.

**Edge Cases Handled**:
1. **NULL pointer**: Equivalent to `mm_malloc(size)`
2. **Zero size**: Equivalent to `mm_free(ptr)`, returns NULL
3. **Normal case**: 
   - Allocates new block with `mm_malloc(size)`
   - Copies data from old block (minimum of old size and new size)
   - Frees old block with `mm_free(ptr)`

**Limitation**: Does not optimize in-place expansion/contraction.

## Helper Functions

### `extend_heap(size_t words)` - Heap Extension

**Purpose**: Extends heap by specified number of words and returns pointer to new free block.

**Process**:
1. Adjusts word count to maintain alignment (even number of words)
2. Calls `mem_sbrk()` to extend heap
3. Initializes new block header/footer as free
4. Updates epilogue header to new heap end
5. Coalesces with previous block if it was free

### `find_fit(size_t asize)` - Free Block Search

**Purpose**: Implements first-fit search through explicit free list.

**Algorithm**:
- Traverses free list from head (`free_listp`)
- Returns first block with size ≥ `asize`
- Returns NULL if no suitable block found

**Time Complexity**: O(number of free blocks)

### `place(void *bp, size_t asize)` - Block Placement and Splitting

**Purpose**: Allocates block and splits if remainder is large enough.

**Splitting Logic**:
- If remaining space ≥ 32 bytes (4 words): split block
- Otherwise: allocate entire block to avoid tiny fragments

**Process**:
1. Removes block from free list
2. If splitting:
   - Marks first part as allocated
   - Creates new free block from remainder
   - Adds remainder to free list
3. If not splitting: marks entire block as allocated

### `coalesce(void *bp)` - Block Coalescing

**Purpose**: Merges newly freed block with adjacent free blocks.

**Four Coalescing Cases**:

1. **Case 1**: Both neighbors allocated
   - Simply add block to free list

2. **Case 2**: Next block free, previous allocated
   - Merge with next block
   - Remove next block from free list
   - Update headers/footers

3. **Case 3**: Previous block free, next allocated  
   - Merge with previous block
   - Remove previous block from free list
   - Update headers/footers

4. **Case 4**: Both neighbors free
   - Merge all three blocks
   - Remove both neighbors from free list
   - Update headers/footers

**Boundary Tag Advantage**: Constant-time access to adjacent block headers/footers enables efficient coalescing.

### Free List Management Functions

#### `add_to_free_list(void *bp)` - Add Block to Free List

**Strategy**: LIFO (Last In, First Out) insertion
- Adds new free block at head of list
- Updates doubly-linked list pointers
- O(1) time complexity

#### `remove_from_free_list(void *bp)` - Remove Block from Free List

**Process**: 
- Updates previous block's next pointer
- Updates next block's previous pointer  
- Handles edge cases (head/tail of list)
- O(1) time complexity

## Key Macros and Pointer Arithmetic

### Block Navigation Macros
```c
#define HDRP(bp)       ((char *)(bp) - WSIZE)           /* Block header */
#define FTRP(bp)       ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)  /* Block footer */
#define NEXT_BLKP(bp)  ((char *)(bp) + GET_SIZE(HDRP(bp)))          /* Next block */
#define PREV_BLKP(bp)  ((char *)(bp) - GET_SIZE(FTRP(PREV_BLKP(bp)))) /* Previous block */
```

### Free List Navigation Macros
```c
#define GET_NEXT_FREE(bp)  (*(void **)(bp))             /* Next free block pointer */
#define GET_PREV_FREE(bp)  (*(void **)((char *)(bp) + WSIZE)) /* Previous free block pointer */
```

## Optimization Techniques

### 1. Explicit Free List
- **Advantage**: Reduces allocation time from O(total blocks) to O(free blocks)
- **Trade-off**: Requires extra pointers in free blocks (16 bytes overhead)

### 2. Immediate Coalescing
- **Advantage**: Reduces external fragmentation
- **Trade-off**: Increased overhead on free operations

### 3. LIFO Free List Policy
- **Advantage**: Better cache locality (recently freed blocks more likely in cache)
- **Simple Implementation**: O(1) insertion/removal

### 4. Boundary Tags
- **Advantage**: Enables constant-time coalescing
- **Trade-off**: 8 bytes overhead per block (header + footer)

### 5. First-Fit Placement
- **Advantage**: Simple and generally fast
- **Trade-off**: May cause fragmentation compared to best-fit

## Memory Layout Visualization

### Heap Structure:
```
Low Address                                    High Address
+----------+----------+----------+----------+----------+
| Prologue | Block 1  | Block 2  | Block 3  | Epilogue |
| (alloc)  | (free)   | (alloc)  | (free)   | (alloc)  |
+----------+----------+----------+----------+----------+
           ^          ^          ^          ^
           |          |          |          |
           heap_listp |          |          |
                      |          |          |
                Free List:       |          |
                bp1 <-> bp3      |          |
                ^                |          |
                free_listp       |          |
```

### Free Block Internal Structure:
```
Free Block (size = 48 bytes):
Offset:  0    8    16   24   32   40
       +----+----+----+----+----+----+
       |Hdr |Next|Prev|    |    |Ftr |
       |48,0|ptr |ptr | ?? | ?? |48,0|
       +----+----+----+----+----+----+
       ^    ^
       |    bp (points here)
       Header
```

## Assumptions and Limitations

### Assumptions:
1. **Alignment**: All requests can be satisfied with 8-byte alignment
2. **Size Range**: Block sizes fit within size_t with 3 low bits available for flags
3. **Memory Model**: Contiguous heap managed by memlib functions
4. **Single-threaded**: No concurrent access protection

### Limitations:
1. **Realloc Optimization**: Always allocates new block instead of in-place expansion
2. **Fragmentation**: First-fit can lead to fragmentation compared to best-fit
3. **Free List Organization**: Single free list may be inefficient for varied request sizes
4. **Minimum Block Size**: 32-byte minimum may waste space for small allocations
5. **Splitting Threshold**: Fixed 32-byte threshold for splitting may not be optimal

### Potential Improvements:
1. **Segregated Free Lists**: Different lists for different size ranges
2. **In-place Realloc**: Check for expansion possibility before malloc/copy/free
3. **Better Placement**: Best-fit or next-fit policies
4. **Adaptive Splitting**: Dynamic splitting thresholds
5. **Footer Elimination**: Remove footers from allocated blocks to reduce overhead

## Performance Characteristics

### Time Complexity:
- **malloc**: O(free blocks) for first-fit search
- **free**: O(1) for marking + O(1) for coalescing = O(1)
- **realloc**: O(free blocks) + O(min(old_size, new_size)) for copying

### Space Overhead:
- **Allocated blocks**: 16 bytes (header + footer)
- **Free blocks**: 16 bytes (header + footer + 2 pointers stored in payload)
- **Minimum allocation**: 32 bytes total (16 overhead + 16 minimum payload)

This implementation provides a solid foundation for dynamic memory allocation with reasonable performance characteristics and clear, maintainable code structure.