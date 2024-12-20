# 01 - Relational Model & Algebra

[Link](https://www.youtube.com/watch?v=APqWIjtzNGE)

### What is a DBMS trying to do for you?

### Example -- Digital Music

```python
class Artist(BaseModel):
  name: str
  year: int
  country: enum
```

### Read files

Ah! This is so slow!!

```python
for line in file.readlines():
  record = parse(line)
  if record[0] == "GZA":
    print(int(record[1]))
```

```golang
func parse(line string) []string {
    return strings.Split(strings.TrimSpace(line), ",")
}

file, err := os.Open("artists.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

scanner := bufio.NewScanner(file)
for scanner.Scan() {
    record := parse(scanner.Text())
    if record[0] == "GZA" {
        year, err := strconv.Atoi(record[1])
        if err == nil {
            fmt.Println(year)
        }
    }
}
```

Why is this slow? (More in a broad sense to introduce the concept of DBs)
- Parsing each line sequentially...
- Disk I/O Bottleneck -- reading from disk is slower than accessing memory.
- Lack of indexing - no quick way to find records matching a condition... (particular to the flat file example)

## Brainstorming -- How do you read files faster?

How do cursors actually work?

```python
import struct

def write_records(filename, records):
    with open(filename, 'wb') as f:
        for record in records:
            name = record['name'].ljust(30)[:30].encode('utf-8')
            year = struct.pack('i', record['year'])
            country = record['country'].ljust(20)[:20].encode('utf-8')
            f.write(name + year + country)

# Sample data
artists = [
    {'name': 'GZA', 'year': 1966, 'country': 'USA'},
    {'name': 'Bjork', 'year': 1965, 'country': 'Iceland'},
    {'name': 'Daft Punk', 'year': 1993, 'country': 'France'},
    # ... more records
]

write_records('artists.dat', artists)

class Cursor:
    RECORD_SIZE = 54  # 30 bytes for name, 4 bytes for year, 20 bytes for country

    def __init__(self, filename):
        self.file = open(filename, 'rb')
        self.position = 0
        self.file_size = self._get_file_size()

    def _get_file_size(self):
        current_pos = self.file.tell()
        self.file.seek(0, 2)  # Move to end of file
        size = self.file.tell()
        self.file.seek(current_pos)
        return size

    def fetch_next(self):
        if self.position >= self.file_size:
            return None  # End of file
        self.file.seek(self.position)
        record_data = self.file.read(self.RECORD_SIZE)
        self.position += self.RECORD_SIZE
        return self._parse_record(record_data)

    def fetch_prev(self):
        if self.position - self.RECORD_SIZE < 0:
            return None  # Start of file
        self.position -= self.RECORD_SIZE
        self.file.seek(self.position)
        record_data = self.file.read(self.RECORD_SIZE)
        return self._parse_record(record_data)

    def _parse_record(self, data):
        name = data[:30].decode('utf-8').strip()
        year = struct.unpack('i', data[30:34])[0]
        country = data[34:].decode('utf-8').strip()
        return {'name': name, 'year': year, 'country': country}

    def close(self):
        self.file.close()
```

# Golang

```golang
package main

import (
    "encoding/binary"
    "fmt"
    "os"
    "strings"
)

const RecordSize = 54

type Artist struct {
    Name    string
    Year    int32
    Country string
}

func writeRecords(filename string, records []Artist) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    for _, record := range records {
        nameBytes := []byte(fmt.Sprintf("%-30s", record.Name)[:30])
        yearBytes := make([]byte, 4)
        binary.LittleEndian.PutUint32(yearBytes, uint32(record.Year))
        countryBytes := []byte(fmt.Sprintf("%-20s", record.Country)[:20])

        data := append(nameBytes, yearBytes...)
        data = append(data, countryBytes...)

        _, err := file.Write(data)
        if err != nil {
            return err
        }
    }
    return nil
}

func main() {
    artists := []Artist{
        {"GZA", 1966, "USA"},
        {"Bjork", 1965, "Iceland"},
        {"Daft Punk", 1993, "France"},
        // ... more records
    }

    err := writeRecords("artists.dat", artists)
    if err != nil {
        fmt.Println("Error writing records:", err)
    }
}
```

# Data Integrity

Flat file limitations

1. How do you find a particular record?

2. What if we now want to create a new application that uses the same database? What if that application is running on a different machine?

3. What if two threads try to write to the same file at the same time?

^ Love this one

4. What if the machine crashes while our program is updating a record?

^^ Even better

5. What if we want to replicate the database on multiple machines for high availability?

With #3, how about two machines try to write to the same file at the same time? And with #4 what if 1 / n or k / n machines crash?

# Data Models

A `data model` is a collection of concepts for describing the data in a database.

```python
class LectureNotes(BaseModel):
  lecture_name: str
  content: str
  duration: date
```

### SCHEMA

Haha - otherwise, you have random bits with no *meaning*.

- Relational
- Key/Values -- _Caching_
NoSQL
- Graph
- Document / JSON / XML / Object
- Wide-Column / Column-family
ML/ Science
- Array (vector, Matrix, Tensor)
Legacy
- Hierarchical
- Network
- Semantic
- Entity-Relationship

### Early DBMS

Early database applications were difficult to build and maintain on available DBMSs in the 1960s. Computer were expensive, humans were cheap.

Tight coupling between logical and physical layers.

Programmers had to know what queries the application would execute before they could deploy the database.

Andy describes the importance of abstractions over hash tables and b-trees in early days of DBMS 🔥

Edgar Codd - `A Relational Model of Data for Large Shared Data Banks`

### Codasyl

Back and forth between relational database people and COBOL.

### Relational Model

The relational model defines a database abstraction based on relations to avoid maintenance overhead.

Key tenets:
- Store database in simple data structures (relations).
- Physical storage left up to the DBMS implementation.
- Access data through high-level language, DBMS figures out best execution strategy.
  
### Structure
The definition of the database's relations and their contents independent of their physical representation.

### Integrity
Ensure the database's contents satisfy constraints

### Manipulation
Programming interface for accessing and modifying a database's contents.

# Data Independence

Isolate the user/application from low-level data representation.

- The user only worries about high-level application logic.
- DBMS optimizes the layout according to operating environment, database contents, and workload.
- DBMS can then re-optimize the database if/when these factors changes.

**Logical Data Independence** VERSUS **Physical Data Independence**

### Relational Model

A relation is an unordered set that contains the relationship of attributes that represent entities.

A tuple is a set of attribute values (aka its domain) in the relation.
- Values are normally atomic / scalar.
- The special value `NULL` is a member of every domain (if allowed).

`n-ary Relation = Table with n columns`

## Relational Model: Primary Keys

A relation's primary key uniquely identifies a single tuple.

Some DBMSs automatically create an internal primary key if a table does not define one.

DBMS can auto-generate unique primary keys via an identity column:
- `IDENTITY` (SQL Standard)
- `SEQUENCE` (PostgreSQL / Oracle)
- `AUTO_INCREMENT` (MySQL)

## Foreign Keys

A **foreign key** specifies that an attribute from one relation maps to a tuple in another relation.

