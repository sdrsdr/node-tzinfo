tzinfo
======


Functions to parse /usr/share/zoneinfo timezone info files.
Parses both v1 and v2 format zoneinfo files.

What is tzinfo
---

tzinfo is a collection of files holding records of past and future timestamps with changes of timezone offsets to universal time at those timestamps. All timestamps are in **seconds** from the epoch. 

Generally speaking a timezone is represented by `info_t` record with fields:
 * `ttimes` - a array of numbers holding the timestamps
 * `types` - a array of numbers, the same size as `ttimes` - the number of offset type valid after the corresponding `ttimes` timestamp; a index into `tzinfo`
 * `tzinfo` - a object of type `tzinfo_change_t` describing a offset from the universal time

API
---

### `tzinfo.getZoneinfoDirectory( )`

Return the auto-detected directory containing the system zoneinfo files.

### `tzinfo.listZoneinfoFiles( zoneinfoDirectory, strip_prefix )`

List all the zoneinfo files contained in the named zoneinfo directory.  Recursively
walks the directory and tests each file found.  This is a blocking operation, so call
only on program load.  The results are small and can be easily cached.

 * The `zoneinfoDirectory` can be set to `undefined` to use the auto-detected directory
 * The `strip_prefix` has default value of `false` and indicates whether entries in the list will contain a `zoneinfoDirectory` prefix or look more like time zone names.
 * Called with no parameters the function will return filenames from the auto-detected directory

### `tzinfo.readZoneinfoFile( tzname, cb )`

Read the zoneinfo file corresponding to the named timezone.  Returns to its callback a
`Buffer` with the file contents, or an `Error`.

### `tzinfo.readZoneinfoFileSync( tzname )`

Read the zoneinfo file corresponding to the named timezone.  Returns a `Buffer`
containing the file contents, or throws an `Error`.

### `tzinfo.parseZoneinfo( buf )`

Parse the zoneinfo file contained in `buf` and return it as an object of type `info_t` .

Returned object format:

    zoneinfo = {
        magic:      // 'TZif'
        version:    // '\0' or '2'

        ttisgmtcnt: // num gmt/local indicators stored in `ttisgmt`
        ttisstdcnt: // num standard/wall indicators stored in `ttisstd`
        leapcnt:    // num leap seconds for which data is stored in `leaps`
        timecnt:    // num transition types stored in `types`
        typecnt:    // num time transition structs stored in `tzinfo`
        charcnt:    // total num chars to store the tz name abbreviations

        ttimes:     // array of `timecnt` transition time timestamps
        types:      // array of `timecnt` tzinfo indices for each time transitioned to
        tzinfo:     // array of `typecnt` tzinfo structs
                    //     { idx: , tt_gmtoff: , tt_isdst: , tt_abbrind: }
        abbrevs:    // concatenated tz name abbreviations (asciiz strings totaling charcnt bytes)
        leaps:      // array of `leapcnt` leap second descriptors
        ttisstd:    // array of `ttisstdcnt` transitions of tzinfo were std or wallclock times
        ttisgmt:    // array of `ttisgmtcnt` transitions of tzinfo were UTC or local time
    };

### `tzinfo.findTzinfo( zoneinfo, date [,firstIfTooOld] )`

Searches for the `date` the parsed `zoneinfo` for the corresponding `tzinfo_change_t` struct and return it or `false` if the `date` is before the earliest
time transition on record or if `date` is not valid.  If `date` precedes the first known
time transition but `firstIfTooOld` is truthy, it returns the oldest tzinfo struct.
If there are no time transitions defined but there is a tzinfo struct, it returns the
tzinfo struct (to always succeed for GMT and UTC).

Tzinfo format:

    tzinfo = {
        idx:        // index of this entry in `zoneinfo.tzinfo`
        tt_gmtoff:  // seconds to add to GMT to get localtime
        tt_isdst:   // whether daylight saving is in effect
        tt_abbrind: // byte offset in abbrevs of tz name abbreviation
        abbrev:     // timezone name abbreviation, eg 'EDT'
    };

To find the POSIX-style timezone environment variable attributes associated with this `tzinfo`,
look at `zoneinfo.ttisstd[tzinfo.idx]` and `zoneinfo.ttisgmt[tzinfo.idx]`.


Change Log
----------

- 0.5.1 - always find GMT zoneinfo
- 0.5.0 - findTzinfo option to return the oldest known tzinfo struct for very old dates
- 0.4.2 - more tests, make `readStringZ` stop on unterminated strings
- 0.4.1 - npm tag
- 0.4.0 - `listZoneinfoFiles()`, `getZoneinfoDirectory()`
- 0.3.0 - `readZoneinfoFile` and `readZoneinfoFileSync`, `findTzinfo`
- 0.2.0 - first published release, with `parseZoneinfo`


Todo
----

- `setZoneinfoDirectory` to override the auto-detected one


Related Work
------------

- [zoneinfo](http://npmjs.com/package/zoneinfo)
- `tzfile(5)`, `zdump(8)`, `zic(8)` - unix zoneinfo manpages
