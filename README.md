# Python File Handling: Complete Guide

## 1. File Basics & Concepts

### **What is a File?**
- Named location on disk storing data
- Two types: **Text files** (human-readable) and **Binary files** (machine-readable)
- Has path, name, extension, permissions

### **File System Hierarchy**
```
Absolute path: /home/user/documents/file.txt (Linux/Mac)
               C:\Users\user\Documents\file.txt (Windows)
Relative path: ./file.txt (current directory)
               ../file.txt (parent directory)
```

### **File Operations Flow**
```
1. Open file    → Get file object (handle)
2. Read/Write   → Perform operations
3. Close file   → Release resources (critical!)
```

### **Why Close Files?**
- **Flush buffers**: Ensures data written to disk
- **Release file locks**: Other processes can access
- **Free system resources**: Prevents resource leak
- **Data integrity**: Incomplete writes without close

## 2. File Modes

| Mode | Description | Creates | Overwrites | Position | Read | Write |
|------|-------------|---------|------------|----------|------|-------|
| `r` | Read (text) | No | No | Start | ✓ | ✗ |
| `w` | Write (text) | Yes | Yes | Start | ✗ | ✓ |
| `a` | Append (text) | Yes | No | End | ✗ | ✓ |
| `x` | Exclusive create | Yes | Error if exists | Start | ✗ | ✓ |
| `r+` | Read + Write | No | No | Start | ✓ | ✓ |
| `w+` | Write + Read | Yes | Yes | Start | ✓ | ✓ |
| `a+` | Append + Read | Yes | No | End | ✓ | ✓ |
| `rb` | Read binary | No | No | Start | ✓ | ✗ |
| `wb` | Write binary | Yes | Yes | Start | ✗ | ✓ |
| `ab` | Append binary | Yes | No | End | ✗ | ✓ |

```python
# Common modes explained
f = open('file.txt', 'r')     # Read only (default)
f = open('file.txt', 'w')     # Write only (DANGER: erases content!)
f = open('file.txt', 'a')     # Append only (safe)
f = open('file.txt', 'r+')    # Read and write (doesn't erase)
f = open('file.txt', 'w+')    # Write and read (ERASES content!)

# Binary mode
f = open('image.png', 'rb')   # Read binary
f = open('image.png', 'wb')   # Write binary

# Exclusive mode (fails if exists)
f = open('file.txt', 'x')     # FileExistsError if file exists
```

## 3. Opening Files

### **Basic Syntax**
```python
# Manual open/close
f = open('file.txt', 'r')
content = f.read()
f.close()                     # Must close!

# With context manager (BEST PRACTICE)
with open('file.txt', 'r') as f:
    content = f.read()
# Auto-closes, even if exception occurs
```

### **Why Context Manager?**
```python
# Manual close (RISKY)
f = open('file.txt', 'r')
data = f.read()
result = process(data)        # If this raises exception...
f.close()                     # ...this never runs!

# Context manager (SAFE)
with open('file.txt', 'r') as f:
    data = f.read()
    result = process(data)    # Even if exception, file closes
```

### **Encoding**
```python
# Text files default encoding
# Linux/Mac: UTF-8
# Windows: cp1252 (locale-dependent)

# Explicit encoding (RECOMMENDED)
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# Common encodings
# utf-8: Universal, supports all languages
# ascii: English only (7-bit)
# latin-1: Western European
# cp1252: Windows default
# utf-16: Wide characters

# Handle encoding errors
with open('file.txt', 'r', encoding='utf-8', errors='ignore') as f:
    content = f.read()        # Skip invalid characters

# errors parameter:
# 'strict': Raise exception (default)
# 'ignore': Skip invalid chars
# 'replace': Replace with �
# 'backslashreplace': Replace with \xNN
```

## 4. Reading Files

### **read() - Read Entire File**
```python
with open('file.txt', 'r') as f:
    content = f.read()        # Returns entire file as string

# Read n bytes
with open('file.txt', 'r') as f:
    chunk = f.read(10)        # Read first 10 characters

# Read all
with open('file.txt', 'r') as f:
    all_content = f.read()    # Empty string if at EOF
    more_content = f.read()   # Returns '' (empty)
```

### **readline() - Read One Line**
```python
with open('file.txt', 'r') as f:
    line1 = f.readline()      # Includes \n at end
    line2 = f.readline()      # Next line
    line3 = f.readline()      # Returns '' at EOF

# Remove newline
line = f.readline().strip()   # Remove leading/trailing whitespace
line = f.readline().rstrip()  # Remove trailing only
```

### **readlines() - Read All Lines**
```python
with open('file.txt', 'r') as f:
    lines = f.readlines()     # Returns list of strings

# Example: ['line1\n', 'line2\n', 'line3']

# Memory concern: Loads entire file into memory
# For large files, use iteration instead
```

### **Iteration (BEST for Large Files)**
```python
# Most efficient way
with open('file.txt', 'r') as f:
    for line in f:            # Reads one line at a time
        print(line.strip())   # Memory efficient!

# Why iteration is best:
# - Memory efficient (doesn't load all at once)
# - Pythonic and readable
# - Fast for large files
```

### **Read Comparison**

```python
# Small files (<10MB)
with open('small.txt', 'r') as f:
    content = f.read()        # OK, load all

# Large files (>10MB)
with open('large.txt', 'r') as f:
    for line in f:            # GOOD, process line by line
        process(line)

# Read in chunks (for binary or huge files)
with open('huge.bin', 'rb') as f:
    while True:
        chunk = f.read(8192)  # Read 8KB at a time
        if not chunk:
            break
        process(chunk)
```

## 5. Writing Files

### **write() - Write String**
```python
with open('file.txt', 'w') as f:
    f.write('Hello, World!')   # Returns bytes written
    f.write('\n')              # Must add newline manually

# write() doesn't add newline!
with open('file.txt', 'w') as f:
    f.write('Line 1')
    f.write('Line 2')          # Result: "Line 1Line 2"

# Add newlines
with open('file.txt', 'w') as f:
    f.write('Line 1\n')
    f.write('Line 2\n')        # Result: "Line 1\nLine 2\n"
```

### **writelines() - Write List**
```python
lines = ['Line 1\n', 'Line 2\n', 'Line 3\n']

with open('file.txt', 'w') as f:
    f.writelines(lines)        # Writes all lines

# writelines() doesn't add newlines!
lines = ['Line 1', 'Line 2']
with open('file.txt', 'w') as f:
    f.writelines(lines)        # Result: "Line 1Line 2"

# Add newlines manually
with open('file.txt', 'w') as f:
    f.writelines(line + '\n' for line in lines)
```

### **Append vs Write**
```python
# Write mode (w) - ERASES existing content
with open('file.txt', 'w') as f:
    f.write('New content')     # Deletes old content!

# Append mode (a) - Adds to end
with open('file.txt', 'a') as f:
    f.write('New line\n')      # Keeps old content

# Example
# file.txt contains: "Hello"

with open('file.txt', 'w') as f:
    f.write('World')
# Result: "World" (Hello is gone!)

with open('file.txt', 'a') as f:
    f.write('World')
# Result: "HelloWorld" (appended)
```

## 6. File Position & Seeking

### **tell() - Get Current Position**
```python
with open('file.txt', 'r') as f:
    print(f.tell())            # 0 (start)
    f.read(5)
    print(f.tell())            # 5 (read 5 bytes)
    f.read()
    print(f.tell())            # EOF position
```

### **seek() - Move Position**
```python
# seek(offset, whence)
# whence: 0 (start), 1 (current), 2 (end)

with open('file.txt', 'r') as f:
    f.seek(0)                  # Go to start
    f.seek(5)                  # Go to byte 5
    f.seek(0, 2)               # Go to end
    f.seek(-5, 2)              # 5 bytes before end

# Text mode only supports:
# - seek(0) or seek(offset, 0)
# - seek(0, 2) for end

# Binary mode supports all:
with open('file.bin', 'rb') as f:
    f.seek(10, 1)              # Move 10 bytes forward
    f.seek(-5, 1)              # Move 5 bytes back
```

### **Rewind File**
```python
with open('file.txt', 'r') as f:
    content = f.read()
    # Now at EOF
    f.seek(0)                  # Back to start
    content_again = f.read()   # Read again
```

## 7. File Object Methods

```python
# Reading
f.read(size=-1)              # Read n bytes (all if -1)
f.readline(size=-1)          # Read one line
f.readlines()                # Read all lines as list

# Writing
f.write(string)              # Write string (returns bytes written)
f.writelines(list)           # Write list of strings

# Position
f.tell()                     # Current position
f.seek(offset, whence=0)     # Move to position

# Buffer
f.flush()                    # Force write buffer to disk

# State
f.closed                     # True if closed
f.mode                       # File mode ('r', 'w', etc.)
f.name                       # File name
f.encoding                   # File encoding (text mode)

# Close
f.close()                    # Close file

# Readable/Writable
f.readable()                 # Can read?
f.writable()                 # Can write?
f.seekable()                 # Can seek?

# Truncate
f.truncate(size=None)        # Resize file

# File descriptor
f.fileno()                   # OS file descriptor (int)

# Check if terminal
f.isatty()                   # Is terminal device?
```

## 8. Binary Files

### **Reading Binary**
```python
# Read image
with open('image.png', 'rb') as f:
    data = f.read()            # Returns bytes object

# Read in chunks (large files)
with open('large.bin', 'rb') as f:
    while True:
        chunk = f.read(8192)   # 8KB chunks
        if not chunk:
            break
        process(chunk)

# Byte operations
with open('file.bin', 'rb') as f:
    data = f.read(10)          # First 10 bytes
    print(data)                # b'\x89PNG\r\n\x1a\n...'
    print(data[0])             # 137 (first byte as int)
    print(hex(data[0]))        # 0x89
```

### **Writing Binary**
```python
# Write bytes
data = b'\x00\x01\x02\x03'
with open('file.bin', 'wb') as f:
    f.write(data)

# Copy binary file
with open('source.png', 'rb') as src:
    with open('dest.png', 'wb') as dst:
        dst.write(src.read())

# Or copy in chunks (better for large files)
with open('source.png', 'rb') as src:
    with open('dest.png', 'wb') as dst:
        while True:
            chunk = src.read(8192)
            if not chunk:
                break
            dst.write(chunk)

# Using shutil (best for copying)
import shutil
shutil.copy('source.png', 'dest.png')
```

### **Binary vs Text**
```python
# Text mode
with open('file.txt', 'r') as f:
    data = f.read()            # Returns str
    type(data)                 # <class 'str'>

# Binary mode
with open('file.txt', 'rb') as f:
    data = f.read()            # Returns bytes
    type(data)                 # <class 'bytes'>

# Convert
text = "Hello"
binary = text.encode('utf-8')  # str → bytes
text_again = binary.decode('utf-8')  # bytes → str
```

## 9. File Buffering

### **What is Buffering?**
```
Program → Buffer (in memory) → Disk
                ↑
           Flush when full or close

Why buffer?
- Disk I/O is slow (microseconds vs nanoseconds)
- Batching writes is more efficient
- Reduces system calls
```

### **Buffer Modes**
```python
# Default buffering (good for most cases)
f = open('file.txt', 'w')

# No buffering (writes immediately)
f = open('file.txt', 'w', buffering=0)  # Binary mode only

# Line buffering (flush on newline)
f = open('file.txt', 'w', buffering=1)  # Text mode only

# Custom buffer size (bytes)
f = open('file.txt', 'w', buffering=8192)  # 8KB buffer

# Examples
with open('file.txt', 'w', buffering=1) as f:
    f.write('Line 1\n')        # Flushed immediately (line buffered)
    f.write('No newline')      # Stays in buffer
    f.flush()                  # Force flush

# When to flush manually
with open('log.txt', 'a') as f:
    f.write('Important log entry\n')
    f.flush()                  # Ensure written to disk NOW
```

### **Buffer Use Cases**
```python
# High-frequency writes (use default buffer)
with open('output.txt', 'w') as f:
    for i in range(10000):
        f.write(f'{i}\n')      # Buffered, efficient

# Critical logs (flush immediately)
with open('critical.log', 'a', buffering=1) as f:
    f.write(f'{timestamp}: Error\n')  # Flushed on \n

# Real-time monitoring (no buffer)
with open('monitor.log', 'wb', buffering=0) as f:
    while True:
        data = get_sensor_data()
        f.write(data)          # Written immediately
```

## 10. Exception Handling

### **Common File Exceptions**
```python
# FileNotFoundError - File doesn't exist
try:
    f = open('nonexistent.txt', 'r')
except FileNotFoundError:
    print("File not found!")

# PermissionError - No permission to access
try:
    f = open('/root/file.txt', 'w')
except PermissionError:
    print("No permission!")

# IsADirectoryError - Path is directory
try:
    f = open('/home/user', 'r')
except IsADirectoryError:
    print("That's a directory!")

# FileExistsError - File exists (with 'x' mode)
try:
    f = open('existing.txt', 'x')
except FileExistsError:
    print("File already exists!")

# UnicodeDecodeError - Encoding mismatch
try:
    with open('file.txt', 'r', encoding='ascii') as f:
        content = f.read()
except UnicodeDecodeError:
    print("Encoding error!")

# IOError / OSError - Generic I/O errors
try:
    f = open('file.txt', 'r')
    content = f.read()
except OSError as e:
    print(f"I/O error: {e}")
```

### **Safe File Operations**
```python
# Check if file exists
import os

if os.path.exists('file.txt'):
    with open('file.txt', 'r') as f:
        content = f.read()

# Check if path is file
if os.path.isfile('file.txt'):
    # It's a file
    pass

# Check if path is directory
if os.path.isdir('folder'):
    # It's a directory
    pass

# Try to open with fallback
try:
    with open('config.txt', 'r') as f:
        config = f.read()
except FileNotFoundError:
    config = get_default_config()

# Safe write (atomic-ish)
import tempfile
import shutil

with tempfile.NamedTemporaryFile('w', delete=False) as tmp:
    tmp.write(content)
    tmp_name = tmp.name

shutil.move(tmp_name, 'file.txt')  # Replace original
```

## 11. File Paths

### **Path Operations (os.path)**
```python
import os

# Join paths (cross-platform)
path = os.path.join('folder', 'subfolder', 'file.txt')
# Linux: folder/subfolder/file.txt
# Windows: folder\subfolder\file.txt

# Get absolute path
abs_path = os.path.abspath('file.txt')
# /home/user/current/file.txt

# Get directory and filename
dir_name = os.path.dirname('/path/to/file.txt')   # /path/to
file_name = os.path.basename('/path/to/file.txt') # file.txt

# Split extension
name, ext = os.path.splitext('file.txt')  # ('file', '.txt')

# Check existence
os.path.exists('file.txt')
os.path.isfile('file.txt')
os.path.isdir('folder')

# Get file size
size = os.path.getsize('file.txt')  # Bytes

# Get modification time
mtime = os.path.getmtime('file.txt')  # Timestamp

# Current directory
cwd = os.getcwd()

# Change directory
os.chdir('/path/to/folder')
```

### **pathlib (Modern Approach)**
```python
from pathlib import Path

# Create path object
p = Path('folder') / 'subfolder' / 'file.txt'
# Path('folder/subfolder/file.txt')

# Read/write directly
p = Path('file.txt')
content = p.read_text()           # Read file
p.write_text('Hello')             # Write file

# Read binary
data = p.read_bytes()             # Read binary
p.write_bytes(b'\x00\x01')        # Write binary

# Check existence
p.exists()                        # File exists?
p.is_file()                       # Is file?
p.is_dir()                        # Is directory?

# File info
p.stat().st_size                  # Size in bytes
p.stat().st_mtime                 # Modification time
p.suffix                          # '.txt'
p.stem                            # 'file' (without extension)
p.name                            # 'file.txt'
p.parent                          # Path('folder/subfolder')

# Create directory
p = Path('new_folder')
p.mkdir(exist_ok=True)            # Create if not exists
p.mkdir(parents=True, exist_ok=True)  # Create parent dirs too

# List files
p = Path('.')
for item in p.iterdir():
    print(item)

# Glob pattern matching
for txt_file in p.glob('*.txt'):
    print(txt_file)

# Recursive glob
for py_file in p.rglob('*.py'):   # All .py files recursively
    print(py_file)

# Delete file
p = Path('file.txt')
p.unlink()                        # Delete file
p.unlink(missing_ok=True)         # No error if missing

# Rename/move
p.rename('new_name.txt')

# Resolve (absolute path)
p = Path('file.txt')
abs_p = p.resolve()               # /home/user/current/file.txt
```

## 12. Working with CSV Files

```python
import csv

# Read CSV
with open('data.csv', 'r') as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)                # List: ['col1', 'col2', 'col3']

# Read CSV as dict
with open('data.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)                # Dict: {'name': 'Alice', 'age': '30'}
        print(row['name'])        # Access by column name

# Write CSV
data = [
    ['Name', 'Age', 'City'],
    ['Alice', 30, 'NYC'],
    ['Bob', 25, 'LA']
]

with open('output.csv', 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['Name', 'Age'])  # Single row
    writer.writerows(data)            # Multiple rows

# Write CSV from dict
data = [
    {'name': 'Alice', 'age': 30},
    {'name': 'Bob', 'age': 25}
]

with open('output.csv', 'w', newline='') as f:
    fieldnames = ['name', 'age']
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()              # Write header row
    writer.writerows(data)

# Custom delimiter
with open('data.tsv', 'r') as f:
    reader = csv.reader(f, delimiter='\t')

# Handle quotes
with open('data.csv', 'r') as f:
    reader = csv.reader(f, quotechar='"', quoting=csv.QUOTE_MINIMAL)
```

## 13. Working with JSON Files

```python
import json

# Read JSON
with open('data.json', 'r') as f:
    data = json.load(f)           # Returns dict/list

# Write JSON
data = {'name': 'Alice', 'age': 30}

with open('output.json', 'w') as f:
    json.dump(data, f)            # Write to file

# Pretty print JSON
with open('output.json', 'w') as f:
    json.dump(data, f, indent=2)  # Indented output

# Convert to/from string
json_str = json.dumps(data)       # Dict → JSON string
data = json.loads(json_str)       # JSON string → Dict

# Handle custom objects
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

def person_encoder(obj):
    if isinstance(obj, Person):
        return {'name': obj.name, 'age': obj.age}
    raise TypeError

person = Person('Alice', 30)
json_str = json.dumps(person, default=person_encoder)

# Or use dataclasses (Python 3.7+)
from dataclasses import dataclass, asdict

@dataclass
class Person:
    name: str
    age: int

person = Person('Alice', 30)
json_str = json.dumps(asdict(person))
```

## 14. Temporary Files

```python
import tempfile

# Temporary file (auto-deleted)
with tempfile.TemporaryFile('w+') as f:
    f.write('Temporary data')
    f.seek(0)
    print(f.read())
# File deleted automatically

# Named temporary file
with tempfile.NamedTemporaryFile('w', delete=False) as f:
    f.write('Data')
    temp_path = f.name
# File exists after context (delete=False)

# Temporary directory
with tempfile.TemporaryDirectory() as tmpdir:
    # tmpdir is a path string
    file_path = os.path.join(tmpdir, 'file.txt')
    with open(file_path, 'w') as f:
        f.write('Data')
# Directory and all contents deleted

# Get temp directory
temp_dir = tempfile.gettempdir()  # /tmp on Linux

# Create temp file with suffix
with tempfile.NamedTemporaryFile(suffix='.txt') as f:
    print(f.name)                 # /tmp/tmp8x9y1z2a.txt
```

## 15. File Locking

```python
import fcntl  # Unix only

# Exclusive lock (write)
with open('file.txt', 'w') as f:
    fcntl.flock(f.fileno(), fcntl.LOCK_EX)
    f.write('Critical section')
    fcntl.flock(f.fileno(), fcntl.LOCK_UN)

# Shared lock (read)
with open('file.txt', 'r') as f:
    fcntl.flock(f.fileno(), fcntl.LOCK_SH)
    content = f.read()
    fcntl.flock(f.fileno(), fcntl.LOCK_UN)

# Non-blocking lock
try:
    fcntl.flock(f.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
except IOError:
    print("File is locked!")

# Cross-platform locking (use portalocker)
# pip install portalocker
import portalocker

with open('file.txt', 'w') as f:
    portalocker.lock(f, portalocker.LOCK_EX)
    f.write('Data')
    portalocker.unlock(f)
```

## 16. Memory-Mapped Files

```python
import mmap

# Memory-map file (efficient for large files)
with open('large.txt', 'r+b') as f:
    mmapped = mmap.mmap(f.fileno(), 0)
    
    # Read
    print(mmapped[0:10])          # First 10 bytes
    
    # Write
    mmapped[0:5] = b'Hello'
    
    # Search
    index = mmapped.find(b'pattern')
    
    # Close
    mmapped.close()

# Benefits:
# - Fast random access
# - Shared between processes
# - OS handles caching
# - Good for large files

# Use case: Database files, large logs
```

## 17. File Descriptors & Low-Level I/O

```python
import os

# Open with file descriptor
fd = os.open('file.txt', os.O_RDONLY)
data = os.read(fd, 1024)          # Read 1024 bytes
os.close(fd)

# Write
fd = os.open('file.txt', os.O_WRONLY | os.O_CREAT)
os.write(fd, b'Hello')
os.close(fd)

# Flags
os.O_RDONLY                       # Read only
os.O_WRONLY                       # Write only
os.O_RDWR                         # Read and write
os.O_CREAT                        # Create if not exists
os.O_TRUNC                        # Truncate to zero
os.O_APPEND                       # Append mode
os.O_EXCL                         # Fail if exists (with O_CREAT)

# Get file descriptor from file object
f = open('file.txt', 'r')
fd = f.fileno()                   # Get underlying FD

# Duplicate file descriptor
fd2 = os.dup(fd)

# Redirect (advanced)
os.dup2(fd, 1)                    # Redirect stdout to file
```

## 18. Performance Optimization

### **Large File Reading**
```python
# BAD (loads entire file)
with open('huge.txt', 'r') as f:
    content = f.read()            # May crash with MemoryError

# GOOD (line by line)
with open('huge.txt', 'r') as f:
    for line in f:
        process(line)

# GOOD (chunks)
with open('huge.bin', 'rb') as f:
    while True:
        chunk = f.read(8192)      # 8KB chunks
        if not chunk:
            break
        process(chunk)

# Using generator (memory efficient)
def read_chunks(filename, chunk_size=8192):
    with open(filename, 'rb') as f:
        while True:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk

for chunk in read_chunks('huge.bin'):
    process(chunk)
```

### **Bulk Writing**
```python
# BAD (many small writes)
with open('output.txt', 'w') as f:
    for i in range(10000):
        f.write(f'{i}\n')         # 10000 write calls

# GOOD (batch writes)
lines = [f'{i}\n' for i in range(10000)]
with open('output.txt', 'w') as f:
    f.writelines(lines)           # One write call

# BETTER (generator + writelines)
with open('output.txt', 'w') as f:
    f.writelines(f'{i}\n' for i in range(10000))
```

### **Buffer Size Tuning**
```python
# Default buffer (good for most)
with open('file.txt', 'w') as f:
    f.write(data)

# Larger buffer for bulk operations
with open('large.txt', 'w', buffering=65536) as f:  # 64KB
    for data in large_dataset:
        f.write(data)

# Benchmark different buffer sizes
import time

sizes = [1024, 8192, 65536]
for size in sizes:
    start = time.time()
    with open('test.txt', 'w', buffering=size) as f:
        for i in range(100000):
            f.write(f'{i}\n')
    print(f'Buffer {size}: {time.time() - start:.2f}s')
```

## 19. Atomic File Operations

```python
import os
import tempfile

# Atomic write (write to temp, then rename)
def atomic_write(filename, content):
    # Write to temporary file
    fd, temp_path = tempfile.mkstemp(dir=os.path.dirname(filename))
    try:
        with os.fdopen(fd, 'w') as f:
            f.write(content)
        os.replace(temp_path, filename)  # Atomic on POSIX
    except:
        os.unlink(temp_path)
        raise

# Why atomic writes matter:
# - Power failure during write
# - Process crash during write
# - Concurrent readers
# Atomic replace prevents partial/corrupt files

# Using context manager
import contextlib

@contextlib.contextmanager
def atomic_file(filename, mode='w'):
    fd, temp_path = tempfile.mkstemp(dir=os.path.dirname(filename))
    try:
        with os.fdopen(fd, mode) as f:
            yield f
        os.replace(temp_path, filename)
    except:
        os.unlink(temp_path)
        raise

# Usage
with atomic_file('config.json') as f:
    json.dump(config, f)
```

## 20. File Compression

### **gzip**
```python
import gzip

# Write compressed
with gzip.open('file.txt.gz', 'wt', encoding='utf-8') as f:
    f.write('Hello, World!')

# Read compressed
with gzip.open('file.txt.gz', 'rt', encoding='utf-8') as f:
    content = f.read()

# Compress existing file
with open('file.txt', 'rb') as f_in:
    with gzip.open('file.txt.gz', 'wb') as f_out:
        f_out.writelines(f_in)

# Or use shutil
import shutil
with open('file.txt', 'rb') as f_in:
    with gzip.open('file.txt.gz', 'wb') as f_out:
        shutil.copyfileobj(f_in, f_out)

# Decompress
with gzip.open('file.txt.gz', 'rb') as f_in:
    with open('file.txt', 'wb') as f_out:
        shutil.copyfileobj(f_in, f_out)
```

### **zipfile**
```python
import zipfile

# Create zip
with zipfile.ZipFile('archive.zip', 'w') as zf:
    zf.write('file1.txt')
    zf.write('file2.txt')
    zf.write('folder/file3.txt', arcname='file3.txt')

# Read zip
with zipfile.ZipFile('archive.zip', 'r') as zf:
    # List contents
    for name in zf.namelist():
        print(name)
    
    # Read file
    content = zf.read('file1.txt')
    
    # Extract all
    zf.extractall('destination_folder')
    
    # Extract specific file
    zf.extract('file1.txt', 'destination_folder')

# Add to existing zip
with zipfile.ZipFile('archive.zip', 'a') as zf:
    zf.write('new_file.txt')

# Compress with different algorithm
with zipfile.ZipFile('archive.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.write('file.txt')

# Password protected (write only)
with zipfile.ZipFile('secure.zip', 'w') as zf:
    zf.setpassword(b'password')
    zf.write('secret.txt')
```

### **tarfile**
```python
import tarfile

# Create tar.gz
with tarfile.open('archive.tar.gz', 'w:gz') as tar:
    tar.add('file1.txt')
    tar.add('folder/', recursive=True)

# Create tar (no compression)
with tarfile.open('archive.tar', 'w') as tar:
    tar.add('file.txt')

# Create tar.bz2
with tarfile.open('archive.tar.bz2', 'w:bz2') as tar:
    tar.add('file.txt')

# Read tar
with tarfile.open('archive.tar.gz', 'r:gz') as tar:
    # List contents
    for member in tar.getmembers():
        print(member.name)
    
    # Extract all
    tar.extractall('destination')
    
    # Extract specific file
    tar.extract('file.txt', 'destination')
    
    # Read file content
    f = tar.extractfile('file.txt')
    content = f.read()
```

## 21. File Watching

```python
# Using watchdog library
# pip install watchdog

from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class MyHandler(FileSystemEventHandler):
    def on_created(self, event):
        print(f'Created: {event.src_path}')
    
    def on_modified(self, event):
        print(f'Modified: {event.src_path}')
    
    def on_deleted(self, event):
        print(f'Deleted: {event.src_path}')
    
    def on_moved(self, event):
        print(f'Moved: {event.src_path} -> {event.dest_path}')

# Watch directory
observer = Observer()
observer.schedule(MyHandler(), path='watched_folder', recursive=True)
observer.start()

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    observer.stop()

observer.join()

# Use case: Auto-reload config, log monitoring, file sync
```

## 22. Stdin/Stdout/Stderr

```python
import sys

# Read from stdin
line = sys.stdin.readline()       # One line
lines = sys.stdin.readlines()     # All lines

# Iterate stdin (pipe-friendly)
for line in sys.stdin:
    process(line.strip())

# Write to stdout
sys.stdout.write('Hello\n')
print('Hello')                    # Same as above

# Write to stderr
sys.stderr.write('Error!\n')
print('Error!', file=sys.stderr)  # Same as above

# Check if piped
if not sys.stdin.isatty():
    # Data is piped in
    data = sys.stdin.read()

if not sys.stdout.isatty():
    # Output is piped/redirected
    pass

# Redirect stdout to file
original_stdout = sys.stdout
with open('output.txt', 'w') as f:
    sys.stdout = f
    print('This goes to file')
sys.stdout = original_stdout

# Context manager for redirection
from contextlib import redirect_stdout, redirect_stderr

with open('output.txt', 'w') as f:
    with redirect_stdout(f):
        print('To file')

# Usage: python script.py < input.txt > output.txt 2> error.txt
```

## 23. File Permissions

```python
import os
import stat

# Get permissions
st = os.stat('file.txt')
mode = st.st_mode

# Check permissions
is_readable = bool(mode & stat.S_IRUSR)
is_writable = bool(mode & stat.S_IWUSR)
is_executable = bool(mode & stat.S_IXUSR)

# Set permissions (Unix)
os.chmod('file.txt', 0o644)       # rw-r--r--
os.chmod('file.txt', 0o755)       # rwxr-xr-x

# Using stat constants
os.chmod('file.txt', stat.S_IRUSR | stat.S_IWUSR | stat.S_IRGRP | stat.S_IROTH)

# Change owner (Unix, requires privileges)
os.chown('file.txt', uid, gid)

# Get file info
st = os.stat('file.txt')
st.st_size                        # Size in bytes
st.st_mtime                       # Last modified time
st.st_atime                       # Last access time
st.st_ctime                       # Creation time (Windows) / metadata change (Unix)
st.st_uid                         # Owner user ID
st.st_gid                         # Owner group ID

# Using pathlib
from pathlib import Path
p = Path('file.txt')
p.chmod(0o644)
st = p.stat()
```

## 24. Directory Operations

```python
import os
import shutil
from pathlib import Path

# Create directory
os.mkdir('new_folder')
os.makedirs('parent/child/grandchild', exist_ok=True)

# Using pathlib
Path('new_folder').mkdir(exist_ok=True)
Path('parent/child').mkdir(parents=True, exist_ok=True)

# Remove directory
os.rmdir('empty_folder')          # Only empty dirs
shutil.rmtree('folder')           # Recursive delete

Path('folder').rmdir()            # Only empty
# No recursive delete in pathlib, use shutil

# List directory
entries = os.listdir('folder')    # Returns list of names

# Better: scandir (returns DirEntry objects)
with os.scandir('folder') as it:
    for entry in it:
        print(entry.name)
        print(entry.is_file())
        print(entry.is_dir())
        print(entry.stat())

# Using pathlib
for item in Path('folder').iterdir():
    print(item)
    print(item.is_file())
    print(item.is_dir())

# Recursive listing
for root, dirs, files in os.walk('folder'):
    for file in files:
        full_path = os.path.join(root, file)
        print(full_path)

# Using pathlib glob
for py_file in Path('.').rglob('*.py'):
    print(py_file)

# Copy directory
shutil.copytree('source', 'destination')

# Copy with overwrite (Python 3.8+)
shutil.copytree('source', 'dest', dirs_exist_ok=True)

# Move/rename directory
shutil.move('old_name', 'new_name')

# Current directory
cwd = os.getcwd()
cwd = Path.cwd()

# Change directory
os.chdir('folder')

# Home directory
home = Path.home()                # /home/user
```

## 25. File System Operations

```python
import os
import shutil

# Copy file
shutil.copy('source.txt', 'dest.txt')           # Copy file
shutil.copy2('source.txt', 'dest.txt')          # Copy with metadata
shutil.copyfile('source.txt', 'dest.txt')       # Copy content only

# Move/rename file
shutil.move('old.txt', 'new.txt')
os.rename('old.txt', 'new.txt')

# Delete file
os.remove('file.txt')
os.unlink('file.txt')                           # Same as remove
Path('file.txt').unlink(missing_ok=True)

# Check before delete
if os.path.exists('file.txt'):
    os.remove('file.txt')

# Get disk usage
total, used, free = shutil.disk_usage('/')
print(f'Total: {total // (2**30)} GB')
print(f'Used: {used // (2**30)} GB')
print(f'Free: {free // (2**30)} GB')

# File links (Unix)
os.symlink('target.txt', 'link.txt')            # Symbolic link
os.link('target.txt', 'hard_link.txt')          # Hard link

# Check if link
os.path.islink('link.txt')
Path('link.txt').is_symlink()

# Read link target
os.readlink('link.txt')
Path('link.txt').readlink()

# Resolve link to real path
os.path.realpath('link.txt')
Path('link.txt').resolve()
```

## 26. Context Managers

### **Multiple Files**
```python
# Multiple with statements
with open('input.txt', 'r') as f_in:
    with open('output.txt', 'w') as f_out:
        content = f_in.read()
        f_out.write(content)

# Single statement (Python 2.7+)
with open('input.txt', 'r') as f_in, open('output.txt', 'w') as f_out:
    content = f_in.read()
    f_out.write(content)

# ExitStack for dynamic number of files
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f'file{i}.txt', 'r')) 
             for i in range(10)]
    # All files auto-close
```

### **Custom Context Manager**
```python
# Using class
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.file:
            self.file.close()
        # Return True to suppress exceptions
        return False

with FileManager('file.txt', 'r') as f:
    content = f.read()

# Using contextlib
from contextlib import contextmanager

@contextmanager
def file_manager(filename, mode):
    f = open(filename, mode)
    try:
        yield f
    finally:
        f.close()

with file_manager('file.txt', 'r') as f:
    content = f.read()
```

## 27. Edge Cases & Gotchas

### **Newline Handling**
```python
# Windows: \r\n (CRLF)
# Unix/Mac: \n (LF)
# Old Mac: \r (CR)

# Text mode auto-converts
with open('file.txt', 'r') as f:      # Auto-converts \r\n to \n
    content = f.read()

# Binary mode preserves
with open('file.txt', 'rb') as f:     # Keeps original \r\n
    content = f.read()

# Disable newline translation
with open('file.txt', 'r', newline='') as f:
    content = f.read()                # Keeps \r\n in text mode

# For CSV, use newline=''
with open('data.csv', 'w', newline='') as f:
    writer = csv.writer(f)
```

### **Encoding Issues**
```python
# Problem: Wrong encoding
with open('file.txt', 'r') as f:      # Uses system default
    content = f.read()                # May fail with UnicodeDecodeError

# Solution: Explicit encoding
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# Detect encoding (use chardet)
# pip install chardet
import chardet

with open('file.txt', 'rb') as f:
    raw = f.read()
    result = chardet.detect(raw)
    encoding = result['encoding']

with open('file.txt', 'r', encoding=encoding) as f:
    content = f.read()

# Handle errors
with open('file.txt', 'r', encoding='utf-8', errors='replace') as f:
    content = f.read()                # Replace bad chars with �
```

### **File Already Open**
```python
# Problem: File left open
f = open('file.txt', 'r')
content = f.read()
# Forgot to close!

# Open again in write mode
# f2 = open('file.txt', 'w')        # May fail or corrupt on some systems

# Solution: Always use context manager
with open('file.txt', 'r') as f:
    content = f.read()
# Auto-closed

# Check if file is closed
print(f.closed)                       # True
```

### **Race Conditions**
```python
# BAD: Check-then-act race
if not os.path.exists('file.txt'):
    # File could be created here by another process!
    with open('file.txt', 'w') as f:
        f.write('data')

# GOOD: Try-except
try:
    with open('file.txt', 'x') as f:  # Exclusive create
        f.write('data')
except FileExistsError:
    pass

# GOOD: Atomic operations
# Use 'x' mode, file locks, or atomic writes
```

### **Path Separators**
```python
# BAD: Hardcoded separators
path = 'folder/subfolder/file.txt'    # Fails on Windows
path = 'folder\\subfolder\\file.txt'  # Fails on Unix

# GOOD: os.path.join
path = os.path.join('folder', 'subfolder', 'file.txt')

# BETTER: pathlib
path = Path('folder') / 'subfolder' / 'file.txt'
```

### **File Descriptor Leaks**
```python
# BAD: Multiple opens without close
for i in range(10000):
    f = open(f'file{i}.txt', 'r')
    content = f.read()
    # No close! Eventually hits OS limit

# GOOD: Always close
for i in range(10000):
    with open(f'file{i}.txt', 'r') as f:
        content = f.read()
```

### **Writing Empty Files**
```python
# 'w' mode creates empty file immediately
with open('file.txt', 'w') as f:
    # File is now empty!
    if error_condition:
        return  # Original content lost!

# Solution: Write to temp, then rename
with tempfile.NamedTemporaryFile('w', delete=False) as tmp:
    tmp.write(content)
    temp_name = tmp.name

os.replace(temp_name, 'file.txt')
```

## 28. Performance Benchmarks

```python
import time

# Benchmark: read methods
file_size = 100_000_000  # 100MB

# Method 1: read() - Fastest for small files
start = time.time()
with open('large.txt', 'r') as f:
    content = f.read()
print(f'read(): {time.time() - start:.2f}s')

# Method 2: readlines() - Fast but memory-intensive
start = time.time()
with open('large.txt', 'r') as f:
    lines = f.readlines()
print(f'readlines(): {time.time() - start:.2f}s')

# Method 3: readline() loop - Slow
start = time.time()
with open('large.txt', 'r') as f:
    while f.readline():
        pass
print(f'readline(): {time.time() - start:.2f}s')

# Method 4: iteration - Fast and memory-efficient
start = time.time()
with open('large.txt', 'r') as f:
    for line in f:
        pass
print(f'iteration: {time.time() - start:.2f}s')

# Typical results:
# read(): 0.5s (loads all)
# readlines(): 0.6s (loads all)
# readline(): 3.0s (many function calls)
# iteration: 0.5s (buffered, efficient)
```

## 29. Common Patterns

### **Process Large File**
```python
def process_large_file(filename):
    with open(filename, 'r') as f:
        for line in f:
            # Process line by line
            data = parse(line)
            yield data

for record in process_large_file('huge.log'):
    handle(record)
```

### **Read Configuration**
```python
def read_config(filename):
    config = {}
    try:
        with open(filename, 'r') as f:
            for line in f:
                key, value = line.strip().split('=')
                config[key] = value
    except FileNotFoundError:
        config = get_defaults()
    return config
```

### **Backup Before Write**
```python
def safe_write(filename, content):
    if os.path.exists(filename):
        # Create backup
        shutil.copy(filename, f'{filename}.bak')
    
    try:
        with open(filename, 'w') as f:
            f.write(content)
    except:
        # Restore backup on error
        if os.path.exists(f'{filename}.bak'):
            shutil.copy(f'{filename}.bak', filename)
        raise
```

### **Tail File (Last N Lines)**
```python
def tail(filename, n=10):
    with open(filename, 'r') as f:
        lines = f.readlines()
        return lines[-n:]

# Memory efficient (for huge files)
from collections import deque

def tail_efficient(filename, n=10):
    with open(filename, 'r') as f:
        return deque(f, maxlen=n)
```

### **Count Lines Efficiently**
```python
def count_lines(filename):
    count = 0
    with open(filename, 'rb') as f:
        for line in f:
            count += 1
    return count

# Faster for very large files
def count_lines_fast(filename):
    with open(filename, 'rb') as f:
        return sum(1 for _ in f)

# Even faster with buffer
def count_lines_buffer(filename):
    with open(filename, 'rb') as f:
        return sum(chunk.count(b'\n') for chunk in iter(lambda: f.read(1024*1024), b''))
```

### **File Rotation**
```python
def rotate_file(filename, max_size=1024*1024):
    """Rotate file when exceeds max_size"""
    if not os.path.exists(filename):
        return
    
    if os.path.getsize(filename) > max_size:
        # Rename to .1, .2, etc.
        i = 1
        while os.path.exists(f'{filename}.{i}'):
            i += 1
        os.rename(filename, f'{filename}.{i}')
```

## 30. Best Practices Summary

### **✅ DO:**
```python
# Always use context manager
with open('file.txt', 'r') as f:
    content = f.read()

# Specify encoding explicitly
with open('file.txt', 'r', encoding='utf-8') as f:
    pass

# Use pathlib for path operations
from pathlib import Path
p = Path('folder') / 'file.txt'

# Iterate for large files
for line in f:
    process(line)

# Use 'x' mode to prevent overwriting
with open('file.txt', 'x') as f:
    f.write('data')

# Use binary mode for non-text
with open('image.png', 'rb') as f:
    data = f.read()

# Check file exists before opening
if Path('file.txt').exists():
    pass

# Use atomic writes for critical data
# (write to temp, then rename)

# Close files in finally block if not using context manager
f = None
try:
    f = open('file.txt', 'r')
finally:
    if f:
        f.close()
```

### **❌ DON'T:**
```python
# Don't forget to close
f = open('file.txt', 'r')
content = f.read()
# No close!

# Don't use system default encoding
with open('file.txt', 'r') as f:  # Encoding depends on system!
    pass

# Don't read huge files with .read()
content = open('huge.txt').read()  # Memory error!

# Don't use 'w' mode carelessly
with open('important.txt', 'w') as f:  # Erases content!
    pass

# Don't hardcode path separators
path = 'folder\\file.txt'  # Fails on Unix

# Don't ignore exceptions
try:
    f = open('file.txt', 'r')
except:
    pass  # Silent failure!

# Don't modify file while reading
with open('file.txt', 'r+') as f:
    for line in f:
        f.write(line.upper())  # Dangerous!

# Don't assume file exists
f = open('file.txt', 'r')  # May raise FileNotFoundError
```

## 31. Quick Reference

```python
# OPEN
with open('file.txt', 'r') as f:              # Read text
with open('file.txt', 'w') as f:              # Write text (overwrites!)
with open('file.txt', 'a') as f:              # Append text
with open('file.bin', 'rb') as f:             # Read binary
with open('file.txt', 'r', encoding='utf-8')  # Explicit encoding

# READ
f.read()                                      # Read all
f.read(n)                                     # Read n bytes
f.readline()                                  # Read one line
f.readlines()                                 # Read all lines as list
for line in f:                                # Iterate (best)

# WRITE
f.write(string)                               # Write string
f.writelines(list)                            # Write list of strings
print('text', file=f)                         # Print to file

# POSITION
f.tell()                                      # Current position
f.seek(0)                                     # Go to start
f.seek(0, 2)                                  # Go to end

# PROPERTIES
f.closed                                      # Is closed?
f.name                                        # Filename
f.mode                                        # Mode ('r', 'w', etc.)

# PATH (pathlib)
from pathlib import Path
p = Path('file.txt')
p.exists()                                    # File exists?
p.read_text()                                 # Read file
p.write_text('data')                          # Write file
p.unlink()                                    # Delete file

# OS
import os
os.path.exists('file.txt')                    # Exists?
os.path.getsize('file.txt')                   # Size in bytes
os.remove('file.txt')                         # Delete
shutil.copy('src.txt', 'dst.txt')             # Copy
```

## 32. Final Reminders

**🔑 Key Concepts:**
- Files have buffers - data sits in memory before disk
- Close files to flush buffers and release resources
- Context managers (`with`) auto-close even on exception
- Text mode handles encoding, binary mode doesn't
- File position matters - reading moves the cursor

**⚡ Performance:**
- Iterate large files line-by-line, don't load all
- Use larger buffer sizes for bulk I/O
- Binary mode is faster than text mode
- Batch writes instead of many small writes

**🛡️ Safety:**
- Always specify encoding for text files
- Use `with` statement (context manager)
- Use 'x' mode to prevent accidental overwrites
- Atomic writes for critical data (temp + rename)
- Handle FileNotFoundError gracefully

**🐛 Common Mistakes:**
- Forgetting to close files
- Using 'w' mode on existing files (data loss!)
- Reading entire large file into memory
- Wrong/missing encoding
- Modifying file while iterating
- Platform-specific path separators

**🎯 Best Tools:**
- `pathlib` for modern path handling
- Context managers for automatic cleanup
- Iteration for memory-efficient reading
- `shutil` for high-level file operations
- `tempfile` for temporary files
