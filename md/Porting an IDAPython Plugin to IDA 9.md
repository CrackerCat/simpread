> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jro.sg](https://jro.sg/IDAPython%208%20to%209.html)

> Alternatives for some APIs removed in IDA 9

In IDA 9, the IDAPython API has been significantly upgraded, consolidating the `ida_struct` and `ida_enum` modules into the existing `ida_typeinf` module.

While this change simplifies the IDAPython API, it also breaks several IDAPython plugins, such as [auto-enum](https://github.com/junron/auto-enum) and [sark](https://github.com/tmr232/Sark). This has caused some frustration among IDA users ([1](https://x.com/domenuk/status/1830949828806791343), [2](https://x.com/ootiosum/status/1828796183516729434)).

Although HexRays has published an [official IDAPython porting guide](https://docs.hex-rays.com/pre-release/developer-guide/idapython/idapython-porting-guide-ida-9), it does not fully document all the changes to the IDAPython API. Additionally, as noted by [@domenuk](https://x.com/domenuk/) on X, several (many) blanks can be observed in the "alternatives" field.

This article documents my [work](https://github.com/tmr232/Sark/compare/main...junron:Sark:main) in adapting the [sark](https://github.com/tmr232/Sark) library for use with IDA 9. Thus, it does not (yet) completely document every single change that you need to make to be compatible with IDA 9. However, it does cover some of the common tasks and operations that are now no longer directly supported and how to re-implement them in IDA 9.

`idaapi.get_inf_structure()` [​](#idaapi-get-inf-structure)
-----------------------------------------------------------

On IDA 8, this method returns a `ida_ida.idainfo` object which contains information about the current IDB.

This method has been removed in IDA 9. Instead, use the various `idaapi.inf_get_*()` methods.

Here are some examples:

<table tabindex="0"><thead><tr><th>IDA 8</th><th>IDA 9</th></tr></thead><tbody><tr><td><code>idaapi.get_inf_structure().procname</code></td><td><code>idaapi.inf_get_procname()</code></td></tr><tr><td><code>idaapi.get_inf_structure().max_ea</code></td><td><code>idaapi.inf_get_max_ea()</code></td></tr><tr><td><code>idaapi.get_inf_structure().is_32bit()</code></td><td><code>idaapi.inf_is_32bit_exactly()</code></td></tr></tbody></table>

Struct/enum error codes [​](#struct-enum-error-codes)
-----------------------------------------------------

Enum and struct related error codes (`idaapi.ENUM_MEMBER_ERROR_*` and `idaapi.STRUC_ERROR_MEMBER_*`) have been removed, along with the `ida_enum` and `ida_struct` modules.

Instead, methods such as `idc.add_struc_member` and `idc.add_enum_member` now return various `ida_typeinf.TERR_*` error codes.

As the error codes for enums and structs have been merged, there is no 1:1 mapping between the errors on IDA 8 and 9. Additionally, the error code values have changed between IDA 8 and 9 too.

Here's some examples:

<table tabindex="0"><thead><tr><th>IDA 8 Name</th><th>IDA 8 Value</th><th>IDA 9 Name</th><th>IDA 9 Value</th></tr></thead><tbody><tr><td><code>ENUM_MEMBER_ERROR_NAME</code></td><td>1</td><td><code>TERR_DUPNAME</code> or <code>TERR_BAD_NAME</code></td><td>-22 or -3</td></tr><tr><td><code>STRUC_ERROR_MEMBER_NAME</code><sup>1</sup></td><td>-1</td><td><code>TERR_DUPNAME</code></td><td>-22</td></tr><tr><td><code>STRUC_ERROR_MEMBER_OFFSET</code></td><td>-2</td><td><code>TERR_OVERLAP</code></td><td>-14</td></tr></tbody></table>

1 I would expect this to return `TERR_BAD_NAME` as well, but apparently more characters are allowed in IDA struct member names now:

![](https://imgur.com/nApQk6G.png)

which makes for some interesting pseudocode:

![](https://imgur.com/taScQ66.png)

`ida_enum` [​](#ida-enum)
-------------------------

This module has been removed entirely, with most of its methods moving to `idc` (refer to the [official IDAPython porting guide](https://docs.hex-rays.com/pre-release/developer-guide/idapython/idapython-porting-guide-ida-9)). Thus, I will be focusing on methods that have been removed entirely or have undergone changes to their function signatures or semantics.

#### `idc.add_enum` [​](#idc-add-enum)

The first parameter, `idx` is still present, but no longer used. New enums will always be added to the end of the list of types, regardless of the `idx` passed.

#### `idc.set_enum_member_cmt` [​](#idc-set-enum-member-cmt)

The last parameter, `repeatable`, is still present, but no longer used. Enum member comments are now never repeatable.

#### `idc.get_enum_cmt` [​](#idc-get-enum-cmt)

The last parameter, `repeatable`, has been removed. However, the `repeatable` parameter of `idc.set_enum_cmt` is still present.

#### Methods relating to enum member serial [​](#methods-relating-to-enum-member-serial)

This includes `get_enum_member_serial` and `idc.op_enum`'s final parameter, `serial`.

Prior to IDA 9, the `serial` field of an enum member was used to distinguish multiple enum members with the same value, but different name. For example, if `MAP_ANON` and `MAP_ANONYMOUS` both have the value of `0x20`, they might receive serial 0 and 1 respectively.

In IDA 9, the `serial` field has been removed. Instead, the `tid` of an enum member is sufficient to uniquely identify that enum member.

To get the `tid` of the `i`th enum member of an enum identified by `eid`, use the `get_type_by_tid` and `get_edm` methods of `ida_typeinf.tinfo_t`:

python

```
def get_enum_member_tid(eid: int, i: int):
    tif = ida_typeinf.tinfo_t()
    tif.get_type_by_tid(eid)
    edm = ida_typeinf.edm_t()
    tif.get_edm(edm, i)
    return edm.get_tid()
```

#### `get_enum_qty` and `getn_enum` [​](#get-enum-qty-and-getn-enum)

These methods allowed for easy enumeration of all enums defined in the IDB. However, with the integration of enums into the unified types list, it seems the only way to iterate through all enums is to iterate through all types and test if that type is an enum:

python

```
def _iter_enum_ids():
    """Iterate the IDs of all enums in the IDB"""
    limit = ida_typeinf.get_ordinal_limit()
    for ordinal in range(1, limit):
        tif = ida_typeinf.tinfo_t()
        tif.get_numbered_type(None, ordinal)
        if tif.is_enum():
            yield tif.get_tid()
```

`ida_struct` [​](#ida-struct)
-----------------------------

Similarly to `ida_enum`, this module also been completely removed, with most of its methods migrated to `idc`.Thus, I will be focusing on methods that have been removed entirely.

The `struc_t` and `member_t` types have been removed in favor of `ida_typeinf.tinfo_t` and `ida_typeinf.udm_t`.

#### `get_struc` [​](#get-struc)

python

```
def get_struc(sid: int) -> Optional[ida_typeinf.tinfo_t]:
    tif = ida_typeinf.tinfo_t()
    if tif.get_type_by_tid(sid) and tif.is_udt():
	    return tif
    return None
```

#### `get_member` [​](#get-member)

python

```
def get_member(sid: int, offset: int) -> Optional[ida_typeinf.udm_t]:
    struc_tif = get_struc(sid)
    if struc_tif is None:
        return None
    udm = ida_typeinf.udm_t()
    udm.offset = offset
    idx = tif.find_udm(udm, ida_typeinf.STRMEM_AUTO)
    if idx != -1:
        return udm
    return None
```

#### `get_member_by_name` [​](#get-member-by-name)

python

```
def get_member_by_name(sid: int, name: str) -> Optional[ida_typeinf.udm_t]:
    struc_tif = get_struc(sid)
    if struc_tif is None:
        return None
    udm = ida_typeinf.udm_t()
    udm.name = name
    idx = tif.find_udm(udm, ida_typeinf.STRMEM_NAME)
    if idx != -1:
        return udm
    return None
```