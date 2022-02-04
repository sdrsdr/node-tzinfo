tzinfoex
======


Functions to parse /usr/share/zoneinfo timezone info files.
Parses both v1 and v2 format zoneinfo files.

What is tzinfo
---

tzinfo is a collection of files holding records of past and future timestamps with changes of timezone offsets to universal time at those timestamps. All timestamps in the database are in **seconds** from the epoch. 

Generally speaking a timezone is represented by `info_t` record with fields:
 * `ttimes` - a array of numbers holding the timestamps
 * `types` - a array of numbers, the same size as `ttimes` - the number of offset type valid after the corresponding `ttimes` timestamp; a index into `tzinfo`
 * `tzinfo` - a object of type `tzinfo_change_t` describing a offset from the universal time

With that information available you can say at (almost) any given universal time what is the current offset for the zone and when the next offset (DST) is about to happen.

# API

```ts
function locateZoneinfoDirectory( ):string;
```

Detects system's zone info directory between  `/usr/share/zoneinfo` and `/usr/lib/zoneinfo`, This is used at module load time for auto configuration. All directory related setup should be automagical.

---
&nbsp;


```ts
function getZoneinfoDirectory( )
```

Return the auto-detected/currently set directory containing the system zoneinfo files.

---
&nbsp;

```ts
function setZoneinfoDirectory(dir:string);
```
Override the auto-detected directory containing the system zoneinfo files

---
&nbsp;

```ts
function listZoneinfoFiles( dirname:string|undefined, strip_prefix:boolean=false ):string[];
```

List all the zoneinfo files contained in the `dirname` directory. Recursively
walks the directory and tests each file found. This is a blocking operation, so call
only on program load.  The results are small and can be easily cached.  

 * The `dirname` can be set to `undefined` to use the auto-detected directory
 * The `strip_prefix` has default value of `false` and indicates whether entries in the list will contain a `zoneinfoDirectory` prefix or look more like time zone names.
 * Called with no parameters the function will return filenames from the auto-detected directory

`precacheZones` function does almost the same amount of work, **asynchronously**, usually completes under one second and can register all zone names while enabling speedy, case insensitive lookups via `getCachedZoneInfo`.

---
&nbsp;


```ts
export function readZoneinfoFile( tzname:string ):Promise<Buffer> ;
export function readZoneinfoFile( tzname:string, cb:(err: NodeJS.ErrnoException | null, data: Buffer) => void ):void ;

```

Read the zoneinfo file corresponding to the named timezone. 

if the `cb` is provided it should take two parameters: `err` and `data`. If err is falsy `data` holds the timezone file content that need to be further parsed via `parseZoneinfo`

if no `cb` is provided a promise that resolves to `Buffer` is returned

---
&nbsp;

```ts
function readZoneinfoFileSync( tzname:string ):Buffer;
```

Read the zoneinfo file corresponding to the named timezone.  Returns a `Buffer`
containing the file contents, or throws an `Error`.

---
&nbsp;

```ts
function parseZoneinfo( buf:Buffer ):info_t|false;
```

Parse the zoneinfo file contained in `buf` and return it as an object of type `info_t`. If `buf` contains invalid data a `false` is returned

Returned object format:
```ts

export interface info_t {
    magic: string;              // 'TZif'
    version: string;            // '\0' or '2'

    ttisgmtcnt: number,         // num gmt/local indicators stored in `ttisgmt`
    ttisstdcnt: number,         // num standard/wall indicators stored in `ttisstd`
    leapcnt:    number,         // num leap seconds for which data is stored in `leaps`
    timecnt:    number,         // num transition types stored in `types'
    typecnt:    number,         // num time transition structs stored in `tzinfo`
    charcnt:    number,         // total num chars to store the tz name abbreviations

    ttimes:     number[],              // transition time timestamps (timecnt)
    types:      number[],              // tzinfo index of each time transitioned to (timecnt)
    tzinfo:     tzinfo_change_t[],     // tzinfo structs (typecnt)
    abbrevs:    string,                // concatenated tz name abbreviations (asciiz strings totaling charcnt bytes)
    leaps:      unknown[],             // leap second descriptors (leapcnt)
    ttisstd:    unknown[],             // transitions of tzinfo were std or wallclock times (ttisstdcnt)
    ttisgmt:    unknown[],             // transitions of tzinfo were UTC or local time (ttisgmtcnt)

    _v1end:  number,
    _v2end:  number,
}
```
---
&nbsp;  



```ts
function findTzinfo( info:info_t, date:number|Date|string, firstIfTooOld:boolean ) : false|tzinfo_change_ex_t;
```
Searches for the `date` in `info` for the corresponding `tzinfo_change_t` struct and return it extended  with the corresponding `ttime` timestamp as `start` and the used index in `ttime`. If `date` is a number it is considered as time in **miliseconds** since the epoch. On error `false` is returned like when the `date` is before the earliest
time transition on record or if `date` is not valid.  If `date` precedes the first known
time transition but `firstIfTooOld` is truthy, it returns the oldest tzinfo struct.
If there are no time transitions defined but there is a tzinfo struct, it returns the
tzinfo struct (to always succeed for GMT and UTC).

tzinfo_change_t is defined as 

```ts
interface tzinfo_change_ex_t extends tzinfo_change_t {
    startat:number; //miliseconds since epoch or 0 if unknown (better use ttimes_index for unknown indicator)
    ttimes_index:number; //index to ttimes/types arrays or -1 if unknown
}
```

To find the POSIX-style timezone environment variable attributes associated with this `tzinfo`,
look at `zoneinfo.ttisstd[tzinfo.idx]` and `zoneinfo.ttisgmt[tzinfo.idx]`.

---
&nbsp;

```ts
function nextTzinfo(info: info_t, current: tzinfo_change_ex_t): false | tzinfo_change_ex_t
```

Finds the next change after `current`. Returns false if no more changes are expected

---
&nbsp;

```ts
function getCachedZoneInfo(zonename:string):Promise<info_t>;
```

Combines `readZoneinfoFile`, `parseZoneinfo` and caches the result

---
&nbsp;

```ts
function precacheZones(capture_canonical_names?:string[]):Promise<true>;
```

Asynchronously precache all zone info data. Post completion `getCachedZoneInfo` is just map lookup. `getCachedZoneInfo` does **not** depend on `precacheZones` but it will benefit significantly from the precache in expense of sub 10MB of RAM. Beside speed improvements zone name lookup becomes **case insensitive** as all zones are now known and such lookup can be done easily. If you want to capture the list of canonical zone names e.g. Europe/Paris etc, you need to pass an empty array as `capture_canonical_names` parameter. Zone names will be pushed there before a lowercase version is stored in internal map. With a SSD disk and i5 circa 2011 the precaching of recent zone info database takes about 600ms. Heap usage goes up by 9MB while the database is calculated as 5MB ondisk.

---
&nbsp;

```ts
function getPreCachedZoneInfo(zonename:string):info_t|false {
```

Do a lookup the precached maps and return the zone info if found. If `precacheZones` has not yet completed (usually under a second) the function will return `false`. `false` is also  returned if unknown zonename is requested.

---
&nbsp;

Example
-------

```ts
import { getCachedZoneInfo, findTzinfo, nextTzinfo } from "tzinfo";
getCachedZoneInfo('Europe/Sofia').then(zi=>{
	if (!zi) process.exit(-1);
	let now=findTzinfo(zi, Date.now(),true);
	if (now==false) {
		console.error("The database seems broken?");
		process.exit(-1);
	}
	console.log("current offset in this tz is "+now.tt_gmtoff+" seconds or "+now.tt_gmtoff/3600+' hours in effect sice '+new Date(now.startat)+' idx:'+now.ttimes_index);
	let next=nextTzinfo(zi,now);
	if (next==false) {
		console.log("no shifts in offset are planned!");
	} else {
		console.log("next change in offset will happen at "+new Date(next.startat-1)+' new offset will be '+next.tt_gmtoff+" seconds or "+next.tt_gmtoff/3600+' hours');
	}
}).catch(err=>{
	console.error(err);
	process.exit(-1);
})
```

Change Log
----------
- 0.8.0 - getPreCachedZoneInfo no-promises lookup of precached data
- 0.7.0 - precacheZones, case insensitive zone name lookup
- 0.6.0 - Port to TS, add getCachedZoneInfo, remove zoneinfoDir export, documentation changes
- 0.5.1 - always find GMT zoneinfo
- 0.5.0 - findTzinfo option to return the oldest known tzinfo struct for very old dates
- 0.4.2 - more tests, make `readStringZ` stop on unterminated strings
- 0.4.1 - npm tag
- 0.4.0 - `listZoneinfoFiles()`, `getZoneinfoDirectory()`
- 0.3.0 - `readZoneinfoFile` and `readZoneinfoFileSync`, `findTzinfo`
- 0.2.0 - first published release, with `parseZoneinfo`



Related Work
------------

- [zoneinfo](http://npmjs.com/package/zoneinfo)
- `tzfile(5)`, `zdump(8)`, `zic(8)` - unix zoneinfo manpages
