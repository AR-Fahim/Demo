# Python File Handling: Deep Dive Guide

> **Philosophy**: Understanding files requires understanding the OS, memory, and I/O architecture. This guide explains the **WHY** behind every pattern.

## Part 1: Core Concepts & Mental Models

### 1.1 The File Abstraction: What Really Happens?

**Mental Model: The Three-Layer Hierarchy**
```
Your Python Code
    ↓
Python File Object (buffer in RAM)
    ↓
OS Kernel (system calls)
    ↓
File System (disk blocks)
    ↓
Physical Disk (magnetic/SSD)
```

**Why This Matters:**
```python
# When you write code like this:
with open('file.txt', 'w') as f:
    f.write('Hello')
# File auto-closes when exiting with block

# What ACTUALLY happens:
# 1. Python asks OS: "Give me file descriptor for 'file.txt'"
# 2. OS checks permissions, creates inode (metadata), returns FD (integer)
# 3. Python wraps FD in file object with buffer (default 8KB)
# 4. write() puts data in BUFFER (not disk yet!)
# 5. On __exit__: flush buffer → OS write queue → disk controller → physical write
# 6. OS closes FD, releases locks

# Time breakdown (approximate):
# - Getting FD from OS: 100 microseconds
# - Writing to buffer: 100 nanoseconds (1000x faster!)
# - Flushing to OS: 500 microseconds
# - Physical disk write: 5-10 milliseconds (100,000x slower than buffer!)
```

**The Psychology: Why Buffering Exists**
- **Problem**: Disk I/O is ~100,000x slower than RAM
- **Solution**: Batch writes in memory, flush occasionally
- **Trade-off**: Data loss if crash before flush vs performance gain

```python
# Without buffering (hypothetical):
for i in range(10000):
    f.write(f'{i}\n')  # 10,000 disk writes = 50-100 seconds!

# With buffering (reality):
for i in range(10000):
    f.write(f'{i}\n')  # ~125 disk writes (8KB buffer) = 0.5 seconds
```

### 1.2 File Descriptors: The Core Abstraction

**Concept**: File descriptor (FD) is an integer the OS uses to track open files

```python
import os

fd = os.open('file.txt', os.O_RDONLY)  # Returns integer (e.g., 3)
# Standard FDs: 0=stdin, 1=stdout, 2=stderr
# Your files get: 3, 4, 5, 6...

# OS maintains per-process table:
# FD → inode (file metadata)
# FD → file offset (current position)
# FD → access mode (read/write)
# FD → file locks

# Why integers? Fast array lookup in kernel
```

**Real Issue: FD Leaks**
```python
# BAD: OS limit on FDs (typically 1024 per process)
for i in range(2000):
    f = open(f'file{i}.txt', 'r')  # Leak!
    # After ~1000 iterations: "Too many open files"

# Debugging FD leaks in production:
import subprocess
pid = os.getpid()
# Linux: ls -la /proc/{pid}/fd | wc -l
# Shows how many FDs your process has open
```

**Best Practice: Context Managers Guarantee Cleanup**
```python
# Even if exception, FD is closed
with open('file.txt', 'r') as f:
    data = f.read()
    1/0  # Exception!
# File is STILL closed here (FD released)

# Why? Python's with statement guarantees __exit__ runs
# Like finally block but cleaner
```

### 1.3 The Buffer: RAM vs Disk Mental Model

**Visualization:**
```
┌─────────────────────────┐
│   Your Program (RAM)    │
│  ┌──────────────────┐   │
│  │  File Buffer     │   │ ← write() goes here (fast)
│  │  (8KB default)   │   │
│  └────────┬─────────┘   │
│           │ flush()     │
└───────────┼─────────────┘
            ↓
┌───────────────────────────┐
│    OS Write Queue         │ ← OS batches writes
└───────────┬───────────────┘
            ↓
┌───────────────────────────┐
│  Disk Controller Cache    │ ← Hardware cache
└───────────┬───────────────┘
            ↓
┌───────────────────────────┐
│   Physical Disk Sectors   │ ← Actual persistence
└───────────────────────────┘
```

**Buffer Sizes: The Psychology**
```python
# Default: 8192 bytes (8KB) - Why?
# - Typical disk block size: 4KB
# - 8KB = 2 blocks = good balance
# - Too small: too many flushes
# - Too large: memory waste

# Tuning for use cases:

# 1. Logging (frequent small writes)
with open('app.log', 'a', buffering=1) as f:  # Line buffering
    f.write('Error\n')  # Flushed immediately on \n
    # Why? Logs must be visible NOW for debugging

# 2. Bulk data export (large sequential writes)
with open('export.csv', 'w', buffering=65536) as f:  # 64KB
    for row in million_rows:
        f.write(csv_format(row))
    # Why? Amortize flush overhead across more writes

# 3. Real-time sensor data (critical writes)
with open('sensor.dat', 'wb', buffering=0) as f:  # No buffer
    f.write(sensor_bytes)
    # Why? Data loss unacceptable, performance secondary

# 4. Network file systems (high latency)
with open('nfs_file.txt', 'w', buffering=1048576) as f:  # 1MB
    # Why? Network latency dominates, batch aggressively
```

### 1.4 Text vs Binary: Encoding Deep Dive

**The Core Problem: Computers Only Understand Bytes**
```python
# Text mode: bytes ↔ strings (with encoding)
with open('file.txt', 'w', encoding='utf-8') as f:
    f.write('Hello')  # String
    # Internally: 'Hello' → b'Hello' (ASCII overlap)
    # But: 'こんにちは' → b'\xe3\x81\x93\xe3\x82\x93\xe3\x81\xab\xe3\x81\xa1\xe3\x81\xaf'

# Binary mode: raw bytes
with open('file.txt', 'wb') as f:
    f.write(b'Hello')  # Must be bytes
```

**Why UTF-8 Won**
```
ASCII:    1 byte per char (English only)
Latin-1:  1 byte per char (Western Europe)
UTF-16:   2-4 bytes per char (fixed width for BMP)
UTF-8:    1-4 bytes per char (variable, ASCII-compatible)

UTF-8 advantages:
✓ ASCII files are valid UTF-8 (backward compatible)
✓ Variable length (efficient for English, supports all Unicode)
✓ No byte-order mark needed (unlike UTF-16)
✓ Self-synchronizing (can find char boundaries)
✗ Variable length = can't random access by char index
```

**Real-World Encoding Issues**
```python
# Issue 1: Platform defaults differ
# Windows: cp1252 (legacy)
# Linux/Mac: UTF-8

# BAD: Relies on platform default
with open('file.txt', 'w') as f:  # Encoding = ???
    f.write('café')
# Works on Linux, breaks on Windows!

# GOOD: Explicit encoding
with open('file.txt', 'w', encoding='utf-8') as f:
    f.write('café')

# Issue 2: BOM (Byte Order Mark)
# UTF-8 file might start with: EF BB BF
# Some editors add BOM, confuses parsers

with open('file.txt', 'r', encoding='utf-8-sig') as f:  # Strips BOM
    content = f.read()

# Issue 3: Mixed encodings in legacy data
def detect_and_read(filename):
    """Handle unknown encodings"""
    try:
        with open(filename, 'r', encoding='utf-8') as f:
            return f.read()
    except UnicodeDecodeError:
        # Fallback to latin-1 (never fails, accepts all bytes)
        with open(filename, 'r', encoding='latin-1') as f:
            return f.read()

# Production approach: Use chardet library
import chardet

def smart_read(filename):
    with open(filename, 'rb') as f:
        raw = f.read()
    detected = chardet.detect(raw)
    encoding = detected['encoding']
    return raw.decode(encoding)
```

### 1.5 File Modes: Decision Tree

```
Need to create new file?
├─ Yes → Need exclusive (fail if exists)?
│  ├─ Yes → 'x' (safe for config init)
│  └─ No → Need to keep existing?
│     ├─ Yes → 'a' (logs, append-only)
│     └─ No → 'w' (DANGER: deletes existing!)
└─ No (file must exist) → Need to write?
   ├─ Yes → 'r+' (read/write, no truncate)
   └─ No → 'r' (read-only, safe)

Binary file? Add 'b' to any mode above
```

**Mode Psychology:**
```python
# 'r' - Safest (read-only, won't corrupt)
with open('data.txt', 'r') as f:
    # Can't accidentally destroy data
    pass

# 'w' - MOST DANGEROUS (truncates immediately)
with open('important.txt', 'w') as f:  # File is EMPTY now!
    if should_cancel:  # Uh oh...
        return  # Original content is GONE!

# 'a' - Safe for logs (append-only)
with open('app.log', 'a') as f:
    f.write('Log entry\n')  # Can't corrupt existing logs

# 'x' - Safe for init (fails if exists)
try:
    with open('config.json', 'x') as f:
        f.write(default_config)
except FileExistsError:
    print("Config already exists, not overwriting")

# 'r+' - For in-place updates
with open('database.dat', 'r+b') as f:
    f.seek(100)  # Go to byte 100
    f.write(b'update')  # Overwrite 6 bytes
    # Rest of file unchanged
```

## Part 2: Production Patterns & Architecture

### 2.1 The Atomic Write Pattern: Preventing Corruption

**Problem: Partial Writes Corrupt Data**
```python
# Scenario: Power failure mid-write
with open('critical.json', 'w') as f:
    f.write('{"user": "alice",')  # Power loss here!
    # Result: Invalid JSON, data lost!

# Or: Process killed mid-write
with open('state.db', 'wb') as f:
    f.write(serialized_data[:5000])  # Killed here!
    # Result: Truncated database, corruption
```

**Solution: Atomic Rename (POSIX Guarantee)**
```python
import os
import tempfile
import json

def atomic_write(filename, content, mode='w', **kwargs):
    """
    Atomic write: writes complete or not at all
    
    How it works:
    1. Write to temporary file (different name)
    2. Flush and fsync (force to disk)
    3. Rename temp → target (atomic on POSIX)
    
    Why atomic? rename() is single syscall:
    - Either: old name → new name (success)
    - Or: nothing changes (failure)
    - Never: partial rename or corruption
    """
    dirname = os.path.dirname(os.path.abspath(filename))
    basename = os.path.basename(filename)
    
    # Create temp file in SAME directory (same filesystem)
    # Why same dir? rename() across filesystems isn't atomic!
    fd, tmp_path = tempfile.mkstemp(
        dir=dirname,
        prefix=f'.{basename}.tmp.',
        suffix='.tmp'
    )
    
    try:
        with os.fdopen(fd, mode, **kwargs) as f:
            f.write(content)
            f.flush()  # Flush Python buffer
            os.fsync(f.fileno())  # Force OS to disk
        
        # Atomic rename
        os.replace(tmp_path, filename)  # Python 3.3+
        # replace() = atomic on POSIX, best-effort on Windows
        
    except:
        # Cleanup on failure
        try:
            os.unlink(tmp_path)
        except:
            pass
        raise

# Usage in production:
def save_config(config):
    content = json.dumps(config, indent=2)
    atomic_write('config.json', content)
    # Guarantee: config.json is always valid or unchanged

# Use case: State persistence
class StatefulService:
    def save_state(self):
        state = self.serialize()
        atomic_write('service.state', state)
    
    def load_state(self):
        try:
            with open('service.state', 'r') as f:
                return self.deserialize(f.read())
        except FileNotFoundError:
            return self.default_state()
```

**When Atomic Writes Matter:**
- Configuration files (service won't start if corrupt)
- Database checkpoints (partial write = corruption)
- State persistence (must be consistent)
- Critical logs (audit trails)

**When NOT to Use:**
- High-frequency writes (overhead of temp + rename)
- Append-only logs (append is naturally atomic)
- Temporary files (don't care about atomicity)

### 2.2 File Locking: Coordinating Multiple Processes

**The Problem: Race Conditions**
```python
# Process A:                    # Process B:
count = int(f.read())           count = int(f.read())  # Both read "5"
count += 1                      count += 1
f.write(str(count))  # "6"     f.write(str(count))  # "6" (should be 7!)

# Result: Lost update! Count is 6, not 7
```

**Solution: Advisory Locks (fcntl on Unix)**
```python
import fcntl
import time

# Exclusive lock (writer)
with open('counter.txt', 'r+') as f:
    fcntl.flock(f.fileno(), fcntl.LOCK_EX)  # Block until acquired
    try:
        count = int(f.read() or '0')
        count += 1
        f.seek(0)
        f.write(str(count))
        f.truncate()
    finally:
        fcntl.flock(f.fileno(), fcntl.LOCK_UN)

# Shared lock (reader) - multiple readers allowed
with open('config.txt', 'r') as f:
    fcntl.flock(f.fileno(), fcntl.LOCK_SH)
    config = f.read()
    fcntl.flock(f.fileno(), fcntl.LOCK_UN)

# Non-blocking lock (try, don't wait)
try:
    fcntl.flock(f.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
except BlockingIOError:
    print("File is locked by another process")

# Cross-platform solution: portalocker
import portalocker

with open('file.txt', 'r+') as f:
    portalocker.lock(f, portalocker.LOCK_EX)
    # Critical section
    portalocker.unlock(f)
```

**Lock Types: Mental Model**
```
Shared (LOCK_SH):
- Multiple processes can hold simultaneously
- For reading
- Blocks exclusive locks

Exclusive (LOCK_EX):
- Only one process can hold
- For writing
- Blocks all other locks

Advisory vs Mandatory:
- Advisory: Processes must cooperate (fcntl)
- Mandatory: OS enforces (rare, Windows has better support)
```

**Production Pattern: Lockfile**
```python
import os
import sys

class LockFile:
    """Ensure only one instance runs"""
    def __init__(self, path='/tmp/myapp.lock'):
        self.path = path
        self.fd = None
    
    def __enter__(self):
        self.fd = os.open(self.path, os.O_CREAT | os.O_RDWR)
        try:
            fcntl.flock(self.fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
        except BlockingIOError:
            print("Another instance is running!")
            sys.exit(1)
        return self
    
    def __exit__(self, *args):
        if self.fd:
            fcntl.flock(self.fd, fcntl.LOCK_UN)
            os.close(self.fd)
            os.unlink(self.path)

# Usage:
with LockFile():
    run_application()  # Guaranteed single instance
```

### 2.3 Memory-Mapped Files: Zero-Copy I/O

**Concept: Map File Directly to Memory**
```
Normal I/O:
Disk → OS buffer → read() → Python bytes → Process
      (copy)        (copy)

Memory-mapped:
Disk → OS buffer ↔ Process memory
      (direct mapping, no copy!)
```

**When to Use mmap:**
```python
import mmap

# ✓ Large files with random access
with open('large_database.dat', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 0)
    
    # Random access (O(1), like array)
    value = mm[1000000:1000100]  # Instant access to byte 1M
    
    # Search (very fast)
    index = mm.find(b'pattern')
    
    # Modify in-place
    mm[100:105] = b'HELLO'
    
    mm.close()

# ✓ Shared memory between processes
# Process A:
with open('shared.dat', 'w+b') as f:
    f.write(b'\x00' * 1000)  # 1KB
with open('shared.dat', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 0)
    mm[0:5] = b'HELLO'

# Process B:
with open('shared.dat', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 0)
    print(mm[0:5])  # b'HELLO' - sees changes!

# ✗ Don't use for:
# - Sequential reading (normal I/O is faster)
# - Small files (overhead not worth it)
# - Files that change size (must remap)
```

**Real Use Case: Large CSV Processing**
```python
def find_in_huge_csv(filename, search_term):
    """Find record in multi-GB CSV without loading into RAM"""
    with open(filename, 'r+b') as f:
        mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
        
        pos = 0
        while True:
            pos = mm.find(search_term.encode(), pos)
            if pos == -1:
                break
            
            # Find line boundaries
            line_start = mm.rfind(b'\n', 0, pos) + 1
            line_end = mm.find(b'\n', pos)
            
            line = mm[line_start:line_end].decode()
            yield line
            
            pos = line_end + 1
        
        mm.close()

# Processes 10GB CSV without loading into RAM
```

### 2.4 Buffering Strategies: Deep Dive

**The Three Buffer Layers:**
```
1. Python's TextIOWrapper buffer (8KB default)
2. C stdio buffer (OS libc)
3. OS page cache (kernel manages)
4. Disk controller cache (hardware)
```

**Tuning Decision Tree:**
```python
# High-frequency tiny writes (logs)
# Problem: Overhead of many small writes
# Solution: Line buffering
f = open('app.log', 'a', buffering=1)  # Flush on \n
f.write('Error\n')  # Immediately visible

# Bulk sequential writes (exports)
# Problem: Default buffer too small, many flushes
# Solution: Large buffer
f = open('export.dat', 'wb', buffering=1024*1024)  # 1MB
for chunk in data:
    f.write(chunk)

# Critical data (financial transactions)
# Problem: Data loss unacceptable
# Solution: No buffering + fsync
f = open('transactions.log', 'ab', buffering=0)
f.write(record)
os.fsync(f.fileno())  # Force to physical disk

# Network filesystem (NFS/SMB)
# Problem: Network latency >> disk latency
# Solution: Aggressive buffering
f = open('/mnt/nfs/file', 'w', buffering=10*1024*1024)  # 10MB

# Database-style (mixed read/write)
# Problem: Thrashing between read/write
# Solution: Disable Python buffer, use OS page cache
f = open('db.dat', 'r+b', buffering=0)  # Let OS handle it
```

**Flush Strategies:**
```python
# 1. Explicit flush (control timing)
with open('log.txt', 'a') as f:
    f.write('Critical event\n')
    f.flush()  # Ensure written to OS NOW
    # (OS may still delay disk write)

# 2. fsync (force physical write)
with open('critical.dat', 'ab') as f:
    f.write(data)
    f.flush()  # Python buffer → OS
    os.fsync(f.fileno())  # OS → disk (slow! ~5-10ms)

# 3. No buffering (auto-flush every write)
f = open('realtime.log', 'ab', buffering=0)

# Performance comparison:
# buffering=8192: 0.5s for 10K writes
# buffering=1: 2s (flush on \n)
# buffering=0: 50s (flush every write!)
# buffering=0 + fsync: 100s (wait for disk!)
```

### 2.5 File Watching: Event-Driven I/O

**Use Cases:**
- Auto-reload config when changed
- Live log monitoring
- File sync services (Dropbox-like)
- Build systems (watch source, rebuild)

**Implementation: watchdog Library**
```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import time

class ConfigReloader(FileSystemEventHandler):
    """Reload config when file changes"""
    
    def __init__(self, config_path, reload_func):
        self.config_path = config_path
        self.reload_func = reload_func
        self.last_modified = 0
    
    def on_modified(self, event):
        if event.src_path.endswith(self.config_path):
            # Debounce (editors trigger multiple events)
            now = time.time()
            if now - self.last_modified < 1:
                return
            self.last_modified = now
            
            print(f"Config changed, reloading...")
            try:
                self.reload_func()
            except Exception as e:
                print(f"Reload failed: {e}")

# Usage:
def reload_config():
    global config
    with open('app.config', 'r') as f:
        config = json.load(f)
    print(f"Config reloaded: {config}")

observer = Observer()
handler = ConfigReloader('app.config', reload_config)
observer.schedule(handler, path='.', recursive=False)
observer.start()

# App runs with live config reload
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    observer.stop()
observer.join()
```

**Lower-Level: inotify (Linux)**
```python
# Using pyinotify (Linux only, kernel integration)
import pyinotify

class EventHandler(pyinotify.ProcessEvent):
    def process_IN_MODIFY(self, event):
        print(f"Modified: {event.pathname}")
    
    def process_IN_CREATE(self, event):
        print(f"Created: {event.pathname}")

wm = pyinotify.WatchManager()
handler = EventHandler()
notifier = pyinotify.Notifier(wm, handler)

# Watch directory
wm.add_watch('/path/to/watch', pyinotify.IN_MODIFY | pyinotify.IN_CREATE)

notifier.loop()

# Advantage over polling:
# - Kernel notifies on change (zero CPU when idle)
# - Instant notification (no poll interval delay)
# - Scales to millions of files
```

## Part 3: Advanced Libraries & Ecosystem

### 3.1 pathlib: Modern Path Handling

**Why pathlib > os.path:**
```python
import os
from pathlib import Path

# Old way (os.path): string manipulation
path = os.path.join('/home', 'user', 'docs', 'file.txt')
dirname = os.path.dirname(path)
basename = os.path.basename(path)
exists = os.path.exists(path)

# New way (pathlib): object-oriented
path = Path('/home') / 'user' / 'docs' / 'file.txt'
dirname = path.parent
basename = path.name
exists = path.exists()

# Psychology: Path is a first-class object
# - Immutable (safe to pass around)
# - Chainable operations
# - Cross-platform (/ works on Windows too!)
```

**Real Project Usage:**
```python
from pathlib import Path

class ProjectStructure:
    """Manage project paths centrally"""
    
    def __init__(self, root):
        self.root = Path(root)
        
        # Define structure
        self.data = self.root / 'data'
        self.logs = self.root / 'logs'
        self.cache = self.root / 'cache'
        self.config = self.root / 'config' / 'app.yaml'
    
    def ensure_dirs(self):
        """Create directories if missing"""
        for dir_path in [self.data, self.logs, self.cache]:
            dir_path.mkdir(parents=True, exist_ok=True)
    
    def list_data_files(self, pattern='*.csv'):
        """Find all CSVs in data directory"""
        return list(self.data.glob(pattern))
    
    def get_log_file(self, name):
        """Get timestamped log file"""
        from datetime import datetime
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        return self.logs / f'{name}_{timestamp}.log'

# Usage:
project = ProjectStructure('/opt/myapp')
project.ensure_dirs()

# Read config
config = yaml.safe_load(project.config.read_text())

# Process all CSVs
for csv_file in project.list_data_files():
    process(csv_file)

# Create log
log_file = project.get_log_file('import')
log_file.write_text('Import complete\n')
```

**Advanced pathlib Patterns:**
```python
# Recursive search
def find_python_files(root):
    """Find all .py files recursively"""
    return Path(root).rglob('*.py')

# Multiple patterns
def find_source_files(root):
    """Find all source files"""
    root = Path(root)
    patterns = ['*.py', '*.js', '*.java']
    for pattern in patterns:
        yield from root.rglob(pattern)

# Filter by date
from datetime import datetime, timedelta

def find_recent_logs(log_dir, days=7):
    """Find logs modified in last N days"""
    cutoff = datetime.now() - timedelta(days=days)
    log_path = Path(log_dir)
    
    for log_file in log_path.glob('*.log'):
        mtime = datetime.fromtimestamp(log_file.stat().st_mtime)
        if mtime > cutoff:
            yield log_file

# Atomic operations
def safe_write_json(path, data):
    """Write JSON atomically using pathlib"""
    path = Path(path)
    temp = path.with_suffix('.tmp')
    
    temp.write_text(json.dumps(data, indent=2))
    temp.replace(path)  # Atomic on POSIX
```

### 3.2 CSV Handling: pandas vs csv Module

**When to Use Each:**
```python
# csv module: Small files, streaming, low memory
import csv

# Good for: Line-by-line processing
with open('data.csv', 'r') as f:
    reader = csv.DictReader(f)
    for row in reader:
        process(row)  # Memory: O(1) per row

# pandas: Analysis, transformations, medium files
import pandas as pd

# Good for: Bulk operations, complex queries
df = pd.read_csv('data.csv')  # Memory: O(n)
result = df[df['age'] > 30].groupby('city').mean()
```

**Production CSV Handling:**
```python
import csv
from typing import Iterator, Dict

def process_large_csv(
    filename: str,
    chunk_size: int = 10000
) -> Iterator[Dict]:
    """
    Process CSV in chunks (memory-efficient)
    
    Use case: 10GB CSV, 8GB RAM
    """
    with open(filename, 'r', newline='', encoding='utf-8') as f:
        reader = csv.DictReader(f)
        
        chunk = []
        for row in reader:
            chunk.append(row)
            
            if len(chunk) >= chunk_size:
                yield chunk
                chunk = []
        
        if chunk:
            yield chunk

# Usage:
for chunk in process_large_csv('huge.csv'):
    # Process 10K rows at a time
    batch_insert_to_db(chunk)

# Error handling
def robust_csv_read(filename):
    """Handle malformed CSV gracefully"""
    errors = []
    
    with open(filename, 'r', newline='') as f:
        reader = csv.DictReader(f)
        
        for line_num, row in enumerate(reader, start=2):  # Line 1 is header
            try:
                # Validate row
                yield validate_and_clean(row)
            except ValueError as e:
                errors.append(f"Line {line_num}: {e}")
                continue
    
    if errors:
        with open('errors.log', 'w') as f:
            f.write('\n'.join(errors))

# Writing with progress
def export_to_csv(data_iterator, filename, total=None):
    """Export with progress tracking"""
    import sys
    
    with open(filename, 'w', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['id', 'name', 'value'])
        writer.writeheader()
        
        for i, row in enumerate(data_iterator):
            writer.writerow(row)
            
            if i % 1000 == 0:
                if total:
                    pct = (i / total) * 100
                    print(f'\rProgress: {pct:.1f}%', end='', file=sys.stderr)
                else:
                    print(f'\rProcessed: {i}', end='', file=sys.stderr)
        
        print('\nDone!', file=sys.stderr)
```

### 3.3 JSON Handling: Beyond Basic Load/Dump

**Production JSON Patterns:**
```python
import json
from decimal import Decimal
from datetime import datetime, date
from pathlib import Path

class EnhancedJSONEncoder(json.JSONEncoder):
    """Handle custom types in JSON"""
    
    def default(self, obj):
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, (datetime, date)):
            return obj.isoformat()
        if isinstance(obj, Path):
            return str(obj)
        if hasattr(obj, '__dict__'):
            return obj.__dict__
        return super().default(obj)

# Usage:
data = {
    'price': Decimal('19.99'),
    'created': datetime.now(),
    'path': Path('/tmp/file.txt')
}

json_str = json.dumps(data, cls=EnhancedJSONEncoder)

# Streaming large JSON arrays (memory-efficient)
import ijson  # pip install ijson

def process_large_json_array(filename):
    """
    Process JSON array without loading entire file
    
    File: [{"id": 1}, {"id": 2}, ...]
    Memory: O(1) not O(n)
    """
    with open(filename, 'rb') as f:
        # Parse incrementally
        objects = ijson.items(f, 'item')
        for obj in objects:
            process(obj)

# Pretty printing with custom format
def pretty_json(data, filename):
    """Human-readable JSON with sorted keys"""
    with open(filename, 'w') as f:
        json.dump(
            data, 
            f,
            indent=2,
            sort_keys=True,
            ensure_ascii=False  # Keep Unicode as-is
        )

# Handling corrupted JSON
def safe_json_load(filename):
    """Try to recover from corrupted JSON"""
    try:
        with open(filename, 'r') as f:
            return json.load(f)
    except json.JSONDecodeError as e:
        print(f"JSON error at line {e.lineno}, col {e.colno}")
        print(f"Error: {e.msg}")
        
        # Attempt recovery: load valid prefix
        with open(filename, 'r') as f:
            content = f.read()
            # Try parsing up to error position
            valid_part = content[:e.pos]
            # Add closing brackets/braces
            return try_fix_json(valid_part)

# Schema validation
import jsonschema  # pip install jsonschema

schema = {
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "age": {"type": "integer", "minimum": 0}
    },
    "required": ["name", "age"]
}

def validate_json_file(filename, schema):
    """Validate JSON against schema"""
    with open(filename, 'r') as f:
        data = json.load(f)
    
    try:
        jsonschema.validate(data, schema)
        return True
    except jsonschema.ValidationError as e:
        print(f"Validation error: {e.message}")
        return False
```

### 3.4 Binary File Formats: struct Module

**Concept: Reading/Writing Binary Data Structures**
```python
import struct

# Pack data into binary format
data = struct.pack(
    'I 10s f',  # Format: unsigned int, 10-byte string, float
    42,         # Integer
    b'Hello',   # String (padded to 10 bytes)
    3.14        # Float
)
# Result: b'*\x00\x00\x00Hello\x00\x00\x00\x00\x00\xc3\xf5H@'

with open('data.bin', 'wb') as f:
    f.write(data)

# Unpack from binary
with open('data.bin', 'rb') as f:
    data = f.read()

number, text, value = struct.unpack('I 10s f', data)
# number=42, text=b'Hello\x00\x00\x00\x00\x00', value=3.14

# Real use case: Reading image headers
def read_png_header(filename):
    """Read PNG file dimensions"""
    with open(filename, 'rb') as f:
        # PNG signature
        signature = f.read(8)
        if signature != b'\x89PNG\r\n\x1a\n':
            raise ValueError("Not a PNG file")
        
        # IHDR chunk
        length, = struct.unpack('>I', f.read(4))  # Big-endian
        chunk_type = f.read(4)  # b'IHDR'
        
        # Dimensions
        width, height = struct.unpack('>II', f.read(8))
        
        return {'width': width, 'height': height}

# Custom binary protocol
class BinaryMessage:
    """Simple binary message format"""
    
    HEADER = b'MSG\x00'  # Magic bytes
    FORMAT = '4s I H H'  # Header, length, msg_type, flags
    HEADER_SIZE = struct.calcsize(FORMAT)
    
    def __init__(self, msg_type, data, flags=0):
        self.msg_type = msg_type
        self.data = data
        self.flags = flags
    
    def serialize(self):
        """Convert to binary"""
        return struct.pack(
            self.FORMAT,
            self.HEADER,
            len(self.data),
            self.msg_type,
            self.flags
        ) + self.data
    
    @classmethod
    def deserialize(cls, binary):
        """Parse from binary"""
        header, length, msg_type, flags = struct.unpack(
            cls.FORMAT,
            binary[:cls.HEADER_SIZE]
        )
        
        if header != cls.HEADER:
            raise ValueError("Invalid message header")
        
        data = binary[cls.HEADER_SIZE:cls.HEADER_SIZE + length]
        return cls(msg_type, data, flags)

# Usage: Custom protocol
msg = BinaryMessage(msg_type=1, data=b'Hello, World!')
binary = msg.serialize()

with open('message.bin', 'wb') as f:
    f.write(binary)

with open('message.bin', 'rb') as f:
    binary = f.read()
    msg = BinaryMessage.deserialize(binary)
    print(f"Type: {msg.msg_type}, Data: {msg.data}")
```

### 3.5 Compression: Strategy & Performance

**Compression Decision Matrix:**
```
Algorithm | Ratio | Speed | Use Case
----------|-------|-------|----------
gzip      | 2-10x | Fast  | Web, logs, general
bzip2     | 3-15x | Slow  | Archives, backups
lzma/xz   | 4-20x | Slower| Maximum compression
zstd      | 2-10x | Fastest| Real-time, databases
lz4       | 1.5-3x| Very Fast| Temp files, cache

Choose based on:
- gzip: Default choice (good balance)
- bzip2: Archival (better compression, slower)
- lzma: Maximum compression (very slow)
- zstd: Production systems (fast + good ratio)
- lz4: Speed critical (caching, temp data)
```

**Production Compression Patterns:**
```python
import gzip
import bz2
import lzma
import zstandard as zstd  # pip install zstandard

def compress_log_rotation(log_file):
    """
    Compress rotated logs (common pattern)
    
    app.log      <- Active (uncompressed)
    app.log.1.gz <- Yesterday (compressed)
    app.log.2.gz <- 2 days ago
    """
    # Rotate existing logs
    for i in range(6, 0, -1):
        old = f'{log_file}.{i}.gz'
        new = f'{log_file}.{i+1}.gz'
        if os.path.exists(old):
            os.rename(old, new)
    
    # Compress current log
    if os.path.exists(log_file):
        with open(log_file, 'rb') as f_in:
            with gzip.open(f'{log_file}.1.gz', 'wb', compresslevel=6) as f_out:
                shutil.copyfileobj(f_in, f_out)
        
        # Clear current log (or delete)
        open(log_file, 'w').close()

# Transparent compression (automatic based on extension)
import contextlib

@contextlib.contextmanager
def smart_open(filename, mode='r', **kwargs):
    """
    Open file with automatic compression
    
    auto.txt    → open()
    auto.gz     → gzip.open()
    auto.bz2    → bz2.open()
    auto.xz     → lzma.open()
    """
    if filename.endswith('.gz'):
        f = gzip.open(filename, mode, **kwargs)
    elif filename.endswith('.bz2'):
        f = bz2.open(filename, mode, **kwargs)
    elif filename.endswith('.xz'):
        f = lzma.open(filename, mode, **kwargs)
    else:
        f = open(filename, mode, **kwargs)
    
    try:
        yield f
    finally:
        f.close()

# Usage: Same code for compressed/uncompressed
with smart_open('data.txt.gz', 'rt') as f:
    content = f.read()

# Streaming compression (memory-efficient)
def compress_stream(input_file, output_file):
    """Compress large file without loading into RAM"""
    with open(input_file, 'rb') as f_in:
        with gzip.open(output_file, 'wb', compresslevel=6) as f_out:
            while True:
                chunk = f_in.read(65536)  # 64KB chunks
                if not chunk:
                    break
                f_out.write(chunk)

# Parallel compression (faster for large files)
import concurrent.futures

def parallel_compress_chunks(input_file, output_file, num_workers=4):
    """
    Compress file in parallel chunks
    
    Split file → compress chunks in parallel → reassemble
    """
    import tempfile
    
    # Split into chunks
    chunk_size = os.path.getsize(input_file) // num_workers
    chunks = []
    
    with open(input_file, 'rb') as f:
        for i in range(num_workers):
            chunk = f.read(chunk_size)
            if not chunk:
                break
            
            # Compress chunk
            temp = tempfile.NamedTemporaryFile(delete=False)
            temp.close()
            chunks.append(temp.name)
            
            with open(temp.name, 'wb') as cf:
                cf.write(gzip.compress(chunk))
    
    # Reassemble
    with open(output_file, 'wb') as f_out:
        for chunk_file in chunks:
            with open(chunk_file, 'rb') as cf:
                f_out.write(cf.read())
            os.unlink(chunk_file)
```

### 3.6 Pickle: Python Object Serialization

**When to Use Pickle:**
```
✓ Use pickle when:
  - Python-to-Python communication
  - Caching computed results
  - Checkpointing ML models
  - Inter-process data sharing

✗ Don't use pickle when:
  - Cross-language (use JSON, Protocol Buffers)
  - Untrusted data (security risk!)
  - Long-term storage (version fragile)
  - Human-readable needed
```

**Advanced Pickle Patterns:**
```python
import pickle
import pickletools

# Basic usage
data = {'model': trained_model, 'metadata': {'accuracy': 0.95}}

with open('model.pkl', 'wb') as f:
    pickle.dump(data, f, protocol=pickle.HIGHEST_PROTOCOL)

with open('model.pkl', 'rb') as f:
    data = pickle.load(f)

# Protocol versions:
# 0: ASCII (backward compatible, large)
# 1: Binary (old)
# 2: Python 2.3+ (efficient for classes)
# 3: Python 3.0+ (default in Python 3.0-3.7)
# 4: Python 3.4+ (huge objects >4GB)
# 5: Python 3.8+ (out-of-band data, faster)

# Custom pickling
class DatabaseConnection:
    """Can't pickle actual connection, recreate on unpickle"""
    
    def __init__(self, host, port):
        self.host = host
        self.port = port
        self.conn = self._connect()
    
    def _connect(self):
        return create_connection(self.host, self.port)
    
    def __getstate__(self):
        """Return state for pickling (exclude conn)"""
        state = self.__dict__.copy()
        del state['conn']
        return state
    
    def __setstate__(self, state):
        """Restore state from pickle"""
        self.__dict__.update(state)
        self.conn = self._connect()  # Recreate connection

# Optimize pickle size
def optimize_pickle(obj, filename):
    """Compress pickle for storage"""
    import gzip
    
    with gzip.open(filename, 'wb') as f:
        pickle.dump(obj, f, protocol=pickle.HIGHEST_PROTOCOL)

# Analyze pickle contents (debugging)
def analyze_pickle(filename):
    """See what's inside a pickle file"""
    with open(filename, 'rb') as f:
        pickletools.dis(f)  # Disassemble pickle opcodes

# Safe pickle loading (untrusted data)
import io

class RestrictedUnpickler(pickle.Unpickler):
    """Only allow safe types"""
    
    SAFE_CLASSES = {
        ('builtins', 'dict'),
        ('builtins', 'list'),
        ('builtins', 'int'),
        ('builtins', 'float'),
        ('builtins', 'str'),
    }
    
    def find_class(self, module, name):
        if (module, name) not in self.SAFE_CLASSES:
            raise pickle.UnpicklingError(
                f"Unsafe class: {module}.{name}"
            )
        return super().find_class(module, name)

def safe_loads(data):
    """Load pickle from untrusted source"""
    return RestrictedUnpickler(io.BytesIO(data)).load()
```

### 3.7 HDF5: Scientific Data Storage

**Concept: Hierarchical Data Format for Huge Datasets**
```python
import h5py  # pip install h5py
import numpy as np

# Why HDF5?
# ✓ Efficient for arrays (chunked storage)
# ✓ Partial I/O (don't load entire dataset)
# ✓ Compression (built-in)
# ✓ Metadata support
# ✓ Concurrent read access
# ✓ Cross-language (C, Python, MATLAB, R)

# Create HDF5 file
with h5py.File('data.h5', 'w') as f:
    # Store dataset
    f.create_dataset('measurements', data=np.random.rand(1000, 1000))
    
    # Store with compression
    f.create_dataset(
        'compressed',
        data=np.random.rand(1000, 1000),
        compression='gzip',
        compression_opts=6
    )
    
    # Chunked storage (efficient partial access)
    f.create_dataset(
        'chunked',
        data=np.random.rand(10000, 10000),
        chunks=(1000, 1000)  # Read/write in 1000x1000 blocks
    )
    
    # Metadata
    f['measurements'].attrs['units'] = 'meters'
    f['measurements'].attrs['timestamp'] = datetime.now().isoformat()

# Read HDF5 (partial access)
with h5py.File('data.h5', 'r') as f:
    # Access slice without loading all
    subset = f['chunked'][0:100, 0:100]  # Only reads needed chunks!
    
    # Read metadata
    units = f['measurements'].attrs['units']
    
    # List contents
    print(list(f.keys()))

# Real use case: ML training data
def create_training_data(images, labels, output_file):
    """Store ML dataset efficiently"""
    with h5py.File(output_file, 'w') as f:
        # Images (compressed)
        f.create_dataset(
            'images',
            data=images,
            compression='gzip',
            chunks=True  # Auto-chunk
        )
        
        # Labels (small, no compression)
        f.create_dataset('labels', data=labels)
        
        # Metadata
        f.attrs['num_samples'] = len(images)
        f.attrs['image_shape'] = images.shape[1:]
        f.attrs['created'] = datetime.now().isoformat()

# Incremental writes (append mode)
def append_to_hdf5(filename, new_data):
    """Add data to existing HDF5 file"""
    with h5py.File(filename, 'a') as f:  # 'a' = append
        if 'data' in f:
            # Resize and append
            dataset = f['data']
            old_size = dataset.shape[0]
            new_size = old_size + len(new_data)
            
            dataset.resize(new_size, axis=0)
            dataset[old_size:] = new_data
        else:
            # Create with resizable dimension
            f.create_dataset(
                'data',
                data=new_data,
                maxshape=(None, *new_data.shape[1:])  # Unlimited first dim
            )
```

### 3.8 SQLite: Embedded Database

**Why SQLite for File Operations:**
```
✓ Use SQLite when:
  - Structured data queries
  - Transactions needed
  - Multiple tables/relationships
  - Concurrent reads (single writer)
  - Better than CSV/JSON for >100MB

✗ Use flat files when:
  - Simple sequential read/write
  - Small data (<1MB)
  - No queries needed
  - Streaming data
```

**Production SQLite Patterns:**
```python
import sqlite3
from contextlib import contextmanager

@contextmanager
def get_db(db_path, **kwargs):
    """Connection pool pattern"""
    conn = sqlite3.connect(db_path, **kwargs)
    conn.row_factory = sqlite3.Row  # Dict-like rows
    try:
        yield conn
        conn.commit()
    except:
        conn.rollback()
        raise
    finally:
        conn.close()

# Usage
with get_db('app.db') as conn:
    cursor = conn.execute('SELECT * FROM users WHERE age > ?', (18,))
    for row in cursor:
        print(dict(row))

# Performance tuning
def optimize_sqlite(db_path):
    """Configure SQLite for performance"""
    conn = sqlite3.connect(db_path)
    
    # Disable synchronous (faster, less safe)
    conn.execute('PRAGMA synchronous = OFF')
    
    # Increase cache size
    conn.execute('PRAGMA cache_size = 10000')  # 10000 pages
    
    # Use WAL mode (better concurrency)
    conn.execute('PRAGMA journal_mode = WAL')
    
    # Memory temp store
    conn.execute('PRAGMA temp_store = MEMORY')
    
    conn.close()

# Bulk insert pattern
def bulk_insert(db_path, records):
    """Efficient bulk insert"""
    with get_db(db_path) as conn:
        # Use transaction
        conn.execute('BEGIN TRANSACTION')
        
        # Prepare statement once
        cursor = conn.cursor()
        cursor.executemany(
            'INSERT INTO users (name, age) VALUES (?, ?)',
            records
        )
        
        conn.execute('COMMIT')

# File as database
class FileDatabase:
    """Use SQLite to index large file"""
    
    def __init__(self, file_path):
        self.file_path = file_path
        self.db_path = f'{file_path}.index.db'
        self._create_index()
    
    def _create_index(self):
        """Create index of file offsets"""
        with get_db(self.db_path) as conn:
            conn.execute('''
                CREATE TABLE IF NOT EXISTS lines (
                    id INTEGER PRIMARY KEY,
                    offset INTEGER,
                    length INTEGER,
                    content TEXT
                )
            ''')
            
            # Index file
            with open(self.file_path, 'r') as f:
                offset = 0
                for line_id, line in enumerate(f):
                    length = len(line)
                    conn.execute(
                        'INSERT INTO lines VALUES (?, ?, ?, ?)',
                        (line_id, offset, length, line.strip())
                    )
                    offset += length
    
    def get_line(self, line_id):
        """Fast random access to any line"""
        with get_db(self.db_path) as conn:
            row = conn.execute(
                'SELECT offset, length FROM lines WHERE id = ?',
                (line_id,)
            ).fetchone()
            
            if row:
                with open(self.file_path, 'r') as f:
                    f.seek(row['offset'])
                    return f.read(row['length']).strip()
    
    def search(self, pattern):
        """Full-text search"""
        with get_db(self.db_path) as conn:
            cursor = conn.execute(
                'SELECT id, content FROM lines WHERE content LIKE ?',
                (f'%{pattern}%',)
            )
            return [dict(row) for row in cursor]

# Usage: Index 10GB log file for fast search
db = FileDatabase('huge.log')
line_1000 = db.get_line(1000)  # Instant, no scanning
results = db.search('ERROR')   # Fast, uses index
```

### 3.9 tempfile: Secure Temporary Files

**Why tempfile > manual temp files:**
```python
# BAD: Predictable names, race conditions, not cleaned up
temp_path = '/tmp/myapp_temp.txt'
with open(temp_path, 'w') as f:
    f.write(data)
# Security: Attacker can predict name, symlink attack
# Reliability: Not cleaned up on crash

# GOOD: Secure, unpredictable, auto-cleanup
import tempfile

with tempfile.NamedTemporaryFile('w', delete=True) as f:
    f.write(data)
    # Use f.name if need path
# Auto-deleted on close

# Patterns:

# 1. Temp file for processing
def process_with_temp(data):
    with tempfile.NamedTemporaryFile('w+', suffix='.json') as tmp:
        json.dump(data, tmp)
        tmp.flush()
        
        # Pass to external tool
        subprocess.run(['tool', tmp.name])
        
        # Read result
        tmp.seek(0)
        return json.load(tmp)

# 2. Temp directory for batch
def batch_process(items):
    with tempfile.TemporaryDirectory() as tmpdir:
        # Process items, write to temp dir
        for i, item in enumerate(items):
            path = os.path.join(tmpdir, f'item_{i}.txt')
            with open(path, 'w') as f:
                f.write(process(item))
        
        # Zip all results
        shutil.make_archive('results', 'zip', tmpdir)
    # tmpdir auto-deleted with all contents

# 3. Atomic write using temp
def atomic_write_via_temp(filename, content):
    """Most portable atomic write"""
    dir_path = os.path.dirname(os.path.abspath(filename))
    
    with tempfile.NamedTemporaryFile(
        'w',
        dir=dir_path,  # Same filesystem
        delete=False
    ) as tmp:
        tmp.write(content)
        tmp.flush()
        os.fsync(tmp.fileno())
        tmp_name = tmp.name
    
    # Atomic rename
    os.replace(tmp_name, filename)

# 4. Secure temp for sensitive data
def process_secret(secret_data):
    # Temp file only readable by current user
    with tempfile.NamedTemporaryFile('w', mode=0o600) as tmp:
        tmp.write(secret_data)
        tmp.flush()
        # Process...
    # Securely deleted (overwritten on some systems)
```

## Part 4: Real-World Project Patterns

### 4.1 Configuration Management

**Multi-Environment Config Pattern:**
```python
from pathlib import Path
import yaml
import os

class Config:
    """
    Load config with environment overrides
    
    Priority (high to low):
    1. Environment variables
    2. Environment-specific file (config.prod.yaml)
    3. Base config file (config.yaml)
    4. Defaults
    """
    
    def __init__(self, base_path='config'):
        self.base_path = Path(base_path)
        self.env = os.getenv('APP_ENV', 'dev')
        self._load()
    
    def _load(self):
        # Start with defaults
        self.data = self._defaults()
        
        # Load base config
        base_file = self.base_path / 'config.yaml'
        if base_file.exists():
            with base_file.open() as f:
                self.data.update(yaml.safe_load(f))
        
        # Load environment-specific
        env_file = self.base_path / f'config.{self.env}.yaml'
        if env_file.exists():
            with env_file.open() as f:
                self.data.update(yaml.safe_load(f))
        
        # Environment variables override all
        self._apply_env_overrides()
    
    def _defaults(self):
        return {
            'database': {
                'host': 'localhost',
                'port': 5432
            },
            'logging': {
                'level': 'INFO'
            }
        }
    
    def _apply_env_overrides(self):
        # APP_DATABASE_HOST overrides database.host
        for key, value in os.environ.items():
            if key.startswith('APP_'):
                parts = key[4:].lower().split('_')
                self._set_nested(self.data, parts, value)
    
    def _set_nested(self, d, keys, value):
        for key in keys[:-1]:
            d = d.setdefault(key, {})
        d[keys[-1]] = value
    
    def get(self, path, default=None):
        """Get nested config: config.get('database.host')"""
        keys = path.split('.')
        value = self.data
        for key in keys:
            if isinstance(value, dict):
                value = value.get(key)
            else:
                return default
        return value if value is not None else default

# Usage:
config = Config()
db_host = config.get('database.host')  # From file or env
```

### 4.2 Logging Architecture

**Production Logging Setup:**
```python
import logging
import logging.handlers
from pathlib import Path

def setup_logging(app_name, log_dir='logs', level=logging.INFO):
    """
    Configure structured logging
    
    Features:
    - Rotating files (avoid huge logs)
    - Separate error log
    - JSON format for parsing
    - Console output for dev
    """
    log_path = Path(log_dir)
    log_path.mkdir(exist_ok=True)
    
    # Root logger
    logger = logging.getLogger(app_name)
    logger.setLevel(level)
    
    # Format
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    
    # Console handler (dev)
    if os.getenv('APP_ENV') == 'dev':
        console = logging.StreamHandler()
        console.setFormatter(formatter)
        logger.addHandler(console)
    
    # Rotating file handler (all logs)
    all_logs = logging.handlers.RotatingFileHandler(
        log_path / f'{app_name}.log',
        maxBytes=10*1024*1024,  # 10MB
        backupCount=5
    )
    all_logs.setFormatter(formatter)
    logger.addHandler(all_logs)
    
    # Separate error log
    error_logs = logging.handlers.RotatingFileHandler(
        log_path / f'{app_name}.error.log',
        maxBytes=10*1024*1024,
        backupCount=5
    )
    error_logs.setLevel(logging.ERROR)
    error_logs.setFormatter(formatter)
    logger.addHandler(error_logs)
    
    # Timed rotation (daily logs)
    daily_logs = logging.handlers.TimedRotatingFileHandler(
        log_path / f'{app_name}.daily.log',
        when='midnight',
        interval=1,
        backupCount=30  # Keep 30 days
    )
    daily_logs.setFormatter(formatter)
    logger.addHandler(daily_logs)
    
    return logger

# Usage:
logger = setup_logging('myapp')
logger.info('Application started')
logger.error('Database connection failed', exc_info=True)
```

.
