# claujson2-document
# first check!
    01. 64bit only
    02. has my own memory allocator.
    03. C++14~ 
    04. experimental!
    05. Array has my_own_vector<_Value>
    06. Object has my_own_vector<Pair<_Value, _Value>> (not a std::map!)
    07. in Object, key can be dupplicated, are not sorted, just in order of input.
    08. scanning - modified? simdjson, parsing - parallel 
    09. it is not read-only!
    10. _Value <- json_Value, (in destructor, no remove data(Array or Object!), no copy, only move or clone!
    11. Value <- wrap _Value, (in destructor, remove data(Array or Object!)
# parallel parse
# parallel write
