---
title: MacOS.Forensics.FSEvents
hidden: true
tags: [Client Artifact]
---

This artifact parses the FSEvents log files.


```yaml
name: MacOS.Forensics.FSEvents
description: |
   This artifact parses the FSEvents log files.

reference:
- https://www.osdfcon.org/presentations/2017/Ibrahim-Understanding-MacOS-File-Ststem-Events-with-FSEvents-Parser.pdf

type: CLIENT

parameters:
   - name: GlobTable
     type: csv
     default: |
       Glob
       /.fseventsd/*
       /System/Volumes/Data/.fseventsd/*
   - name: Glob
     type: string
     description: Instead of providing the globs in a table, a single glob may e given.
   - name: PathRegex
     description: Filter the path by this regexp
     default: .
     type: regex
   - name: FlagsRegex
     description: Filter by flags
     type: regex
     default: .
export: |
    LET FSEventProfile = '''[
    ["Header", 0, [
      ["Signature", 0, "String", {
          length: 4
      }],
      ["StreamSize", 8, uint32],
      ["Items", 12, "Array", {
          count: 10000,
          max_count: 10000,
          type: FSEventEntry,
          sentinel: "x=>len(list=x.path) = 0",
      }]
    ]],
    ["FSEventEntry", "x=>len(list=x.path) + 21", [
      ["path", 0, "String"],
      ["id", "x=>len(list=x.path) + 1", "uint64"],
      ["flags", "x=>len(list=x.path) + 9", "Flags", {
          type: "uint32",
          bitmap: {
            FSE_CREATE_FILE: 0,
            FSE_DELETE: 1,
            FSE_STAT_CHANGED: 2,
            FSE_RENAME: 3,
            FSE_CONTENT_MODIFIED: 4,
            FSE_EXCHANGE: 5,
            FSE_FINDER_INFO_CHANGED: 6,
            FSE_CREATE_DIR: 7,
            FSE_CHOWN: 8,
            FSE_XATTR_MODIFIED: 9,
            FSE_XATTR_REMOVED: 10,
            FSE_DOCID_CREATED: 11,
            FSE_DOCID_CHANGED: 12,
            FSE_UNMOUNT_PENDING: 13,
            FSE_CLONE: 14,
            FSE_MODE_CLONE: 16,
            FSE_TRUNCATED_PATH: 17,
            FSE_REMOTE_DIR_EVENT: 18,
            FSE_MODE_LAST_HLINK: 19,
            FSE_MODE_HLINK: 20,
            IsSymbolicLink: 22,
            IsFile: 23,
            IsDirectory: 24,
            Mount: 25,
            Unmount: 26,
            EndOfTransaction: 29
          }
      }],
    ]]
    ]'''

sources:
  - query: |
      LET files = SELECT OSPath, Mtime
      FROM glob(globs=(Glob || GlobTable.Glob))
      WHERE log(message=OSPath)

      SELECT * FROM foreach(row=files,
      query={
        SELECT OSPath.Basename AS File, Mtime, path,
               id, join(array=flags, sep=", ") AS flags
        FROM foreach(row=parse_binary(
           filename=read_file(filename=OSPath, accessor="gzip", length=1000000),
           accessor="data",
           profile=FSEventProfile, struct="Header").Items)
      })
      WHERE path =~ PathRegex AND flags =~ FlagsRegex

```
